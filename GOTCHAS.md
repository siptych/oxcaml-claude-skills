# Gotchas

Lessons from building a multi-package OxCaml workspace on the `5.2.0+ox` switch with
Base, Core and Async, grouped by topic. Every entry was hit in practice in October 2026
on the versions it names; check it again after an upgrade. Several come from running
many coding agents in one dune workspace at once. Stable entries are promoted into
`skills/`.

## opam and switch setup

- **Name**: the project is `OxCaml` (oxcaml.org), not `0xCaml`. The opam switch
  is `5.2.0+ox` from `git+https://github.com/oxcaml/opam-repository.git`.
- **depext on macOS**: `opam switch create 5.2.0+ox` needs `autoconf`
  (Homebrew). Non-interactive opam aborts on missing depexts with exit 10
  instead of installing them; run `brew install autoconf` first, or pass
  `--confirm-level=unsafe-yes`.
- **Build time**: the OxCaml compiler is built from source; budget 15-30 min
  on an M-series Mac. Start the switch build in the background before
  scaffolding anything.
- Opam pins warn about missing `maintainer`/`authors` fields; harmless but
  add them to `dune-project` to keep logs clean.
- `opam install async_websocket` on the ox switch can pick v0.13.0, an empty
  stub with no modules. Pin the ox preview version
  (`async_websocket.v0.18~preview.130.106+341`); it pulls cryptokit and zarith.
- `async_ssl` does not compile against Homebrew `openssl@4` (`ASN1_STRING_data`
  removed), and even when built against `openssl@3` the link picks up
  `/opt/homebrew/lib/libssl` via zarith's `-L`. Workaround: write an
  `openssl.pc` under `~/.opam/5.2.0+ox/share/openssl3-pkgconfig/` whose `Libs`
  names the openssl@3 libraries by absolute path (first entry must start with
  `-` or dune rejects it), then `PKG_CONFIG_PATH=<dir> opam reinstall async_ssl`.
  Without async_ssl, conduit-async silently has no TLS.
- **Package cycle.** When a repo's engine needs library B and B needs the same repo's
  primitives, one opam package cannot hold both halves (opam rejects the cycle, even when
  dune would resolve it per library). Ship two packages from one repo (`core`,
  `core-engine`). Its own CI pins the checkout first (`opam pin add core . --no-action`),
  then pins B from GitHub, then `opam install ./core-engine.opam --deps-only --with-test`.
  Consumers pin the same URL twice, once per package name, in dependency order.
- The shared opam switch can be broken mid-session by another agent's `opam install`
  (here `core_unix` and `async` vanished and `core`'s sub-library directories were
  emptied while `core_kernel` rebuilt). Install packages only from the main session,
  never from parallel agents.
- Installing `mdx` (or any package whose solution pulls a non-preview `core`) on the ox switch makes opam downgrade the entire switch including the compiler; interrupted, it leaves `core` half-removed. Recovery: `opam reinstall core`, then `opam switch import ~/.opam/5.2.0+ox/.opam-switch/backup/state-<before>.export` with the openssl3 `PKG_CONFIG_PATH`. Always `opam install --dry-run` first and abort if the plan touches `ocaml-variants`, `core`, `base`, `async` or `ppx_jane`.
- `opam pin add --no-action` to a new commit does not rebuild an installed package; `opam reinstall <pkg>` is needed. A composite action shows as one step in the run summary, so per-step timings come from job log timestamps.
- `digestif` 1.3.0 does not build on the ox switch (local-mode mismatch in `By.blit`), which blocks `mirage-crypto-ec`; the older mirage-crypto-ec 1.x would downgrade mirage-crypto. For P-256/P-521 ECDSA verification write it over zarith (verification uses only public data, so constant time is not needed).

## dune and workspaces

- A `_build` directory produced by one switch confuses the other; `dune clean`
  when swapping switches.
- A `dune-workspace` + `dune-project` at the parent directory of all checkouts
  makes dune resolve sibling repos without opam pins; `dune build @all
  @runtest` inside a sub-repo then builds via the parent root. `opam pin` with
  a local path only sees committed HEAD, not the working tree.
- Across projects in one dune workspace only public library names resolve:
  `(libraries my-lib)`, not `my_lib`.
- `dune fmt` re-wraps doc comments; apply text edits before formatting.
- Multiple agents in one dune workspace share the `_build` lock; a run can
  queue behind another and return an empty output file. Re-run rather than
  trust an empty result.
- A finished agent can leave a `dune build` process alive holding `_build/.lock`
  for everyone in the workspace. Parallel agents should each set
  `DUNE_BUILD_DIR=/abs/path/_build-<repo>` (dune honours it for build, test,
  fmt and promote) so locks never collide; add `_build-*/` to `.gitignore` at
  the workspace root.
- A `dune build` can hold `_build/.lock` with no child processes for 30+
  minutes; wrap agent builds in `timeout`.
- `dune build ./engine` from inside a repo in the workspace builds just that directory;
  handy while a sibling repo is mid-refactor and does not compile.
