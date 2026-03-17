+++
title = "[WIP][LRW] 1. Build your first WASM"
date = "2026-03-15"
[taxonomies]
authors = ["Seungjin Kim"]
tags = ["wasm", "rust"]
+++

## 1. Code First

A wise man once said, ‘Talk is cheap. Show me the code.’ Now let’s make your WASM file first.

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

  만약 목록에서 `wasm32-unknown-unknown` 항목이 보이지 않는다면, 이는 아직 WASM 파일을 컴파일할 준비가 되지 않았음을 의미한다.

  ```text
    `wasm32-unknown-unkown`의 의미
    wasm32: 대상 CPU 아키텍처가 32비트 WebAssembly임을 의미한다.
    unknown (Vendor): 이 바이너리를 제조하거나 배포하는 특정 벤더(제조사)가 정의되지 않았음을 뜻한다.
    unknown (OS/ABI): 대상 운영체제나 시스템 인터페이스(ABI)가 정의되지 않았음을 뜻한다.
    종합하면, 특정 운영체제나 하드웨어 제약 없이 어디서든 실행 가능한 순수한 32비트 WebAssembly 바이너리를 의미한다.

    다른 타겟의 예시
    x86_64-unknown-linux-gnu: x86 아키텍처의 64비트 환경을 대상으로 하며, 특정 벤더에 한정되지 않는다. Linux 운영체제와 GNU C 라이브러리(glibc)가 필요하며, 리눅스 C 코드를 컴파일할 때 사용하는 표준적인 타겟이다.
    aarch64-apple-darwin: aarch64(Apple Silicon) 아키텍처를 대상으로 하며, Apple 기기 및 Darwin OS/런타임에 종속적임을 의미한다.
  ```

  `wasm32-unknown-unknown` 타깃 설치하기
    
  ```shell
    > rustup target add wasm32-unknown-unknown
  ```

  설치를 마쳤다면, 다시 한번 `rustc --print target-list` 명령을 실행하여 `wasm32-unknown-unknown` 타깃이 목록에 정상적으로 추가되었는지 확인한다.
    
### 2. Hello World 코드

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




