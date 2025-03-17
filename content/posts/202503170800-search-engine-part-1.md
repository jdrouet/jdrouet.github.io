+++
title = "Search engine storage layer"
description = "Or how to write on disk efficiently in the browser or any other device."
date = 2025-03-17

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

This is how simple the [File System API](./https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) can be accessed.

This `FileSystemSyncAccessHandle` provides the capability to read and write from/at any point in the file. Starting from this, it's now possible to implement [`AsyncRead`](https://docs.rs/futures-io/0.3.31/futures_io/trait.AsyncRead.html), [`AsyncSeek`](https://docs.rs/futures-io/0.3.31/futures_io/trait.AsyncSeek.html) and [`AsyncWrite`](https://docs.rs/futures-io/0.3.31/futures_io/trait.AsyncWrite.html) which gives all the capabilities to implement the storage layer.

I'm not going to elaborate more about this, it's probably enough already, but if you're interested about its implementation, I've made a crate out of it, [browser-fs](https://crates.io/crates/browser-fs), that can be used like `web-fs`, but without the overhead.

### Merging the different implementations

Now that there is a way to write to the filesystem from any platform, creating a module for accessing the filesystem will be as simple as

```rust
#[cfg(target_arch = "wasm32")]
use browser_fs::*;
#[cfg(not(target_arch = "wasm32"))]
use async_fs::*;
```
