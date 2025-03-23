+++
title = "Building a search engine from scratch, in Rust: part 3"
description = "Or how we'll implement sharding and transactions for our search engine."
date = 2025-04-05

[extra]
emoji = "🦀"

[taxonomies]
tags = ["rust", "search-engine", "webassembly", "encryption", "cross-platform", "tutorial", "performance"]
+++

In the [previous article](../202503191700-search-engine-part-2/), we implemented the core document structure and indexes for our search engine. Now, let's tackle how we'll make our search engine scalable through sharding and reliable through transactions.

### Shard Definition

Here's how our sharding architecture organizes data across multiple shards:

```ascii
                 Manifest
                +--------+
                | shards |
                +--------+
                     |
          +----------+-----------+
          v          v           v
    +---------+ +---------+ +---------+
    | Shard 0 | | Shard 1 | | Shard 2 |
    |---------| |---------| |---------|
    | 0 - 100 | | 101-200 | | 201-300 |
    +---------+ +---------+ +---------+
        |           |           |
    Collection  Collection  Collection
     Indexes     Indexes     Indexes
```

At this point, we have everything we need to build a shard: a collection to list all the documents in the shard and an index for each type. A simple implementation of that shard would look like this.

```rust
// an abstraction to be able to map the indexes
enum AnyIndex {
    Boolean(BooleanIndex),
    Integer(IntegerIndex),
    Tag(TagIndex),
    Text(TextIndex),
}

struct Shard {
    collection: Collection,
    indexes: HashMap<Kind, AnyIndex>,
}
```

But this is a single shard representation, we might have several an need to have a representation as well.

```rust
struct Manager {
    shards: BTreeMap<u64, Shard>,
}
```

With this representation, the `u64` in the `BTreeMap` will represent the minimum in the range of partition handled by that shard. When initialized, the first shard key will be `0`.

But the two previous representation are actually wrong: this would mean that we'll load in memory the entire search engine, which doesn't scale. Instead, the `Shard` structure will only contain the filenames of the collection and indexes, which will be loaded in memory only when needed, and written to disk when they are not needed anymore.

The `Manager` structure can then be renamed to `Manifest` and will be, as well, persisting on disk, representing the state of the search engine at a given point in time.

```rust
struct Manifest {
    shards: BTreeMap<u64, Shard>,
}

struct Shard {
    collection: Filename,
    indexes: HashMap<Kind, Filename>,
}
```

This manifest will be stored in the working directory as `manifest.bin` and every file (collections and indexes) will have a random name.

Sharding architecture highlights:
- Manifest-based shard management
- File-based storage with lazy loading
- Transaction support for concurrent operations
- Dynamic shard splitting based on size
- Recovery mechanism for incomplete transactions

#### Transaction Mechanism

This level of abstraction for the manifest allows us to add or delete shards when needed but there's an issue: we cannot block the access to the search engine each time we insert a document. We should be able to insert a set of documents while using the index and just block its access when writing the updated manifest to disk.

Following a similar mechanism to a transactional database, inserting data will require initializing a transaction, which will create a temporary manifest file which will contain the names of all the original indexes and the names of the indexes that have been updated. Updating a collection or an index will create a new file on disk but non updated indexes will remain the same.

```ascii
    Original State         Transaction             Committed State
    +--------------+       +--------------+        +--------------+
    | manifest.bin |       | manifest.tx  |        | manifest.bin |
    +--------------+       +--------------+        +--------------+
    | idx1.bin     |       | idx1.bin     |        | idx1.bin     |
    | idx2.bin     |  -->  | idx2_new.bin |  -->   | idx2_new.bin |
    | idx3.bin     |       | idx3.bin     |        | idx3.bin     |
    +--------------+       +--------------+        +--------------+
```

This would give use this code for the shard management

```rust
/// represents a file during a transaction
struct TxFile {
    /// original file path, if it exists
    // a shard can not have any boolean index but it can be created after an update
    base: Option<Filename>,
    /// new file path after changes, if modified
    // the filename once the transaction is committed
    next: Option<Filename>,
}

/// represents a shard during a transaction
struct TxShard {
    /// collection file state
    collection: TxFile,
    /// index files state for each kind
    indexes: HashMap<Kind, TxFile>,
}

/// manages the state of all shards during a transaction
struct TxManifest {
    /// maps shard keys to their transaction state
    /// uses BTreeMap to maintain order for efficient splits
    shards: BTreeMap<u64, TxShard>,
}
```

This transaction manifest would be written down to the filesystem depending on the platform: in the browser, we cannot know when the page will be closed so better write it after each operation, while on mobile, the app can do a simple operation before closing. This provides a nice way of being able to recover a transaction that has not been committed.

That commit operation simply consists in, for each file of each shard, taking the `next` filename if exists or the `base` one, and write it in the `manifest.bin`. This commit operation is atomic, and then less prone to errors.

#### Sharding, Or Not Sharding

Before talking about how to shard, we should talk about when we should decide to shard.

Considering I've decided to leave the limit configurable depending on the size of the files, we have to be able to determine the size of a index file, once serialized and encrypted. Considering the time needed to serialize and encrypt is CPU bound (and after some experiments), writing the encrypted file to disk in order to determine its size brings too much overhead and kills the performance.

The second option I came up with was to compute the size of the index each time it gets updated. It's quite time consuming, it's the bruteforce way, but compared still peanuts compared to the time needed to serialize it and encrypting it. And we'll be able to improve this performance later, there's an entire section dedicated for that.

Considering the redondancy in the structure of the indexes, we can make something smart that won't require too much repeat. Let's implement a `ContentSize` that evaluates the size of the structure.