- A half-written `dune` file in any sibling repo breaks `dune build` for the
  whole workspace. Build `@./all @./runtest` from inside your repo, or set up a
  private workspace of symlinks for the repos you need.
- `dune fmt` does not format `dune` files; edit library lists by hand carefully.
- A repo with no sibling dependencies can build with `dune build --root .` and is then immune to another agent's half-written sibling `dune` files.
- A library with a `.mli` listing its modules needs the new module added there as well as in the `.ml`, or the library builds but tests cannot see it.
- Parallel agents in one dune workspace see each other's half-written `dune` files. A private workspace in the scratchpad with symlinks to the needed repos, and `git archive origin/main` snapshots of repos others are editing, isolates a build.
- **dune's shared cache does not serve per-agent build directories.** dune 3.22's
  default (`enabled-except-user-rules`) restored nothing here: after `rm -rf _build` the
  same library recompiled every module, and `dune cache size` stays 0B. With
  `DUNE_CACHE=enabled` (or `--cache=enabled`) a rebuild into the *same* build directory
  path is fully restored, but a different `DUNE_BUILD_DIR` recompiles everything, so
  `_build-<repo>` directories per agent never share artifacts. Budget a cold build per
  agent directory; reuse one `DUNE_BUILD_DIR` per repo across sessions instead of
  creating new ones.
- `dune fmt` at the root of a symlink workspace formats every linked repo; use `dune build @<repo>/fmt --auto-promote`. ocamlformat rewrites `(*@@X@@*)` marker comments. `cmd | head && git commit` commits on a failed build; set `pipefail`. Tee-ing an e2e script into `head` kills it with SIGPIPE before cleanup.
- The root `dune-workspace` covers every repo, so `dune fmt --root .` from inside one repo still formats all of them. An unquoted zsh variable holding several names stays one argument in a loop; use an array or `bash -c`. The Obj.magic check flags `Obj.repr` too; allowlist test files that use it on purpose via `.obj-magic-allowlist`.
- **Copying build outputs.** Files under `_build` are read-only; `cp` onto an earlier
  copy fails with `Permission denied`. `rm -f` the destination first.
- **`core_kernel.iobuf` needs `core_kernel` in the opam depends.** It builds locally
  because the switch has it, and fails in CI with `Library "core_kernel.iobuf" not
  found`. The words-per-call figures and the pinned zero_alloc errors matched on amd64 CI.
- Each test library must be attached to its package (`(package ...)` in the dune stanza)
  or `--only-packages` builds skip or break them.

## CI and containers

- **CI**: apt's opam on `ubuntu-latest` (2.1.x) fails to add the ox repository
  (`cp: cannot stat .git/objects/maintenance.lock`). Use `ocaml/setup-ocaml@v3`
  with `ocaml-compiler: ocaml-variants.5.2.0+ox` and an `opam-repositories`
  block listing `ox` before `default`; it ships a current opam and caches the
  switch.
- Every push to CI rebuilds the OxCaml compiler until the first run finishes
  and seeds the setup-ocaml cache (30-60 min). Add a `concurrency` group with
  `cancel-in-progress: true` to every workflow, and let one run finish before
  pushing more.
- When a repo gains a sibling dependency, put the CI pin in the same commit.
- On GitHub Actions, pushing while the first run is still building the compiler cancels it; let that first run finish so the compiler cache exists. `gh run rerun --failed` is the fix for transient opam download errors.
- A sibling library can gain a new dependency on GitHub and every consumer's CI then fails with "unknown package" until it pins it too. Pin order is a dependency order, write it down once and reuse it.
- Core's `Time_ns` crashes at startup without `/etc/localtime`; runtime images need `tzdata` and a UTC symlink.
- An Async binary as PID 1 ignores SIGTERM; run it under `tini` or pods hang until the grace period ends.
- OxCaml under QEMU is too slow for multi-arch builds; use native `ubuntu-24.04-arm` runners and merge per-arch digests. Building the compiler in a 2 CPU / 6 GB Rancher Desktop VM needs swap on `/mnt/data` (`/` is tmpfs).
- `RUN git clone` layers cache forever; `ADD https://github.com/<org>/<repo>.git#ref` re-checks the remote, and a `PIN_EPOCH` build arg busts the sibling pin cache.
- hadolint 2.12 (the default in hadolint-action) rejects `ADD --keep-git-dir`; download 2.15+. The `docker/dockerfile:1.7` frontend crashed under Rancher Desktop buildkit; the built-in frontend works.
- MinIO no longer publishes images on Docker Hub; `pgsty/minio` and `pgsty/mc` are the community builds that include `mc` and `curl`.
- GHCR makes every new package private by default, even when pushed from a public repo. Put `org.opencontainers.image.source=https://github.com/<org>/<repo>` in the Dockerfile `LABEL` and in build-push-action `labels` so the package links to the repo and inherits its visibility; an already-created package must be flipped in the web UI, there is no API.
- GitHub pushes can fail transiently with `remote: fatal error in commit_refs` or `[remote rejected] (failure)`; retry. A fresh repo's first CI run can fail on "Bad checksum" for the compiler tarball; rerun.
- setup-ocaml v3 creates its switch at `$GITHUB_WORKSPACE/_opam`; that directory is what to cache after `opam install`, keyed on OS, image, compiler, ox commit and every pin's resolved commit. Its own cache key includes the repository URLs, so adding `#commit` to the ox URL forces a one-time compiler rebuild of about 16 minutes per repo. With a warm switch a CI run drops from 17 minutes to under 2; a cached switch is 2.5 GB against a 10 GB per-repo cache.
- A CI cache hit on the opam switch skips system packages: depexts such as `libsecp256k1-dev` live outside the switch, so a restored switch fails at link time. Install `opam list --installed --depexts` with apt on every cache hit, exact or partial.
- proctopus does not build on macOS (`mmap` needs `mremap`/`off64_t`, `procfs` reads `/proc`), and its opam packages do not declare Linux-only, so the failure comes at C compile time; on Linux it installs in 24 minutes. The duckdb CLI releases ship gzipped per architecture (`duckdb_cli-linux-arm64.gz`), so an image needs only `gunzip`. hadolint wants the `pipefail` `SHELL` line before any piped `RUN`. An empty `MEMTRACE` is a no-op in `trace_if_requested`, so wrappers can always set it. `docker compose --profile '*' config` validates every profile with no engine running.
- Compose: `docker compose --profile X up` also starts every service without a profile; name services explicitly. A named volume shared by two containers takes its owner from the first mount point, so mount it read-only in the second and order the start. `make` recipes that pipe into `python3 -c ...; git add` keep going after a failure; chain with `&&`. An API that rate-limits per client IP sees one IP behind an nginx proxy.
- `COPY --from=build .../_build/install/default/share/<repo>/` copied dangling symlinks:
  dune installs `share/` files as symlinks into `_build/default`, so nginx served 404 for
  `index.html` and `main.js`. Dereference first in an intermediate stage
  (`FROM build AS site` + `cp -rL`), then copy from it.

