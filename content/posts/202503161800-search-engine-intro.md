+++
title = "Building a search engine from scratch, in Rust: introduction"
description = "Or how to build a secure, blazingly fast search engine that works everywhere, from your browser to your phone, without compromising on features or performance."
date = 2025-03-16

[extra]
emoji = "🦀"

[taxonomies]
tags = ["rust", "search-engine", "webassembly", "encryption", "cross-platform", "tutorial", "performance"]
+++

Have you ever wondered how search engines work under the hood? I'm not talking about vector search or anything fancy, but just the good old search engines that could really find what we would look for.
Even though we can already find some articles about how search engine work from a high level, it could be interesting to implement our own and add some constraints to make it fun (and maybe simpler).

## The Challenge

Most search engine implementations you'll find online target the server-side. They're great for large-scale applications but don't address the unique challenges of building a search engine that runs directly in your browser, your mobile app or in a desktop app. That's exactly what we're going to tackle: creating a client-side search engine that works seamlessly across platforms. And to spice things up, we'll add an encryption layer so that the indexed data stays safe and encrypted.

Running a search engine in a browser brings its own set of interesting constraints and challenges. We'll need to work within the limitations of WebAssembly, handle browser-specific storage mechanisms, and ensure our engine performs well even with limited resources.
But WebAssembly is not the only limiting factor. Running a computation intensive process in a mobile app could be challenging: what happens if the mobile app goes to the background while indexing?
These constraints make the project even more interesting from a learning perspective.

**Main Challenges:**
- Browser limitations (storage, threading)
- Mobile limitations (limited CPU time)
- Cross-platform compatibility
- Resource constraints

## Why Build From Scratch?

There are several existing solutions we could use. [Meilisearch](https://www.meilisearch.com/) and [Elasticsearch](https://www.elastic.co/elasticsearch) are powerful search engines, but they're designed to run on servers and would not be suitable for client-side search. [Tantivy](https://github.com/quickwit-oss/tantivy) is an excellent Rust-based search engine that we can draw inspiration from, but its heavy use of threading makes it unsuitable for browser environments. [SQLite](https://www.sqlite.org/) with full-text search capabilities could work with [SQLCipher](https://github.com/sqlcipher/sqlcipher), but the binary would be big, the persistence layer in the browser would be complicated to implement and we wouldn't learn much how to implement a search engine.

Building our own search engine gives us complete control over the architecture and implementation. We can make it exactly as lightweight or feature-rich as we need, and most importantly, we can learn about every component that goes into making a search engine work.

**Existing Solutions Limitations:**
- [Meilisearch](https://www.meilisearch.com/)/[Elasticsearch](https://www.elastic.co/elasticsearch): Server-focused
- [Tantivy](https://github.com/quickwit-oss/tantivy): Not browser-compatible
- [SQLite](https://www.sqlite.org/) and [SQLCipher](https://github.com/sqlcipher/sqlcipher): Limited search capabilities

## Project Goals

Our search engine needs to work seamlessly across different platforms. Whether it's running in a web browser through WebAssembly, in a mobile application, or on the desktop, the core functionality should remain consistent. This cross-platform requirement introduces interesting challenges in handling platform-specific quirks while maintaining a unified codebase.

Resource efficiency is crucial, especially when running in a browser environment. We need to keep our memory footprint small, ensure our binary size doesn't bloat the application, and maintain snappy search performance. These constraints will push us to make thoughtful design decisions and optimizations.

We'll support various data types in our search engine. From basic text content to tags, numbers, and boolean values, each type brings its own indexing and searching challenges. We'll implement flexible querying capabilities that allow users to search across these different types effectively.

As an interesting twist, we'll add encryption capabilities to our engine. While not strictly necessary for a basic search engine, implementing encryption adds an extra layer of complexity and learning opportunity. It's a great way to understand how encryption can be integrated into a search system while keeping performance in mind.

To finish, in order to not compute our indexes on every platforms we'll run the search engine, we'll make the encrypted indexes sharable between devices.

**Core Features:**
- Cross-platform compatibility
- Multiple data type support
- Efficient resource usage
- Encryption
- Index synchronization between devices

## High-Level Architecture Overview

Here's a glimpse of what we'll be building:

```rust
/// The main search engine interface
pub trait SearchEngine {
    /// Index a new document
    async fn index(&mut self, document: Document) -> Result<()>;
    /// Search through indexed documents
    async fn search(&self, query: &str) -> Result<SearchResults>;
    /// Delete a document from the index
    async fn delete(&mut self, id: &str) -> Result<()>;
}
```

Our engine will be built in layers, starting with basic storage operations with the encryption layer, then adding indexing capabilities, the search functionality, and, finally, how to synchronize those indexes between devices. This modular approach will help us understand each component's role and how they work together to create a functioning search engine.

**Architecture Layers:**
- Storage: File operations and data persistence, with encryption
- Index: Document management and retrieval
- Search: Query processing and results
- Synchronization: Sharing the computed indexes across devices

## The Journey Ahead

Throughout this series, we'll explore every aspect of building a search engine. We'll start with the fundamentals of storage and work our way up to complex search algorithms. Along the way, we'll tackle cross-platform challenges, optimize performance, and add encryption capabilities.

Next up, we'll dive into implementing the storage layer, where we'll face our first cross-platform challenges and lay the groundwork for our search engine. We'll explore different storage strategies for browsers and native applications, and begin adding our cryptographic spice to the mix.

**Series Outline:**
- Part 0: Introduction (this article)
- [Part 1: Storage Layer Implementation](../202503170800-search-engine-part-1/)
- [Part 2: Document Schema and Indexes](../202503191700-search-engine-part-2/)
- [Part 3: Sharding and Transactions](../202503231000-search-engine-part-3/)
- Part 4: Search Implementation
- Part 5: Performance Optimization
- Part 6: Index Synchronization

Stay tuned for the next article in this series, where we'll get our hands dirty with actual implementation. Whether you're a seasoned Rust developer or just getting started, this journey promises to be both educational and entertaining.
