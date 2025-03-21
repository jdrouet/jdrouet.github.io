+++
title = "Building a search engine from scratch, in Rust: part 2"
description = "Or how we'll structure go from a set of documents to a set of structured indexes."
date = 2025-03-19

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
enum Kind {
    Boolean,
    Integer,
    Tag,
    Text,
}

struct Schema {
    attributes: HashMap<String, Kind>,
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
enum Value {
    Boolean(bool),
    Integer(u64),
    Tag(String),
    Text(String),
}

struct Document {
    id: String,
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

## Destructuring The Documents

Now that we know how the documents will be structured, it's time to dive in the serious stuff: the indexes.

The indexes will contain the information we need to find a document based on the query parameters. We'll try to keep the content of the index fairly small when serialized to maximise the content of indexed data.

So basically, we'll do a link between the data and the document identifier: from a given term (boolean, integer, tag or text), what document contains that term, and how many times. A basic data structure would look like this.

```rust
type Index = Map<Term, Map<DocumentIdentifier, Count>>;
```

And this would be reproduced for every index, and for every term. When you think about the `DocumentIdentifier`, which would be a `String`, each term would have a cost of `size_of(DocumentIdentifier)` which is **at least** equal to the size of the string (plus some bytes depending on if we use `String` or `Box<str>`). This doesn't scale for big documents containing many terms and big identifiers, we need to use a different approach.

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

The next problem we'll have to tackle on that tructure: when deleting a document from the index, how to efficiently go from the `DocumentIdentifier` to the `EntryIndex`? For that, we need to introduce a reverse map as follow.

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

/// Collection used in memory to have performant access to the entries
struct Collection {
    entries_by_index: HashMap<EntryIndex, DocumentIdentifier>,
    entries_by_name: HashMap<DocumentIdentifier, EntryIndex>,
    sharding: BTreeMap<ShardingValue, HashSet<EntryIndex>>,
}

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

### Boolean Index

Now let's start with a simple index, the boolean index. This will only have a simple use cases: fetching the entries where an attribute is `true` or `false`.

So if we follow the structure we defined earlier, we'd end up with the following tructure.

```rust
type ValueIndex = u8;

struct BooleanIndex {
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
            res || changed
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
struct IntegerIndex {
    content: BTreeMap<u64, HashMap<AttributeIndex, HashMap<EntryIndex, HashSet<ValueIndex>>>>,
}
```

And that's it, we take the same code as above, and it will work.

### Tag Index

Let's pause a bit and clarify something: what's the difference between a tag index and a text index. Well, in this implementation, a tag index is an index where the term, the tag is a single item, that doesn't get preprocessed and that we'll search for as it is. If we insert `Alice`, then searching `alice` won't find it, it's only exact match. This is good for matching email addresses, folder names, etc.
The text index, on the opposite, the input data gets processed, but we'll talk about it right after.

Once again, we can follow a similar architecture than the two other indexes.

```rust
struct TagIndex {
    content: HashMap<Box<str>, HashMap<AttributeIndex, HashMap<EntryIndex, HashSet<ValueIndex>>>>,
}
```

You can notice here that, instead of using `String`, we are using `Box<str>`. This is because the size of a `String` is of 24 bytes, plus the characters, while `Box<str>` has a size of 16 bytes, plus the characters. This might not seem much, but every byte is worst keeping when you use a Nokia 3310.

### Text Index

As I mentioned earlier, the text index is a bit more complicated than the tag index. When the tag index is made to return entries that contain a term with exact matching, the text index will provide functionalities to search inside the provided text: the tag index is for a single value like `alice` or `personal` while you give the text index complete sentences, articles, documents, etc. And on top of that, we'll want to be able to find some elements in that text even if there are some mistages in the query: `personnal` should match documents containing `this is a personal document`.

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

And we'll require to store those positions in the index as well, considering a term could occure several time in the same attribute value.

```rust
type Position = u32;
type TokenIndex = u16;

type TermPostings = HashMap<AttributeIndex, AttributePostings>;
type AttributePostings = HashMap<EntryIndex, EntryPostings>;
type EntryPostings = HashMap<ValueIndex, HashSet<(TokenIndex, Position)>>;

struct TextIndex {
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

### Shard Definition

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

#### Transaction Mechanism

This level of abstraction for the manifest allows us to add or delete shards when needed but there's an issue: we cannot block the access to the search engine each time we insert a document. We should be able to insert a set of documents while using the index and just block its access when writing the updated manifest to disk.

Following a similar mechanism to a transactional database, inserting data will require initializing a transaction, which will create a temporary manifest file which will contain the names of all the original indexes and the names of the indexes that have been updated. Updating a collection or an index will create a new file on disk but non updated indexes will remain the same.

This would give use this code for the shard management

```rust
struct TxFile {
    // a shard can not have any boolean index but it can be created after an update
    base: Option<Filename>,
    // the filename once the transaction is committed
    next: Option<Filename>,
}
struct TxShard {
    collection: TxFile,
    indexes: HashMap<Kind, TxFile>,
}
struct TxManifest {
    shards: BTreeMap<u64, TxShard>,
}
```

This transaction manifest would be written down to the filesystem depending on the platform: in the browser, we cannot know when the page will be closed so better write it after each operation, while on mobile, the app can do a simple operation before closing. This provides a nice way of being able to recover a transaction that has not been committed.

That commit operation simply consists in, for each file of each shard, taking the `next` filename if exists or the `base` one, and write it in the `manifest.bin`. This commit operation is atomic, and then less prone to errors.