## Base, Core and stdlib shadowing

- `include Comparable.Make (T)` shadows `<`, `=` for type `t`; later int code
  like `if n < 0` breaks. Use `Int.(n < 0)` or put int helpers before the include.
- Distinct newtypes over one fixed-point implementation need a generative
  functor (`Make ()`), otherwise Price and Size unify.
- Inside a local open like `Price.( a + b )`, record field access `o.limit_px`
  fails with "Unbound record field" when the record type is not in `Price`;
  bind the field outside the local open.
- Constructors in tuple lists in test code need qualifying (`Side.Buy`), the
  type is not inferred early enough.
- Keep extracted libraries wrapped (Jane Street style); submodules become
  `Order_book.Fill`, not top-level `Fill`. Do not `open! Order_book` when it
  defines a `Result` module: it shadows Core's.
- Base `List.init` calls `f` from the last index down.
- A library module named `Info` shadows `Core.Info` for siblings and for
  anyone who opens the library; use `Core.Info` explicitly.
- **Re-export with type equality.** To make `Outer.Price.t = Inner.Price.t`
  write `include Inner.Price` in the `.ml` and
  `include module type of struct include Inner.Price end` in the `.mli`; that
  keeps `comparator_witness` equal too, so `Price.Map.t` values pass between libraries.
  A hand-written `include Comparable.S with type t := t` creates a fresh witness; add
  `and type comparator_witness = M.comparator_witness`.
- Re-export a variant with its constructors: `type t = M.t = Buy | Sell [@@deriving ...]`
  in the `.mli`, `include M` plus the extra codecs in the `.ml`.
- After `include M`, redefining `of_string` (e.g. lowercasing addresses) changes only
  that function; `Comparable`/`Hashable` sexp converters from `M` still use the base
  parser. Override `t_of_sexp` as well when it must validate.
- Never name a library module `Error` or `Info`: `open! Core` hides it even
  inside the library. Use `Api_error`.
- `(*` inside an OCaml comment opens a nested comment, so `count(*)` in a comment breaks the build.
- Inside `Int64.( ... )`, literals and `*` become int64 operations.
- In `Command.Param`, `optional_with_default` takes the parsed value as default, not the string form.
- **`Core.X` moved to `X_unix`.** Core has no Unix-dependent code; the old names are
  deprecated stubs. Library names on the ox switch (`core_unix` 0.18 preview):
  `Command.run` is `Command_unix.run` (`core_unix.command_unix`); `Core.Unix` is
  `Core_unix` (`core_unix`); `Sys.command`, `Sys.readdir`, `Sys.remove`,
  `Sys.executable_name` are `Sys_unix` (`core_unix.sys_unix`); `Filename.temp_dir` is
  `Filename_unix` (`core_unix.filename_unix`); zone-aware `Time_ns` is `Time_ns_unix`
  (`core_unix.time_ns_unix`), `Time_float_unix`, `Date_unix`; `Bigstring_unix`
  (`core_unix.bigstring_unix`), `Iobuf_unix`, `Signal_unix`, `Linux_ext`. The
  deprecation text on `Core.Sys.*` says "available from opam as `core_kernel.sys_unix`",
  which is wrong for dune: the library is `core_unix.sys_unix`.
