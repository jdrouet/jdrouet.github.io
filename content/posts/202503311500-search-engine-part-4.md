+++
title = "Building a search engine from scratch, in Rust: part 4"
description = "Or how we'll find something in all this."
date = 2025-04-12

[extra]
emoji = "🦀"

[taxonomies]
tags = ["rust", "search-engine", "webassembly", "encryption", "cross-platform", "tutorial", "performance"]
+++

In the [part 2](../202503191700-search-engine-part-2/) and [part 3](../202503231000-search-engine-part-3/), we implemented the way to store the indexed data and to split them in small enough pieces for the search engine to remain efficient, due to the encryption bottleneck. Now, it's time to implement read access and search functionality for our indexed entries.

## Query Definition

In order to execute a search, the user first needs to define its query. Considering the indexes we have, we'll have to define, for each index, a set of filters that could be executed, but we'll define them later in that article.

These filters can be applied to specific attributes, though this isn't always necessary. For example, we might want to search for text across all attributes, like filtering all articles having "Hello" in them, in the title or the content, with a single condition. On the other side, it's hard to imagine a use case where the user will want any article with a boolean value, whatever the attribute. That being said, this responsibility will be left to the user building the query.

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

**Key Points:**
- Flexible query language supporting complex boolean expressions
- Typed filters for different data types
- Optional attribute targeting for conditions
- Composable expressions using AND/OR operators

### Filter Definition

The tag and boolean filters are straightforward: an entry either matches the expected term or it doesn't. This leads to the following simple filter implementations:

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

**Filter Implementation Achievements:**
- Boolean filters for simple true/false matching
- Integer filters supporting range queries
- Tag filters for exact string matching
- Memory-efficient implementation using numeric indexes

#### Text Filter

Now let's tackle the most complex piece. Searching through text is only easy when looking for exact values. We need something more clever here.

