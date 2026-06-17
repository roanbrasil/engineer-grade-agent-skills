---
name: wasm-patterns
description: Apply WebAssembly patterns for browser compute offload, WASI server-side plugins, edge functions (Cloudflare Workers, Spin), and Rust-to-Wasm compilation — including when to use Wasm and when to avoid it.
---

# WebAssembly Patterns

## What WebAssembly Is

WebAssembly (Wasm) is a binary instruction format for a stack-based virtual machine.
It runs at near-native speed, is sandboxed by default, and is portable across environments.

Key properties:
- **Performance**: within 10-20% of native speed for CPU-bound workloads
- **Security**: capability model — no system access unless the host explicitly grants it
- **Portability**: compile once, run in browser, server, edge, IoT
- **Language agnostic**: compile from Rust, C/C++, Go, Kotlin, Python (Pyodide), and others

```
  Source Code         Compiler          WebAssembly
  +-----------+      +----------+      +----------+
  | Rust / C  | ---> | wasm32   | ---> | .wasm    |
  | Go / Py   |      | target   |      | binary   |
  +-----------+      +----------+      +----------+
                                            |
              +-----------------------------+-------------+
              |                             |             |
         +--------+                  +---------+    +---------+
         |Browser |                  | Wasmtime|    |  Edge   |
         | (V8)   |                  | (WASI)  |    | Worker  |
         +--------+                  +---------+    +---------+
```

---

## Browser WebAssembly

### Use Cases

- Image processing, video encoding/decoding
- Cryptographic operations (hashing, signing)
- Game engines, physics simulations
- Audio processing, speech recognition
- Running portable libraries (e.g., SQLite in browser via sql.js)

### Loading a Wasm Module

```javascript
// Preferred: streaming compilation (doesn't wait for full download)
const { instance } = await WebAssembly.instantiateStreaming(
  fetch('/module.wasm'),
  importObject  // host functions the module can call
);

// Call an exported function
const result = instance.exports.add(1, 2);
```

### Memory Model

Wasm uses a flat linear memory (`WebAssembly.Memory`). Numbers pass natively;
strings and objects must be serialized through memory.

```javascript
// Allocate memory in Wasm, write a string, call function
const memory = new Uint8Array(instance.exports.memory.buffer);
const ptr = instance.exports.alloc(str.length);
for (let i = 0; i < str.length; i++) {
  memory[ptr + i] = str.charCodeAt(i);
}
const resultPtr = instance.exports.process_string(ptr, str.length);
instance.exports.dealloc(ptr, str.length);
```

```
  JavaScript Side          Wasm Linear Memory
  +-------------+          +------------------+
  | JS String   | -copy--> | ptr: 0x1000      |
  | "hello"     |          | bytes: 68 65 ... |
  +-------------+          +------------------+
                                    |
                           +------------------+
                           | Wasm function    |
                           | processes bytes  |
                           +------------------+
```

### Rust + wasm-bindgen (Browser)

wasm-bindgen auto-generates JS bindings, handling string/object serialization.

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[wasm_bindgen]
pub struct ImageProcessor {
    width: u32,
    height: u32,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> ImageProcessor {
        ImageProcessor { width, height }
    }

    pub fn invert(&self, pixels: &mut [u8]) {
        for pixel in pixels.iter_mut() {
            *pixel = 255 - *pixel;
        }
    }
}
```

```bash
# Build: Rust -> Wasm -> npm package
wasm-pack build --target web --out-dir pkg

# Output: pkg/
#   module_bg.wasm   (the binary)
#   module.js        (generated JS glue)
#   module.d.ts      (TypeScript types)
#   package.json
```

```javascript
// JavaScript usage (ESM)
import init, { greet, ImageProcessor } from './pkg/module.js';

await init(); // loads and compiles the .wasm
console.log(greet('world')); // "Hello, world!"

const proc = new ImageProcessor(800, 600);
proc.invert(pixelData);
```

### Offload to Web Worker (Critical Pattern)

Wasm runs synchronously and blocks the main thread. Always run heavy Wasm in a Worker.

```javascript
// worker.js
import init, { heavyCompute } from './pkg/module.js';

await init();

self.onmessage = async ({ data }) => {
  const result = heavyCompute(data.input);
  self.postMessage({ result });
};