- **`Bigstring_unix.write ?min_len`.** With the default `min_len = 0` it is one
  `write()` call: it can return a short count and does not retry on `EINTR`. Pass
  `~min_len:len` to loop until everything is written, or use `really_write` (unit
  result). `read ?min_len` is the same on the read side. Both raise `Invalid_argument` if
  `min_len > len` and `IOError` if the minimum is not reached.
- A library module named `Session` cycles with the wrapped library's own `Session` when sibling modules open the library; give it a distinct name. `open! Async` hides a library's own `Clock`; alias it in both `.ml` and `.mli`.
- `Hashtbl.sum` does not exist; `List.sum (module Int64)` fails under local mode, convert to Int. Under `set -euo pipefail`, `x=$(find missing | head -1)` aborts; add `|| true`. `export DUNE_BUILD_DIR` does not persist between agent shell calls.
- A module named like a stdlib or Core one (for example `Asset_ctx` next to `Ctx`) shadows silently inside `open!`; check `dune build @check` of dependents, not only the library.
- **Iobuf on v0.18** takes three type parameters: `(read, Iobuf.seek, Iobuf.global)
  Iobuf.t`; library `core_kernel.iobuf`.

## ppx derivers and JSON

- Generated code calls `string_of_yojson`, `int_of_yojson` etc. unqualified;
  `open! Core` does not provide them. Add
  `open Ppx_yojson_conv_lib.Yojson_conv.Primitives` in files that derive
  yojson, and only there (warning 33 fails the build on an unused open).
- `[@yojson.option]` accepts a missing field but rejects explicit `null`. For
  APIs that send `null`, use `[@default None] [@yojson_drop_if Option.is_none]`.
  For bools: `[@default false] [@yojson_drop_if not]`.
- Records reject unknown fields unless `[@@yojson.allow_extra_fields]`; put it
  on every response type decoded from a third-party API.
- Derived variants encode as `["Ctor", ...]`, never matching tagged objects.
  Hand-write variant codecs over a derived helper record. Inline-record
  constructors cannot be passed as values; destructure field by field.
- `int64` encodes as `Intlit`; emit `Int` when it fits so the tree equals what
  Yojson parses.
- Tuples encode as JSON arrays, handy for pairs like `users: [buyer, seller]`.
- `Uri` has no `sexp_of_t` without `uri-sexp`.
- `[@sexp.opaque]` on a `Uri.t` field still fails under `[@@deriving sexp_of]` on ox; hand-write `sexp_of_t`. `[%of_yojson: X.t list]` needs `Primitives` opened. ppx_yojson_conv has no `Yojson.Safe.t_of_yojson`; pass raw JSON through with an identity function.
- Inside `[%message ...]` a labelled argument cannot come first (start with `""`); inside `[%sexp ...]` a type annotation cannot follow `{ r with ... }`.
- `[%message]` must start with a string before labelled fields; bare variables print without labels, so annotate `(x : t)`. An `[@@alert]` must also appear on the module type in the `.ml`. `type t [@@immediate]` in a signature needs `[@@warning "-unused-type-declaration"]`. For must-fail compile tests pass `-color never` and a literal file name.
- `ppx_sexp` cannot call a hand-written `sexp_of_t` on a phantom-typed value through `[%sexp]`; call it directly. `module X = Info` must be bound before `open! Core`. Inline tests that read fixtures need `(deps (source_tree fixtures))` or `(glob_files ...)`.
- `[%sexp (a : int), (b : float)]` fails with "ppx_sexp_value: don't know how to handle this construct"; use `[%message "" ~a:(...) ~b:(...)]`. `[@sexp_drop_default.equal]` and `[@sexp.option]` on new record fields keep dependents' printed sexps unchanged.
- Inside `[%sexp ...]` a field access needs the whole expression parenthesised: `[%sexp ((f x).field : T)]`. For an enumeration with no quickcheck generator use `Quickcheck.Generator.of_list Side.all`. `Process.run_expect_no_output_exn` is a deprecation error on the ox Async; use `Unix.system_exn` in tests. `Time_source.create` is read-write; pass `Time_source.read_only ts` to APIs and keep the original to advance.

## Async, HTTP and websockets

- `Conduit_async.V3.resolve_uri` builds an OpenSSL config without SNI hostname
  for `wss`; rebuild it with `Ssl.Config.create ~hostname ()`. cohttp-async
  sets SNI itself for `https`.
- Async's `eprintf`/`eprint_s` are deprecated as errors; use `Print.eprintf`.
- Async: `Bus.iter_exn` does not exist in the ox Async; use
  `Bus.subscribe_permanently_exn`. `Cohttp.Code.meth` has no `equal`; use
  `Poly.equal`.
- `Clock_ns.every ~start ~stop:(Pipe.closed w)` is the idiom for per-connection
  ping timers that stop on close.
- `Websocket.frame_received` is a Bus delivering every frame including control
  frames, and async_websocket answers server pings itself, so a silence
  watchdog can count pongs with no custom frame code. Check `Bus.is_closed`
  before subscribing to a socket that is closing.
- `Uri` re-encodes query strings (`%3A` becomes `:`); sign the query in the
  form `Uri` actually sends, or signatures fail against the real API.
