+++
draft = false
title = "When type recursion gets out of control"
date = "2026-04-19"
[taxonomies]
authors = ["Charles Johnson"]
tags = ["rust", "async", "compile time"]
+++

# Introduction

Hi, I'm Charles and I'd like to share issues I had with Rust compile times at work. Most of the advice you see online about reducing compile times addresses the different stages with the standard remedies:

- Compiling 3rd party dependencies: locally using cargo is fine but in CI you could use cargo-chef or sccache
- Compiling your crates: use a workspace and parallelise the workspace dependency graph to compile multiple smaller crates on parallel process instead of using a single process for one big crate
- Linker: use alternative linkers e.g. mold

Going deeper you could follow fasterthanlime's blog post, [Why is my rust build so slow](https://fasterthanli.me/articles/why-is-my-rust-build-so-slow), where he demonstrated the problems with `warp`'s tendency to create very big types and the cost of monomorphisation from generic types as well as show you a host of other tools to debug compile times.

However, none of the blogs I found cut to the core of the problem I faced at work so I'd like to share my experience so others that run into something similar can apply it.

# Problem

At [Clear](getclearapp.com), we used to suffer from unreliable rust release builds. When compiling the GraphQL server, the build sometimes failed, reporting an `overflow evaluating the requirement` for a type deep wrapped in a generic type from our 3rd party dependencies but originating from `juniper::graphql_object` macro calls for GraphQL types. If we tried to rebuild the exact same code, the build would often succeed.

# Splitting one crate into many

I later split the original binary crate, `clear-server` into multiple library crates (as well as later other binary crates that weren't used for our GraphQL server). I then isolated the `juniper::graphql_object` calls to three library crates: `clear_public_graphql_api` (types exclusive to the GraphQL API for the mobile app), `clear_admin_graphql_api` (types exclusive to the GraphQL API for the admin panel) and `clear_shared_graphql_api` (types common to both mobile app and admin panel). I split the remaining code that didn't depend on `juniper` but interacted with the database into a `clear_db_client` crate.

This is what the workspace crate dependency graph looked like

{% mermaid() %}

graph TD
A(clear-server) --> B(clear_public_graphql_api)
A --> C(clear_admin_graphql_api)
B --> D(clear_shared_graphql_api)
C --> D
A --> E(clear_db_client)
B --> E
C --> E
D --> E
{% end %}

Since increasing the `recursion_limit` (e.g. `#![recursion_limit = 1024]` at the top of `lib.rs`), from the default of 128, to 1024 for `clear_public_graphql_api` and 256 for `clear_admin_graphql_api`, no overflows have been triggered. I still, however, observe `clear_public_graphql_api` taking noticeably longer to compile than any other crate indicating that its recursion depth is the compile time bottleneck.

Right now, I can reproduce the error for a debug build if I set the recursion limit for `clear_public_graphql_api` a lot lower, to 49. In this case, the top level error message is `overflow evaluating the requirement &str: Sync` which comes from our application code to insert image URLs into the database which the `graphql_object` macro call for the `Mutation` type directly depends on. 

# Inversion of control to flatten crate dependency graph

We can remove the dependency on the database query code from the `graphql_object` macro call by adding a trait object to the GraphQL context type which has trait methods to interact with the database.

For example, if the code below had a type recursion problem,
```rust
use juniper::{
    graphql_object,
    Context
};
use diesel::{
    r2d2::{Pool, ConnectionManager},
    pg::PgConnection,
    dsl::count_star
};
use crate::schema::users;

type PgPool = Pool<ConnectionManager<PgConnection>>;

struct MyContext {
    pool: PgPool,
}

impl MyContext {
    fn total_users(&self) -> i32 {
        let mut conn = self.pool.get().unwrap();
        let total_users: Result<i32, _> = users::table.select(count_star())
            .first::<i64>(&mut conn)
            .try_into();
        total_users.unwrap()
    }
}

impl Context for MyContext {}

struct MyType;

#[graphql_object(context = MyContext)]
impl MyType {
    fn foo(&self, ctx: &MyContext) -> i32 {
        ctx.total_users()
    }
}
```
we could define a trait
```rust
trait GraphQlContext {
    fn total_users(&self) -> i32;
}

struct DbContext {
    pool: PgPool,
}

impl GraphQLContext for DbContext {
    fn total_users(&self) -> i32 {
        let mut conn = self.pool.get().unwrap();
        let total_users: Result<i32, _> = users::table.select(count_star())
            .first::<i64>(&mut conn)
            .try_into();
        total_users.unwrap()
    }
}
```
that can be part of the existing GraphQL context type
```rust
struct MyContext {
    pool: PgPool,
    dynamic_part: Arc<dyn DynContext>,
}

impl MyContext {
    fn total_users() -> i32 {
        ctx.dynamic_part.total_users()
    }
}
```

For simplicity, this example didn't use `async fn` (because a database connection is acquired synchronously) so the type recursion wouldn't actually be a problem; the type information is encapsulated within `fn` blocks. The codebase originally asyncronously acquired connections for diesel queries which forced the majority of GraphQL methods to be `async fn`. We also started using `sqlx` which both asyncronously acquired connections and executed queries. Because `async fn` actually generates a unique type based on the contents in the block, type information floats up the call graph, increasing the type recursion.

What we have done already is define a `GraphQLContext` trait using the [`async-trait`](https://crates.io/crates/async-trait) macro in a `clear_graphql_context` crate and implement it for `DbGraphQLContext` in the `clear_db_graphql_context` crate which the `clear_*_graphql_api` crates don't depend on. 

Here's what the dependency graph looks like:

{% mermaid() %}
graph TD
A(clear-server) --> B(clear_public_graphql_api)
A --> C(clear_admin_graphql_api)
B --> D(clear_shared_graphql_api)
C --> D
A --> E(clear_db_client)
B --> E
C --> E
D --> E
B --> F(clear_graphql_context)
C --> F
D --> F
G(clear_db_graphql_context) --> F
G --> E
A --> G
A --> F
F --> E
{% end %}

# Result

We have been implementing new GraphQL queries and mutations by using the `GraphQLContext` trait as an interface to hide type information from the compiler. This has reduced the debug cycle when we are tweaking the application logic in the `clear_db_graph_context` crate and allows for a stable interface for integration tests. 

We could also start refactoring existing GraphQL resolvers, starting with those that insert image URLs but even evaluating `Sync` requirement for the GraphQL context requires a type recursion depth of 38 due to database pool types. However, we'd have to refactor every single GraphQL resolver so that database pools aren't accessed directly but via a `GraphQLContext` trait object in order to unlock further compilation time reductions. This is why I haven't tried to completely overcome the compilation time issue and have instead looked for alternative client-server architectures that I will explain in another blog entry.
