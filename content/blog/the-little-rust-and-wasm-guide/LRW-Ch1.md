+++
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

  Cargo를 이용한 hello-world 프로젝트 생성  
  ```shell
       ❯ cargo new --lib hello-world
       Creating library `hello-world` package
       note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
  ```

  `cargo new hello-wasm --lib` 명령어로 생성된 프로젝트의 디렉터리 구조는 다음과 같다.  
  ```shell
    ❯ eza --tree hello-world
    hello-world
    ├── Cargo.toml
    └── src
    └── lib.rs
  ```

  ```text
  `eza`가 모죠?
  eza는 기존의 ls 명령어를 현대적으로 재해석하여 대체하기 위한 도구이다. Rust 언어로 작성되어 매우 빠른 속도를 자랑하며, 사용자에게 더 직관적이고 풍부한 정보를 제공한다. 공식 웹사이트인 eza.rocks에서 더 자세한 정보를 확인할 수 있다.
  ```

  WASM 바이너리를 생성하기 위해서는 Cargo.toml 파일에 [lib] 섹션을 추가하고, crate-type을 설정해야 한다.  
  ```toml
  [lib]
  crate-type = ["cdylib"]
  ```

  
  완성된 Cargo.toml
  ```toml
  [package]
  name = "hello-world"
  version = "0.1.0"
  edition = "2024"

  [lib]
  crate-type = ["cdylib"]

  [dependencies]
  ```
  
  

  아래와 같이 `src/lib.rs`를 작성한다.  
  ```rust
  pub fn hello_world() -> &'static str' {
    "Hello World"
  }
  ```

### 3. 컴파일하기

  `cargo build`로 `wasm` 파일 빌드  
  ```shell
  ❯ cargo build --target wasm32-unknown-unknown --release
    Finished `release` profile [optimized] target(s) in 0.01s  
  ```
  `target/wasm32-uknown-unkown/release` 폴더에서 `hello_world.wasm` 파일을 확인해본다.

  ```shell
  ❯ file target/wasm32-unknown-unknown/release/hello_world.wasm
  target/wasm32-unknown-unknown/release/hello_world.wasm: WebAssembly (wasm) binary module version 0x1 (MVP)
  ```

아주 간단한 Rust 코드를 통해 `hello_world.wasm` 파일을 생성해 보았다. `file` 명령어를 사용하여 해당 파일을 확인하면 `WebAssembly (wasm) binary`라는 결과를 얻을 수 있다.

이제 이 `.wasm` 파일을 어떻게 실행하고 활용할 수 있는지 궁금할 것이다. 이에 대해서는 다른 언어들과 달리 이야기할 거리가 아주 많다. 여기서부터가 진정한 WebAssembly의 세계로 들어가는 시작이기 때문이다.

다음 글에서는 우리가 만든 이 WASM 파일의 실체가 무엇인지, 그리고 어떤 환경에서 어떻게 실행할 수 있는지에 대해 자세히 알아보도록 하겠다.