We want to support fuzzy matching, where searching for "Moovies" would match "movie" and this is done by implementing some [fuzzy search](https://en.wikipedia.org/wiki/Approximate_string_matching).

We also want something that allows to find words starting with a value (searching `title:starts_with("Artic")` should catch "Article"). This is a subset of the [wildcard search](https://en.wikipedia.org/wiki/Wildcard_character).

This gives us the following filter

```rust
enum TextFilter {
    // searching for the exact value
    Equals { value: Box<str> },
    // matching the prefix
    StartsWith { prefix: Box<str> },
    // fuzzy search
    Matches { value: Box<str> },
}
```

The `Equals` implementation is similar to the previous indexes so we'll skip it.

##### Prefix Search

In order to implement the `StartsWith` filter without going through the entire content of the index, we need to precompute a structure. This structure will be a simple tree where each node is a character.

```rust
#[derive(Debug, Default)]
pub(super) struct TrieNode {
    children: BTreeMap<char, TrieNode>,
    term: Option<Box<str>>,
}
```

That way, finding the words matching a prefix will just need to go through each letters of that prefix in the tree, and all children are the potential words.

Finding the final node for the `StartsWith` filter is done as following

```rust
impl TrieNode {
    fn find_starts_with(&self, mut value: Chars<'_>) -> Option<&TrieNode> {
        let character = value.next()?;
        self.children
            .get(&character)
            .and_then(|child| child.find_starts_with(value))
    }
}
```

And once we get the node, we need to iterate through the entire tree structure of the children to collect the matching words. This can be done by implementing an iterator.

```rust
#[derive(Debug, Default)]
struct TrieNodeTermIterator<'n> {
    queue: VecDeque<&'n TrieNode>,
}

impl<'t> TrieNodeTermIterator<'t> {
    fn new(node: &'t TrieNode) -> Self {
        Self {
            queue: VecDeque::from_iter([node]),
        }
    }
}

impl Iterator for TrieNodeTermIterator<'_> {
    type Item = Box<str>;

    fn next(&mut self) -> Option<Self::Item> {
        let next = self.queue.pop_front()?;
        self.queue.extend(next.children.values());
        if let Some(value) = next.term {
            Some(value.clone())
        } else {
            self.next()
        }
    }
}

impl TrieNode {
    fn search(&self, prefix: &str) -> impl Iterator<Item = Box<str>> {
        if let Some(found) = self.find_starts_with(prefix.chars()) {
            TrieNodeTermIterator::new(found)
        } else {
            TrieNodeTermIterator::default()
        }
    }
}
```

Once we have those words, we can simply deduce matching the entries.

##### Fuzzy Search

Fuzzy searching is done by extracting, for each words, all the possible subsets of `N` letters and creating an index from that. Then, when queried, we take the value to match against, extract all the possible subsets of `N` words, and the words *matching the most*, should be kept.

In our case, we'll use `N=3`, because it's what's used in Postgres to improve the fuzzy search performance and it's suggested in [some research papers](https://onlinelibrary.wiley.com/doi/abs/10.1002/spe.4380180407).

Building that trigram index will come at a cost: we'll have to process the entire index. But this should be relatively fast considering this is a simple process. Building the trigrams can be done using the [`substr-iterator` crate](https://crates.io/crates/substr-iterator).

```rust
#[derive(Default)]
struct Trigrams<'a>(HashMap<[char; 3], HashSet<&'a str>>);

impl<'a> Trigrams<'a> {
    fn insert(&mut self, value: &str) {
        TrigramIter::from(value).for_each(|trigram| {
            let set: &mut HashSet<usize> = self.0.entry(trigram).or_default();
            set.insert(value);
        });
    }
}

impl TextIndex {
    fn trigrams(&self) -> Trigrams<'_> {
        let mut res = Trigrams::default();
        self.content.keys().for_each(|word| {
            res.insert(word);
        });
        res
    }
}
```

Once this is done, we can search all the possible terms with the following function

```rust
impl<'a> Trigrams<'a> {
    fn search(&self, term: &str) -> impl Iterator<Item = Box<str>> {
        TrigramIter::from(term)
            .filter_map(|tri| self.0.get(&tri))
            .flatten()
            // this is provided by the itertools crate
            .unique()
            .map(Box::from)
    }
}
```

##### Scoring The Matching entries

Now that we have all the possible terms matching the filter, we need to provide some kind of score. But first, should we accept all the matching terms? Should querying `macro` match `pneumonoultramicroscopicsilicovolcanoconiosis`?

Doing such a filtering can be done computing the [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) of the queried word with the word we found and only keep the word having a distance smaller than half of the length of the queried word. No need to reinvent the wheel here, the [`distance` crate](https://crates.io/crates/distance) implements it.

Next, we'll compute scores for each term using the [Okapi BM25 algorithm](https://en.wikipedia.org/wiki/Okapi_BM25). For this, we'll need several metrics including the collection size (number of entries in the shard), the size of the entry (number of tokens for the requested attribute) and the average length of the attribute accross all the collection.

In order to avoid recomputing this for each search, we'll update the `TextIndex` to persist these values each time we update the text index.

```rust
pub struct TextIndex {
    /// content of the text index
    content: ...,
    /// number of tokens per attribute
    attributes: HashMap<AttributeIndex, u32>,
    /// number of tokens per entry per attribute
    entries: HashMap<EntryIndex, HashMap<AttributeIndex, u32>>,
}
```

With these values persisted, querying the search engine can be done this way.

```rust
// FYI this piece of code is not supposed to compile, nor to be efficient

impl TextIndex {
    /// computes the score for each entry based on the found terms
    fn compute_scores(
        &self,
        attribute: Option<AttributeIndex>,
        matchings: HashMap<EntryIndex, HashMap<&str, usize>>,
    ) -> HashMap<EntryIndex, f64> {
        let collection_size = self.collection_size();
        let avg_length = self.collection_avg_length(attribute);
        matching
            .into_iter()
            .map(|(entry_index, terms)| {
                let entry_length = self.entry_length(entry_index, attribute);
                let mut score = 0.0;
                for (term, freq_in_entry) in terms {
                    let entries_with_term = self.count_entries_with_term(term, attribute);
                    // have a look at the wikipedia page for the formula, it will be simpler to read
                    let idf = compute_idf(collection_size, entries_with_term);
                    score += compute_bm25_score(idf, freq_in_entry, entry_length, avg_length);
                }
                (entry_index, score)
            })
            .collec()
    }

    /// reduce the list of terms and return the entries matching the terms
    fn reduce_matches(
        &self,
        attribute: Option<AttributeIndex>,
        terms: impl Iterator<Item = &str>,
    ) -> HashMap<EntryIndex, HashMap<&str, usize>> {
        let mut res = HashMap::default();
        for term in terms {
            for entry_index in self.find_entries_for_term(term) {
                let entry = res.entry(entry_index).or_default();
                // count the occurences of that term in the entry
                entry.entry(term).and_modify(|v| {
                    *v += 1;
                }).or_insert(1);
            }
        }
        res
    }

    /// search through the index
    fn search(
        &self,
        attribute: Option<AttributeIndex>,
        filter: &TextFilter,
    ) -> HashMap<EntryIndex, f64> {
        let matching_terms = match filter {
            TextFilter::StartsWith { prefix } => self.trienodes().search(prefix),
            TextFilter::Matches { value } => self.trigrams().search(value),
            TextFilter::Equals { value } => self.inner.get_term(value).into_iter(),
        };
        let matching_entries = self.reduce_matches(attribute, matching_terms)
        self.compute_scores(attribute, matchings)
    }
}
```

With that code, we end up with a map of all the matching entries with a score.

**Text Search Achievements:**
- Prefix search using trie data structure
- Fuzzy search using trigram indexing
- Levenshtein distance filtering for result relevance
- BM25 scoring for accurate result ranking
- Efficient caching of collection statistics

## Query Execution

Now that we have a way to build the expression of the query, we can query individualy each index, it's time to plug everything together in order to execute a complete search.

As a reminder, considering the engine is organised in shards, the search will simply being executing the search on every shard. But considering the search might take some time, the search should return the results as each shard is processed.

```rust
impl SearchEngine {
    async fn search<Cb>(&self, expression: &Expression, callback: Cb) -> std::io::Result<usize>
    where
        Cb: Fn(HashMap<EntryIndex, f64>)
    {
        let mut found = 0;
        for (_shard_key, shard) in self.shards {
            let result = shard.search(expression).await?;
            found += result.len();
            callback(result);
        }
        Ok(found)
    }
}
```

After defining the high level, we can go one level down, and look at how it works at the shard level.

### Caching File Content

A shard is only defined by the names of the files it's composed of.

```rust
struct ShardManifest {
    /// the collection is mandatory, it's the index of all the entries
    /// if it's none, there's no entry, so there's no shard
    collection: Box<str>,
    /// then every index is optional (except the integer index, but we'll keep the same idea)
    boolean: Option<Box<str>>,
    integer: Option<Box<str>>,
    tag: Option<Box<str>>,
    text: Option<Box<str>>,
}
```

Which means that, for each condition in the search expression, we'll need to load the collection to find the `AttributeIndex` for the given attribute name, and then load the corresponding index and execute the query. We could load all of the indexes when starting a search in the shard but we might not need all of them and considering the decryption cost, we should avoid that.

Let's add an abstraction layer for the shard.

```rust
struct Shard {
    /// this is loaded anyway
    collection: Collection,
    /// then we create a cache
    boolean: Option<CachedEncryptedFile<BooleanIndex>>,
    integer: Option<CachedEncryptedFile<IntegerIndex>>,
    tag: Option<CachedEncryptedFile<TagIndex>>,
    text: Option<CachedEncryptedFile<TextIndex>>,
}

struct CachedEncryptedFile<T> {
    file: EncryptedFile,
    cache: async_lock::OnceCell<T>,
}

impl<T: serde::de::DeserializedOwned> CachedEncryptedFile<T> {
    async fn get(&self) -> std::io::Result<&T> {
        self.cache
            // here we only deserialize the file when we need to access it
            // and it remains cached
            .get_or_try_init(|| async { self.file.deserialize::<T>().await })
            .await
    }
}
```

With that level of abstraction, we're sure the files are only loaded once in memory.

### Calling Async Code In Every Condition

Now it's time to face the next problem: executing the query. From a first point of view, it should be fairly simple, the expression is a dumb tree where the leaves are conditions. So, to run this, we could just recursively call the indexes in the conditions and reduce at the expression level.

```rust
impl Shard {
    async fn search(&self, expression: &Expression) -> HashMap<EntryIndex, f64> {
        expression.execute(self).await
    }
}

impl Expression {
    async fn execute(&self, shard: &Shard) -> HashMap<EntryIndex, f64> {
        match self {
            Expression::Condition(condition) => condition.execute(shard).await,
            Expression::And(left, right) => {
                let left_res = left.execute(shard).await?;
                let right_res = right.execute(shard).await?;
                reduce_and(left_res, right_res)
            },
            Expression::Or(left, right) => {
                let left_res = left.execute(shard).await?;
                let right_res = right.execute(shard).await?;
                reduce_or(left_res, right_res)
            }
        }
    }
}
```

However, this approach won't work well because recursive async calls are problematic. So we should convert this to a sequential process.

To make this sequential, the trick is to create an iterator in the `Expression` tree. That way, the following expression tree should be serialized as follow

```
             author:"alice"
           /
        OR
      /    \
     /       author:"bob"
    /
AND
    \
     \
      \
        title:"Hello"
e
[cond(author:"alice"), cond(author:"bob"), OR, cond(title:"Hello"), AND]
```

So that we can use a [Reverse Polish Notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation) to compute the scores.

```rust
impl Expression {
    async fn execute(&self, shard: &Shard) -> HashMap<EntryIndex, f64> {
        let mut stack = Vec::new();

        for item in self.iter() {
            match item {
                Item::Condition(cond) => {
                    let result = condition.execute(shard).await?;
                    stack.push(result);
                }
                // kind is AND/OR
                Item::Expression(kind) => {
                    let right = stack.pop().unwrap();
                    let left = stack.pop().unwrap();
                    stack.push(aggregate(kind, left, right));
                }
            }
        }

        Ok(stack.pop().unwrap_or_default())
    }
}
```

Which fixes the problem of not being able to execute async calls.

> We could have used a crate like [`async-recursion`](https://crates.io/crates/async-recursion) to handle this but I find this solution more elegant.

### Joining The results

We have some inconsistencies in our search implementations. The `TextIndex` returns a `HashMap<EntryIndex, f64>` while the others return a `HashSet<EntryIndex>`. This is due to the lack of score in the other indexes. In order to be able to search with only those indexes (boolean, integer or tag), we need to implement that scoring mechanism as well.

> The logic is fairly straightforward and similar to what was done in the text index, so I'll leave it up to you here.

Now we need to implement the `aggregate` function we used in the piece of code above. We will try to normalise the scores in `0..1` to make them equally important and to prevent a score inflation.

```rust
fn normalise(mut scores: HashMap<EntryIndex, f64>) -> HashMap<EntryIndex, f64> {
    let max = scores.values().copied().fold(0.0, f64::max);
    scores.values_mut().for_each(|value| {
        *value /= max;
    });
    scores
}

fn aggregate(kind: Kind, mut left: HashMap<EntryIndex, f64>, mut right: HashMap<EntryIndex, f64>) -> HashMap<EntryIndex, f64> {
    let left = normalise(left);
    let right = normalise(right);
    match kind {
        // we want the intersection
        Kind::And => left.into_iter()
            .filter_map(|(entry_index, score_left)| {
                right
                    .remove(&entry_index)
                    // we use a multiplication to join scores
                    .map(|score_right| (entry_index, score_left * score_right))
            })
            .collect(),
        // we want the union
        Kind::Or => {
            right.drain().for_each(|(entry_index, right_score)| {
                left.entry(entry_index)
                    .and_modify(|left_score| {
                        // we use the addition to join scores
                        *left_score += right_score;
                    })
                    .or_insert(right_score);
            });
            left
        },
    }
}
```

With this implementation, each index and each field is equally important. To make the result more relevant, we could introduce a system of weights (or boost) in the equation.

With our final query scores calculated, we can simply match the `EntryIndex` to the original identifier from the collection and return the content in the callback.

**Query Engine Achievements:**
- Lazy loading of indexes through caching
- Efficient async execution using sequential processing
- Memory-efficient query evaluation using RPN
- Normalized scoring across different index types
- Extensible aggregation system for combining results

## Conclusion

In this fourth part of our search engine series, we've implemented a flexible and efficient search system that works across our different index types. We've tackled several key challenges:

**Key Achievements:**
- Built a flexible query language supporting complex boolean expressions
- Implemented specialized filters for different data types
- Created an efficient fuzzy search system using trigrams
- Added prefix search capabilities using tries
- Developed a robust scoring system based on BM25

Our implementation carefully balances functionality with performance. By using cached file access and converting recursive operations to sequential ones, we've maintained good performance even with encryption overhead. The scoring system ensures results are relevant while remaining computationally efficient.

The search engine now supports various query types:
- Simple boolean matching (`public:true`)
- Range queries for numbers (`created_at > 123456789`)
- Exact matching for tags (`author:"alice"`)
- Fuzzy text search with prefix matching (`title:starts_with("prog")`)
- Complex expressions combining multiple conditions
