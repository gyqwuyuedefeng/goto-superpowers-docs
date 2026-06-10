# TempPathRoot Cleanup Design

## Summary

Add a scheduled cleanup task that scans `tempPathRoot` and removes expired temporary content. The task is intentionally filesystem-only: it does not classify files by business type and does not depend on Redis, database state, or download completion callbacks.

This design covers every file and directory under `tempPathRoot`, including:

- files created directly under the root
- tool output directories such as `excel-convert`, `image-convert`, and `merge-image`
- any other temporary content written by the system under the same root

## Goals

- Reclaim disk space for all temporary content under `tempPathRoot`
- Keep the cleanup rule simple and uniform across business flows
- Avoid coupling cleanup behavior to specific task metadata or download timing
- Follow the existing task style used by Quartz cleanup jobs in the project

## Non-Goals

- No immediate deletion after download
- No Redis-driven or database-driven retention decisions
- No special-case retention rules per business type
- No cleanup outside `tempPathRoot`

## Current Context

`tempPathRoot` is configured in:

- `goto/common/src/main/resources/application-common-home.yml`
- `goto/common/src/main/resources/application-common-public.yml`
- `goto/common/src/main/resources/application-common-wsl.yml`

Known flows already writing into this root include:

- Excel format conversion input and output
- image conversion input and output
- merge-image task input and output
- root-level temporary files used by tool flows

The download framework currently has `postprocessFiles`, but that hook runs during download URL generation rather than after the HTTP response finishes. It is therefore not a reliable place to implement "delete after download".

## Proposed Approach

Implement a dedicated scheduled cleanup task for `tempPathRoot` only.

Suggested task shape:

- create a class in `goto/system` similar in style to `LongTimeUnusedUserDrawRemoveTask`
- expose a single `run()` entrypoint
- in `run()`, perform scan, expiration check, deletion, and summary logging

Retention rule:

- default retention period: `12` hours
- expiration basis: `lastModified`
- delete when `now - lastModified > retention`

Scheduling:

- recommended scan frequency: once per hour
- deletion threshold remains based on `12` hours, not on the scheduling interval

## Deletion Scope

The task scans the normalized absolute `tempPathRoot`.

Deletion behavior:

- if an expired path is a regular file, delete the file
- if an expired path is a directory, delete the entire directory tree
- if a path is not expired, keep it unchanged

The cleanup is recursive, so nested files and directories are covered automatically.

## Safety Rules

### Root Path Guard

Before deletion, normalize the configured `tempPathRoot` to an absolute path. Every candidate path must also be normalized and verified as a child of that root. If a path falls outside the root, skip it and log a warning.

### Symlink Handling

Treat symbolic links conservatively:

- do not follow symlinks into external locations
- if a symlink entry itself is expired, delete only the link entry

### Recursive Deletion Order

For directory deletion, remove children before parents so that non-empty directory errors do not interrupt normal cleanup.

### Failure Isolation

Deletion failures for individual paths must not fail the whole run. Typical issues such as file already removed, permission denial, or concurrent writes should be logged per path and counted in summary statistics.

## Logging

The task should log:

- task start
- root path being scanned
- no-op exit when root does not exist or contains no removable content
- per-path failure logs with path and exception
- final summary statistics

Suggested summary fields:

- scanned count
- expired count
- deleted success count
- deleted failure count
- skipped count

## Configuration

Keep the first version minimal.

Required configurable item:

- retention period, default `12` hours

Optional future config, not required now:

- scan cron or fixed delay
- maximum delete count per run
- excluded subdirectories

## Testing Strategy

At minimum, verify:

1. `tempPathRoot` does not exist
   The task exits cleanly and logs a no-op result.

2. Mixed root content
   Expired files and directories are deleted, unexpired ones remain.

3. Recursive directory deletion
   Nested directory trees are fully removed.

4. Path guard
   Out-of-root paths are never deleted.

5. Failure tolerance
   A single delete failure does not stop the remaining cleanup work.

6. Symlink behavior
   The task does not traverse outside the root through a symlink.

## Implementation Notes

Recommended implementation split:

- `TempPathRootCleanupTask`: orchestration and summary logging
- small internal helper methods for expiration check, safe path normalization, and recursive deletion

This keeps the task readable and consistent with the existing cleanup task style while avoiding unnecessary framework expansion.

## Risks And Tradeoffs

- Using `lastModified` means active writes can extend retention. This is acceptable because it avoids deleting recently touched temporary content.
- A filesystem-only policy may keep some stale files until the next scheduled run. This is acceptable because the chosen strategy explicitly prefers simplicity over immediate cleanup.
- Scanning the full root each run is broader than business-specific cleanup, but it matches the requirement that every temporary artifact under `tempPathRoot` must be covered.

## Decision

Approved design decisions:

- use a pure scheduled cleanup task
- scan only `tempPathRoot`
- include all files and directories under that root
- use `lastModified` as the expiration basis
- use a default retention period of `12` hours
- do not implement delete-after-download behavior
