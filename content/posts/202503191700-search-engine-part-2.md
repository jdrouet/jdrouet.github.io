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