- `Clock_ns.with_timeout` around `Pipe.read` leaves the read pending and it
  swallows the next value; wait on `Pipe.values_available` then `read_now`.
- Read websocket frames with your own `iter_without_pushback`; a
  `Pipe.filter_map` stage stalls acks when nobody reads downstream.
- A library module that opens Async cannot refer to its own `Stream` module; alias it before the open.
- `print_endline` inside `Command.basic` prints nothing when Async is opened because the scheduler never starts; use `Command.async`.
- `open! Async` shadows `Sys`; use `Core.Sys.get_argv`. `Deferred.List.exists` takes no `~how`.
- `print_s` under `open! Async` in a `Command.basic` body goes to Async's stdout writer and never flushes; use `Core.print_s` or `Command.async`.
- On the ox Async, `Clock_ns.with_timeout` returns `Clock_ns.Or_timeout.t`; annotate it.
- `Command.async` takes `?behave_nicely_in_pipeline`, not `~behave_nicely_with_pipes`.
- `Throttle.capacity_available` waits for one slot; `Throttle.prior_jobs_done` waits for all running jobs.
- TCP feed servers need `Writer.set_raise_when_consumer_leaves w false` and should watch `Writer.consumer_left`. Async `exit` returns a Deferred; CLI error paths use `Stdlib.exit`. Hung Async expect tests print nothing; append to a file to debug.
- Async `Writer.close` still accepts writes while flushing; a write queued behind the close fails with `EBADF unix_fcntl` from the writer's error handler. A failed writer never reports flushed: use `Writer.flushed_or_failed_unit`, give `close` a `~force_close` timeout, and `Monitor.detach_and_iter_errors (Writer.monitor w)` to keep writer errors from killing the program.
- `match x with None -> fun _ -> true | Some ...` parses the `Some` case into the lambda; parenthesise the `fun`. Async has no `Pipe.Read_result` module; write `` [ `Eof | `Ok of 'a ] ``. zsh does not word-split a command held in a variable (`$B build` fails); use a wrapper script. A default `on_error` that logs with `Log.Global` leaks wall-clock timestamps into expect output; tests pass their own handler.
- macOS bash 3.2 under `set -u` treats an empty array `"${a[@]}"` as unbound; write `${a[@]+"${a[@]}"}`. Async's `Unix` has no `realpath`; use `In_thread.run (fun () -> Filename_unix.realpath p)`. `Sys.is_directory` follows symlinks, so a directory of symlinks is a cheap filtered view of a data tree.
- Filtering build output with `grep` hid a compile error and a run looked green; filter whole `File "..."` blocks by repo with awk. `Bvar.wait` inside `Deferred.any` on every paced frame piles up waiters until the next broadcast; use a replaceable `Ivar`. `Pipe.map'` with one string per batch keeps chunked HTTP bodies from sending a chunk per line.
- `Command.async` does not run `Shutdown.at_shutdown` on SIGINT/SIGTERM by default; add `Signal.handle Signal.terminating ~f:(fun _ -> shutdown 0)` so journals flush. Warning 23 (useless record `with`) is an error when the literal lists every field. `dune promote` promotes every pending correction in the shared build dir, not only yours.
- TLS host names are not verified by cohttp-async + conduit-async 8.0.0: async_ssl checks the chain (`Verify_peer`, system roots), but `~hostname` is used only for SNI and conduit's `verify` defaults to accept, so `https://wrong.host.badssl.com/` returns 200. Pass `~ssl_config:(Conduit_async.V2.Ssl.Config.create ~hostname ~verify_modes:[Verify_peer] ~verify ())` where `verify` calls `Async_ssl.Ssl.Connection.check_peer_certificate_host conn host`. A host mismatch then fails as `Failure "Connection verification failed."`, a bad chain as `Ssl_error "certificate verify failed"`.
- V2 and V3 `Conduit_async` `Ssl.Config.t` are distinct abstract types with the same `create`, so one shared function cannot serve both cohttp's `ssl_config` (V2) and a `Conduit_async.V3.OpenSSL` websocket address (V3); write one per version.

## Time and clocks

- Async `Clock` is `Time_float`; use `Clock_ns` for `Time_ns.Span`, converting
  with `Time_ns.Span.of_span_float_round_nearest`.
- `Time_ns` sexps print in the local zone, so expect tests that print timestamps differ between a Mac and CI (UTC). Rewrite to UTC in the test helper, or print `Time_ns.to_int63_ns_since_epoch`.
- `Time_source.create ~now ()` plus `advance_directly` is a good fake clock for rotation tests.
- `Time_ns.Span.to_int63_ns` returns `Int63.t`; divisions need `Int63.( / )`. `Time_ns.Alternate_sexp` keeps sexps in UTC and avoids time-zone diffs in expect tests.
- Async synthetic clocks: `Time_source.next_alarm_fires_at` shows the sleep the code scheduled, so a test can wait on it before advancing; alarms land on timing-wheel slots, so round printed delays to milliseconds. Compare `Sys.readdir "/dev/fd"` lengths to test that descriptors are released. `Cohttp_async.Client.Connection.close` cannot cancel an in-flight request.
- Synthetic-clock live tests need `Time_source.advance_by_alarms_by` then `Scheduler.yield_until_no_jobs_remain`, and every component must stamp times from the same source. In lists like `[ name, fun t x -> ...; ... ]` an unparenthesised `fun` swallows the `;`.
- Never run `dune fmt --root <workspace>`: it formats every repo, including files other agents are mid-edit on. The default sexp of `Time_ns` prints in the local zone (+01:00 locally, Z in CI); print through `Time_ns.to_string_iso8601_basic ~zone:Time_float.Zone.utc` in expect tests. ppx_yojson_conv error text includes the source path, which differs between the workspace and CI; keep it out of expectations.
- `Time_ns.sexp_of_t` prints in the local zone; use `Time_ns.Alternate_sexp` in error messages so expect output is the same on every machine. `git revert -q` is not a flag. Tightening a library's input validation changes the error text dependents' expect tests print; promote them in the same wave.

## Testing

- `dune test` fails with `Sys_error "../test_foo.ml: No such file or directory"`
  for every inline-test partition. The `ox` ppx_expect (v0.18~preview) records
  bare file names, but dune calls the runner from the test directory with
  `-source-tree-root ..`. Fix in the test `dune` stanza:
  `(inline_tests (flags (:standard -source-tree-root .)))`.
- With `open Async`, every `let%expect_test` in the file must return
  `unit Deferred.t`, even pure ones.
- Under ox Base, `Random.State.make_self_init` raises inside inline tests
  ("nondeterministic seed is forbidden"); seed from `Time_ns.now`.
- ppx_yojson_conv error text embeds the source path, which differs between
  workspace and opam builds; keep it out of expect output.
- Keeping the old expect tests in the adapter, unchanged, is a cheap proof that
  extracting a core library is behaviour-preserving; put the neutral copies, with
  promoted output, in the core.
- `dune promote` overwrites test edits made after the failing run; promote
  first. Filtering build output with `grep '^[-+] '` hides compile errors.
- Exception sexps embed the source path; keep them out of expect output.
- `Expect_test_helpers_core` is not on the ox switch; use `Or_error.try_with` and print.
- Pinning compiler error text in a dune rule (`with-accepted-exit-codes 2 (run %{ocamlopt} -i f.ml)`
  diffed against `f.expected`) works, but on GitHub Actions the compiler colours its
  output and the diff fails; pass `-color never`. A unique value passed to an `@ local`
  parameter without `borrow_` counts as a use, so consuming it later fails.
- core_bench shows 3 minor words for an empty benchmark; count allocations with `Gc.minor_words` in an expect test. `Time_ns.Stable.Span.V2` has no quickcheck functions; `Quickcheck.Shrinker.atomic` does not exist, use `Shrinker.empty ()`. Opening `Quickcheck.Generator.Let_syntax` at the top of a file shadows Async's `let%bind`.
- The inline test runner always writes a `.corrected` file even on success; compare with the original to detect failure, and `-partition <file>.ml` reruns one file. `dune-build-info` reports `version=unknown` through a symlinked workspace.
- A must-fail compile test run with plain `ocamlc` and only the library's `.cmi` files can fail with "layout ... is any because the .cmi is missing" instead of the intended error; pick a call whose types need no other library. There is no `ocamlfind` on the switch.
- `ppx_inline_test` on the ox switch rejects `let%test_module`; write `module%test Name = struct ... end`. In a `Command.async` body after `open Command.Param`, `return` is the Param one: use `Deferred.unit`, and `let%map.Command` for flags.

## OxCaml modes and layouts

- OxCaml rejects `exception E of t [@@deriving sexp_of]` when a field's sexp
  converter is not portable; carry a tagged sexp or an int code instead.
- On OxCaml `List.split_n` returns an unboxed tuple; use `List.take`/`List.drop`. `let%bind x : T = e` annotates `e`; write `let%bind (x : T) = e`. Partial applications passed as labelled arguments fail with "This value is local"; wrap in `fun`. Nested variant constructors inside a list need qualification.
- `Zstd.decompress` handles concatenated frames; only `get_decompressed_size` stops at the first, so grow the buffer on "Destination buffer is too small". OxCaml alerts on `Domain.spawn` (wants `Domain.Safe.spawn` with portable closures), so child processes are the simple way to use several cores from Core/Async code. One `In_thread` hand-off per pipe chunk cost 4x in kernel time; batch hand-offs.
- **Loopify does not unbox floats.** A self-tail-recursive sum with a `float`
  accumulator allocates 2 words per iteration (2000 per 1000 elements) in dev, release
  and release `-O3`, with or without `[@@loop]`. A `let mutable` loop or a `float#`
  accumulator drops it to the 2 words of the boxed result.
- **`[@unrolled n]` is a call-site attribute.** On a definition (`[@@unrolled 3]`) it is
  warning 53, an error in the dev profile; write `(f [@unrolled 3]) x`.
- **Two `Float_u` modules.** `Stdlib_upstream_compatible.Float_u` has `add`, `sub`,
  `mul`, `div`; the `unboxed.float_u` library's `Float_u` has `add` and `sub` but no
  top-level `mul`/`div` (use `Float_u.O.( * )`), and is the one with `compare`, `equal`,
  sexp and quickcheck. `[@@deriving compare, equal, hash, bin_io]` on a field typed
  `float#` fails with `Unbound value "compare_float_u"` etc.; type the field `Float_u.t`
  from `unboxed.float_u`. `[@@deriving sexp]` on a bare `float#` field works.
- **Mixed blocks break polymorphic compare at run time.** `x = x` on a record with a
  `float#` field compiles and raises `Invalid_argument("compare: mixed block value")`.
- **Docs lag the compiler.** `type t = K of int64# | L` compiles although oxcaml.org lists
  it as a restriction; `exception E of float#` is still rejected. `ppx_template`
  instantiation (`f [@mode local]`) works outside a `%template` block.
- **Gated features.** `ocaml_simd*`, `parallel`, the `Multicore` library and
  `ocaml-probes` are not installed. The vector type `int64x2#` type-checks; `[%probe]`
  compiles and is silently ignored (`probes: false`, `-probes` "option not available").
  `Domain.spawn` carries alerts `unsafe_multidomain` and `do_not_spawn_domains`; use
  `Domain.Safe.spawn [@alert "-do_not_spawn_domains"]`.
- **Extension universe.** The default enables layouts, small numbers, `let mutable`,
  labeled tuples and iarrays with no flag; `-extension-universe upstream_compatible`
  rejects `float32` and unboxed records.
- **zero_alloc details.** `val f : ... [@@zero_alloc]` and `val[@zero_alloc] f : ...`
  both work and are enforced against an unannotated implementation.
  `[@zero_alloc strict]` passes with `failwith "literal"` and fails with a formatted
  message. Must-fail tests for zero_alloc need `ocamlopt -c` (the check is in the
  backend, `-i` passes) and the symbol names in the error (`(camlM__f_2_5_code)`) stripped
  with sed.
- A fixed-point decimal made zero-alloc: parsing went 85 ns/14 words to 18 ns/0, printing 126 ns/48 words to 22 ns/3. `rem` is not infix under `open! Core`; use `Int.rem`. Aliases like `let mul = mul_units` or `let ( + ) = Int.( + )` in a functor body fail a `[@@zero_alloc]` signature ("weaker guarantee"); write `let[@zero_alloc] mul a b = ...`. A module type in the `.ml` must repeat the `.mli`'s `[@@zero_alloc]` attributes. `val compare ... [@@zero_alloc]` can be restated after `include Comparable.S` but the implementation must override with plain `Int` comparisons. A 0-words test can pass while `[@zero_alloc]` rejects: a rare slow path allocating a tuple is only caught by the checker. A functor's functions pass once the functor argument's functions carry the attribute.

## Performance and measurement

- **perf, bpftrace and magic-trace.** None runs on macOS: there is no `perf` or
  `bpftrace`, `dtrace` is limited by SIP, and Apple silicon has no Intel Processor Trace.
  Profile on Linux. magic-trace needs an Intel CPU with PT exposed, which means bare
  metal or a VM that passes PT through; GitHub runners, Rancher Desktop VMs on a Mac
  and most cloud instances do not. `perf` and bpftrace uprobes inside a container need
  `--cap-add SYS_ADMIN` (or `CAP_PERFMON` plus `CAP_BPF`) and a host
  `kernel.perf_event_paranoid` of 1 or lower. OxCaml symbols carry build stamps,
  e.g. `camlMy_lib__Writer__append_32_100_code`; the numbers change between
  builds, so attach with a glob (`uprobe:/app/bin/my-app:camlMy_lib__Writer__append_*_code`).
- **Syscall tracing an Async program**: most `strace` output is the scheduler's
  `epoll_wait`/`futex`/`timerfd` noise; filter with `strace -f -e trace=network,read,write`
  (or a viewer such as strace-ui) before looking for socket problems.
- Order-book depth diffs decode at about 6 minor words per byte with Yojson; JSON decoding was about 60% of a recorder's per-frame cost.
- A pipe-merge raw reader does about 0.5 to 0.7 M lines/s; a hand-written timestamp parser changed nothing, the per-line pipe hand-off dominates. Adding a field to a widely implemented module type (`Feed.S`) breaks every implementor; add a wrapper (`Feed.With_gaps`) with `of_feed`/`to_feed` instead.
- **Measure words, not nanoseconds.** Removing an option, tuple or float box changed
  time by 0.3-1.5 ns on a hot minor heap; words went to 0. `-O3` changed no words-per-call
  figure in these cases. The `core_bench` command has no test-name filter.
- **JSON.** One small trade frame: `Yojson.Safe.from_string` plus member lookups
  298 words, 2.4 us; a `[@zero_alloc]` scanner 0 words, 0.26 us. The first scanner
  (restart every key search, compare every byte) was 0.88 us.
- `Pipe.read_now` allocates about 13 words per call even with no consumer, and `Pipe.iter_without_pushback` calls it per element; `Pipe.read_now'` pays it once per batch. A relay that paid it twice per frame cost 26 words; batched delivery is 13 and a direct callback is 0. Most of the remaining 35 words per frame are inside async_websocket (payload copy, frame record). Word counts are exact and reproducible across dev, release, O3 and amd64; nanoseconds on a loaded laptop are not, so pin words.
- Hand-written curves over zarith, dev build, arm64: Ed25519 sign 0.73 ms,
  verify 0.67 ms; P-256 verify 1.1 ms; P-521 verify 4.5 ms. Roughly 3-30x slower than
  libsodium/OpenSSL, which is fine behind a REST call or a webhook delivery and not on a
  hot path. For a hot use, add C stubs over libcrypto's EVP_DigestSign/Verify (already
  linked through async_ssl).

