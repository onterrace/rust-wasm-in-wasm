# 1.8 웹어셈블리 안의 어셈블리

이 페이지에서는 코드를 상세히 설명하지 않습니다. 어셈블리를 어떻게 임포트하여 사용할 수 있는지에 대해 집중합니다. 

**원문 제목: js-sys: WebAssembly in WebAssembly**

Rust 프로그램은 WebAssembly(wasm)의 함수를 호출할 수 있습니다.WebAssembly는 웹 브라우저 또는 독립 실행형 WebAssembly 런타임에서 실행되도록 설계되었지만 Rust와 같은 언어로 작성된 기본 애플리케이션을 포함하여 다른 애플리케이션에 내장될 수도 있습니다.

Rust에서 wasm 함수를 호출하려면 wasm 모듈용 Rust 바인딩을 생성하는 편리한 방법을 제공하는 wasm-bindgen 크레이트를 사용할 수 있습니다. wasm-bindgen은 #[wasm_bindgen]이라는 절차적 매크로를 사용하여 Rust에서 wasm 함수를 가져오고 호출하는 데 필요한 코드를 생성합니다


이 문서의 원본은 [js-sys: WebAssembly in WebAssembly](https://rustwasm.github.io/docs/wasm-bindgen/examples/wasm-in-wasm.html)에서 찾을 수 있습니다.

js-sys 크레이트를 사용하여 WebAssembly 모듈 내부에서 예쁜 메타를 얻고 WebAssembly 모듈을 인스턴스화할 수 있습니다!


> 이 코드의 전체 코드는 [Github](https://github.com/latteonterrace/rust-wasm-in-wasm)에서 확인할 수 있습니다.

우리는 두 개의 웹어셈블리 크레이트를 만들 것입니다. 하나는 간단한 덧셈 함수를 가진 크레이트이고 이것을 웹어셈블리로 빌드할 것입니다.  다른 하나는 그 만들어진 웹 어셈블리를 불러와서 덧셈 함수를 사용하는 웹 어셈블리입니다. 


add라는 크레이트를 생성하고 lib.rs에 다음과 같이 작성합니다.



**add/src/lib.rs**  
```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```

Cargo.toml을 다음과 같이 작성합니다. 
```toml
[package]
name = "add"
version = "0.1.0"
edition = "2018"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.83"
```
이 크레이트를 다음을 실행하여 어셈블리로 빌드합니다.

```shell
wasm-pack build --target web
```
이제 방금 우리가 빌드한 웹어셈블리를 이용하는 크레이트를 만들어 봅시다.

```shell
cargo new wasm-in-wasm --lib
```


add 크레이트의 pkg 디렉토리에서 생성된 add_bg.wasm 파일을 wasm-in-wasm 크레이트의 src 디렉토리로 복사합니다. lib.rs 파일을 열고 다음과 같이 입력합니다. 

**wasm-in-wasm/src/lib.rs**    
```rust
use js_sys::{Function, Object, Reflect, WebAssembly};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};


// lifted from the `console_log` example
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(a: &str);
}

macro_rules! console_log {
    ($($t:tt)*) => (log(&format_args!($($t)*).to_string()))
}


const WASM: &[u8] = include_bytes!("add_bg.wasm");

async fn run_async() -> Result<(), JsValue> {
    // console_log!("instantiating a new wasm module directly");

    let a = JsFuture::from(WebAssembly::instantiate_buffer(WASM, &Object::new())).await?;
    let b: WebAssembly::Instance = Reflect::get(&a, &"instance".into())?.dyn_into()?;

    let c = b.exports();

    let add = Reflect::get(c.as_ref(), &"add".into())?
        .dyn_into::<Function>()
        .expect("add export wasn't a function");

    let three = add.call2(&JsValue::undefined(), &1.into(), &2.into())?;
    console_log!("1 + 2 = {:?}", three);
    let mem = Reflect::get(c.as_ref(), &"memory".into())?
        .dyn_into::<WebAssembly::Memory>()
        .expect("memory export wasn't a `WebAssembly.Memory`");
    console_log!("created module has {} pages of memory", mem.grow(0));
    console_log!("giving the module 4 more pages of memory");
    mem.grow(4);
    console_log!("now the module has {} pages of memory", mem.grow(0));

    Ok(())
}

#[wasm_bindgen(start)]
pub fn run() {
    spawn_local(async {
        run_async().await.unwrap_throw();
    });
}
```

이 코드는 조금 복잡하지만 지금 모든 것을 알 필요는 없습니다. 코드에 대해서는 자세히 설명하지 않습니다. 이 페이지에서는 웹어셈블리를 이용하여 웹어셈블리를 실행하는 방법에 대해서만 설명합니다.  


코드에 보면 웹 어셈블리를 로드하는 코드가 있습니다. 
```rust
const WASM: &[u8] = include_bytes!("add_bg.wasm");
```
이 코드 아래에 보면 add를 호출하는 코드가 있습니다. 
```rust
    let add = Reflect::get(c.as_ref(), &"add".into())?
        .dyn_into::<Function>()
        .expect("add export wasn't a function");

    let three = add.call2(&JsValue::undefined(), &1.into(), &2.into())?;
```    

프로젝트의 디렉토리를 자세히 들여다 보면 다음과 같은 구조를 가지고 있습니다. 

```shell
📂project-root
├── 📂add
│   ├── 📂pkg
│   │   ├── 📄add_bg.wasm  # 이 파일을 복사할 것입니다. 
├── 📂wasm-in-wasm
│   ├── 📂src
|       ├── 📄lib.rs
|       ├── 📄add_bg.wasm # 여기에 복사합니다. 
├── 📄index.html
```


wasm-in-wasm 크레이트의 Cargo.toml을 다음과 같이 작성합니다. 

**wasm-in-wasm/Cargo.toml**
```shell
[package]
name = "wasm-in-wasm"
version = "0.1.0"
authors = ["The wasm-bindgen Developers"]
edition = "2018"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.83"
js-sys = "0.3.60"
wasm-bindgen-futures = "0.4.33"
```

wasm-in-wasm에 index.html을 다음 코드를 추가합니다. 

**wasm-in-wasm/index.html**    
```html
<body>
  <script type="module">
    import init  from './pkg/wasm_in_wasm.js';

    async function run() {
      await init();
    }
    run();
  </script>
</body>
```
wasm-in-wasm를 빌드합니다. 
```shell
wasm-pack build --target web
```

VSCode에서 index.html을  Live Server로 실행합니다. 브라우저에서 개발자 도구를 열고 콘솔을 확인하면 로그를 볼 수 있을 것입니다. 