// main.js
const worker = new Worker('./worker.js', { type: 'module' });
worker.postMessage({ input: largeDataset });
worker.onmessage = ({ data }) => renderResult(data.result);
```

---

## WASI — WebAssembly System Interface

WASI gives Wasm access to system resources outside the browser:
file I/O, networking, clocks, random numbers — all capability-gated.

```
  Wasm Module
  +------------------+
  | your code        |
  | (no syscalls)    |
  +------------------+
          |
          | WASI calls (wasi_snapshot_preview1 / Preview 2)
          v
  +------------------+
  | Runtime Host     |
  | (Wasmtime etc.)  |
  | grants only what |
  | you allow        |
  +------------------+
          |
   +------+------+
   |             |
  Filesystem   Network
  (preopened   (allowed
   dirs only)   hosts only)
```

### WASI Preview 2 / Component Model

Preview 2 introduces typed, language-agnostic composition using **WIT (Wasm Interface Types)**:

```wit
// counter.wit
package example:counter@0.1.0;

interface counter {
  increment: func(by: u32) -> u32;
  reset: func();
  value: func() -> u32;
}

world counter-world {
  export counter;
}
```

Components are composable: link a Rust HTTP handler with a Go database layer, etc.

### Runtimes

| Runtime   | Focus                        | Language |
|-----------|------------------------------|----------|
| Wasmtime  | Reference impl, WASI P1+P2   | Rust     |
| WasmEdge  | Cloud-native, network ext.   | C++      |
| WAMR      | Embedded, tiny footprint     | C        |
| wasmer    | Universal, many backends     | Rust     |

---

## Wasmtime — Server-Side Plugin System

Embed Wasmtime in your application to run untrusted user-provided Wasm modules safely.

### Host Application (Rust)

```rust
use wasmtime::*;
use wasmtime_wasi::WasiCtxBuilder;

fn run_plugin(wasm_bytes: &[u8], input: &str) -> anyhow::Result<String> {
    let engine = Engine::default();
    let mut store = Store::new(&engine, ());

    // --- Define what the plugin is allowed to call ---
    let mut linker: Linker<()> = Linker::new(&engine);

    // Host function: logging (plugin can call this)
    linker.func_wrap("env", "log", |caller: Caller<'_, ()>, ptr: i32, len: i32| {
        // read string from plugin memory and log it
        let mem = caller.get_export("memory")
            .and_then(|e| e.into_memory())
            .unwrap();
        let data = mem.data(&caller);
        let s = std::str::from_utf8(&data[ptr as usize..(ptr + len) as usize]).unwrap();
        println!("[plugin] {}", s);
    })?;

    // Compile and instantiate the plugin
    let module = Module::new(&engine, wasm_bytes)?;
    let instance = linker.instantiate(&mut store, &module)?;

    // Call the plugin's exported function
    let process = instance.get_typed_func::<(i32, i32), i32>(&mut store, "process")?;
    let result_ptr = process.call(&mut store, (input_ptr, input.len() as i32))?;
    // ... read result from memory
    Ok(result_string)
}
```

### Fuel — Limit Computation

```rust
let mut config = Config::new();
config.consume_fuel(true);

let engine = Engine::new(&config)?;
let mut store = Store::new(&engine, ());
store.set_fuel(1_000_000)?; // ~1M instructions before trap

// Plugin is killed if it exceeds fuel
let result = plugin_func.call(&mut store, args);
match result {
    Err(e) if e.downcast_ref::<Trap>().map_or(false, |t| *t == Trap::OutOfFuel) => {
        eprintln!("Plugin exceeded computation limit");
    }
    other => other?,
}
```

---

## Spin — Wasm Microservices (Fermyon)

Spin builds HTTP services and event handlers as Wasm components.
Cold start under 1ms — no container image, no OS boot.

```
  HTTP Request
       |
  +----v------+       spin.toml
  |  Spin     |<------[component routes]
  |  Runtime  |
  +----+------+
       |
  +----v-----------+
  | .wasm component|   (stateless; one request = one invocation)
  |  your handler  |
  +----------------+
       |
  Outbound HTTP, KV store, SQLite, Redis (only allowed hosts)
```

```toml
# spin.toml
spin_manifest_version = 2

[application]
name = "my-api"
version = "0.1.0"

[[trigger.http]]
route = "/api/..."
component = "api-handler"

[component.api-handler]
source = "target/wasm32-wasi/release/api_handler.wasm"
allowed_outbound_hosts = ["https://api.example.com"]
[component.api-handler.build]
command = "cargo build --target wasm32-wasi --release"

[[trigger.http]]
route = "/health"
component = "health-check"