## Bonsai and js_of_ocaml

- A fresh switch resolves upstream dune 3.24.2, whose ppx_css inliner writes `include ./app__generated` and breaks bonsai_web with "Syntax error"; keep `dune < 3.23` in the opam file (ox dune 3.22.2+ox works). `let%arr` needs `bonsai.ppx_bonsai` (not in ppx_jane); `let%map` with `Bonsai.Let_syntax` works. `Bonsai.state_opt` takes no unit; the `For_proc` section of `cont.mli` has different signatures from the top-level ones. `Bonsai_web` exports `View`, shadowing a local one; inside `Float.( ... )`, `min`/`max` are the Float ones. js_of_ocaml `int` is 32 bits: millisecond timestamps must be floats; silence the effects warning with `-w no-missing-effects-backend` in `link_flags`; ojs libraries need `(modes byte)`; `Async_js.Http` only does GET/POST, PATCH/DELETE need raw `XmlHttpRequest`. Read `window.ethereum` inside `Effect.of_thunk`, not at render. Node 22 defines `navigator` as a getter: fake DOM globals need `defineProperty`. Release JS 1.58 MB (465 KB gzipped); dev 49 MB; CI cold 25 min, warm 2.3 min.

## bin_prot and stable types

- Adding a constructor to a shared wire type breaks exhaustive matches in every dependent repo at once. Route new data through a separate module reachable from an existing `Unknown` branch, and only add the real constructor after dependents carry wildcards.
- Extending a stable variant without breaking dependents: keep the in-memory type separate and hand-write its bin_io sharing the stable type's tags, pinned by a byte-equality test; append the new digest to the compatible list and check in a fixture written by the old code. 64k-row Parquet row groups cut a five-minute range query from 12 ms to 2 ms on 2M rows for 2% more bytes.

