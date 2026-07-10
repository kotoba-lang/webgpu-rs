# kotoba-lang/webgpu-rs

**SSoT for `kotoba.webgpu-rs.*`** — pure-CLJC port of the retired Rust
`kami-webgpu-rs` CPU-side domain (render-IR parse, mat4, geometry, demo scene, …).

This is **not** the browser WebGPU executor (`kotoba-lang/webgpu` / `kami.webgpu`).
ADR-2607102200 addendum 8 restored this package as the home after a temporary
merge into webgpu.

## Test

```sh
clojure -M:test
```
