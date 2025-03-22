+++
title = "Building a search engine from scratch, in Rust: part 1"
description = "Or how to write on disk efficiently in the browser or any other device."
date = 2025-03-22

[extra]
emoji = "🦀"

[taxonomies]
tags = ["rust", "search-engine", "webassembly", "encryption", "cross-platform", "tutorial", "performance"]
+++

In the [previous article](../202503161800-search-engine-intro/), I introduced what project we're going to address in the following weeks: how to build a cross-platform search engine with encryption capabilities. Today, we'll have a look at the first technical challenge: how to store things on disk.

You might be thinking that we start with a simple topic, to warm up and get ready for the serious parts. That's both right and wrong at the same time. Writing on the filesystem from a mobile application or a desktop application is an easy task, but doing so in the browser while using the same interface is not as easy as it seems.

And on top of that, we want our storage layer to read and write encrypted files, without compromising too much on performance.

## The Storage Challenge

Before diving into search algorithms and indexing, we need to solve a fundamental challenge: how do we store data consistently across different platforms? While writing to the filesystem is straightforward on desktop and mobile platforms, browsers present unique constraints that require careful consideration.

**Key Challenges:**
- Cross-platform filesystem access
- Async operations requirement
- Performance considerations
- Encryption integration (coming up next)

## Exploring Available Solutions

When I first approached this problem, I naturally looked at what the browser ecosystem offers for storage. The most obvious solution seemed to be [LocalStorage]((https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)) - it's simple, widely supported, and easy to use. However, its limitations quickly became apparent: the storage size is restricted (usually to just a few megabytes), and it only supports string values, which would force us to serialize and deserialize our data constantly.

Next, I considered [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API). It's more powerful and supports larger datasets, which initially seemed promising. But working with [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) directly is notoriously complex - its API is verbose and requires handling multiple edge cases. While there are wrapper libraries that make it more palatable, they would add unnecessary bulk to our final bundle size.

That's when I discovered the [File System API](./https://developer.mozilla.org/en-US/docs/Web/API/File_System_API), a relatively new addition to modern browsers. This API caught my attention because it provides something remarkable: a filesystem-like interface that feels natural and familiar. Instead of working with abstract storage concepts, we can work with files and directories in a way that's conceptually similar to how we'd handle storage on desktop or mobile platforms.

The [File System API](./https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) offers several compelling advantages:
- We can work with files directly, making our code more intuitive
- Performance is generally better than alternatives since we're closer to the system
- The API aligns well with how we'll handle storage on other platforms
- We get proper file handling capabilities including streaming and random access

However, there's a catch - browsers isolate this filesystem access to a specific directory for security reasons. We can't just access any file on the user's system (which is actually good for our use case). Also, the API is relatively new, which means we need to consider browser compatibility.

**Key Points:**
- LocalStorage is too limited in size and capabilities
- IndexedDB is powerful but complex and heavy
- File System API provides the best balance of features and simplicity
- We'll need to handle the async nature of browser storage
- Security constraints actually benefit our use case

Let's dive into how we can work with this API in practice...

## Storage layer

Considering the browser will be the weak point here, let's spend some time figuring out how to address this.

### A filesystem for the Browser

Addressing this head down could lead us to use the [local storage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) or [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) as a virtual filesystem. But modern browsers now come with a [File System API](./https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) that provides an access to the filesystem on the device. This API doesn't provide a direct access to `/home` or any other directory, but instead, provides an isolated directory in which it's possible to do any kind of filesystem operation (expect links).

After taking a look at the documentation, here is how it works in javascript

```javascript
// we get access to the isolated directory for our page
const root = await navigator.storage.getDirectory();
// we open a file
const draftHandle = await root.getFileHandle("draft.txt", { create: true });
// we write to the file
const encoder = new TextEncoder();
const encodedMessage = encoder.encode(message);
const writeBuffer = accessHandle.write(encodedMessage, { at: readBuffer });
```

With this piece of code, there's one thing that's introduced: the storage layer will have to be `async`.

#### Existing crates

