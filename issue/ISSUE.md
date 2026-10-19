# Issue #1621: Test Store.GC with missing untagged index entries

**Link:** https://github.com/oras-project/oras-go/issues/1621

## Summary

The `oras-go` library includes garbage collection (`Store.GC`) to clean up unreferenced blobs and metadata from an OCI layout store.

Prior to this issue, PR #1611 (and related issue #1603) implemented the logic in `gcReachable` to handle untagged index entries that point to non-existent blobs without error. However, only the low-level internal helper `gcReachable` was tested in memory (`TestGCReachable_MissingUntaggedEntry`). The high-level `Store.GC` end-to-end flow—loading from disk via `New(root)`, executing `s.GC(ctx)`, updating `index.json`, and cleaning disk blobs—lacked end-to-end regression test coverage.

## Nature of the Issue

This was a **missing test coverage issue**, not an unfixed bug in the library code. The core logic was present, but end-to-end regression testing was needed to ensure the public `Store.GC` API handles missing untagged entries properly across disk I/O boundaries.