## Parquet, DuckDB and zstd

- Thrift compact field headers carry the field id *delta* from the previous field, not the id; nested structs restart the delta at 0. Bool fields put their value in the type nibble and have no body. Union types (`LogicalType`, `TimeUnit`) encode as a struct with exactly one empty-struct field.
- `TIMESTAMP(NANOS, isAdjustedToUTC=true)` makes DuckDB read the column as `TIMESTAMP WITH TIME ZONE`, which is microsecond-only and silently drops nanoseconds. Default to `isAdjustedToUTC=false` to get `TIMESTAMP_NS`.
- DuckDB rejects a whole file if any STRING value is invalid UTF-8; use `bytes` for opaque payloads. Without `ORDER BY`, rows across row groups come back in any order, so tests should read with `file_row_number = true` and order by it.
- The `zstd` opam binding (0.4, ctypes based) builds on the ox switch. Compress only the page body; the page header stays uncompressed and records both sizes.
- Multiplying two `DECIMAL(18,8)` columns in DuckDB overflows ("Overflow in multiplication of DECIMAL(18)"); cast to DOUBLE or DECIMAL(38,16) first.
- DuckDB refuses `CREATE VIEW` over a `read_parquet` glob that matches no files; create views only for tables that have files.
- Write Parquet to `<path>.tmp` and rename on close so globs never see a partial file.
- Concatenated zstd frames decode as one stream with `zstd -d`, so appending a frame to an existing `.zst` is a valid restart strategy; the OCaml binding's `get_decompressed_size` reads only the first frame.
- Bigbuffer plus a reused bin_prot scratch buffer makes appends allocation-free. pyarrow skips `_`-prefixed names, so side files next to Parquet go in `_manifest/`. DuckDB `ASOF LEFT JOIN` and `epoch_ns` work on `TIMESTAMP_NS`.

## Shell

- Never chain `dune test; git commit; git push` with `;` — a killed test
  still commits. Use `&&`.
- `dune exec` runs the binary from `_build-*/install/default/bin/<name>`, so `pkill -f main.exe` misses it; kill by pid or by the `<name> -port` pattern. Check `lsof -iTCP:<port> -sTCP:LISTEN` before starting a server on a fixed port.
- `cmd | grep -v ... && git commit` skips the commit when grep filters every line; never put a filtering grep in an `&&` chain. A `;` chain once pushed a non-compiling commit. In zsh `echo ======` fails with "not found".
- zsh does not word-split variables: `opam install $PKGS` passes one argument; use `${=PKGS}` or `bash -c`. Serialize opam installs across agents.
