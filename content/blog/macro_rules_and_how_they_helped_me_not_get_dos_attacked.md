+++
draft = false
title = "macro_rules! and how they helped us not get DOS attacked"
date = "2026-04-18"
[taxonomies]
authors = ["Charles Johnson"]
tags = ["rust", "macros", "GraphQL"]
+++

# Introduction

Hi, I'm Charles and I'd like to share some of the problems I had with a GraphQL server written in Rust and how writing a `macro_rules!` macro reduced the boilerplate of the solution to the problem

# Problem

## Cyclic type references in a GraphQL schema

One of the downsides of GraphQL is that it allows for cyclic type references that allow for an arbitrarily deep query to be served that can lead to excessive amounts of resources to be expended for a single request. This is something that I avoided at a previous company when building a GraphQL server from scratch but after joining [Clear](https://getclearapp.com), I noticed there were several cyclic type references in the schema. 

An example of cyclic type references in a [GraphQL schema](https://graphql.org/learn/schema/#type-language):

```graphql
type Query {
    foo: Foo
}

type Foo {
    bar: Bar
    result: Int
}

type Bar {
   foo: Foo
   result: Int
}
```

In this case, the following arbitrarily deep [GraphQL query](https://spec.graphql.org/draft/#root-operation-type) could be made:
#### Arbitrarily deep query
```graphql
query {
    foo {
        bar {
            foo {
                bar {
                    result
                }
                result
            }
            result
        }
        result
    }
    result
}
```
The chain of `foo { bar { foo { bar {...` can go on indefinitely and standard GraphQL servers will try to process the whole query.

## Consequences

Eventually, a large enough query will overwhelm the GraphQL server, slowing down other requests and even crashing the server or other downstream services. These queries don't even have to be that deep when one of the fields within the cycle is a [List](https://graphql.org/learn/schema/#list) type. The presence of lists that return multiple items causes the size of the response to grow exponentially with query depth, providing an asymmetric attack vector.

Failure to solve the [N+1 problem](https://graphql.org/learn/performance/#the-n1-problem) e.g. by implementing the [DataLoader pattern](https://github.com/graphql-rust/juniper/blob/master/book/src/advanced/dataloader.md) for fields within the cycle can also lead to excessive number of database queries that can quickly consume the number of available database connections, causing request time outs or even crashing database servers that run out of memory.

So there are a lot of reasons to avoid shipping a GraphQL server with these cyclic type references to production and having clients depend on it. Alas, that ship had already sailed and I needed to find a way to prevent these cyclic type references from being exploited.

# Solutions

I may be able been update the server quite simply without breaking clients depending on the possible queries the Clear app could make that relied on the cyclic type references. 

## Simple nested query

For example, if a client was only using the cyclic type reference for the single query:
```graphql
query {
    foo {
        bar {
            foo {
                result
            }
        }
    }
}
```
you could change the definition of `Bar` to the following
```graphql
type Bar {
    foo: FooWithJustResult
    result: Int
}

type FooWithJustResult {
    result: Int
}
```

This would break the cyclic type reference without breaking the clients.

However, Clear has queries factored into [fragments](https://spec.graphql.org/draft/#sec-Language.Fragments) which unfortunately ties the client to a specific GraphQL type even if another type would be compatible for the equivalent in-line query. 

## Fragmented query

For example, if a client factors the in-line query above as below:

```graphql
query {
    foo {
        bar {
            foo {
                ...FooFragment
            }
        }
    }
}

fragment FooFragment on Foo {
    result
}
```
then updating the type of `Bar` 's `foo` field to `FooWithJustResult` will break the query which expects a `Foo` object at runtime. In this example, you could redefine the schema to be

```graphql
type Query {
    foo: FullFoo
}

type FullFoo {
    bar: Bar
    result: Int
}

type Bar {
   foo: Foo
   result: Int
}

type Foo {
    result: Int
}
```
but this wouldn't work if you have other fragments `on Foo` that require the `bar` field.

## More complex sets of queries

In practice, it is very difficult to remove cyclic type references once you have clients relying on them and even if you go through the significant amount of work to rewrite the client to avoid the cyclic type references, which would likely require implementing alternative GraphQL fields on the server, you'll have to allow time for existing clients to update before you can finally remove the cyclic type references.

Instead of removing the cyclic type references from the schema, there are techniques to limit query complexity with server runtime checks. Careful analysis needs to be done before applying these techniques as the contract between the server and client can't be enforced with the standard schema generation (e.g. [get-graphql-schema](https://github.com/prisma-labs/get-graphql-schema)) and validation (e.g. [relay-compiler](https://relay.dev/docs/next/guides/compiler/) with the [TypeScript compiler](https://github.com/microsoft/TypeScript)) tools.

The GraphQL website states the following techniques to avoid issues with cyclic type references

* [Depth limiting](https://graphql.org/learn/security/#depth-limiting): Rejecting queries that exceed a certain depth
* [Query complexity analysis](https://graphql.org/learn/security/#query-complexity-analysis): Calculating the cost of a query before processing it and rejecting queries that exceed the maximum cost

Applying these techniques to the whole GraphQL schema would limit the damage a single malicious query could make but, for existing applications, requires thorough analysis of all possible queries to make sure clients won't break. Even after analysing clients to find the maximum depth of query they could make, this can be too weak a constraint and still allow maliciously crafted requests to take down the server. Query complexity analysis is more a general solution but has so many variables to tune and if no compatible, out of the box implementation exists for your server, implementing this is time consuming.

The solution we came up with at Clear was something in between these two techniques: limiting the number of specific GraphQL fields within a nested chain of a query. This made it a lot easier to work out whether the Clear mobile app would break as we incrementally deprecated each GraphQL field that formed a cyclic type reference.

### Implementation

At Clear, we use the [juniper](https://github.com/graphql-rust/juniper) crate to build a GraphQL server using it's derive and procedural macros on Rust types for each GraphQL type in our schema. We'd implement the example schema as follows 

```rust
use juniper::graphql_object;

/// This would be used to store database connection pools and data loaders
struct Context;

impl Context {
    async fn get_bar(&self) -> Option<DbBar> {
        unimplemented!()
    }
    async fn get_foo(&self) -> Option<DbFoo> {
        unimplemented!()
    }
    async fn get_bar_result(&self) -> Option<i32> {
        unimplemented!()
    }
    async fn get_bar(&self) -> Option<i32> {
        unimplemented!()
    }
}

struct DbFoo;

struct DbBar;

/// Passed to `juniper::RootNode::new` to register this as the root query type
struct Query;

#[graphql_object(context = Context)]
impl Query {
    async fn foo(&self, context: &Context) -> Option<Foo> {
        context.get_foo().await.map(|db_item| Foo {db_item})
    }
}

struct Foo {
    db_item: DbFoo
}

#[graphql_object(context = Context)]
impl Foo {
    async fn bar(&self, context: &Context) -> Option<Bar> {
        context.get_bar().await.map(|db_item| Bar {db_item})
    }
    async fn result(&self, context: &Context) -> Option<i32> {
        context.get_foo_result().await
    }
}

struct Bar {
    db_item: DbBar
}

#[graphql_object(context = Context)]
impl Bar {
    async fn foo(&self, context: &Context) -> Option<Foo> {
        context.get_foo().await.map(|db_item| Foo {db_item})
    }
    async fn result(&self, context: &Context) -> Option<i32> {
        context.get_bar_result().await
    }
}
```

In order to limit the depth, we can keep track of how many `foo` and `bar` parent fields have already been resolved to get to the given GraphQL object and return an error if this is 3 or more before resolving a `Foo` object or is 2 or more before resolving a `Bar` object which is the most that the example client requires.

```rust
use juniper::{FieldResult, FieldError};

#[graphql_object(context = Context)]
impl Query {
    async fn foo(&self, context: &Context) -> Option<Foo> {
        let db_item = context.get_foo().await?;
        Some(Foo {depth: 1, db_item})
    }
}

struct Foo {
    depth: usize,
    db_item: DbFoo,
}

#[graphql_object(context = Context)]
impl Foo {
    async fn bar(&self, context: &Context) -> FieldResult<Option<Bar>> {
        let Some(db_item) = context.get_bar().await else {
            return Ok(None)
        };
        if previous_depth >= 2 {
            log::warn!("Depth limit of 2 exceeded");
            return Err(FieldError::from("Depth limit exceeded"));
        }
        Some(Bar {depth: self.depth + 1, db_item})
    }
    async fn result(&self, context: &Context) -> i32 {
        context.get_foo_result().await
    }
}

struct Bar {
    depth: usize,
    db_item: DbBar,
}

#[graphql_object(context = Context)]
impl Bar {
    async fn foo(&self, context: &Context) -> FieldResult<Option<Foo>> {
        let Some(db_item) = context.get_foo().await else {
            return Ok(None)
        };
        if previous_depth >= 3 {
            log::warn!("Depth limit of 3 exceeded");
            return Err(FieldError::from("Depth limit exceeded"));
        }
        Some(Foo {depth: self.depth + 1, db_item})
    }
    async fn result(&self, context: &Context) -> i32 {
        context.get_bar_result().await
    }
}
```

The same kind of code needed to be applied in many places in Clear's back end codebase so, to reduce the boilerplate, I wrote a macro, `impl_set_depth`. This allows blanket implementation of the `DepthLimited` trait for the GraphQL type by implementing a `SetDepth` trait which is a super trait. Here's what it looks like:

```rust
macro_rules! impl_set_depth {
    ($graphql_item: path, $db_item:path, $depth_limit:expr) => {
        impl SetDepth for $graphql_item {
            type DbItem = $db_item;
            const DEPTH_LIMIT = $depth_limit;

            fn set_depth(&mut self, depth: usize) {
                self.depth = depth;
            }
        }
    }
}

trait SetDepth {
    type DbItem;
    const DEPTH_LIMIT: usize;

    fn set_depth(&mut self, depth: usize);
}

trait DepthLimited: SetDepth {
    fn increment_depth(db_item: SElf::DbItem, previous_depth: usize) -> Result<Self>
    where
        Self: Sized,
    {
        if previous_depth >= Self::DEPTH_LIMIT {
            log::warn!("Depth limit of {} exceeded", Self::DEPTH_LIMIT);
            bail!("Depth limit exceeded");
        }

    fn from_db_item(db_item: Self::DbItem, depth: usize) -> Self
    where
        Self: Sized
    }
}

impl<T> DepthLimited for T
where
    T: SetDepth,
    T::DbItem: Into<T>
{
    fn from_db_item(db_item: Self:DbItem, depth: usize) -> Self
    where
        Self: Sized,
    {
        let mut item = db_item.into();
        item.set_depth(depth);
        Ok(item)
    }
}
```

Here's how it would have been applied to the example:

```rust

#[graphql_object(context = Context)]
impl Query {
    async fn foo(&self, context: &Context) -> FieldResult<Option<Foo>> {
        let Some(db_item) = context.get_foo().await else {
            return Ok(None);
        };
        Some(Foo::increment_depth(db_item, 0)?)
    }
}

impl From<DbFoo> for Foo {
    fn from(db_item: DbFoo) -> Self {
        Self {
            db_item,
            depth: 0
        }
    }
}

impl_set_depth!(Foo, DbFoo, 3);

#[graphql_object(context = Context)]
impl Foo {
    async fn bar(&self, context: &Context) -> FieldResult<Option<Bar>> {
        let db_item = context.get_bar().await?;
        Ok(Some(Bar::increment_depth(db_item, self.depth)?))
    }
    async fn result(&self, context: &Context) -> i32 {
        context.get_foo_result().await
    }
}

impl From<DbBar> for Bar {
    fn from(db_item: DbBar) -> Self {
        Self {
            db_item,
            depth: 0
        }
    }
}

impl_set_depth!(Bar, DbBar, 2);

#[graphql_object(context = Context)]
impl Bar {
    async fn foo(&self, context: &Context) -> FieldResult<Option<Foo>> {
        let db_item = context.get_foo().await?;
        Ok(Some(Foo::increment_depth(db_item, self.depth)?))

    }
    async fn result(&self, context: &Context) -> i32 {
        context.get_bar_result().await
    }
}
```

This will allow the [fragmented query](#fragmented-query) to be served whilst preventing deeper queries [like above](#arbitrarily-deep-query).

# Result

Using this technique alone for Clear's public GraphQL server prevented arbitrarily deep queries to be served without breaking the mobile app. We identified all the GraphQL fields responsible for the cyclic type references and looked at how deeply nested they are within queries that the mobile app could make. From this analysis we were able to deduce the approriate depth limits for each GraphQL object that was part of a cyclic type reference. We also made a [tool](https://gitlab.com/haut-technologies/graphql-analyzer) to help with this analysis.

Another tool was also developed to parse the GraphQL schema and automatically construct the most expensive query possible and then load test the GraphQL server running in a review environment as part of the merge request pipeline. This allows us to know what request rate limit to set for the GraphQL server to prevent a DOS attack as well as measure the impact of performance improvements such as implementing data loaders.
