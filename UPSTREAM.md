# Upstream

This fork is n0's `iroh-blobs` at its latest release plus one fix that upstream has not shipped yet; it stands on `v0.103.1` (`7a4c53a`). Upstream is <https://github.com/n0-computer/iroh-blobs>. The fork follows upstream release by release: each new release tag is taken whole and the fix is carried onto it, and nothing from between releases is taken. It adds nothing beyond the fix, and it exists until an upstream release carries an equivalent fix. `mee-pdn` uses it through `[patch.crates-io]` in its root `Cargo.toml`, as a git dependency pinned by revision.

## What the fork changes

**A store runtime dropped on one of its own threads no longer waits for itself.** `FsStore::load_with_opts` builds a private multi-thread runtime for the store and hands it to the store's actor as an `RtWrapper`. The actor is created and run as tasks on that same runtime. In upstream, `RtWrapper::drop` calls `block_in_place(|| drop(rt))`, and dropping a runtime waits for every thread of its blocking pool to stop. Tokio launches the multi-thread runtime's workers through that pool, so when the drop runs on one of the runtime's own threads, the thread waits for itself forever.

That happens on two paths. When `Actor::new` fails (the store's directory cannot be created, or redb refuses the database, for example with `DatabaseAlreadyOpen`), the failing task drops the runtime: the task never completes, `load_with_opts` never returns, and the error is lost. When a store's actor stops, it drops the runtime on its own worker the same way, and that thread stays parked for the rest of the process.

The fix is in `src/store/fs.rs`. Each runtime `load_with_opts` builds gets a number, and `on_thread_start` marks every thread of that runtime with it in a thread-local. `RtWrapper::drop` on a marked thread of its own runtime calls `shutdown_background()`, which signals the runtime to stop and returns without waiting. Anywhere else it drops the runtime as upstream does. The marker is used instead of `Handle::id()` because `Handle::id()` is stable only since tokio 1.49, while the crate requires 1.43.

The cost: on a thread of its own runtime, the drop no longer waits for the runtime's remaining tasks. On the failing load nothing else has started yet. On a stopping actor, the database task is dropped as the workers stop rather than before the drop returns, so a store loaded again on the same path immediately afterwards can find redb still locked — and now gets that error back instead of hanging.

The tests are `test_load_on_a_held_database_returns_the_error` and `test_load_on_an_unusable_directory_returns_the_error` in `src/store/fs.rs`. Both hang until their 10-second timeout without the fix. The stopping path has no test of its own: tokio offers no way to enumerate a runtime's threads, so a parked thread is invisible to a test. It goes through the same `Drop`.

## Upstream issues this fork waits on

- <https://github.com/n0-computer/iroh-blobs/issues/252>: a failed `FsStore::load` hangs forever. This is the failing-load path above. It was opened on 2026-08-11, and it had no reply as of 2026-10-08.
- <https://github.com/n0-computer/iroh-blobs/issues/260>: store teardown can deadlock. This is the stopping path above. It was opened on 2026-09-10, and it had no reply as of 2026-10-08.
- <https://github.com/n0-computer/iroh-blobs/issues/249>: a request not to build a runtime in `FsStore::load` at all. If upstream does that, both paths disappear with it. It was opened on 2026-07-26, and it had no reply as of 2026-10-08.

## Moving to a new upstream release

When upstream tags a release that does not fix both paths:

1. Fetch the tag: `git fetch https://github.com/n0-computer/iroh-blobs.git 'refs/tags/<release>:refs/tags/<release>'`.
2. Start a branch from the tag and cherry-pick the fix onto it. Any conflict is in `src/store/fs.rs`.
3. Run the two tests above with the fix, and then the whole suite as upstream's CI runs it: `cargo nextest run --locked --workspace --lib --bins --tests` with `--all-features`, with the default features and with `--no-default-features`, the doc tests, `cargo fmt --check` with the options in `Makefile.toml`, and clippy in the three ways `.github/workflows/ci.yaml` runs it, under `RUSTFLAGS=-Dwarnings`.
4. Push the branch, move the `rev` in `mee-pdn`'s `[patch.crates-io]` to it, and run `mee-pdn`'s checks.
5. Update the release this file names at the top.

## Returning to upstream

The fork is dropped once an upstream release fixes both paths. It does not matter whether the fix is like this one or removes the private runtime. To confirm, run the two tests above against that release. Then remove the `[patch.crates-io]` entry from `mee-pdn`, raise its `iroh-blobs` requirement to that release, and archive this repository.

## Weekly check

Once a week:

- Read the three issues and any pull request linked from them.
- List upstream releases with `git ls-remote --tags https://github.com/n0-computer/iroh-blobs.git 'v*'`.
- For a new release, read `git log <release this fork stands on>..<new release> -- src/store/fs.rs` for a change to `RtWrapper` or `load_with_opts`. Then either move the fork onto that release or return to upstream, as the two sections above describe.

Record each check below, newest first: the date, what was found, and what was done. A week with no entry means the check was not made.

## Log

- 2026-10-08: fork created from `v0.103.1`. The latest upstream release is `v0.103.1`, upstream `main` has no fix, and none of the three issues has a reply.