[component.health-check]
source = "target/wasm32-wasi/release/health.wasm"
```

```rust
// Rust Spin handler
use spin_sdk::http::{IntoResponse, Request, Response};
use spin_sdk::http_component;

#[http_component]
fn handle(req: Request) -> anyhow::Result<impl IntoResponse> {
    let body = format!("Path: {}", req.uri().path());
    Ok(Response::builder()
        .status(200)
        .header("content-type", "text/plain")
        .body(body)
        .build())
}
```

```python
# Python Spin handler
from spin_sdk import http
from spin_sdk.http import Request, Response

class IncomingHandler(http.IncomingHandler):
    def handle_request(self, request: Request) -> Response:
        return Response(
            200,
            {"content-type": "application/json"},
            b'{"status": "ok"}'
        )
```

### Deploy

```bash
# Local dev
spin up

# Fermyon Cloud
spin deploy

# Kubernetes (SpinKube)
kubectl apply -f spinapp.yaml
```

---

## Cloudflare Workers — Edge Wasm

Cloudflare Workers run JavaScript at 300+ edge locations. Wasm modules can be imported directly.

```javascript
// worker.js
import wasmModule from './image-transform.wasm';

let instance;

export default {
  async fetch(request, env) {
    // Instantiate once per isolate (cached)
    if (!instance) {
      instance = await WebAssembly.instantiate(wasmModule, {
        env: {
          log: (ptr, len) => { /* host logging */ }
        }
      });
    }

    const url = new URL(request.url);
    if (url.pathname.startsWith('/transform')) {
      const imageData = await request.arrayBuffer();
      const result = transformImage(instance, imageData);
      return new Response(result, {
        headers: { 'Content-Type': 'image/webp' }
      });
    }

    return fetch(request);
  }
};
```

```toml
# wrangler.toml
name = "image-transformer"
main = "worker.js"
compatibility_date = "2024-01-01"

[[rules]]
type = "CompiledWasm"
globs = ["**/*.wasm"]
```

```
  Client Request
       |
  [Cloudflare Edge — nearest PoP]
       |
  +----v----------+
  | Worker Isolate|   <1ms cold start
  | + Wasm module |   no container spin-up
  +---------------+
       |
  [origin or cache]
```

### Use Cases for Edge Wasm

- Image resizing/format conversion at edge (avoid round-trip to origin)
- Auth token validation without origin request
- A/B testing with complex bucketing logic
- Geo-based routing with heavy custom logic
- Request/response transformation

---

## When to Use Wasm

```
Decision Tree:

Is the work CPU-bound (not I/O-bound)?
  NO  --> Use async JS/Python/Go; Wasm won't help
  YES -->
    Is it in the browser?
      YES --> Is it > ~10ms of JS work?
               YES --> Use Wasm (+ Worker thread)
               NO  --> Stick with JS (serialization overhead not worth it)
      NO  -->
        Is it a plugin/sandboxing use case?
          YES --> Wasmtime or similar embedded runtime
          NO  -->
            Is it a serverless/edge function with cold-start sensitivity?
              YES --> Spin (Fermyon) or Cloudflare Worker + Wasm
              NO  --> Consider native binary or container
```

### Wasm Shines

| Scenario | Why Wasm |
|----------|----------|
| Browser image/video processing | Near-native speed; offload from JS |
| Third-party plugin execution | Sandboxed; can't escape host process |
| Edge functions | Sub-millisecond cold start |
| Portable CLI tools | One binary runs on any OS/arch |
| Embedded scripting | Safe user-defined logic in your app |

### Wasm Is Not the Answer

| Scenario | Why Not |
|----------|---------|
| Simple CRUD API | Network I/O dominates; Wasm adds complexity |
| String parsing in browser | JS is faster; no serialization cost |
| Greenfield microservices | Containers/native binaries are simpler |
| Heavy I/O workloads | Async JS/Python handles I/O; Wasm does not help |

---

## Complete Rust → Wasm Example (Browser)

```
Project layout:
  my-wasm-lib/
  ├── Cargo.toml
  ├── src/
  │   └── lib.rs
  └── pkg/               (generated by wasm-pack)
      ├── my_wasm_lib.js
      ├── my_wasm_lib.d.ts
      └── my_wasm_lib_bg.wasm
```

```toml
# Cargo.toml
[package]
name = "my-wasm-lib"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"
web-sys = { version = "0.3", features = ["console"] }

