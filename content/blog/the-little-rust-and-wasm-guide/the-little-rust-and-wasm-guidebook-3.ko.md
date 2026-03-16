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
  `Cargo.toml` 과 `src/lib.rs` 을 다음의 파일들로 바꾸어 보도록하자.

  `Cargo.toml`
  ```toml
  [package]
  name = "hello-wasm"
  version = "0.1.0"
  edition = "2024"

  [lib]
  crate-type = ["cdylib"]

  [dependencies]
  chrono = "0.4"
  wasm-bindgen = "0.2"
  ```
  앞서서 처음 생성해보았던 WASM 파일에서와은 다른점은 `dependencies`로 `chrono`와 `wasm-bindgen`이 추가되었다는 것이다. `chrono`는 rust 상에서 현재시간을 가져오는 예시상의 비즈니스 로직을 위한것이며 `wasm-bindgen`이 바로 Rust 코드로 생성된 WASM 파일이 Runtime을 통해 외부은 자바스크립트와 통신을 가능하게 해주는 것이다.  
  
  `src/lib.rs`
  ```rust
  use chrono::Local;
  use wasm_bindgen::prelude::*;
  
  #[wasm_bindgen]
    extern "C" {
        fn alert(s: &str);
    }
  
  #[wasm_bindgen]
    pub fn now() -> String {
        Local::now().to_rfc3339()
    }
  
  #[wasm_bindgen]
    pub fn pop_message(msg: &str) {
        alert(msg)
    }
  
  #[wasm_bindgen]
    pub fn add(a: f64, b: f64) -> f64 {
        a + b
    }
  ```
  wasm_bindgen 마크로로 쌓여있는 함수들이 뒤에서 자바스크립트를 통해 호출될 함수들이다. 현재 시간을 나타내는 now(), 자바스크립트의 alert() 함수를 호출하는 pop_message(&str) 그리고 더하기 연산을 하는 add(f64,f64)를 볼수있다.  
  

### 3.1.2 wasm 파일만들기
  `cargo build --release --target wasm32-unknown-unknown`로 `hello-wasm.wasm` 파일을 빌드한다.  


### 3.1.3 wasm-bindgen
  wasm-bindgen은 러스트 크레이트로 컴파일러로 생성되는 Wasm 파일과 자바스크립트 사이에 연동을 가능하게 해준다. 단순히 WASM 파일을 빌드할뿐 아니라 해당 WASM 파일을 호출하기위한 자바스크립트와 타입스크립트 파일도 생성해준다.  
  
  설치하기. 
  ```
  > cargo binstall wasm-bindgen-cli
  ```
  혹은 [wasm-bindgen 리포지토리](https://github.com/wasm-bindgen/wasm-bindgen) 에서 코드를 가져와 직접 빌드해도 된다.(추천)
  ```shell
  > git clone --depth 1 https://github.com/wasm-bindgen/wasm-bindgen.git && cd wasm-bindgen
  > cargo build --release --package wasm-bindgen-cli
  > install -s -Dm755 target/release/wasm-bindgen -t ~/.cargo/bin
  ```

  wasm-bindgen 명령어로 web에서 이용가능한 wasm으로 가공해보자
  ```
  > wasm-bindgen ./target/wasm32-unknown-unknown/release/hello_wasm.wasm --target web --out-dir ./pkg
  ```
  `target`이 `web`이고 `pkg` 디렉토리에 결과물을 생성한다.  

  ```shell
  ❯ eza --tree pkg/
  pkg
  ├── hello_wasm.d.ts
  ├── hello_wasm.js
  ├── hello_wasm_bg.wasm
  └── hello_wasm_bg.wasm.d.ts
  ```

  `pkg/hello_wasm.js`를 열어보면 자바스크립트상에서 이용하기 위해 자바스크립트로 export 된 WASM(rust) 파일의 함수들을 볼수있을 것이다. 자바스크립트와의 연동을위해 일일이 손으로 함수들을 재정은 할필요없이 자동으로 wase-bindgen 명령어에은해 생성된다.  

  이제 WEB에서 실제전으로 호출될 html 파일을 다음과 같이 생성하자. `index.html`을 `Cargo.toml` 이 위치한 프로젝트 루트에 생성하자.  

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
        import init, {now, pop_message}
        from './pkg/hello_wasm.js';
  
        async function run() {
          await init();
          pop_message("Current time is " + now());
        }
  
        run();
      </script>
  
      <h1> The Little Rust and Wasm Guidebook </h1>
      <form id="sumForm">
        <input type="number" id="num1" placeholder="First number" required>
        <input type="number" id="num2" placeholder="Second number" required>
        <button type="submit">Add</button>
      </form>
  
      <p>Result: <span id="result">0</span></p>
  
    <script type="module">
     import init, { add }
        from './pkg/hello_wasm.js';
  
      const form = document.getElementById('sumForm');
      const resultDisplay = document.getElementById('result');
  
      form.addEventListener('submit', (event) => {
        event.preventDefault();
        const val1 = document.getElementById('num1').value;
        const val2 = document.getElementById('num2').value;
  
        const sum = add(val1, val2);
  
        resultDisplay.textContent = sum;
      }
      );
    </script>
  
    </body>
  </html>
  ```

  `miniserve`로 이제 준비된 파일들을 실제로 실행해보자.
  ```
  > miniserve -p 9099 . --index index.html
  ```

  ```text
  miniserve가 모죠?  
  Miniserve: a CLI tool to serve files and dirs over HTTP  
  https://github.com/svenstaro/miniserve
  ```

  웹브라우저에서http://localhost:9099 를 확인해보자.  



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