After looking on [crates.io](https://crates.io), I encountered the nice [web-fs](https://crates.io/crates/web-fs) crate. The creator promotes the use of [web-fs](https://crates.io/crates/web-fs) as an wasm alternative for [async-fs](https://crates.io/crates/async-fs). With a simple `#[cfg(target_arch = "wasm32")]`, one can switch from one implementation or the other.

But after trying things out and digging a bit more in the code itself, something surprised me. Most of the filesystem operations are offloaded to a separate web worker, which is fair considering the `createSyncAccessHandle()` [method](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle/createSyncAccessHandle) is only available in a dedicated web worker.

Taking a step back here: would I run a search engine, that does some heavy operations in the main thread? I'd probably have a dedicated web worker doing everything. So this abstraction level is therefore not needed anybody, making things simpler.

#### Implementing a simple wrapper

Implementing the [File System API](./https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) wrapper in rust is quite an easy task considering [web-sys](https://crates.io/crates/web-sys) already extracts most of the javascript API to Rust.

Here is now how to get the root directory directly in Rust

```rust
// getting the worker scope
let scope = js_sys::global().unchecking_into::<WorkerGlobalScope>();
// getting the storage manager
let manager = scope.navigator().storage();
// getting the root directory
let root = JsFuture::from(manager.get_directory())
    .await
    .unwrap()
    .dyn_into::<FileSystemDirectoryHandle>()
    .unwrap();
```

Starting from this, we can create a file and write into it

```rust
// definiting the options for opening the file
// surprisingly, no need to be mutable
let opts = FileSystemGetFileOptions::new();
opts.set_create(true);
let promise = root.get_file_handle_with_options("draft.txt", &opts);
let file = JsFuture::from(promise)
    .await
    .unwrap()
    .dyn_into::<FileSystemFileHandle>()
    .unwrap();
// creating an access handle
let promise = file.create_sync_access_handle();
let access = JsFuture::from(promise)
    .await
    .unwrap()
    .dyn_into::FileSystemSyncAccessHandle<>()
    .unwrap();
```

This is how simple the [File System API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) can be accessed.

This `FileSystemSyncAccessHandle` provides the capability to read and write from/at any point in the file. Starting from this, it's now possible to implement [`AsyncRead`](https://docs.rs/futures-io/0.3.31/futures_io/trait.AsyncRead.html), [`AsyncSeek`](https://docs.rs/futures-io/0.3.31/futures_io/trait.AsyncSeek.html) and [`AsyncWrite`](https://docs.rs/futures-io/0.3.31/futures_io/trait.AsyncWrite.html) which gives all the capabilities to implement the storage layer.

I'm not going to elaborate more about this, it's probably enough already, but if you're interested about its implementation, I've made a crate out of it, [browser-fs](https://crates.io/crates/browser-fs), that can be used like `web-fs`, but without the overhead.

### Merging The Different Implementations

Now that there is a way to write to the filesystem from any platform, creating a module for accessing the filesystem will be as simple as

```rust
#[cfg(target_arch = "wasm32")]
use browser_fs::*;
#[cfg(not(target_arch = "wasm32"))]
use async_fs::*;
```

## The Encryption Layer

I started with a quite simple question: how do we encrypt data efficiently without compromising security? After evaluating several options, I landed on [`aes-gcm`](https://en.wikipedia.org/wiki/Galois/Counter_Mode) (AES Galois/Counter Mode). This choice wasn't random, AES-GCM gives us two crucial guarantees: our data stays confidential and we can verify its authenticity. Plus, the [aes-gcm](https://crates.io/crates/aes-gcm) crate provides a battle-tested implementation we can trust.

I initially considered implementing streaming encryption, which would let us read and write encrypted content directly on the filesystem. It seemed elegant on paper - why load entire files into memory when we could stream them? But as I dug deeper, I realized I was potentially overcomplicating things. Here's why:

1. Implementing streaming encryption with both authenticity and confidentiality is tricky
2. Making it play nice with async operations would add even more complexity
3. Given our use case (small index files optimized for mobile sync), the benefits wouldn't justify the complexity

What sealed the deal for me was discovering that modern browsers support AES-GCM through their [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API). This means we could potentially offload encryption tasks to the browser's native implementation, saving both processing power and WASM bundle size.

### The Cipher Implementation

Let's walk through how I built the encryption layer. First, we need a way to carry our cipher around. Since we'll be using it across different file instances and possibly different threads, we want something that's both clone-able and thread-safe:

```rust
use std::sync::Arc;

#[derive(Clone)]
pub struct Cipher(Arc<aes_gcm::Aes256Gcm>);
```

Now, how do we create new ciphers? We need a way to initialize them with a key. Here's what I came up with:

```rust
impl Cipher {
    pub fn from_key(input: &[u8]) -> Result<Self, Error> {
        aes_gcm::Aes256Gcm::new_from_slice(input)
            .map(|inner| Self(Arc::new(inner)))
            .map_err(|_| Error::InvalidKeyLength)
    }
}
```

The interesting part comes with encryption and decryption. AES-GCM requires a unique `nonce` (number used once) for each message. Instead of asking users to manage nonces themselves, which would be error-prone, I decided to generate a new one for each encryption and prefix it to the encrypted data. This way, decryption becomes straightforward as the nonce is right there in the data:

```rust
impl Cipher {
    pub fn encrypt(&self, input: &[u8]) -> std::io::Result<Vec<u8>> {
        // Each encryption gets its own 96-bit nonce
        let nonce = aes_gcm::Aes256Gcm::generate_nonce(&mut OsRng);
        let ciphertext = self.0
            .encrypt(&nonce, input)
            .map_err(|_| Error::EncryptionFailed)?;

        // We pack the nonce with the encrypted data
        let mut result = nonce.to_vec();
        result.extend(ciphertext);
        Ok(result)
    }

    pub fn decrypt(&self, input: &[u8]) -> Result<Vec<u8>> {
        // First 12 bytes are our nonce
        let Some((nonce, payload)) = input.split_at_checked(12) else {
            return Err(Error::DecryptionFailed);
        };

        let nonce = aes_gcm::Nonce::<U12>::from_slice(nonce);
        self.0.decrypt(nonce, payload)
            .map_err(|_| Error::DecryptionFailed)
    }
}
```

Putting it all together with proper error handling, here's our complete encryption solution:

```rust
use std::sync::Arc;

use aes_gcm::aead::consts::U12;
use aes_gcm::aead::{Aead, OsRng};
use aes_gcm::{AeadCore, KeyInit};

pub enum Error {
    InvalidKeyLength,
    EncryptionFailed,
    DecryptionFailed,
}

#[derive(Clone)]
pub struct Cipher(Arc<aes_gcm::Aes256Gcm>);

impl Cipher {
    pub fn from_key(input: &[u8]) -> Result<Self, Error> {
        aes_gcm::Aes256Gcm::new_from_slice(input)
            .map(|inner| Self(Arc::new(inner)))
            .map_err(|_| Error::InvalidKeyLength)
    }

    pub fn encrypt(&self, input: &[u8]) -> std::io::Result<Vec<u8>> {
        // Each encryption gets its own 96-bit nonce
        let nonce = aes_gcm::Aes256Gcm::generate_nonce(&mut OsRng);
        let ciphertext = self.0.encrypt(&nonce, input)
            .map_err(|_| Error::EncryptionFailed)?;
        // We pack the nonce with the encrypted data
        let mut result = nonce.to_vec();
        result.extend(ciphertext);
        Ok(result)
    }

    pub fn decrypt(&self, input: &[u8]) -> Result<Vec<u8>> {
        // First 12 bytes are our nonce
        let Some((nonce, payload)) = input.split_at_checked(12) else {
            return Err(Error::DecryptionFailed);
        };

        let nonce = aes_gcm::Nonce::<U12>::from_slice(nonce);
        self.0.decrypt(nonce, payload)
            .map_err(|_| Error::DecryptionFailed)
    }
}
```

This encryption layer might seem simple, but it's intentionally so. It provides strong security guarantees while remaining easy to use and understand. When building complex systems, I've learned that simple, well-understood solutions often outperform clever but complicated ones.

## Wrapping Things Up

Now that we can write on the filesystem and encrypt the data, let's write a piece of code that plug them both together. But to do so, we need to define what will be written or what hierarchy it will have.

### The File Trait

Let's stay open minded here, considering this is the early stage of the implementation. We need to consider that we need to store encrypted files, but maybe some clear files as well, for things what wouldn't be at risk.

If we start with a simple item, let's define what a file should look like.

```rust
trait File {
    fn write(&self, payload: &[u8]) -> impl Future<Output = std::io::Result<()>>;
    fn read(&self) -> impl Future<Output = std::io::Result<Vec<u8>>>;
    fn delete(self) -> impl Future<Output = std::io::Result<()>>;
}
```

Where, when using an unencrypted file, we get the following

```rust
struct ClearFile {
    path: std::path::PathBuf,
}

impl File for ClearFile {
    async fn write(&self, payload: &[u8]) -> std::io::Result<()> {
        fs::write(&self.path, payload).await
    }

    async fn read(&self) -> std::io::Result<Vec<u8>> {
        fs::read(&self.path).await
    }

    async fn delete(self) -> std::io::Result<()> {
        fs::delete(&self.path).await
    }
}
```

And the encrypted file is slightly more complicated

```rust
struct EncryptedFile {
    path: std::path::PathBuf,
    cipher: Cipher,
}

impl File for ClearFile {
    async fn write(&self, payload: &[u8]) -> std::io::Result<()> {
        let payload = self.cipher.encrypt(payload)
            .map_err(std::io::Error::other)?;
        fs::write(&self.path, &payload).await
    }

    async fn read(&self) -> std::io::Result<Vec<u8>> {
        let content = fs::read(&self.path).await?;
        self.cipher.decrypt(&content)
            .map_err(std::io::Error::other)
    }

    async fn delete(self) -> std::io::Result<()> {
        fs::delete(&self.path).await
    }
}
```

It's now event possible to extend the `File` trait to make it more practical and add serialization support.

```rust
trait File {
    // keep the other trait functions here

    // Serialize and write a value to the file
    async fn serialize<V: serde::Serialize>(&self, value: &V) -> std::io::Result<()> {
        let payload = serde_cbor::to_vec(value).map_err(|err| {
            std::io::Error::new(std::io::ErrorKind::InvalidData, err)
        })?;
        self.write(&payload).await
    }

    /// Read and deserialize a value from the file
    async fn deserialize<V: serde::de::DeserializeOwned>(&self) -> std::io::Result<V> {
        let content = self.read().await?;
        serde_cbor::from_slice(&content).map_err(|err| {
            std::io::Error::new(std::io::ErrorKind::InvalidData, err)
        })
    }
}
```

You can see in the code above that I'm using [`serde_cbor`](https://crates.io/crates/serde_cbor): this is for simplicity and for performance, you can keep whichever you prefer.

### Directory Management

Now that we have file operations, let's write something at the directory level that provides us ways to create instances of `ClearFile` or `EncryptedFile`.

```rust
struct Directory {
    cipher: Cipher,
    path: std::path::PathBuf,
}

impl Directory {
    /// creates a new instance of a ClearFile, without creating it on the filesystem
    fn clear_file(&self, relative: impl AsRef<Path>) -> ClearFile {
        ClearFile {
            path: self.path.join(relative),
        }
    }

    /// creates a new instance of a EncryptedFile, without creating it on the filesystem
    fn encrypted_file(&self, relative: impl AsRef<Path>) -> EncryptedFile {
        EncryptedFile {
            path: self.path.join(relative),
            cipher: self.cipher.clone(),
        }
    }
}
```

And if we add some extra functions to do basics operations on the directory, like listing files, creating the directory or deleting the directory and its content

```rust
impl Directory {
    /// creates the directory if not exists
    async fn create(&self) -> std::io::Result<()> {
        fs::create_dir_all(&self.path).await
    }

    /// list all the files in the current directory
    async fn files(&self) -> std::io::Result<Vec<PathBuf>> {
        use futures_lite::StreamExt;

        let mut result = Vec::new();
        if let Ok(mut res) = fs::read_dir(&self.path).await {
            while let Ok(Some(item)) = res.try_next().await {
                let filepath = item.path();
                if filepath.is_file() {
                    result.push(filepath);
                }
            }
        }

        Ok(result)
    }

    /// removes the current directory and all its content
    async fn delete(&self) -> std::io::Result<()> {
        fs::remove_dir_all(&self.path).await
    }
}
```

At this point, we have everything we need to interact with the filesystem, read and write files, drop files and handle the working directory of the search engine.

## Conclusion

In this first part of this series on building a search engine, we've laid a solid foundation by addressing one of the most fundamental challenges: cross-platform storage with encryption support. We've successfully created:

- A unified storage interface that works across desktop, mobile, and browser platforms
- An efficient encryption layer using AES-GCM that provides both security and authenticity
- A clean API that abstracts away the complexities of different filesystems and encryption

The resulting implementation is not just functional, but also maintainable and extensible. By choosing the [File System API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) for browser storage and implementing a straightforward encryption layer, we've avoided unnecessary complexity while maintaining strong security guarantees.

This storage layer will serve as the backbone for our search engine, allowing us to reliably store and retrieve indexed data across all platforms. In the next article, we'll build upon this foundation to start implementing the core indexing functionality of our search engine.

Stay tuned for the next part where we'll dive into the exciting world of search algorithms and index structures!
