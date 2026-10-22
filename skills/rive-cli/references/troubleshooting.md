# Troubleshooting

Symptoms the CLI does not explain. For anything else, the error text and
`rive docs gotchas` come first.

| Symptom | Cause | Fix |
|---|---|---|
| A headless `--screenshot` is blank or one flat colour, but the live window looks right | The work needs several draws (chunked bake, deferred GPU setup), and a headless capture draws once. `--advance` does not draw. | Use the live window's `screenshot <path>`, or a throwaway copy that does all the work in the first draw. |
| `key: nothing is focused` after a `--pointer` click | A headless click never gives focus, even with `canTouch`. | Put a `FocusActionTarget` in the entry state, `--advance` a frame or two, then send `--key`. |
| Native `--data` captures look right, but the browser ignores host writes | `--data` lands before the first advance, so the path for later writes never ran. Once `advance` returns `false` Rive stops calling it, so it cannot poll. | Register `addListener` on each property you draw from and call `context:markNeedsUpdate()`. Prove it in the signed browser run. |
| `Unknown require: name` for a script that exists | The script is a symlink, so its module name is the target's path. | Copy the file. For declaration order and `isModule`, see `rive docs luau/protocols`. |
| `Key 'image' not found in external type 'GPUTarget'` (1.3+) | `GPUCanvas` became `GPUTarget & {…}`, and the checker narrows a `self.canvas` field read to `GPUTarget`. | Read it once into a local at the top of `draw` (`local canvas = self.canvas`), or cast with `(self.canvas :: GPUCanvas).image`. |
| `push`, `pull` or `library publish` stops with "Rerun with --yes" | 1.5.0+ asks before overwriting, and an agent shell cannot answer. | Add `--yes` after confirming the target file is the right one. |
| A scripted `.riv` plays nothing in a web page | It was built unsigned (`--once`, watch, or a copied `build/` file), or the page reads frames during the splash. | Build with `--publish`, and gate the reveal on a script-set "drawn" flag plus a pixel check. |
| The signed file is still watermarked after binding | The publish service decides, probably by workspace plan; this is not a CLI fault. | Record it, hide the canvas during the splash, and do not strip the watermark. |
| `--test` is green but nothing ran | No script returns the Tests protocol (`noTestsFound: true`). | Gate on `noTestsFound == false` and on named cases. |
| Long cases fail with `the case ran past its 2000ms budget` after a CLI update | Since 1.3.0 every case, and the declaration of the cases, has a fixed 2000 ms budget. It cannot be configured, and `--optimize` does not lift it. | Split cases per item and shorten them, with margin (about 1 s on the slowest host). Move match-length simulations into a soak project that steps them across frames in `advance()` and reports through the console, and gate on a fail-closed parser of that output. |
| `--verify` fails on a fresh clone with missing asset files | Generated inputs (audio, RML) have not been produced yet. | Run the project's prepare/generator step first. |
| The live window ignores terminal commands, or `pkill -f rive` kills your own shell | stdin is closed, or `-f` matched the calling shell's command line. | Keep stdin open (`tail -f /dev/null \| rive <dir>`, or a pty) and stop it with `pkill -x rive`. |
| Captures differ from the export on Linux/Mesa | A capture-only `LD_PRELOAD` shim (GLSL fixes, frame grabs) leaked into the build. | Unset preloads for `--once`, `--publish` and `--verify`, and set them only for capture runs. |
