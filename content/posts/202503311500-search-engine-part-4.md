+++
title = "Building a search engine from scratch, in Rust: part 4"
description = "Or how we'll find something in all this."
date = 2025-04-05

[extra]
emoji = "🦀"

[taxonomies]
tags = ["rust", "search-engine", "webassembly", "encryption", "cross-platform", "tutorial", "performance"]
+++

In the [part 2](../202503191700-search-engine-part-2/) and [part 3](../202503231000-search-engine-part-3/), we implemented the way to store the indexed data and to split them in small enough pieces for the search engine to remain efficient, due to the encryption bottleneck. Now, it's time to get read access in all this and search for the entries we have in the indexes.

## Query Definition

In order to execute a search, the user first needs to define its query. Considering the indexes we have, we'll have to define, for each index, a set of filters that could be executed, but we'll define them later in that article.

Those filters will be applied to a given attribute, but not always. We can totally imagine filtering on some text, whatever the attribute, like filtering all articles having "Hello" in them, in the title or the content, with a single condition. On the other side, it's hard to imagine a use case where the user will want any article with a boolean value, whatever the attribute. That being said, this responsibility will be left to the user building the query.

And finally, those conditions can be combined into an expression, with `AND` or `OR`.

With such a structure, we could query something like `title:matches("search") AND author:"jeremie" AND (tags:"webassembly" OR tags:"rust") AND public:true`.

This gives us the following rust implementation, which is nothing more than a tree structure.

```rust
/// Representation of a filter for each index
enum Filter {
    Boolean(...),
    Integer(...),
    Tag(...),
    Text(...),
}

/// Representation of a condition on an attribute
struct Condition {
    /// attribute it refers to
    attribute: Option<Box<str>>,
    /// filter to apply
    filter: Filter,
}

/// Representation of a complex expression
enum Expression {
    Condition(Condition),
    And(Box<Expression>, Box<Expression>),
    Or(Box<Expression>, Box<Expression>),
}
```

### Filter Definition

The tag and boolean filters should be fairly simple: the entry matches the expected term or not. Giving us the following simple filters.

```rust
enum BooleanFilter {
    Equals { value: bool },
}
enum TagFilter {
    Equals { value: Box<str> },
}
```

Considering the index structure defined in [the part 2](../202503191700-search-engine-part-2/), looking for the related entries will be fairly simple for a given attribute.

```rust
impl BooleanIndex {
    fn search(&self, attribute: Option<AttributeIndex>, filter: &BooleanFilter) -> HashSet<EntryIndex> {
        let postings = match filter {
            BooleanIndex::Equals { value } => self.content.get(value),
        };
        let Some(postings) = postings else {
            // no need to go further if the term is not found
            return Default::default();
        };
        if let Some(attribute) = attribute {
            postings.get(&attribute).iter().flat_map(|attr_postings| attr_postings.keys().copied())
        } else {
            // if there is not attribute specifier, we just return all the entries
            postings.iter().flat_map(|attr_postings| attr_postings.keys().copied())
        }
    }
}
```

The `TagIndex` being exactly the same, we can skip its implementation.

#### Integer Filter

The integer filter can be a bit more complicated. We want to allow the user to be able to query a date range for example. So we'll need to implement `GreaterThan` and `LowerThan` on top of the previously defined filter.

```rust
enum IntegerFilter {
    Equals { value: u64 },
    GreaterThan { value: u64 },
    LowerThan { value: u64 },
}
```

But the `IntegerIndex` indexes the possible values with a `BTreeMap`, which allows us to query the values by range. And with all the possible attributes and entries, we can fetch the different entries.

```rust
impl IntegerIndex {
    fn filter_content(&self, filter: &IntegerFilter) -> impl Iterator<Item = &HashMap<AttributeIndex, HashMap<EntryIndex, HashSet<ValueIndex>>>> {
        match filter {
            IntegerFilter::Equals { value } => self.content.range(*value..=*value),
            IntegerFilter::GreaterThan { value } => self.content.range((*value + 1)..),
            IntegerFilter::LowerThan { value } => self.content.range(..*value),
        }
    }

    fn search(&self, attribute: Option<AttributeIndex>, filter: &IntegerFilter) -> HashSet<EntryIndex> {
        if let Some(attribute) = attribute {
            self.filter_content(filter)
                // here we filter for the given attribute
                .flat_map(|postings| postings.get(&attribute).iter())
                .flat_map(|entries| entries.keys().copied())
                .collect()
        } else {
            self.filter_content(filter)
                // here we take all the attributes
                .flat_map(|postings| postings.values())
                .flat_map(|entries| entries.keys().copied())
                .collect()
        }
    }
}
```

Once again, we end up having a faily simple implementation.

#### Text Filter

And now it's time to tackle the big piece.

## Query Execution
