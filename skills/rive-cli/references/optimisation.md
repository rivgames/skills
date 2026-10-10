# Optimisation

## Measure the right thing

- `--bench=<frames>` times the artboard at its **authored** size and ignores
  `--viewport`. To bench another size, change the artboard's width and height
  in a throwaway copy.
- Since 1.4 `--bench` runs the release-baked scripts a publish ships. Numbers
  from either side of 1.4 are not comparable.
- Bench reports CPU-side advance/render time and wasm memory, not GPU frame
  time. GPU cost and real frame rate come from the browser runtime on target
  hardware. Record the runtime version, CLI version and file hash with every
  baseline.

## Scripted rendering

- `init` runs more than once and only one instance draws. Create GPU
  pipelines, buffers and textures lazily, on the first `draw` with a non-zero
  size.
- Redraw only on change: let `advance` return `false` when idle, and wake it
  with view-model `addListener` plus `context:markNeedsUpdate()`. Redrawing
  only the damaged region of a full-screen target cut redrawn pixels by about
  92% in one project.
- Splitting heavy one-off work (texture bakes) across draws keeps the first
  frame short, but headless captures then come out blank. Plan the capture
  workaround from the troubleshooting table.
- `context:gpuTarget()` (1.3+) renders straight into Rive's own target and
  skips a full-size `GPUCanvas` plus a `drawImage` composite. It returns nil
  on hosts that cannot expose one, so keep the canvas path as a fallback and
  confirm which path the web runtime actually takes.
- `MeshInstances` with `renderer:drawImageMeshInstanced` (1.3+) draws many
  copies of one mesh in a single call.
