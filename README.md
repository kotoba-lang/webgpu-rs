# kotoba-lang/webgpu-rs

**CPU-side EDN render-IR domain logic in pure Clojure.** A
[kotoba-lang](https://github.com/kotoba-lang) capability library: EDN
render-IR parsing (globals/instances/lights/camera/environment/materials/
meshes), the `scene.edn` -> render-IR bridge, a deterministic procedural
demo scene, pure vertex/index mesh generation, minimal matrix math, and
the per-draw-call pure math (model matrices, view-projection, uniform/
instance buffer packing) a host renderer needs before it ever touches a
GPU API.

This is a `.cljc` port of `kami-webgpu-rs`, a Rust crate that lived in
`kami-engine` (`orgs/kotoba-lang/kami-engine`), the native (Rust/wgpu)
twin of the CLJS `kami.webgpu` render executor — both interpreted the
*same* EDN render-IR (ADR-0001/0040/0044), one driving the browser
WebGPU API, the other driving wgpu directly. `kami-webgpu-rs` is being
retired from `kami-engine`'s working tree as part of retiring the Rust
game-engine workspace in favor of pure-Clojure "kotoba" authority repos
(ADR-2607010000). This repo exists so that retirement loses no domain
knowledge — `kami-webgpu-rs`'s full source is still recoverable via `git
show HEAD:kami-webgpu-rs/src/lib.rs` in `kami-engine`'s history for as
long as that history exists, but the authority for its CPU-side domain
logic now lives here.

**Not** `kotoba-lang/webgpu` (an unrelated, already-existing foundational
repo — this port never touched it).

## Maturity

| | |
|---|---|
| Role | capability |
| Tests | 39 tests, 128 assertions across 8 namespaces, all green |
| Lint | `clj-kondo`, 0 errors, 0 warnings |
| Scope | EDN render-IR parsing + deterministic scene generation + per-draw-call pure math; **no GPU calls anywhere** (see "What was intentionally left unported") |

## What was ported

Namespace-for-module mapping from `kami-webgpu-rs/src/lib.rs`:

| `kami-webgpu-rs` (Rust) | `kotoba.webgpu-rs.*` (Clojure) namespace |
|---|---|
| `parse_ir`, `parse_render_ir`, `Globals`, `Instance`, `Light`, `Camera`, `Environment`, `Material`, `Mesh`, `RenderIr`, `mat4_from_flat` | `kotoba.webgpu-rs.render-ir` |
| `scene_to_ir` | `kotoba.webgpu-rs.scene` |
| `demo_city` | `kotoba.webgpu-rs.demo` |
| the `xorshift` closure (`scene_to_ir`/`demo_city`'s `let mut rnd = ...`) | `kotoba.webgpu-rs.rng` |
| `cube()`, plus test-only `geo_sphere`/`geo_cylinder` | `kotoba.webgpu-rs.geometry` |
| the `glam::Mat4` ops actually called (`from_translation`, `from_rotation_y`, `from_scale`, `*`, `perspective_rh`, `look_at_rh`, `orthographic_rh`, `transform_point3`) | `kotoba.webgpu-rs.mat4` |
| `model_mat` + the pure math inside `Renderer::draw` (camera/light view-projection, the `gf`/`idata` float packing) | `kotoba.webgpu-rs.frame` |
| `MAX_INST`, buffer sizes, shader entry points, formats, vertex-attribute layout, the `light_a..d` tuning vec4s, `align256` | `kotoba.webgpu-rs.config` |
| `kami_scene`'s `num`/`vec3`/`ident` tolerant-default helpers (the narrow subset kami-webgpu-rs itself called) | `kotoba.webgpu-rs.edn` |

`kotoba.webgpu-rs` is the top-level overview namespace.

### Render-IR parsing

```clojure
(require '[kotoba.webgpu-rs.render-ir :as ir])

(ir/parse-ir "{:globals {:sky {:horizon [0.7 0.8 0.9]}}
               :instances [{:pos [0 0 0] :color [1 0 0] :size [2 5]}]}")
;=> [{:horizon [0.7 0.8 0.9] :sun-dir [...] :sun [...] :eye nil :target nil}
;    [{:pos [0.0 0.0 0.0] :color [1.0 0.0 0.0] :size [2.0 5.0] :yaw 0.0
;      :metallic 0.0 :roughness 0.65 :emissive 0.0}]]

(ir/parse-render-ir "{:instances []
                       :lights [{:kind :point :color [1 0.5 0.2] :pos [2 3 0]}]
                       :materials [{:id :skin :model :mtoon}]}")
;=> {:globals {...} :instances [] :lights [{...}] :camera nil :env {...}
;    :materials [{...}] :meshes []}
```

Direct 1:1 port — parsing is pure EDN-in/data-out, so there's no
GPU-vs-CPU split to make here. Every tolerant-default rule from the Rust
is preserved exactly, including a subtlety easy to get wrong: a
present-but-wrong-typed field (e.g. `:color 5`, a number instead of a
vector) falls back to the *field's own default* (`opt_vec3(...).unwrap_or(default)`
semantics), not to zeros — `kotoba.webgpu-rs.edn/opt-vec3` exists
specifically to distinguish "absent/wrong-shaped" from "present, coerce
to zero", and `render-ir_test.cljc`'s
`light-color-and-dir-default-when-non-vector`/
`material-base-and-shade-default-when-non-vector` tests pin that down
(this monorepo caught and fixed the same distinction while writing this
port — see git history if `repos.edn`/PR context is needed).

Where Rust used an enum with a `by_name` string-match fallback
(`LightKind`, `MaterialModel`, `AlphaMode`), this port uses a keyword
plus a `by-name` fn with the identical fallback rule — idiomatic
Clojure, no information lost. `NodeId`/`ElementId`-style newtype
wrappers don't exist here because kami-webgpu-rs's Rust never had any
(unlike e.g. `kami-cae`/`kotoba-lang/fea`).

### Scene bridge + procedural demo

```clojure
(require '[kotoba.webgpu-rs.scene :as scene]
         '[kotoba.webgpu-rs.demo :as demo])

(scene/scene->ir my-kami-clj-scene-edn-string) ;=> [globals instances]
(demo/demo-city)                                ;=> [globals instances] (170-prop royale scatter + ball)
```

Both are deterministic: the exact 32-bit xorshift PRNG (seed
`2654435769`, ported bit-for-bit including Rust `u32` wraparound
semantics) that the web (CLJS `game.cljs`) uses, so a host rendering the
same EDN gets the same procedurally-scattered world.
`kotoba.webgpu-rs.rng-test/scatter-rng-matches-the-web` checks the first
five draws against values captured from the Rust originally (which
itself asserted parity against the CLJS sequence).

Porting `scene_to_ir`'s scatter loop surfaced two real bugs in an
earlier draft of this port, both now fixed and covered by tests: (1) the
tree branch must consume **zero** extra RNG draws (tree
w/h/metallic/roughness/color are static config, not randomized) — an
earlier draft accidentally drew and discarded one extra value per tree,
silently desyncing the RNG stream from the Rust; (2) the building-index
draw must be scaled by the building-table length before truncating
(`(rnd() * buildings.len() as f32) as usize % buildings.len()`) — an
earlier draft used the raw `[0,1)` draw directly, which always selected
building index 0. Both are now exact draw-for-draw ports.

### Geometry

```clojure
(require '[kotoba.webgpu-rs.geometry :as geo])

(geo/cube)              ;=> [verts indices] — 24 verts (pos3+normal3) / 36 indices
(geo/sphere 1.0 4 6)     ;=> UV sphere
(geo/cylinder 1.0 2.0 6) ;=> capped cylinder
```

`cube` is kami-webgpu-rs's only `src/lib.rs`-level mesh generator.
`sphere`/`cylinder` port `geo_sphere`/`geo_cylinder`, which lived in
kami-webgpu-rs's `#[cfg(test)]` module as local re-implementations of
`kami.webgpu.geometry` (the cljc canonical, in the separate
`kotoba-lang/webgpu` repo — **not** touched by this port) used there only
to assert cross-platform parity against fixtures co-located in the old
Cargo workspace. The generation algorithms are pure geometry logic, not
test-only by nature, so they're ported here as first-class functions;
the golden-fixture *comparison itself* (which needed `kami-webgpu`'s
`fixtures/` directory co-located) is out of scope — `geometry_test.cljc`
covers vertex/index-count structural correctness instead.

### Matrix math + per-frame math

```clojure
(require '[kotoba.webgpu-rs.mat4 :as m]
         '[kotoba.webgpu-rs.frame :as frame])

(frame/model-mat {:pos [10.0 0.0 20.0] :size [2.0 4.0] :yaw 0.0 ...})
;=> a 16-float column-major mat4 (translate * rotate-y * scale)

(frame/pack-globals globals instances w h) ;=> 60 floats (the `G` WGSL uniform struct)
(frame/pack-instances instances)           ;=> 24 floats/instance, capped at MAX_INST
```

`kotoba.webgpu-rs.mat4` ports the handful of `glam::Mat4` operations
kami-webgpu-rs actually called (not a general linear-algebra library).
`kotoba.webgpu-rs.frame` extracts the *pure* CPU-side math that lived
inline in `Renderer::draw` — instance model matrices, the main camera's
and the shadow map's view-projection matrices (derived from the instance
centroid when `Globals` doesn't set an explicit eye/target), and the
exact flat-float layout of the two buffers `draw` uploaded to the GPU
each frame. None of it touches a `wgpu::*` type; only the actual
`queue.write_buffer`/`draw_indexed` calls are adapter-only and stay
unported. `kotoba.webgpu-rs.config` holds the buffer-layout and
light-tuning constants (`light_a..d`, `MAX_INST`, `align256`, shader
entry-point names, vertex-attribute offsets) this packing code needs, as
plain data.

## What was intentionally left unported, and why

kami-webgpu-rs is genuinely GPU/rendering-bootstrap code, unlike e.g.
`kami-cae`/`kotoba-lang/fea` (which had zero GPU surface). What's left
unported is real GPU-API surface, not a porting shortcut:

- **`Renderer`** (`device`/`queue`/pipelines/bind-groups/buffers/
  `depth_view`/`shadow_view`) and its `new`/`resize`/`draw`/`device`/
  `queue` methods — wgpu resource creation (textures, samplers, bind
  group layouts, render pipelines) and the actual `RenderPass`/
  `draw_indexed`/`set_vertex_buffer` calls. `kotoba.webgpu-rs.frame`
  ported every *pure* computation `draw` did before it touched wgpu
  (matrices, buffer contents); only the upload/draw calls stay adapter.
- **`render_async`/`render_to_pixels`/`render`** — requests a GPU
  adapter/device, drives the headless offscreen-texture + pixel-readback
  path (`align256` is ported to `kotoba.webgpu-rs.config`; the actual
  `copy_texture_to_buffer`/`map_async`/`poll` calls are not, since they
  have no meaning without a real wgpu device).
- **`lit_shader.wgsl` / `shadow_shader.wgsl`** — WGSL shader source,
  compiled by wgpu. Not reproduced verbatim anywhere in this repo (their
  entry-point names, `"vs"`/`"fs"`, are captured as data in
  `kotoba.webgpu-rs.config` since the vertex-buffer layout code needs
  them); the shader bodies themselves are GPU-language source with no
  CPU-side counterpart to port.
- **`examples/live.rs`** (a `winit` window + event loop rendering
  `demo_city` live) and **`examples/render_png.rs`** (calls `demo_city`
  + `render`, writes a PNG via `image::save_buffer`) — both 100%
  host-adapter (window/event-loop or file I/O) wrapped around functions
  that are otherwise fully ported (`demo_city` -> `demo/demo-city`); the
  wrapping itself has nothing pure left to extract.

Representation-only changes (no information lost):

- Rust's `Vec<f32>`/array buffer layouts -> plain Clojure vectors of
  doubles; `[[f32;4];4]` (joint palettes) -> nested 4-vectors, matching
  shape 1:1.
- `serde_json` (used only by kami-webgpu-rs's `#[cfg(test)]` golden-
  fixture loader, itself unported per the geometry section above) -> not
  needed; EDN is Clojure's native serialization.
- `Option<T>` fields (`Globals.eye`/`.target`, `Camera` fallbacks) ->
  `nil`, with `or`-chains replacing `.unwrap_or(...)`.

## Tests

Parity tests for kami-webgpu-rs's `#[cfg(test)]` module and
`render_ir_ext_tests` module, using the same EDN fixtures and expected
values as the Rust originals, plus new coverage for logic that had no
isolated Rust unit test of its own (it lived inline in `Renderer::draw`,
which needs a real GPU to test at all in Rust — `frame_test.cljc` covers
the pure math extracted from it instead):

- `rng_test` <- the `scatter_rng_matches_the_web` xorshift parity assertion.
- `geometry_test` <- `cube_mesh_shape` (+ structural coverage for the
  ported `sphere`/`cylinder` generators, whose golden-fixture comparison
  is out of scope — see above).
- `render_ir_test` <- all of `mod tests`' `parse_ir`/`parse_render_ir`
  cases and all of `mod render_ir_ext_tests`, plus two new tests
  (`light-color-and-dir-default-when-non-vector`,
  `material-base-and-shade-default-when-non-vector`) pinning down the
  `opt_vec3`-vs-`contains?` default-semantics distinction noted above.
- `scene_test` <- all of `scene_to_ir_*`'s inline tests.
- `demo_test` <- new coverage for `demo_city` (no isolated Rust test
  existed; it was only exercised via the two examples).
- `mat4_test` <- `model_mat_translates_lifts_and_scales`, plus new
  coverage for `look-at-rh`/`perspective-rh` (untested in isolation in
  the Rust, since `Renderer::draw` only ever composed them together).
- `frame_test` <- `model_mat_translates_lifts_and_scales` (again, at the
  `frame/model-mat` call site), plus new coverage for the centroid/
  uniform/instance-buffer packing logic.
- `config_test` <- `align256_rounds_up_to_256`.

```bash
clojure -M:test
clojure -M:lint
```

## License

Apache License 2.0.
