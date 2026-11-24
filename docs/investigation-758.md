# Investigation: texture ownership after host writes (#758)

Status: static source review only; no Rust build, D3D12 execution, or game data used. AI-assisted investigation. This is a working note on the personal fork, not a proposed upstream patch.

## Scope

Upstream issue: https://github.com/Marvisak/VIctory/issues/758
Reviewed matching upstream and fork versions of texture.rs and host.rs on main (2026-11-24). Recheck main before implementation.

## Confirmed source behavior

- texture.rs:69-86: HOST_WRITTEN is a resource-keyed set; host_written inserts a key, and forget removes it.
- texture.rs:103-116: upload returns before creating staging or recording a copy if the key is present.
- list.rs:454,550,634: CopyResource, CopyTextureRegion and resolve mark the destination during command-list recording.
- texture.rs:167-225: staging refresh is deferred; it detects changes via watch::writes for watched memory and BLAKE3 for committed unwatched memory.
- host.rs:493-511,1143-1163,1219-1225: the host runs deferred steps on submission, then executes the recorded lists. first_use is per recording, reset with the list. An upload skipped at recording has neither a deferred refresh nor a recorded copy.
- resource.rs:976: resource drop forgets its texture state.

The reported stale texture is independently reproduced by DrDarkDK on WARP and RTX 4070 Ti using synthetic pixels (issue comment 7127987785). Those are their results, not ours.

## Additional design hazard: recording versus submission

A permanent set cannot express which version of title memory was current when a host copy took ownership. Removing the early return would also be incorrect: a recorded copy from unchanged staging could overwrite newer host pixels, even if refresh correctly detects no change.

A recording-time generation check alone is insufficient when a closed command list is reused: recording it while host ownership is active can permanently omit the upload even after a later CPU write. Conversely, an upload recorded before an intervening host copy can still execute afterward. These are source-derived risks to test, not verified GPU failures.

A second hazard is partial copies. Current upload loops over every staging footprint; a change to one mip or region must not silently destroy still-authoritative GPU data in another. The write-watch journal tracks 64 KiB pages, not exact pixel changes.

## Related existing work

- https://github.com/Marvisak/VIctory/pull/230 (open when reviewed): proposes conditional submission-time texture uploads via a prelude; explicitly preserves host_written skipping. This is a useful integration point, not a complete #758 fix.
- https://github.com/Marvisak/VIctory/pull/594 (open when reviewed): tracks subresource states. Coordinate with this work before changing barriers or partial copies.
- https://github.com/Marvisak/VIctory/pull/143 and https://github.com/Marvisak/VIctory/pull/641: related unwatched memory safety and partially resident textures. Recheck before changing the hash path.

## Proposed solution direction (not implemented)

1. Track a title-memory baseline associated with the host write: a journal generation for watched memory, a valid content snapshot/digest for unwatched memory. Unknown/unmapped contents must not be treated as confirmed changes. Keep ownership separate from staging freshness.
2. Preserve host data while the title memory has not changed. When a later title write is detected, refresh staging and schedule only the necessary copy. Do not unconditionally replay a stale staging copy.
3. Make the upload decision reflect execution order, including host writes and repeated submissions. Investigate PR #230's prelude mechanism first; a prelude cannot by itself represent all possible intra-list copy/read orderings. Do not claim arbitrary ordering support until a test establishes it.
4. Define partial-copy/subresource ownership explicitly, or narrow the first patch to whole-resource copies and keep the remaining limitation documented. Preserve resource lifetime and clear state on drop.

## Synthetic regression matrix (proposed, not run)

Use generated RGBA8 pixels, no title files.

| Case | Expected |
| --- | --- |
| Initial title upload A | Host reads A |
| Full host copy B, unchanged title A | Host still reads B |
| Full host copy B, later title write C | Host reads C |
| Record a reader under host ownership; write C before submit | Reader observes C under intended title-memory semantics |
| Reuse a closed reader across title writes | Does not permanently suppress later uploads |
| Record reader before host copy, submit after copy, title unchanged | B is preserved |
| Partial copy of one subresource, unrelated title write | Host-owned untouched subresource survives (define granularity) |
| Unwatched/unmapped range | No invalid memory read or fabricated dirty result |
| Resource release and key reuse | New resource does not inherit old ownership |

Repeat the applicable cases on write-watched and unwatched memory. Verify readback on WARP first. Report any skipped adapters separately; do not run a retail world workload.

## Next steps

Coordinate with the issue author and the author of #230 before implementing. Establish the intended semantics for mixed/partial ownership and recording versus submission. Implement a focused synthetic regression first; then a minimal fix. Run Windows GPU crate tests, rustfmt, Clippy and asset policy when a supported environment is available. Do not open an upstream PR while external contributions are restricted.
