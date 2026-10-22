# riv.games agent skills

Agent skills ([agentskills.io](https://agentskills.io) `SKILL.md` format) for
building with Rive.

| Skill | For | Checked against |
|---|---|---|
| [`rive-cli`](skills/rive-cli/SKILL.md) | Projects built with the official Rive CLI (`rive`): which loop proves what, headless-capture blind spots, push/sign/watermark traps, test gates, CLI upgrades, troubleshooting, performance measurement | Rive CLI 1.7.0 |

## Install

Into the current project, pinned to a release tag:

```sh
npx skills add rivgames/skills#rive-cli-v1.1.0 --skill rive-cli
```

Update a project later with `npx skills update rive-cli`, or re-run `add`
with a newer tag.

## Versioning

Each skill's frontmatter carries:

- `metadata.skill-version`: semver for the skill's content.
- `metadata.rive-cli-checked`: the newest Rive CLI release the skill was
  reviewed against. A bump means it was checked, not necessarily changed.

Releases are tagged `<skill>-v<skill-version>`.

## Licence

MIT, copyright Riv Games, Inc. See [LICENSE](LICENSE).
