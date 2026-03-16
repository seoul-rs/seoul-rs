+++
draft = true
title = "The Little Rust & Wasm Guidebook (3)"
date = "2026-03-18"
[taxonomies]
authors = ["Seungjin Kim"]
tags = ["wasm", "rust"]
+++

# 3. WASM과 웹브라우저



## 3.1 wasm 파일과 브라우저 연동하기
### 3.1.1 프로젝트 생성
  ```shell
  > cargo new --lib hello-world
  ```
  Cargo.toml 과 src/lib.rs

  `Cargo.toml`
  ```toml
  [package]
  name = "hello-wasm"
  version = "0.1.0"
  edition = "2024"

  [lib]
  crate-type = ["cdylib"]

  [dependencies]
  wasm-bindgen = "0.2"
  ```

  `src/lib.rs`
  ```rust
  use std::ffi::FromBytesWithNulError;

  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  extern "C" {
    fn alert(s: &str);
  }

  #[wasm_bindgen]
  pub fn greet(name: &str) -> String {
    format!("Hello, {}! This was built manually.", name)
  }

  #[wasm_bindgen]
  pub fn add(a: f64, b: f64) -> f64 {
    a + b
  }
  ```

### 3.1.2 wasm 파일만들기
  `cargo build --release --target wasm32-unknown-unknown`로 `hello-wasm.wasm` 파일을 빌드한다.  


### 3.1.3 wasm-bindgen
  wasm-bindgen은 러스트 크레이트로 컴파일러로 생성되는 Wasm 파일과 자바스크립트 사이에 연동을 가능하게 해준다.
  인스톨
  ```
  > cargo binstall wasm-bindgen-cli
  ```
  혹은 [wasm-bindgen 리포지토리](https://github.com/wasm-bindgen/wasm-bindgen) 에서 코드를 가져와 직접 빌드해도 된다.(추천)
  ```
  > git clone --depth 1 https://github.com/wasm-bindgen/wasm-bindgen.git && cd wasm-bindgen
  > cargo build --release --package wasm-bindgen-cli
  > install -s -Dm755 target/release/wasm-bindgen -t ~/.cargo/bin
  ```

  wasm-bindgen 으로 web에서 이용가능한 wasm으로 가공해보자
  ```
  > wasm-bindgen ./target/wasm32-unkown-unkown/release/hello_wasm --target web --out-dir ./pkg
  ```

  ```shell
  ❯ eza --tree pkg/
  pkg
  ├── hello_wasm.d.ts
  ├── hello_wasm.js
  ├── hello_wasm_bg.wasm
  └── hello_wasm_bg.wasm.d.ts
  ```

  `index.html`
  ```html
  <!DOCTYPE html>
  <html>
    <head>
      <meta charset="utf-8">
      <title>Manual Wasm Bindgen</title>
    </head>
    <body>
      <script type="module">
        // The CLI tool generated this file for you!
        import init, { greet } from './pkg/hello_wasm.js';
  
        async function run() {
          // Initialize the wasm module
          await init();
  
          // Call the function
          const message = greet("seoul.rs");
          console.log(message);
  
          const display = document.createElement("h1");
          display.textContent = message;
          document.body.appendChild(display);
        }
  
        run();
      </script>
    </body>
  </html>
  ```

  Run
  ```
  > miniserve -p 9099 . --index index.html
  ```

  ```text
  miniserve가 모죠?  
  Miniserve: a CLI tool to serve files and dirs over HTTP  
  https://github.com/svenstaro/miniserve
  ```



## 3.2 wasm-pack
  [wasm-pack](https://github.com/drager/wasm-pack) 은 앞의 `cargo init`, `wasm-bindgen` 등의 거맨드들을을 하나의 툴로 묶어 개발을 좀더 편하게 해준다.
  
### 3.2.1 설치
  `curl https://drager.github.io/wasm-pack/installer/init.sh -sSf | sh` https://drager.github.io/wasm-pack/installer/
  혹은 직접 빌드해서 설치한다.
  ```shell
  > git clone --depth 1 https://github.com/drager/wasm-pack.git && cd wasm-pack
  > cargo build --release
  > install -s -Dm755 target/release/wasm-pack -t ~/.cargo/bin
  ```

  ```shell
  ❯ wasm-pack help
  📦 ✨  pack and publish your wasm!
  
  Usage: wasm-pack [OPTIONS] <COMMAND>
  
  Commands:
    build    🏗️  build your npm package!
    pack     🍱  create a tar of your npm package but don't publish!
    new      🐑 create a new project with a template
    publish  🎆  pack up your npm package and publish!
    login    👤  Add an npm registry user account! (aliases: adduser, add-user)
    test     👩‍🔬  test your wasm!
    help     Print this message or the help of the given subcommand(s)
  
  Options:
    -v, --verbose...             Log verbosity is based off the number of v used
    -q, --quiet                  No output printed to stdout
        --log-level <LOG_LEVEL>  The maximum level of messages that should be logged by wasm-pack. [possible values: info, warn, error] [default: info]
    -h, --help                   Print help
    -V, --version                Print version
  ```

  
### 3.2.2 

  ```shell
  ❯ wasm-pack new hello-wasm
 [INFO]: ⬇️  Installing cargo-generate...
 🐑  Generating a new rustwasm project with name 'hello-wasm'...
 🔧   Destination: /tmp/hello-wasm ...
 🔧   project-name: hello-wasm ...
 🔧   Generating template ...
 [ 1/14]   Done: .appveyor.yml
 [ 2/14]   Done: .github/dependabot.yml
 [ 3/14]   Done: .github
 [ 4/14]   Done: .gitignore
 [ 5/14]   Done: .travis.yml
 [ 6/14]   Done: Cargo.toml
 [ 7/14]   Done: LICENSE_APACHE
 [ 8/14]   Done: LICENSE_MIT
 [ 9/14]   Done: README.md
 [10/14]   Done: src/lib.rs
 [11/14]   Done: src/utils.rs
 [12/14]   Done: src
 [13/14]   Done: tests/web.rs
 [14/14]   Done: tests
 🔧   Moving generated files into: `/tmp/hello-wasm`...
 🔧   Initializing a fresh Git repository
 ✨   Done! New project created /tmp/hello-wasm
 [INFO]: 🐑 Generated new project at /hello-wasm
  > 
  ```


  ```shell
  ❯ eza --tree hello-wasm
  hello-wasm
  ├── Cargo.toml
  ├── LICENSE_APACHE
  ├── LICENSE_MIT
  ├── README.md
  ├── src
  │   ├── lib.rs
  │   └── utils.rs
  └── tests
      └── web.rs
  ```

  wasm-pack build --target web

  ```shell
  ❯ eza --tree pkg/
  pkg
  ├── hello_wasm.d.ts
  ├── hello_wasm.js
  ├── hello_wasm_bg.js
  ├── hello_wasm_bg.wasm
  ├── hello_wasm_bg.wasm.d.ts
  ├── package.json
  └── README.md
  ```

## 3.3 wasm_bindgen_futures