[profile.release]
opt-level = "z"      # optimize for size
lto = true
```

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

// Called once when module loads
#[wasm_bindgen(start)]
pub fn main() {
    console_error_panic_hook::set_once(); // better panic messages
}

/// Fast Fourier Transform for audio processing
#[wasm_bindgen]
pub fn fft(samples: &[f32]) -> Vec<f32> {
    // ... compute FFT ...
    samples.to_vec() // placeholder
}

/// Compress image pixels in-place
#[wasm_bindgen]
pub fn compress_rgba(pixels: &mut [u8], quality: u8) {
    for chunk in pixels.chunks_mut(4) {
        // reduce color depth based on quality
        let factor = 256u16 / (quality as u16 + 1);
        chunk[0] = ((chunk[0] as u16 / factor) * factor) as u8;
        chunk[1] = ((chunk[1] as u16 / factor) * factor) as u8;
        chunk[2] = ((chunk[2] as u16 / factor) * factor) as u8;
        // alpha unchanged
    }
}
```

```bash
# Build commands
rustup target add wasm32-unknown-unknown
cargo install wasm-pack

wasm-pack build --target web --out-dir pkg
# For Node.js: --target nodejs
# For bundlers (webpack/vite): --target bundler
```

```html
<!-- index.html -->
<script type="module">
  import init, { fft, compress_rgba } from './pkg/my_wasm_lib.js';

  await init();  // fetches and compiles .wasm

  // Use canvas to get pixel data
  const canvas = document.getElementById('canvas');
  const ctx = canvas.getContext('2d');
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

  // Direct view into Wasm memory — zero copy!
  compress_rgba(imageData.data, 50);
  ctx.putImageData(imageData, 0, 0);
</script>
```

---

## Anti-Patterns

### 1. Using Wasm for Simple String Work

```javascript
// WRONG: serialization cost exceeds any speed gain
const result = wasmInstance.exports.to_uppercase(str); // copies str into Wasm memory

// CORRECT: JS handles strings natively with zero overhead
const result = str.toUpperCase();
```

### 2. Running Wasm on the Main Thread

```javascript
// WRONG: blocks UI for 500ms
const result = heavyWasmCompute(largeData);
renderUI(result);

// CORRECT: offload to Worker
const worker = new Worker('./wasm-worker.js', { type: 'module' });
worker.postMessage(largeData);
worker.onmessage = ({ data }) => renderUI(data.result);
```

### 3. Memory Leaks (C/C++ Wasm)

```javascript
// WRONG: ptr allocated but never freed
const ptr = instance.exports.malloc(1024);
const result = instance.exports.process(ptr);
// forgot: instance.exports.free(ptr);

// CORRECT
const ptr = instance.exports.malloc(1024);
try {
  const result = instance.exports.process(ptr);
  return result;
} finally {
  instance.exports.free(ptr);
}
```

Note: Rust with wasm-bindgen handles memory automatically via JS references.
C/C++ compiled with Emscripten requires manual `free()` calls.

### 4. Recompiling on Every Request (Server)

```rust
// WRONG: slow; compiles module on each request
async fn handle_request(wasm_bytes: &[u8]) {
    let module = Module::new(&engine, wasm_bytes)?; // expensive!
    // ...
}

// CORRECT: compile once, instantiate many times
lazy_static! {
    static ref MODULE: Module = Module::new(&ENGINE, include_bytes!("plugin.wasm")).unwrap();
}

async fn handle_request() {
    let instance = linker.instantiate(&mut store, &*MODULE)?; // cheap
}
```

---

## Production Checklist

- [ ] `wasm-opt -O3` run on output binary (Binaryen optimizer; wasm-pack does this automatically)
- [ ] Wasm binary served with `Content-Type: application/wasm` (enables streaming compile)
- [ ] Heavy work offloaded to Web Worker in browser (never block main thread)
- [ ] Module compiled once and cached; not recompiled per request/invocation
- [ ] Fuel/timeout configured for untrusted plugin code (Wasmtime)
- [ ] Allowed hosts/directories explicitly configured (WASI capability model)
- [ ] Panic hook set (`console_error_panic_hook`) for Rust debug builds
- [ ] Memory leaks audited for C/C++ modules (no wasm-bindgen safety net)
- [ ] Wasm module size measured; use `wasm-opt` + `wasm-strip` if > 500 KB
- [ ] CORS headers set if Wasm loaded cross-origin
- [ ] Component model (WASI P2) considered for polyglot composition use cases
