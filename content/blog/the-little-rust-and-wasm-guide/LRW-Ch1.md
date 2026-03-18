+++
draft = true
title = "[WIP][LRW] 1. Build your first WASM"
date = "2026-03-15"
[taxonomies]
authors = ["Seungjin Kim"]
tags = ["wasm", "rust"]
+++

**Code First.**  
A wise man once said, *‘Talk is cheap. Show me the code.’*  
Keep Calm and Code On.  

### 1. Check your Rust compiler can handle WASM build
  ```shell
        ❯ rustc --print target-list | grep wasm
        wasm32-unknown-emscripten
        wasm32-unknown-unknown
        wasm32-wali-linux-musl
        wasm32-wasip1
        wasm32-wasip1-threads
        wasm32-wasip2
        wasm32-wasip3
        wasm32v1-none
        wasm64-unknown-unknown
  ```

  If you don't see the wasm32-unknown-unknown entry in the list, it means you aren't ready to compile WASM files yet.

  ```text
    wasm32-unknown-unkown means  
    wasm32: Indicates that the target CPU architecture is 32-bit WebAssembly.  
    unknown(Vendor): Signifies that no specific vendor (manufacturer) is defined for producing or distributing this binary.
    unknown(OS/ABI): Signifies that the target operating system or System Interface(ABI) is not defined.  
    Taken together, it refers to a "pure" 32-bit WebAssembly binary that can run anywhere, free from specific OS or hardware constraints.  

    Examples of Other Targets  
    x86_64-unknown-linux-gnu: Targets the 64-bit environment of the x86 architecture and is not limited to a specific vendor. It requires the Linux operating system and the GNU C library (glibc). This is the standard target used when compiling Linux C code.  
    aarch64-apple-darwin: Targets the aarch64 (Apple Silicon) architecture and is dependent on Apple devices and the Darwin OS/runtime.  
  ```

  Installing `wasm32-unknown-unknown` build target  
    
  ```shell
    > rustup target add wasm32-unknown-unknown
  ```
  Once the installation is complete, run the `rustc --print target-list` command again to verify that the `wasm32-unknown-unknown` target has been successfully added to the list.
    
### 2. Code Hello World 

  Creating a hello-world project using Cargo  
  ```shell
       ❯ cargo new --lib hello-world
       Creating library `hello-world` package
       note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
  ```

  The command `cargo new hello-wasm --lib` creates the following directory structure:   
  
  ```shell
    ❯ eza --tree hello-world
    hello-world
    ├── Cargo.toml
    └── src
    └── lib.rs
  ```

  ```text
  What is `eza`?
  eza는 기존의 ls 명령어를 현대적으로 재해석하여 대체하기 위한 도구이다. Rust 언어로 작성되어 매우 빠른 속도를 자랑하며, 사용자에게 더 직관적이고 풍부한 정보를 제공한다. You can find more at eza.rocks(https://eza.rocks) site.
  ```

  Generating a WASM binary requires adding a [lib] section and configuring the crate-type in Cargo.toml.  
  ```toml
  [lib]
  crate-type = ["cdylib"]
  ```
  
  Completed `Cargo.toml`    
  ```toml
  [package]
  name = "hello-world"
  version = "0.1.0"
  edition = "2024"

  [lib]
  crate-type = ["cdylib"]

  [dependencies]
  ```
  
  Write the contents of `src/lib.rs` as follows:  
  ```rust
  pub fn hello_world() -> &'static str' {
    "Hello World"
  }
  ```

### 3.Compiling

  `wasm` file build with `cargo build`

  ```shell
  ❯ cargo build --target wasm32-unknown-unknown --release
    Finished `release` profile [optimized] target(s) in 0.01s  
  ```

  The `hello_world.wasm` file can now be found in the `target/wasm32-unknown-unknown/release` folder.  

  ```shell
  ❯ file target/wasm32-unknown-unknown/release/hello_world.wasm
  target/wasm32-unknown-unknown/release/hello_world.wasm: WebAssembly (wasm) binary module version 0x1 (MVP)
  ```

  We have successfully created a `hello_world.wasm` file using a very simple Rust code. Running the `file` command on it confirms that it is indeed a WebAssembly (wasm) binary.  

  Now, you might be wondering how to execute and utilize this `.wasm file`. Unlike other languages, there is quite a lot to discuss here. This is where the true journey into the world of WebAssembly begins.  

  In the next post, we will take a closer look at what this WASM file actually is and how it can be executed in various environments.    





