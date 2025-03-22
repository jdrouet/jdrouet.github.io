+++
title = "Building a search engine from scratch, in Rust: part 2"
description = "Or how we'll structure go from a set of documents to a set of structured indexes."
date = 2025-03-19

[extra]
emoji = "🦀"

[taxonomies]
tags = ["rust", "search-engine", "webassembly", "encryption", "cross-platform", "tutorial", "performance"]
+++

In the [previous article](../202503170800-search-engine-part-1/), I explained how we'll write on disk and how we'll implement an abstraction so that it works on any device as well as in the browser. Now, it's time to start thinking about what we'll be storing and in what shape.

## Defining A Document

In the majority of time, the use of a search engine is coupled with a database, because it's rarely the search-engine responsibility to store the document. The database stores the things and the search-engine indexes the important attributes that need to be searched on. Indexing content is memory intensive so better not abuse it.

The consequence of this it that the document that are being indexed, most of the time, have a defined structure, a schema that they follow.

If we look at an example of search engine like [Tantivy](https://github.com/quickwit-oss/tantivy), indexing a document requires the definition of a schema. Using a similar pattern here seems fairly legit.

On the other side, if we look at [Elasticsearch](https://www.elastic.co/elasticsearch), the data structure is a bit more free. And indexed document can contain some nested attributes that are then flattened when being indexed. This means that the cardinality of attributes is almost infinite. This works well when you work on a server with a huge about of RAM and a but CPU, but when it comes to the browser on an old mobile, it's better to give it some limitations.

That being said, a use case has to be considered: if a document has tags (like the names of the recipients of a message), it would be great to be able to search through them without having to define `n` independent attributes. To avoid this, it should support arrays.

Now, to optimise querying later on and avoid querying with some integers an attribute that contains only text, the schema should be aware of the type of an attribute.

So let's write down some ground rules:
- a schema has a finite number of attributes
- an attribute has a defined attribute
- a document attribute cannot contain another document
- a document attribute can contain multiple values

If we follow those rules, we should end up with the following API

```rust
let schema = Schema::builder()
    .attribute("content", Kind::Text)
    .attribute("sender", Kind::Tag)
    .attribute("recipient", Kind::Tag)
    .attribute("timestamp", Kind::Integer)
    .attribute("encrypted", Kind::Boolean)
    .build();

let message = Document::new("message_id")
    .attribute("content", "Hello World!") // text
    .attribute("sender", "Alice") // tag
    .attribute("recipient", "Bob") // tag
    .attribute("recipient", "Charles") // multiple times the same attribute
    .attribute("timestamp", 123456789) // integer
    .attribute("encrypted", true); // boolean
```

It looks like a fairly simple.

Now, in [the introduction](../202503161800-search-engine-intro/), it was mentioned that the content of the index should be sharded. Sharding means we have to define a way to group documents. As a matter of simplicity, the sharding mechanism will be just based on an integer attribute defined in the document but if you want to look at more complex mechanism, I'll invite you to have a look at how [PostgreSql does it](https://www.postgresql.org/docs/current/ddl-partitioning.html).

This is something that should be defined in the schema, so that it's common across the entire search engine. So let's introduce a function to define it as follow.

```rust
let schema = Schema::builder()
    .attribute("content", Kind::Text)
    .attribute("sender", Kind::Tag)
    .attribute("recipient", Kind::Tag)
    .attribute("timestamp", Kind::Integer)
    .attribute("encrypted", Kind::Boolean)
    .shard_by("timestamp")
    .build()?;
```

And now, building the schema should fail if the sharding attribute is not defined or if it's not an `integer`.

It gives us the following structure for the schema.

```rust
/// Represents the possible data types that can be indexed
enum Kind {
    /// Simple true/false values
    Boolean,
    /// Unsigned 64-bit integers, used for numeric queries and sharding
    Integer,
    /// Single tokens that require exact matching (e.g. email addresses, IDs)
    Tag,
    /// Full-text content that will be tokenized and stemmed
    Text,
}

/// Defines the structure and rules for indexable documents
struct Schema {
    /// Maps attribute names to their types
    attributes: HashMap<String, Kind>,
    /// The attribute used to determine document sharding
    /// Must be of Kind::Integer
    shard_by: String,
}

impl Schema {
    // let's follow the builder pattern
    pub fn builder() -> SchemaBuilder {
        SchemaBuilder::default()
    }
}

enum Error {
    /// when the user didn't specify a sharding attribute
    ShardingAttributeNotSet,
    /// when the user specified how to shard but the attribute is not defined
    UnknownShardingAttribute
}

#[derive(Default)]
struct SchemaBuilder {
    attributes: HashMap<String, Kind>,
    shard_by: Option<String>,
}

impl SchemaBuilder {
    pub fn attribute(mut self, name: impl Into<String>, kind: Kind) -> Self {
        self.attributes.insert(name.into(), kind);
        self
    }

    pub fn shard_by(mut self, name: impl Into<String>) -> Self {
        self.shard_by = Some(name.into());
        self
    }

    pub fn build(self) -> Result<Schema, Error> {
        let Some(shard_by) = self.shard_by else {
            return Err(Error::ShardingAttributeNotSet);
        }
        if !self.attributes.contains_key(&shard_by) {
            return Err(Error::UnknownShardingAttribute);
        }

        Ok(Schema {
            attributes: self.attributes,
            shard_by,
        })
    }
}
```

**Note:** if we want to optimize some extra bytes, instead of using a `String`, we can use a `Box<str>` as we won't update the content of those strings.

Now, let's have a look at the `Document` API. we could create a document builder that will analyse every attribute value that gets inserted in the document, but this could become a bit complicated to handle all the possible errors that should never happen.

Instead of that, the search-engine will validate the document before inserting it in the indexes.

This gives use a relatively simple API for the `Document` as well.

```rust
/// A value that can be indexed
enum Value {
    /// Boolean values are stored as-is
    Boolean(bool),
    /// Integer values are used for range queries and sharding
    Integer(u64),
    /// Tags are stored without any processing
    Tag(String),
    /// Text values will be tokenized and processed
    Text(String),
}

/// Represents a document to be indexed
struct Document {
    /// Unique identifier for the document
    id: String,
    /// Maps attribute names to their values
    /// A single attribute can have multiple values
    attributes: HashMap<String, Vec<Value>>,
}

impl Document {
    pub fn new(id: impl Into<String>) -> Self {
        Self {
            id: id.into(),
            attributes: Default::default(),
        }
    }

    pub fn attribute(mut self, name: impl Into<String>, value: impl Into<Value>) -> Self {
        self.attributes.entry(name.into()).or_default().push(value.into());
        self
    }
}
```

This kind of API provides the flexibility to build any kind of attribute without having to think too much about the errors and then handle the validation when it gets inserted. The schema being fixed, this kind of errors should be covered by the users tests, by doing so, I decided to prioritise usability.

Now that we have defined our schema and document structure, here's how all the pieces fit together:

```ascii
+-------------+     +----------------+
| Document    |     | Collection     |
+-------------+     +----------------+
| id: String  |     | entries_by_idx |
| attributes  +---->| entries_by_name|
+-------------+     | sharding       |
                    +----------------+
                           |
                           v
        +----------------------------------+
        |           Indexes                |
        |----------------------------------|
        |                                  |
    +--------+   +---------+   +--------+  |
    |Boolean |   |Integer  |   |Text    |  |
    |Index   |   |Index    |   |Index   |  |
    +--------+   +---------+   +--------+  |
        |            |            |        |
        +------------+------------+--------+
```

Key takeaways:
- Documents have a fixed schema with typed attributes
- Supported types: Boolean, Integer, Tag, Text
- Documents can have multiple values for an attribute
- Sharding is based on an integer attribute
- Schema validation happens at document insertion

## Destructuring The Documents

Now that we know how the documents will be structured, it's time to dive in the serious stuff: the indexes.

The indexes will contain the information we need to find a document based on the query parameters. We'll try to keep the content of the index fairly small when serialized to maximise the content of indexed data.

So basically, we'll do a link between the data and the document identifier: from a given term (boolean, integer, tag or text), what document contains that term, and how many times. A basic data structure would look like this.

```rust
type Index = Map<Term, Map<DocumentIdentifier, Count>>;
```

And this would be reproduced for every index, and for every term. When you think about the `DocumentIdentifier`, which would be a `String`, each term would have a cost of [`size_of(DocumentIdentifier)`](https://doc.rust-lang.org/std/mem/fn.size_of.html) which is **at least** equal to the size of the string (plus some bytes depending on if we use `String` or `Box<str>`). This doesn't scale for big documents containing many terms and big identifiers, we need to use a different approach.

### The Collection File

If we introduce, for each shard, a `Collection` file that will contain a list of all the document identifiers and a `u32` to identify them, then in each index, we can use that `u32` to identify the document.

```rust
type EntryIndex = u32;
/// the collection file in each shard
type Collection = Map<EntryIndex, DocumentIdentifier>;
/// for each index type
type Index = Map<Term, Map<EntryIndex, Count>>;
```

This should reduce the cost significantly.

Now, in order to shard the collections and indexes, we need to store the attribute used for sharding close. If we use the index to find the sharding value of every document, considering the structure of the index, doing so will not be performant enough.

Persisting that attribute in the collection should make it easier to access.

```rust
type EntryIndex = u32;
type ShardingValue = u64;

struct Collection {
    entries: HashMap<EntryIndex, DocumentIdentifier>,
    sharding: BTreeMap<ShardingValue, HashSet<EntryIndex>>,
}
```

That way, when one of our indexes reaches a critical size, we can just split in half the shard by taking all the entries based on the sharding `BTreeMap`. The `BTreeMap`, being sorted by design, provides the perfect API for that.

The next problem we'll have to tackle on that structure: when deleting a document from the index, how to efficiently go from the `DocumentIdentifier` to the `EntryIndex`? For that, we need to introduce a reverse map as follow.

```rust
type EntryIndex = u32;
type ShardingValue = u64;

struct Collection {
    entries_by_index: HashMap<EntryIndex, DocumentIdentifier>,
    entries_by_name: HashMap<DocumentIdentifier, EntryIndex>,
    sharding: BTreeMap<ShardingValue, HashSet<EntryIndex>>,
}
```

This improves the performance for getting an entry by name, but duplicates all the document identifiers on disk. We can do better by doing that duplication when serializing or deserializing our collection.

```rust
type EntryIndex = u32;
type ShardingValue = u64;
type DocumentIdentifier = Arc<str>;

/// Representation on disk of an entry
struct Entry {
    index: EntryIndex,
    name: DocumentIdentifier,
    shard: ShardingValue,
}

/// Representation on disk of a collection
struct PersistedCollection {
    entries: Vec<Entry>
}
```

This represents how we'll store our collection on disk. Each entry maintains its numeric index, document identifier, and sharding value in a simple vector structure.

```rust
/// Manages document identifiers and sharding information
struct Collection {
    /// Maps numeric indexes to document identifiers
    /// Uses u32 to optimize memory usage while supporting large datasets
    entries_by_index: HashMap<EntryIndex, DocumentIdentifier>,
    /// Reverse mapping for quick document lookups
    entries_by_name: HashMap<DocumentIdentifier, EntryIndex>,
    /// Maps sharding values to sets of document indexes
    /// Using BTreeMap for ordered access during shard splitting
    sharding: BTreeMap<ShardingValue, HashSet<EntryIndex>>,
}
```

In memory, we maintain bidirectional mappings between indexes and document identifiers for efficient lookups in both directions. The sharding map uses a `BTreeMap` to maintain order, which will be crucial for our sharding operations.

And here is the function to build a `Collection` based on its persisted state on disk.

```rust
/// Let's build the collection from the persisted state
impl From<PersistedCollection> for Collection {
    fn from(value: PersistedCollection) -> Collection {
        let mut entries_by_index = HashMap::with_capacity(value.entries.len());
        let mut entries_by_name = HashMap::with_capacity(value.entries.len());
        let mut sharding = BTreeMap::default();
        for entry in value.entries {
            entries_by_index.insert(entry.index, entry.name.clone());
            entries_by_name.insert(entry.name.clone(), entry.index);
            sharding.entry(entry.shard).or_default().insert(entry.index);
        }
        Collection {
            entries_by_index,
            entries_by_name,
            sharding,
        }
    }
}
```

Notice the use of [`Arc<str>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) instead of `String`. We need to have multiple reference to the same string in memory. If we use `String`, we'll pay several time the cost of that string length. When using `Arc<str>`, we only pay the price of the string length once and just the price of the pointer each time we clone it.

> One could ask, considering the advantage of `Arc<str>`, why not writing that directly to disk or use it in the other indexes. Well, it doesn't work when serialized. [`Arc<str>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) contains a pointer in memory of where the string is. When serialize/deserialize, this memory address changes, so the serializer just replaces the pointer with its actual value, which means duplication of data.

Now, let's take a step back. In the current `Index` representation, there's no mention of attribute, but the attribute is quite similar to the document identifier: we don't know how big it could be and the size on disk is related to the size of the string. Might be worst adding it in our collection structure. And considering we'll need to access the attribute name by an `AttributeIndex` and the other way around, we need to implement a similar mechanism.

```rust
/// Let's keep a fairly low number of attributes, 256 attributes should be enough
type AttributeIndex = u8;

struct Attribute {
    index: AttributeIndex,
    name: Arc<str>,
}

struct PersistedCollection {
    attributes: Vec<Attribute>,
    entries: Vec<Entry>
}

struct Collection {
    attributes_by_index: HashMap<AttributeIndex, Arc<str>>,
    attributes_by_name: HashMap<Arc<str>, AttributeIndex>,
    // ...other fields
}
```

At this point, we have everything ween need to build our collection.

Key points about collections:

- Uses numeric indexes to reduce storage overhead
- Maintains bidirectional mappings for efficient lookups
- Stores sharding information for easy partitioning
- Uses `Arc<str>` for memory-efficient string handling
- Attributes are also indexed for space optimization

Before we dive into each index type, here's how the hierarchical structure works for our indexes:

```ascii
Term Index
+---------+
|'rust'   |--+
|'fast'   |  |
|'search' |  |
+---------+  |
             v
        Attribute Index
        +-------------+
        |'content'    |--+
        |'title'      |  |
        +-------------+  |
                         v
                    Document Index
                    +------------+
                    |Doc1: [0,5] |
                    |Doc2: [3]   |
                    +------------+
```

### Boolean Index

Now let's start with a simple index, the boolean index. This will only have a simple use cases: fetching the entries where an attribute is `true` or `false`.

So if we follow the structure we defined earlier, we'd end up with the following tructure.

```rust
type ValueIndex = u8;

/// stores boolean values for quick true/false queries
struct BooleanIndex {
    /// maps boolean values to their postings
    /// structure: bool -> attribute -> document -> value positions
    content: HashMap<bool, HashMap<AttributeIndex, HashMap<EntryIndex, HashSet<ValueIndex>>>>,
}
```

Where `ValueIndex` is the index of that term, for an attribute, for that entry.

That way, inserting an value can be done this way.

```rust
impl BooleanIndex {
    fn insert(
        &mut self,
        entry_index: EntryIndex,
        attribute_index: AttributeIndex,
        value_index: ValueIndex,
        term: bool,
    ) -> bool {
        let term_postings = self.content.entry(term).or_default();
        let attribute_postings = term_postings.entry(attribute_index).or_default();
        let entry_postings = attribute_postings.entry(entry_index).or_default();
        entry_postings.insert(value_index)
    }
}
```

For deletion, we get the following implementation

```rust
impl BooleanIndex {
    fn delete(&mut self, entry_index: EntryIndex) -> bool {
        let mut changed = false;
        for (term, term_postings) in self.content.iter_mut() {
            for (attribute, attribute_postings) in term_postings.iter_mut() {
                changed |= attribute_postings.remove(&entry_index).is_some();
            }
        }
        changed
    }
}
```

This is fairly simple. But there is an invisible problem here: empty postings don't get removed.

We could, after each removal, check that the posting is not empty, and if empty, remove it from the parent.

#### Self Cleaning Postings

To avoid doing that in every indexes, writing a trait would make things simpler.

First, we need to provide a way for the parent container to know if the posting is empty. Considering there is no trait shared between all the map and set implementations, we should define one.

```rust
/// Trait to check if an element is empty
trait IsEmpty {
    fn is_empty(&self) -> bool;
}

impl<K, V> IsEmpty for HashMap<K, V> {
    fn is_empty(&self) -> bool {
        HashMap::is_empty(self)
    }
}
```

It's also possible to write a macro so that we can implement this on `HashMap`, `HashSet`, `BTreeMap` and all other maps.

And now, we can define a trait to have a self cleaning container

```rust
trait WithMut<K, V> {
    fn iter_with_mut<Cb>(&mut self, callback: Cb) -> bool
    where
        Cb: FnMut((&K, &mut V)) -> bool;
}

impl<K, V> WithMut<K, V> for HashMap<K, V>
where
    K: Clone,
    V: IsEmpty,
{
    fn iter_with_mut<Cb>(&mut self, mut callback: Cb) -> bool
    where
        cb: FnMut((&K, &mut V)) -> bool
    {
        let mut keys_to_remove = Vec::new();
        let changed = self.iter_mut().fold(false, |acc, (key, value)| {
            let changed = callback((key, value));
            if value.is_empty() {
                // we gather all the keys that have to be removed
                keys_to_remove.push(key.clone());
            }
            acc || changed
        });
        // we cleanup after giving back the mutability
        for key in keys_to_remove {
            self.remove(&key);
        }
        changed
    }
}

// similar code for HashSet, etc
```

And with that fix, the deletion code can be simplified to a way more elegant way

```rust
impl BooleanIndex {
    fn delete(&mut self, entry_index: EntryIndex) -> bool {
        self.content.iter_with_mut(|(_, term_postings)| {
            term_postings.iter_with_mut(|(_, attribute_postings)| {
                attribute_postings.remove(&entry_index).is_some()
            })
        })
    }
}
```

### Integer Index

Now that we have the boolean index, writing the integer index will be quite trivial. We'll just have a small difference. When on the boolean index we only query `true` or `false` for a given attribute, on the integer index, one might want to query for a range, below, above and so on. So the terms should be stored sorted. Luckily, doing so just involves switching a `HashMap` to become a `BTreeMap`.

```rust
/// stores integer values for range queries
struct IntegerIndex {
    /// uses BTreeMap for efficient range queries
    /// structure: number -> attribute -> document -> value positions
    content: BTreeMap<u64, HashMap<AttributeIndex, HashMap<EntryIndex, HashSet<ValueIndex>>>>,
}
```

And that's it, we take the same code as above, and it will work.

### Tag Index

Let's pause a bit and clarify something: what's the difference between a tag index and a text index. Well, in this implementation, a tag index is an index where the term, the tag is a single item, that doesn't get preprocessed and that we'll search for as it is. If we insert `Alice`, then searching `alice` won't find it, it's only exact match. This is good for matching email addresses, folder names, etc.
The text index, on the opposite, the input data gets processed, but we'll talk about it right after.

Once again, we can follow a similar architecture than the two other indexes.

```rust
/// stores tag values for exact matching
struct TagIndex {
    /// uses Box<str> to optimize memory usage
    /// structure: tag -> attribute -> document -> value positions
    content: HashMap<Box<str>, HashMap<AttributeIndex, HashMap<EntryIndex, HashSet<ValueIndex>>>>,
}
```

You can notice here that, instead of using `String`, we are using `Box<str>`. This is because the size of a `String` is of 24 bytes, plus the characters, while `Box<str>` has a size of 16 bytes, plus the characters. This might not seem much, but every byte is worst keeping when you use a Nokia 3310.

### Text Index

As I mentioned earlier, the text index is a bit more complicated than the tag index. When the tag index is made to return entries that contain a term with exact matching, the text index will provide functionalities to search inside the provided text: the tag index is for a single value like `alice` or `personal` while you give the text index complete sentences, articles, documents, etc. And on top of that, we'll want to be able to find some elements in that text even if there are some mistakes in the query: `personnal` should match documents containing `this is a personal document`, event with a typo.

So we'll have to adjust a bit the interface we used for the other indexes considering now, an input value will carry more information: an indexed attribute value will be composed of several tokens. A token will be defined by its term, it's position in the original text and the index in the list of tokens.

Processing that input will consist in extracting the words from the input text, lowercase them and apply some [stemming](https://en.wikipedia.org/wiki/Stemming) before inserting all the terms in the index. The first ones are fairly simple but the stemming mechanism can be quite complicated, especially when we consider doing something that handles multiple languages.

Here is a simple example of how the preprocessing process should work. I removed the complexity from the `capture_all` and `stem` function to make it simpler to read. Those functions will just be the ones provided by the [`regex` crate](https://crates.io/crates/regex) and the [`rust-stemmers` crate](https://crates.io/crates/rust-stemmers).

```rust
let input = "Hello World! Do you want tomatoes?";
// extracting the words
let tokens = capture_all(r"(\w{3,20})", input)
    // lowercase them
    .map(|(position, term)| (position, term.to_lower_case()))
    // keep track of the token index
    .enumerate()
    // stemming
    .map(|(index, (position, term))| Token {
        index,
        position,
        term: stem(term),
    })
    .collect::<Vec<_>>();
// should return
// { term: "hello", index: 0, position: 0 }
// { term: "world", index: 1, position: 6 }
// { term: "you", index: 2, position: 16 }
// { term: "want", index: 3, position: 20 }
// { term: "tomato", index: 4, position: 25 }
```

And we'll require to store those positions in the index as well, considering a term could occur several time in the same attribute value.

```rust
type Position = u32;
type TokenIndex = u16;

type TermPostings = HashMap<AttributeIndex, AttributePostings>;
type AttributePostings = HashMap<EntryIndex, EntryPostings>;
type EntryPostings = HashMap<ValueIndex, HashSet<(TokenIndex, Position)>>;

/// stores processed text for full-text search
struct TextIndex {
    /// maps terms to their positions in documents
    /// structure: term -> attribute -> document -> value -> (token_index, position)
    content: HashMap<Box<str>, TermPostings>,
}
```

Now, for each attribute value, we keep a `HashSet` of the `TokenIndex` and `Position`.

Considering the wasm binary will only be able to handle 4GB of data, the maximum index length of a string would fit in a `u32` and considering the words have a minimum of 3 characters, using `u16` to index them should be enough. Therefore, `Position` and `TokenIndex` are respectively an alias to `u32` and `u16`.

Now, if we implement the `insert` method, it gives us the following.

```rust
impl TextIndex {
    fn insert(
        &mut self,
        entry_index: EntryIndex,
        attribute_index: AttributeIndex,
        value_index: ValueIndex,
        term: &str,
        token_index: TokenIndex,
        position: Position,
    ) -> bool {
        let term_postings = self.content.entry(term).or_default();
        let attribute_postings = term_postings.entry(attribute_index).or_default();
        let entry_postings = entry_postings.entry(entry_index).or_default();
        let value_postings = attr_postings.entry(value_index).or_default();
        value_postings.insert((token_index, position))
    }
}
```

The `delete` method, on the other hand, remains the same.

Index implementation summary:

- Boolean Index: Simple true/false lookups
- Integer Index: Range-based queries using BTreeMap
- Tag Index: Exact match lookups with `Box<str>` optimization
- Text Index: Full-text search with position tracking
- All indexes implement self-cleaning for empty postings


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

In this second part of our series on building a search engine, we've laid out the core architecture for how documents are stored, indexed, and sharded. We've tackled several fundamental challenges:

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

In the part 3: Search Implementation, we'll build upon these foundations to implement the actual search functionality.

Stay tuned to learn how we'll turn these carefully designed data structures into a fully functional search engine that can handle complex queries across all our supported data types!
