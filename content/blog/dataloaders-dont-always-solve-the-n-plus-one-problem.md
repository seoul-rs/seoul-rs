+++
draft = false
title = "Dataloaders don't always solve the N+1 problem"
date = "2026-07-26"
[taxonomies]
authors = ["Charles Johnson"]
tags = ["rust", "dataloader", "GraphQL"]
+++

## Introduction

Hi, I'm Charles, the CTO of [Clear](https://getclearapp.com).
I wanted to share my experience of using dataloaders in Rust
within the context of GraphQL servers at different companies.
I found that they didn't always give the best performance
and came up with an alternative pattern to overcome this.

## Motivation

The [N + 1 problem](https://graphql.org/learn/performance/#the-n1-problem) is well known problem in GraphQL.

Say you have the schema,

```graphql
type Query {
    foos: [Foo]
}

type Foo {
    bar: Int
}
```

and you want to make the following query.

```graphql
query FooBarQuery {
    foos {
        bar
    }
}
```

A naive implementation of this in Rust using the [juniper crate](https://crates.io/crates/juniper) could look like

```rust
use juniper::graphql_object;
use sqlx::{query_as, query_scalar, PgPool};

struct Context {
    pool: PgPool
}


impl Context {
    async fn get_foos(&self) -> sqlx::Result<Vec<Foo>> {
        query_as!(Foo, "SELECT id FROM foo").fetch_all(&self.pool).await
    }
    async fn get_bar(&self, foo_id: i32) -> sqlx::Result<i32> {
        query_scalar!("SELECT id FROM bar b WHERE b.foo = $1", foo_id).fetch_one(&self.pool).await
    }
}

/// Passed to `juniper::RootNode::new` to register this as the root query type
struct Query;

#[graphql_object(context = Context)]
impl Query {
    async fn foos(&self, context: &Context) -> FieldResult<Foo> {
        context.get_foos()
            .await
            .map(|db_item| Foo {db_item})
            .map_err(|e| {
                error!("Failed to get foos: {e:?}");
                FieldError::from("Internal Server Error")
            })
    }
}

struct Foo {
    id: i32
}

#[graphql_object(context = Context)]
impl Foo {
    async fn bar(&self, context: &Context) -> FieldResult<i32> {
        context.get_bar(self.id)
            .await
            .map_err(|e| {
                error!("Failed to get bar: {e:?}");
                FieldError::from("Internal Server Error")
            })
    }
}
```

With this implementation, the `FooBarQuery` GraphQL request will make `N+1` database queries
where `N` is the number of rows in the `foo` table.
The request latency therefore grows linearly with the number of rows returned.

## An off-the-shelf-solution solution

A well known solution to this problem in GraphQL is using Facebook's Dataloader
which was originally implemented in javascript but ported to many other languages,
including Rust.
It relies on the async runtime to batch database queries
which can turn linear time complexity GraphQL requests to constant time.

Here's how you'd use the [`dataloader`](https://crates.io/crates/dataloader) crate for the above problem:

```rust
use juniper::graphql_object;
use sqlx::{query_as, query_scalar, PgPool};
use tracing::error;
use dataloader::cached::Loader;
use dataloader::BatchFn;

struct Context {
    pool: PgPool,
    bar_loader: Loader<BarLoadFn>
}

impl Context {
    async fn get_foos(&self) -> sqlx::Result<Vec<Foo>> {
        query_as!(Foo, "SELECT id FROM foo")
            .fetch_all(&self.pool)
            .await
    }
}

struct BarLoadFn {
    pool: PgPool
};

struct Bar {
    id: i32,
    foo: i32
}

impl BarLoadFn {
    async fn get_bars(&self, foo_ids: &[i32]) -> sqlx::Result<Vec<Bar>> {
        query_as!(
            Bar,
            "SELECT id, foo FROM bar b WHERE b.foo = ANY($1)",
            foo_ids
        )
            .fetch_all(&self.pool)
            .await
    }
}

impl BatchFn<i32, sqlx::Result<i32>> for BarLoadFn {
    async fn load(&self, keys: &[i32]) -> HashMap<i32, sqlx::Result<i32>> {
        match self.get_bars(keys)
            .await {
                Ok(bars) => bars
                    .into_iter()
                    .map(|b| (b.foo, b.id))
                    .collect(),
                Err(e) => keys
                    .iter()
                    .map(|foo_id| (foo_id, Err(e)))
                    .collect()
            }
    }
}

/// Passed to `juniper::RootNode::new` to register this as the root query type
struct Query;

#[graphql_object(context = Context)]
impl Query {
    async fn foos(&self, context: &Context) -> FieldResult<Foo> {
        context.get_foos()
            .await
            .map(|db_item| Foo {db_item})
            .map_err(|e| {
                error!("Failed to get foos: {e:?}");
                FieldError::from("Internal Server Error")
            })
    }
}

struct Foo {
    id: i32
}

#[graphql_object(context = Context)]
impl Foo {
    async fn bar(&self, context: &Context) -> FieldResult<i32> {
        // The `dataloader` crate documentation shows how to use the `Loader::load` method
        // instead of the `Loader::try_load` method used below.
        // The `Loader::load` method panics if the argument isn't a key in the `HashMap`
        // returned by the `BatchFn::load` method instead of returning an `Err` like `try_load`
        context
            .bar_loader
            .try_load(self.id)
            .await
            .map_err(|e| {
                error!("Couldn't find bar with ID={}: {e:?}", self.id);
                FieldError::from("Internal Server Error")
            })
            .and_then(|r| r
                .map_err(|e| {
                    error!("Failed to fetch bar with ID={}: {e:?}", self.id);
                    FieldError::from("Internal Server Error")
                })
            )
    }
}
```

As well as encouraging use of the `Loader::load` method,
the `dataloader` crate also has the footgun of using the `async-std` runtime as a default feature
which I've seen misconfigured in servers running the `tokio` runtime.

Whilst `dataloader` generally batches database queries as intended
and reduces request latencies,
I've observed suboptimal batching behaviours with batch sizes of just 2 or 3 keys
even when setting the maximum batch size much higher.
`dataloader` relies on the async runtime's `yield_now` function (e.g. [for tokio](https://docs.rs/tokio/latest/tokio/task/fn.yield_now.html)) to wait until other previously scheduled tasks have completed until running the batched database query.
The problem I ran into was not all keys were scheduled to be loaded before the first key start to load.
This caused several batched queries to be executed instead of one,
effectively nullifying the solution to the `N + 1` problem.

## Hand rolling an alternative

I decided to scrap using the `dataloader` crate
and use a mutex instead
to minimise the database queries
in a more predictable way.
This greatly reduce the request latency
but I had to be careful to avoid deadlocks.

[This](https://github.com/Charles-Johnson/damogo-graphql/commit/f91944cc959d09978f4dc020f4ec359f28df66de) is the first commit that delivered the performance improvements which encouraged me to remove other dataloaders in a [subsequent commit](https://github.com/Charles-Johnson/damogo-graphql/commit/79f193c0bf630b5f865eb35a461133d67076c1ea).
Below is how I'd apply that strategy to the problem in this blog post.

```rust
use juniper::graphql_object;
use sqlx::{query_as, query_scalar, PgPool};
use tracing::error;
use tokio::sync::Mutex;

struct Context {
    pool: PgPool,
    bar_cache: Arc<Mutex<HashMap<i32, Option<i32>>>>
}

impl Context {
    async fn get_foos(&self) -> sqlx::Result<Vec<Foo>> {
        query_as!(Foo, "SELECT id FROM foo")
            .fetch_all(&self.pool)
            .await
    }
    async fn get_bars(&self, foo_ids: &[i32]) -> sqlx::Result<Vec<Bar>> {
        query_as!(
            Bar,
            "SELECT id, foo FROM bar b WHERE b.foo = ANY($1)",
            foo_ids
        )
            .fetch_all(&self.pool)
            .await
    }
}

/// Passed to `juniper::RootNode::new` to register this as the root query type
struct Query;

#[graphql_object(context = Context)]
impl Query {
    async fn foos(&self, context: &Context) -> FieldResult<Foo> {
        let foos = context.get_foos()
            .await
            .map_err(|e| {
                error!("Failed to get foos: {e:?}");
                FieldError::from("Internal Server Error")
            })?;
        let mut bar_lock = self
            .bar_cache
            .lock()
            .await;
        // Remember the IDs of the foos and mark their bar as unknown
        bar_lock.extend(foos.iter().map(|foo| (foo.id, None)));
        Ok(foos)
    }
}

struct Foo {
    id: i32
}

#[graphql_object(context = Context)]
impl Foo {
    async fn bar(&self, context: &Context) -> FieldResult<i32> {
        // Crucial to reuse the lock to batch everything in one database query
        let mut bar_lock = context
            .bar_cache
            .lock()
            .await;
        match bar_lock.get(&self.id) {
            // Cache hit
            Some(Some(value)) => Ok(*value),
            // Cache miss
            other => {
                if other.is_none() {
                    // Possible if not all foo IDs were marked in the `foos` query 
                    error!("foo ID={} not marked for batch in advance", self.id);
                }
                bar_lock.extend(context
                    .get_bars(bar_lock.keys())
                    .await
                    .map_err(|e| {
                        error!("Failed to fetch bar with ID={}: {e:?}", self.id);
                        FieldError::from("Internal Server Error")
                    })
                );
                bar_lock
                    .get(&self.id)
                    .cloned()
                    .ok_or_else(|| {
                        error!("Couldn't find bar with ID={}", self.id);
                        FieldError::from("Internal Server Error")        
                    })
            }
        }
    }
}
```

## Other solutions I could have tried

- [`Loader::with_yield_count`](https://github.com/cksac/dataloader-rs/issues/12#issuecomment-603330462):
this essentially gives more time for all key loads to be scheduled so multiple batched database queries are less likely.
However, it's not obvious what number would be sufficient and too large a number may add significant latency.
- [The dataloader extracted from the `async-graphql` crate](https://docs.rs/versatile-dataloader):
appears to be the same algorithm
but with more natural error handling
and the ability to set a delay in milliseconds
instead of a yield count.
