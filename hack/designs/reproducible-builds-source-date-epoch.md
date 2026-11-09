# Reproducible Builds: `SOURCE_DATE_EPOCH` as a client default

*Builds on [#14950](https://github.com/dagger/dagger/pull/14950). Tracks [discussion #12313](https://github.com/dagger/dagger/discussions/12313). Supersedes [#10318](https://github.com/dagger/dagger/pull/10318).*

## Problem

[#14950](https://github.com/dagger/dagger/pull/14950) added an explicit `sourceDateEpoch` argument to `Container.publish`, `export`, `exportImage` and `asTarball`. Three gaps remain for the standard [`SOURCE_DATE_EPOCH`](https://reproducible-builds.org/docs/source-date-epoch/) workflow:

1. **Directory exports drift.** `Directory.export` writes files with their wall-clock mtimes.
2. **The environment variable is ignored.** A CI job that sets `SOURCE_DATE_EPOCH` still has to pass it to every export call, through every module that exports.
3. **Build tools never see it.** Compilers and packagers inside exec containers that honor `SOURCE_DATE_EPOCH` don't get it.

## Solution

| Piece | Behavior |
|-------|----------|
| `Directory.export(sourceDateEpoch: Int)` | Clamps exported file mtimes later than the epoch down to it. |
| Client default | The CLI reads `SOURCE_DATE_EPOCH` into `ClientMetadata.SourceDateEpoch`. An omitted `sourceDateEpoch` argument defaults to it. An explicit argument wins. |
| Exec env | `withExec` sets `SOURCE_DATE_EPOCH` in the container from the calling client, unless the container already sets it (`withEnvVariable` wins). |

Without `SOURCE_DATE_EPOCH` on the client, behavior is unchanged.

## Implementation

- **Client** (`engine/client/client.go`, `engine/opts.go`): `clientMetadata()` parses `SOURCE_DATE_EPOCH` into `ClientMetadata.SourceDateEpoch`. SDK clients get this for free: their requests go through the CLI session proxy, which attaches the CLI's client metadata to every request.
- **Argument default** (`core/schema/container.go`): `sourceDateEpochArg(ctx, arg)` returns the explicit argument, else the client's value, else nil.
- **Directory export** (`engine/engineutil/filesync.go`): `LocalDirExport` wraps the export FS in an `fsutil` filter whose map hook clamps `Stat.ModTime` as entries stream to the client. The snapshot is untouched. Changeset export keeps calling `LocalFSExport` directly and is unaffected.
- **Exec env** (`core/schema/container.go`, `engine/engineutil/executor_spec.go`): the `withExec` resolver copies the client's value into `ExecutionMetadata.SourceDateEpoch`. The executor's `setSourceDateEpoch` setup phase adds it to the process env unless the container env already has it. Nested clients started by the exec (module functions, dagger-in-dagger) inherit this value as their own client default.

## Cache identity

A default that comes from the client, not from an argument, must still be part of the cache key, or results computed for one client's epoch would be served to another's.

- `publish`, `export`, `exportImage` and `Directory.export` are `DoNotCache`, so the resolver-side default needs nothing more.
- `asTarball` and `withExec` are cached. Their dynamic input hooks call `setClientSourceDateEpochInput`, which adds a `clientSourceDateEpoch` implicit input when the client sets an epoch (and, for `asTarball`, omits the argument). Calls from clients without `SOURCE_DATE_EPOCH` keep their current identity, so existing cache entries stay valid.
- The exec captures the epoch at resolve time, not when it runs. Lazy evaluation by a different client therefore uses the value the call was keyed by. Nested client metadata is taken from the exec's metadata for the same reason.

## Open question

Module function calls are cached by their arguments. A function that reads `SOURCE_DATE_EPOCH` from its runtime container's env, or relies on the default inside, can return a result computed under another client's epoch. Keying function calls by the client epoch would fix this at the cost of splitting their cache. Left for review.

## Testing

- `TestContainer/TestSourceDateEpochFromClientEnv`: three clients (two epochs, none) run the same exec and `asTarball`. Each sees its own value, which shows the cache doesn't leak across clients. `withEnvVariable` and an explicit argument override the default.
- `TestDirectory/TestExportSourceDateEpoch`: mtimes are clamped with the argument and with the client env default, and are left alone without either.
