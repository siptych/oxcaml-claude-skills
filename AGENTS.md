# AGENTS.md

Claude Code skills and a gotchas log for OxCaml and the Jane Street library stack,
installable as a plugin (`.claude-plugin/plugin.json`). The tree is exported from an
upstream workspace; each export replaces it.

| Path | Holds |
| --- | --- |
| `skills/` | One directory per skill, each with `SKILL.md`; see `skills/AGENTS.md` |
| `skills/*/examples/` | Dune projects that prove the modes and performance claims |
| `scripts/` | `check-obj-magic.sh` and its test; see `scripts/AGENTS.md` |
| `GOTCHAS.md` | Short lessons grouped by topic |
| `.github/workflows/ci.yml` | shellcheck, the script test, and both example projects on the `5.2.0+ox` switch |

## Checks

- Scripts: `shellcheck scripts/*.sh && scripts/test-check-obj-magic.sh`.
- Examples: in `skills/<skill>/examples`, `opam install . --deps-only --with-test`, then
  `dune build @all` and `dune test`, on the `5.2.0+ox` switch.
- Markdown edits need no build.

## Rules

- A claim about compiler behaviour or allocation goes into an `examples/` project that
  CI builds, with the error text or the word count pinned, not only into prose.
- Pin zero words, or a range. A non-zero count differs between macOS arm64 and Linux
  amd64.
- Keep entries generic: no project, company or person names, no absolute paths.
- A new skill gets a row in the README "Skills" table in the same commit.
- Never install an opam package without `opam install --dry-run` first; abort if the
  plan touches `ocaml-variants`, `core`, `base`, `async` or `ppx_jane`.
