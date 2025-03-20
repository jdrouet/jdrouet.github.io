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

This new update introduce multiple changes.

First, notice the use of [`Arc<str>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) instead of `String`. We need to have multiple reference to the same string in memory. If we use `String`, we'll pay several time the cost of that string length. When using `Arc<str>`, we only pay the price of the string length once and just the price of the pointer each time we clone it.

> One could ask, considering the advantage of `Arc<str>`, why not writing that directly to disk or use it in the other indexes. Well, it doesn't work when serialized. [`Arc<str>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) contains a pointer in memory of where the string is. When serialize/deserialize, this memory address changes, so the serializer just replaces the pointer with its actual value, which means duplication of data.