```rust
/// provides size estimation for optimizing shard splits
trait ContentSize {
    /// returns estimated size in bytes when serialized
    fn estimate_size(&self) -> usize;
}

// for constant value sizes
macro_rules! const_size {
    ($name:ident) => {
        impl ContentSize for $name {
            fn estimate_size(&self) -> usize {
                std::mem::size_of::<$name>()
            }
        }
    }
}

// and for other types like u16, u32, u64 and bool
const_size!(u8);

// let's consider a string just for its content size
impl<T: AsRef<str>> ContentSize for T {
    /// estimates string size based on character count
    /// note: This is an approximation and doesn't account for encoding overhead
    fn estimate_size(&self) -> usize {
        self.as_ref().len()
    }
}

// for maps, similar
impl<K: ContentSize, V: ContentSize> for HashMap<K, V> {
    /// recursively estimates size of all keys and values
    /// used to determine when to split shards
    fn estimate_size(&self) -> usize {
        self.iter().fold(0, |acc, (key, value)| acc + key.estimate_size() + value.estimate_size())
    }
}
```

With this in hand, we can implement it for all the indexes and we'll have a rough idea of the size of the file. Considering that encryption will add a bit of overhead in size, we can decide to split the index when it reaches 90% of the limit size.

But now the question is: how to implement the sharding mechanism? It's quite simple and will be based on what we put in place earlier in this article.

Here's a visual example of how a shard splits:

```ascii
Before Split:
    Shard 0 [0-200]
    +---------------+
    | Doc1  [50]    |
    | Doc2  [75]    |
    | Doc3  [125]   |
    | Doc4  [175]   |
    +---------------+

After Split:
    Shard 0 [0-100]     Shard 1 [101-200]
    +---------------+   +----------------+
    | Doc1  [50]    |   | Doc3  [125]    |
    | Doc2  [75]    |   | Doc4  [175]    |
    +---------------+   +----------------+
```

In the `Collection` structure lives a `BTreeMap` of all the entries by sharding value. A simple way to shard is just to split that `BTreeMap` at its center of gravity so that we have almost the same number of documents in both shard.

Now we just have to implement a splitting function on the collection and all the indexes.

```rust
impl Collection {
    /// attempts to split the collection into two roughly equal parts
    /// returns None if splitting is not possible (e.g., all documents have same shard value)
    fn split(&mut self) -> Option<Collection> {
        // if all the entries have the same sharding value, it's not possible to split considering
        // a sharding value can only be in one shard.
        if self.sharding < 2 {
            return None;
        }

        // calculate target size for new collection
        let total_count = self.entries_by_name.len();
        let half_count = total_count / 2;

        let mut new_collection = Collection::default();

        // keep moving entries until we reach approximately half size
        while new_collection.entries_by_name.len() < half_count && self.sharding.len() > 1 {
            // if this happens, it means we moved everything from the old shard,
            // which shouldn't happen considering that we check the number of shards
            let Some((shard_value, entries)) = self.sharding.pop_last() {
                return new_collection;
            }
            // moving from the old collection to the new collection one by one
            for entry_index in entries {
                if let Some(name) = self.entries_by_index.remove(&entry_index) {
                    self.entries_by_name.remove(&name);
                    new_collection.entries_by_index.insert(entry_index, name.clone());
                    new_collection.entries_by_name.insert(name, entry_index);
                }
            }
            new_collection.sharding.insert(shard_value, entries);
        }

        new_collection
    }
}
```

This will give use a new collection if it was possible to split it. If it's possible, we can now split all the indexes.

```rust
// similar for each index
impl BooleanIndex {
    // we just create a new index an move every item from the old entries to the new index
    fn split(&mut self, entries: &HashSet<EntryIndex>) -> BooleanIndex {
        let mut next = BooleanIndex::default();
        self.content.iter_with_mut(|(term, term_postings)| {
            term_postings.iter_with_mut(|(attribute, attribute_postings)| {
                // fetch the intersection
                let intersection = entries.iter().filter(|entry_index| attribute_postings.contains_key(entry_index)).collect();
                if !intersection.is_empty() {
                    // only create the postings if there is an intersection
                    let next_term_postings = next.content.entry(term.clone()).or_default();
                    let next_attribute_postings = next_term_postings.entry(attribute.clone()).or_default();
                    for entry_index in intersection.iter() {
                        // remove from the old one and insert in the new one
                        if let Some(entry_posting) = attribute_postings.remove(entry_index) {
                            next_attribute_postings.insert(entry_index, entry_posting);
                        }
                    }
                }
                !intersection.is_empty()
            })
        });
        next
    }
}
```

After this creation of a new shard, we can inject it in the transaction manifest, with the sharding key being the minimum of all the sharding keys, which can simply be accessed using the [`first_key_value` function of the `BTreeMap`](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.first_key_value). And at the next commit, it will be possible to search in it.

## Conclusion

In this third part of our series on building a search engine, we've implemented the architecture for sharding and transactions. We've tackled several fundamental challenges:

- Designing a flexible yet efficient document schema
- Implementing specialized indexes for different data types
- Creating a robust sharding mechanism
- Building a transaction-safe storage system

Our implementation prioritizes both performance and safety, with careful consideration for resource constraints across different platforms. The use of numeric indexes instead of strings, lazy loading of shards, and atomic transactions all contribute to making our search engine efficient and reliable.

**Key Achievements:**
- Type-safe document schema with support for multiple values
- Memory-efficient index structures
- Platform-independent storage abstraction
- Transaction-safe operations
- Dynamic sharding capability

### What's Next?

In the part 4: Search Implementation, we'll build upon these foundations to implement the actual search functionality.

Stay tuned to learn how we'll turn these carefully designed data structures into a fully functional search engine that can handle complex queries across all our supported data types!
