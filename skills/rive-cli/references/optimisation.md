# Optimisation

## Measure the right thing

- `--bench=<frames>` times the artboard at its **authored** size and ignores
  `--viewport`. To bench another size, change the artboard's width and height
  in a throwaway copy.
- Since 1.4 `--bench` runs the release-baked scripts a publish ships. Numbers
  from either side of 1.4 are not comparable.
- Bench reports CPU-side advance/render time and wasm memory, not GPU frame
  time, `init` or a module's top level. GPU cost and real frame rate come from
  the browser runtime on target hardware. Record the runtime version, CLI
  version and file hash with every baseline.

## Scripted rendering

- `init` runs more than once and only one instance draws. Create GPU
  pipelines, buffers and textures lazily, on the first `draw` with a non-zero
  size.
- Redraw only on change: let `advance` return `false` when idle, and wake it
  with view-model `addListener` plus `context:markNeedsUpdate()`. Redrawing
  only the damaged region of a full-screen target cut redrawn pixels by about
  92% in one project.
- `context:gpuTarget()` (1.3+) renders straight into Rive's own target and
  skips a full-size `GPUCanvas` plus a `drawImage` composite. It returns nil
  on hosts that cannot expose one, so keep the canvas path as a fallback and
  confirm which path the web runtime actually takes.
- `MeshInstances` with `renderer:drawImageMeshInstanced` (1.3+) draws many
  copies of one mesh in a single call.

## Script time budgets

Seen in Rive's showcase games (rive.app/games), November 2026, and in the
web runtime source.

- **200 ms per call, `init` included.** Rive's scripting security docs give
  each Luau and AnimaScript call that budget (AnimaScript background jobs
  and workers excepted), and native runtimes enforce it
  (`rive docs gpu-performance`). Keep each call well under it, AnimaScript
  too while the browser leaves your build unbounded, so a later fuel build
  does not cut it off on slow phones. A heavy `init` in a fuel-built
  AnimaScript module was stopped at 200 ms in 6 of 9 headless web loads,
  on web runtimes 2.45.0 and 2.49.1 and at phone, tablet and desktop sizes.
  A phone page stayed blank. Count `init`'s cost each time it runs.
- **1 s for a Luau module's top level, from web runtime 2.48.0.** Code
  outside functions is stopped with `execution exceeded 1.0 second timeout`,
  and every script that requires that module then fails with `require could
  not find a script named …`. Web runtimes 2.45.0 and 2.47.1 set no such
  limit, so a file that passes there can fail after a runtime upgrade.
- **Precompute, then slice.** Build heavy data in the generator or prepare
  step. Spread what remains over the first advances with a fixed per-frame
  time slice (in Luau, checked with `os.clock()`). A slice tuned on a
  desktop can still overrun: one bake stage that usually took 22 ms took
  515 ms under tablet emulation. A spread bake comes out blank in headless
  captures; see the troubleshooting table.
- **Time setup in an `--optimize` copy, not the live window.** The live
  window's debug builds run slower than what ships: AnimaScript's run tight
  loops several times slower (Rive's build-settings docs), and Luau compiles
  at O1, not O2. In a throwaway copy run with `--optimize`, wrap `init` or a
  Luau module body in `os.clock()` calls (`os.clock` is Luau's). Timing
  prints change the signed bytes, so keep them out of the build you sign.
- **Prove the budget under CPU throttling.** Load the signed file in a
  browser at 6x and 20x (CDP `Emulation.setCPUThrottlingRate`), on the
  runtime you ship and on the next one you plan to move to. Fail on console
  lines containing `execution exceeded` or `require could not find`, or when
  the script's first-draw flag never sets. 6x was enough to reproduce the 1 s
  module stop on 2.49.3.
- **Prove Luau's 200 ms natively or by timing.** Rive's docs say runtimes
  interrupt a Luau call over 200 ms, but no such stop was seen on web runtime
  2.45.0, even with 7-26 s main-thread stalls at 60x throttling. A clean
  browser run does not prove the budget.

### AnimaScript in the browser

How the browser bounds an AnimaScript (`scripting: wasm`) call depends on how
the module was built. The `.riv` embeds the wasm module, and its export names
show the build:

| Exports | Build | In the browser |
|---|---|---|
| `__riveFuelCheck` | Fuel-metered | The module stops itself at the budget. Web runtimes 2.45.0 and 2.49.1 stopped a long fuel-built `init` with `wasm call trapped in host_obj_user_init`. Web runtime 2.44.1 has no fuel support. |
| `__riveHostBudget` only | Watchdog | Unbounded. The runtime logs `wasm script module expects a watchdog to bound its calls, which the browser does not have: a runaway script will hang the page`. A runaway loop freezes the tab, and page timers cannot fire to catch it. |
| neither | `dangerouslyFast` | No budget on any host. |

- CLI 1.7.0 publishes watchdog builds (or `dangerouslyFast`) and has no fuel
  option. Its gpu-performance doc says the browser has no limit for wasm
  scripts; that holds only for those two builds. 8 of the 9 AnimaScript
  showcase games were watchdog builds.
- Rive's current build-settings docs add `wasmFuel: true` (off by default;
  a few percent slower in loops). Check `rive docs` for your CLI before
  relying on it.
- A host that plays AnimaScript files from outside its team should accept
  only fuel builds, checked from the exports. Rive's scripting security page
  gives the same advice.

## File size

Seen in Rive's showcase games, November 2026 (0.4 to 72 MB each).

- **Measure bytes per asset type, raw and compressed, before signing.**
  Re-encoding later means signing again. Large files are not always media:
  several showcase files were mostly meshes, raw textures or data blobs.
- **Subset fonts in the prepare step** (for example with fontTools). The CLI
  embeds font bytes as given (CLI 1.7.0 docs). Its sample `Inter.ttf` is
  877 KB with 2,933 glyphs, and four showcase games shipped it whole; a
  printable-ASCII cut is 84 KB. Keep every character the file can show,
  player-entered text included.
- **Choose each sound's format by its size after the compression your server
  applies** (brotli or gzip). MP3 suits music and long sounds. Short
  synthesized WAV effects can shrink to almost nothing, where MP3 would be
  larger: in one project, 267 KB of short WAV effects became 1.3 KB with
  brotli.
- **Resample high-rate audio before import.** One showcase game spent 29 MB
  on 107 s of sound, mostly 96 or 192 kHz 24-bit stereo FLAC.
- **Ship images at the size they are displayed, as WebP or JPEG.** One
  showcase game carried 36.7 MB of PNG, 33 of its 35 images over 1024 px. The
  CLI strips ImageAsset format and quality on export (CLI 1.7.0 docs), so
  resize and encode in the prepare step.
- **Check the served `Content-Encoding` and transfer size.** Some CDNs skip
  large files: CloudFront compresses only objects of 1,000 to 10,000,000
  bytes, so the biggest showcase files arrived uncompressed. If the header
  is missing, change how the file is served (a content type the CDN
  compresses, or a host that compresses files that size), then check that
  the transfer size has dropped below the raw size.
- **Leave `shaderOutputs` at its default of all five targets.** A runtime
  reads only its own backend's target, and a target left out does not load
  on that backend. The cost lands on the web: WebGL2 reads only GLSL ES 300,
  so 56-85% of the raw shader bytes in the showcase files are never read
  there. Compression shrinks the total (1.67 MB of shaders became 0.32 MB
  with gzip), but the unread share stays about 80%. Trim only for a file
  that will only ever run on one backend, after measuring that share of the
  compressed download. The value is fixed when you sign.
