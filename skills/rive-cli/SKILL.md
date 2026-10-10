---
name: rive-cli
description: "Field lessons for projects built with the official Rive CLI (`rive`, rive.yaml + scene.rml + .luau): which loop proves what (native unsigned vs signed web), headless-capture blind spots, push/sign/watermark traps, test gates, CLI upgrades, troubleshooting and performance measurement. Use when building, testing, signing, pushing, upgrading or optimising a Rive CLI project."
compatibility: "Official Rive CLI on PATH (`rive --version`); its bundled docs via `rive docs`"
metadata:
  skill-version: "1.0.0"
  rive-cli-checked: "1.5.1"
  checked-on: "2026-10-09"
---

# Rive CLI

The CLI's own docs are version-matched and authoritative: `rive docs`,
`rive docs --search <text>`, `rive schema <Type>`, `rive samples`, and the
AGENTS.md that `rive create` writes. Read them for anything about RML, Luau,
flags or error messages. This skill holds only what they do not tell you:
lessons that cost rework in real projects.

## Rules that prevent rework

1. **Two loops, different proof.** Iterate natively and unsigned (live window,
   `--verify`, headless captures, `--test`). Only a signed build in the real web
   runtime proves the web path (host listeners, WebGL specifics, readiness).
   Signing calls Rive's service and is rate-limited: sign once at the end of a
   session, from a fresh copy holding only the listed inputs. If the service
   refuses, stop and report; never retry in a loop.
2. **Headless captures draw one frame.** `--advance` steps time without
   drawing. Work spread over several draws (chunked texture bakes, deferred GPU
   setup) comes out blank or partial. Use the live window's `screenshot <path>`,
   or a throwaway copy that does all the work in the first draw.
3. **Headless input skips some real paths.** `--data` lands before the first
   advance, so a script's `addListener` path for later host writes never runs.
   A headless `--pointer` click never gives focus, even with `canTouch`; seed
   focus with a `FocusActionTarget` in the entry state before testing `--key`.
4. **`rive push` overwrites the bound editor file.** It is for inspection,
   never a signing or publish path. Since 1.5.0 `push`, `pull` and
   `library publish` ask first, so agent and CI shells need `--yes`.
5. **Watermark: a binding is necessary, not proven sufficient.** Without
   `push.fileId` in rive.yaml every signed export is watermarked. The docs say
   binding removes it, yet a bound project has still shipped watermarked
   (probably the workspace plan). Check the signed bytes in a stock web
   runtime; never strip the splash or treat it as a failed login.
6. **`--test` passes with no tests.** It exits 0 with
   `"noTestsFound": true`. Gate on `noTestsFound == false` and on named cases.
7. **Record CLI versions; do not hard-gate everyday checks on them.** Store
   `rive --version` and the binary's sha256 with each piece of evidence. An
   exact-version assert in routine checks breaks every gate on each update.
   Keep exact pins only in tools that reproduce receipts, and run those
   against a cached binary (`~/.rive/versions/<v>/rive`) instead of the
   default `rive` on PATH.
8. **Copy shared scripts; never symlink them.** A symlinked `.luau` takes its
   target's path as its module name, so `require('name')` fails.
9. **Generated sources are canonical.** When RML or assets come from a
   generator, run the generator/prepare step before `--verify`. Edit the
   generator, not its output.

## Where to go next

| Task | Read |
|---|---|
| Starting a project or animation; scripted scenes for a web host; games with tests and headless proof; editor round-trip and libraries; CI or agent shells; updating the CLI | [references/workflows.md](references/workflows.md) |
| Something renders blank, a key does nothing, a require fails, push refuses, the web build is silent, numbers disagree | [references/troubleshooting.md](references/troubleshooting.md) |
| Measuring or improving frame time and scripted GPU work | [references/optimisation.md](references/optimisation.md) |

## Maintaining this skill

- `metadata.rive-cli-checked` is the newest CLI release this skill was
  reviewed against. Bump it after each review, even if nothing else changes.
- Add a lesson only when one line of direction reliably prevents redoing work
  and the CLI's docs or errors do not already say it. Delete lessons the CLI
  now documents or fixes.
- Bump `skill-version` (semver) on any content change.
