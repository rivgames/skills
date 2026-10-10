# Workflows

Each section adds only what `rive docs` does not say. Read the matching doc
topic first.

## Starting a project or animation

- Keep authored timelines small. A per-vertex geographic morph produced an
  18.9 MB scene.rml with 125k `KeyFrameDouble` elements: right for that one
  morph, slow to diff and to open in the editor. Prefer layout, data binding,
  `DataConverterInterpolator` and state machines, and reach for dense keyframes
  only when the motion is truly per-vertex.
- Key by property name (`<KeyedProperty property="rotation">`, 1.4+) rather
  than numeric `propertyKey`. Diffs stay readable.
- Shared artboards, view models and scripts across files belong in a published
  library (`rive library publish`, `libraries:` with a local `override:`), not
  in copied RML.

## A scripted scene inside a web host

- Build the signed file from a fresh temporary copy that holds only an
  explicit list of inputs, never from existing build output. Record each
  input's hash next to the signed bytes so "signed" can go stale.
- `onLoad` does not mean rendered. Signed exports can show a Rive splash
  (about 1.6 s) first. Have the script set a view-model boolean after its first
  real draw, and sample pixels for expected content before revealing the
  canvas.
- In `@rive-app/webgl2`, view-model `.on()` callbacks also fire for the host's
  own writes. Track host echoes explicitly instead of comparing against staged
  values.
- Native `--viewport` is in points: for a device-pixel-ratio 2 target of
  1440x1212, capture at 720x606.
- For pixel comparisons, freeze time- and pointer-driven motion with a data
  flag (for example a reduced-motion boolean). Otherwise frames differ run to
  run.
- Luau can call `fetch()` and `context:decodeFile()` (1.4+), and the host
  decides whether scripts may use the network. A host that plays untrusted
  uploads must check its runtime's network policy before trusting it.

## A game with tests and headless proof

A check that holds up, in order:

1. Prepare generated inputs (audio, RML), then `--verify --format=json`.
2. `inspect <dir> --json` and require an empty `problems` list. `inspect`
   takes `--json`, not `--format=json`.
3. `--test --format=json`: require `noTestsFound: false`, zero failures and
   the named cases you expect.
4. Scenario runs with `--pointer`/`--key`/`--touch` plus `--data-dump`, and
   assert on dumped values rather than on screenshots.
5. `--once` twice and compare sha256 to prove the export is byte-identical.

JSON goes to stdout and logs to stderr, so keep the two streams separate when
parsing.

## Editor round-trip and libraries

- `rive create <dir> --from-remote-file=<id>` starts from an editor file.
  `rive pull --yes` refreshes it. `rive push --yes --name="<label>"` writes a
  named revision.
- With a generator, its source stays canonical. Bring editor edits back into
  the generator before the next push, or the push erases them.

## CI and agent shells

- No TTY means pickers list their choices and exit, and anything that
  overwrites (`push`, `pull`, `library publish`, `--unpublish=web`) refuses
  without `--yes`. Pass `--project=` and `--team=` explicitly.
- Signing needs `rive login`, which an agent cannot complete. Exit 3 means not
  logged in and 7 means the service is unreachable. Report either; do not
  retry.

## Updating the CLI

1. Before touching the installed CLI, diff the old and new releases in `/tmp`.
   `https://releases.rive.app/cli/v<version>/manifest.json` names each
   platform's tarball and its sha256: download, check the hash, extract
   `rive`, `docs/` and `samples/`, then diff `--help` and the `docs/` tree. Do
   not use `rive switch` for this, because it changes the global selection.
2. Run `--verify` (and `--test`) for every project with both binaries. A
   project that fails only on the new one is a CLI change; bisect across the
   intermediate releases the same way.
3. Re-pin evidence deliberately. Exact-version asserts in check tools fail
   by design after an update.
4. Review this skill against the new release and bump `rive-cli-checked`.
