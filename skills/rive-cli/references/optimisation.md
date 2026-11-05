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
  every Luau and AnimaScript call that budget, and native runtimes enforce it
  (`rive docs gpu-performance`). Keep each call well under it: a heavy
  `init` in a fuel-built AnimaScript module was stopped at 200 ms in 6 of 9
  headless web loads, all under phone or tablet emulation, and the page
  stayed blank. Desktop loads passed. Count `init`'s cost each time it runs.
- **1 s for a Luau module's top level, from web runtime 2.48.0.** Code
  outside functions is stopped with `execution exceeded 1.0 second timeout`,
  and every script that requires that module then fails with `require could
  not find a script named …`. Web runtimes 2.45.0 and 2.47.1 set no such
  limit, so a file that passes there can fail after a runtime upgrade.
- **Precompute, then slice.** Build heavy data in the generator or prepare
  step. Spread what remains over the first advances with a fixed per-frame
  time slice checked with `os.clock()`. A slice tuned on a desktop can still
  overrun: one bake stage that usually took 22 ms took 515 ms under tablet
  emulation. A spread bake comes out blank in headless captures; see the
  troubleshooting table.
- **Time setup in dev builds only.** `os.clock()` around a module body or
  `init` gives its cost. Timing prints change the signed bytes, so keep them
  out of the release build.
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
- Keep AnimaScript calls inside the 200 ms budget even while your build is
  unbounded in the browser, so a later fuel build does not cut them off on
  slow phones.
- A host that plays AnimaScript files from outside its team should accept
  only fuel builds, checked from the exports. Rive's scripting security page
  gives the same advice.

