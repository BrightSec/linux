# Fuzzing scope: nfsd4_sequence ONLY

Target ONE function: `nfsd4_sequence` in `fs/nfsd/nfs4state.c`.

## Why this function
This is the NFSv4.1 SEQUENCE operation handler. It manages DRC (Duplicate
Request Cache) slots per session. A known use-after-free exists where
`free_session_slots` frees a slot and the same function then writes to it.

## The function and its helpers
- `nfsd4_sequence` (fs/nfsd/nfs4state.c, ~line 4440) — the main target
- `free_session_slots` — frees slots when the shrinker reduced target_maxslots
- `check_slot_seqid` — validates slot sequence IDs
- `nfsd4_alloc_slot` — allocates new slots

## How to harness — USE EXTRACTION, NOT whole-file inclusion

`nfs4state.c` is 7000+ lines with 100+ external dependencies. Including
the whole file causes a stub avalanche. DO NOT `#include "nfs4state.c"`.

Instead, EXTRACT the function:

1. Copy `nfsd4_sequence`, `free_session_slots`, `check_slot_seqid` into
   `extracted.c`. Keep the bodies verbatim.
2. Create `types.h` with minimal struct definitions — only the fields
   these functions read/write:
   - `nfsd4_session`: `se_slots` (xarray), `se_fchannel.maxreqs`,
     `se_target_maxslots`, `se_slot_gen`
   - `nfsd4_slot`: `sl_seqid`, `sl_flags`, `sl_generation`
   - `nfsd4_sequence` (args): `sessionid`, `seqid`, `slotid`,
     `highest_slotid`, `cachethis`, `maxslots`
   - `nfsd4_compound_state`: `session`, `slot`
3. Stub in `stubs.c`:
   - `kfree` → REAL `free()` (ASan needs this for use-after-free!)
   - `kmalloc`/`kzalloc` → real `malloc`/`calloc`
   - `xa_load` → simple array lookup by index
   - `xa_store`/`xa_erase` → array store/remove
   - `spin_lock`/`spin_unlock` → no-op
   - `nfsd4_get_session_locked` → return pre-allocated session
   - `nfsd4_put_session_locked` → no-op
   - `nfsd4_sequence_check_conn` → return nfs_ok
   - `nfsd4_replay_cache_entry` → no-op
   - `is_session_dead` → return false
   - All `trace_*`, `printk`, `pr_*` → no-op
4. Compile with `-Wno-everything -fsanitize=address`

## Fuzz input (32 bytes)
- [0:16) sessionid — 16 raw bytes
- [16:20) seqid — u32
- [20:24) slotid — u32
- [24:28) highest_slotid — u32
- [28:32) target_maxslots — u32 (simulates shrinker state)

The harness sets up a session with 10 pre-allocated slots, sets
`se_target_maxslots` from the fuzz input to simulate the memory-pressure
shrinker having reduced the target, then calls `nfsd4_sequence`.

## Do NOT target
Everything else. This run is for `nfsd4_sequence` only.
