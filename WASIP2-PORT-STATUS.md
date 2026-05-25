# wasip2 port — status checkpoint

Date: 2026-05-25
Branch: `flawless-neo/wasip2`
Plan: consuming repo's `docs/plans/2026-05-19-002-feat-lancedb-user-owned-schema-instances-plan.md`

## Per-crate status (wasm32-wasip2)

| Crate | `--no-default-features` | `--no-default-features --features bitpacking` |
|---|---|---|
| `lance-arrow` | ✅ clean | — |
| `lance-core` | ✅ clean | — |
| `lance-encoding` | ❌ needs `bitpacking` | ✅ clean |
| `lance-io` | ✅ clean | — |
| `lance-file` | ⏳ `testing` module gated + bitpacking forwarded; needs ~3 more source gates | ⏳ same |
| `lance-table` | ⏳ ~6 call-site gates (`ObjectStore::put`, `lance_io::object_writer`, `Path::from_filesystem_path`, `lance_io::local::*`) | ⏳ same |
| `lance` (umbrella) | ⏳ blocked on lance-table + lance-file clean | ⏳ same |

## Build invocation that works today

```bash
export WASI_SDK_PATH="$HOME/.local/opt/wasi-sdk-33.0-arm64-macos"
export CC_wasm32_wasip2="$WASI_SDK_PATH/bin/clang"
export CFLAGS_wasm32_wasip2="--sysroot=$WASI_SDK_PATH/share/wasi-sysroot --target=wasm32-wasip2"

cargo check -p lance-arrow      --target wasm32-wasip2 --no-default-features
cargo check -p lance-core       --target wasm32-wasip2 --no-default-features
cargo check -p lance-encoding   --target wasm32-wasip2 --no-default-features --features bitpacking
```

## lance-io — exact remaining surgery

Module-level gates already applied to `src/lib.rs`:
- `pub mod local`            — `#[cfg(not(target_arch = "wasm32"))]`
- `pub mod object_writer`    — `#[cfg(not(target_arch = "wasm32"))]`

What's left in `src/object_store.rs`:

The file imports + uses `LocalObjectReader` (from `super::local`) and
`LocalWriter` / `ObjectWriter` / `WriteResult` (from
`crate::object_writer`). Imports are already gated. The remaining
work is to gate every CALL SITE — there are ~14, all in fns that
serve the local-fs backend path:

```
src/object_store.rs:632  LocalObjectReader::open_with_tracker
src/object_store.rs:694  LocalObjectReader::open_with_tracker
src/object_store.rs:738  fn put_async (uses Path::from_absolute_path)
src/object_store.rs:741  ditto
src/object_store.rs:742  ditto
src/object_store.rs:749  tokio::fs::create_dir_all
src/object_store.rs:757  tokio::fs::File::from_std
src/object_store.rs:760  ditto
src/object_store.rs:771  ditto
src/object_store.rs:784  ditto
src/object_store.rs:798  ditto
src/object_store.rs:841  ditto
src/object_store.rs:855  ditto
```

The cleanest pattern: cfg-gate each `fn` that contains a local-fs
call site as a whole, with a `#[cfg(target_arch = "wasm32")]`
alternative that returns `Err(Error::NotSupported(...))`. Lance-io's
high-level API stays the same shape on both targets; calling a
local-fs op on wasm just errors at runtime.

The functions to gate (from inspection of the `impl ObjectStore`
block around lines 600-900):
- `LocalObjectReader::open_with_tracker` callers in two read paths
- `put_async` (line ~738) — writes to local backend
- A handful of helpers that construct `LocalWriter` / wrap
  `tokio::fs::File::from_std`

## Other remaining work

- `src/object_store/providers/local.rs` — entire file: gate or
  cfg-out the `LocalStoreProvider` impl block. (Single
  `#[cfg(not(target_arch = "wasm32"))]` at the top of the file.)
- `src/traits.rs` line 16 imports `WriteResult` from `object_writer`
  — gate the import + the trait methods that reference
  `WriteResult` if any (need to verify; may need to move
  `WriteResult` out of `object_writer` into a non-fs module).

## After lance-io is clean

Mechanical cascade: `lance-table` → `lance-file` (parquet `zstd`
feature off, same pattern as `arrow-ipc`) → `lance` (umbrella; may
need feature-gate audits of `lance-index`, `lance-datafusion`,
`lance-namespace-*` — all of which are NOT in v1 wasip2 scope per
the R13+R14 deferral decision in the consuming repo at
`docs/decisions/2026-05-20-r13-r14-deferral-to-v1.5.md`).
