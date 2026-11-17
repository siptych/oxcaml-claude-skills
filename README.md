# oxcaml-claude-skills

Claude Code skills, a gotchas log and a small CI check for writing
[OxCaml](https://oxcaml.org) with the Jane Street library stack (Base, Core, Async,
ppx_jane). Everything here was collected while building a multi-package workspace on
the `5.2.0+ox` switch; the claims about compiler behaviour and allocation are backed by
dune projects that CI builds and tests.

## Install

As a Claude Code plugin:

```sh
claude plugin marketplace add siptych/oxcaml-claude-skills
claude plugin install oxcaml-claude-skills@oxcaml-claude-skills
```

Or copy a single skill directory into `~/.claude/skills/` (user-wide) or
`.claude/skills/` (one project).

## Skills

| Skill | What it covers |
|-------|----------------|
| `skills/oxcaml-setup` | Creating and maintaining an OxCaml opam switch locally and in GitHub Actions, depexts, package pitfalls, inline tests under the ox `ppx_expect` |
| `skills/oxcaml-modes` | Verified `local_`/`exclave_`/`global_`/`@ local`, `@ unique` and `borrow_` patterns with the compiler's error texts and the fix for each |
| `skills/oxcaml-performance` | A checklist to follow before optimising, then verified `or_null`, unboxed numbers and tuples, `float#` record fields, `let mutable`, `[@zero_alloc]`, inlining attributes, `-O3` and zero-allocation parsing, with measured words per call |
| `skills/async-http-websocket` | cohttp-async and async_websocket servers and clients, same-port websocket upgrades, logging in tests |
| `skills/janestreet-ocaml-style` | Writing Base/Core/Async code the Jane Street way, ppx_jane, expect tests, and a workflow for agents (signatures first, failing test before the fix) |

Each skill opens with outcomes and short exercises for checking a fresh session against
it.

## Gotchas

[`GOTCHAS.md`](GOTCHAS.md) holds about 160 short entries grouped by topic: opam and the
switch, dune workspaces, CI and containers, Core shadowing, ppx derivers, Async, clocks,
testing, modes and layouts, measurement, Bonsai, bin_prot, Parquet and DuckDB. Point an
agent at it before it starts on an OxCaml code base, or grep it for the error in front
of you.

## Examples

`skills/oxcaml-modes/examples` and `skills/oxcaml-performance/examples` are standalone
dune projects. `src/` holds the patterns, `test/` pins behaviour and words per call
with expect tests, and `rejected/` holds programs the compiler must refuse, each diffed
against the expected error text.

```sh
cd skills/oxcaml-modes/examples
opam install . --deps-only --with-test
dune build @all && dune test
```

## Scripts

`scripts/check-obj-magic.sh [root]` fails when `Obj.magic`, `Obj.repr` or `Obj.obj`
appears in a `.ml`/`.mli` file (tracked or untracked, `_build*/` and `_opam/` skipped)
that is not listed in `<root>/.obj-magic-allowlist`. The allowlist takes one path or
shell glob per line, relative to the root; `#` starts a comment, so the reason for each
entry can sit next to it.

To enforce it in another repository, add a job (no OCaml needed, it runs in seconds):

```yaml
  obj-magic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          curl -fsSL https://raw.githubusercontent.com/siptych/oxcaml-claude-skills/main/scripts/check-obj-magic.sh \
            | bash -s -- .
```

`scripts/test-check-obj-magic.sh` is its test.

## Versions

Verified on `5.2.0+ox` with the Jane Street v0.18 preview libraries from
[oxcaml/opam-repository](https://github.com/oxcaml/opam-repository), on macOS arm64 and
Linux amd64. OxCaml moves quickly: when a pattern here stops compiling, the examples'
CI run is the first place it shows.

## Contributing

This repository is exported from a private workspace where the skills are used daily,
and each export replaces the tree. Issues and pull requests are welcome; an accepted
change is applied upstream and arrives here with the next export, so it lands as a new
commit rather than a merge of the pull request.

## License

MIT, see [LICENSE](LICENSE).
