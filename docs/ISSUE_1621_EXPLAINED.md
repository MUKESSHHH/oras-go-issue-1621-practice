# Understanding oras-go Issue #1621: Store.GC with Missing Untagged Entries

> **Honest Limits Note:** This report documents a local learning exercise and practice run. No commits or pull requests were pushed to the upstream repository. Upstream issue #1621 is related to duplicate issue #1603 and PR #1611.

---

## 1. The Issue in 3 Sentences

Sometimes a store’s index lists an untagged manifest whose actual file on disk is gone or was never written. When the store runs garbage collection (GC), it must quietly drop that ghost entry from the index without crashing or throwing an error. A low-level test already proved that the graph-walking logic worked, but issue #1621 asked for a true end-to-end test verifying that `Store.GC` correctly reloads the index from disk, cleans it up, and saves the updated index back to the filesystem.

---

## 2. Everyday Example: The Library and the Card Catalog

To make everything easy to picture, imagine a public library:

| Library Analogy | Real OCI Concept | What It Means |
| :--- | :--- | :--- |
| **The Library** | `Store` (OCI Store) | The storage folder on your computer that holds everything. |
| **Card Catalog** | `index.json` | The inventory file listing which items exist in the library. |
| **Catalog Card** | Index Entry / Descriptor | A single card with details (size, fingerprint) describing a book. |
| **Book on the Shelf** | Blob | The actual data file stored on the disk. |
| **Name Tag (Sticky Note)** | Tag (e.g. `"kept"`, `"v1.0"`) | A human-friendly label stuck on a card so people can find it by name. |
| **Unlabeled Card** | Untagged Entry | A catalog card that only has a fingerprint number, no human name. |
| **Missing Book** | Dangling / Missing Entry | A catalog card points to a shelf spot, but the book is physically missing. |
| **Uncataloged Box on Floor** | Orphan Blob | A file sitting on the shelf that no catalog card points to. |
| **Night Cleaner** | Garbage Collector (`GC`) | A background worker that inspects the cards, keeps what matters, removes junk, and throws away orphan boxes. |

In this analogy:
- If a card has a name tag (like `"kept"`), the cleaner must protect it and all books it references.
- If an uncataloged box is lying on the shelf (an orphan blob), the cleaner throws it out.
- If someone left an unlabeled card for a book that does not exist (a missing untagged entry), the cleaner must safely throw away the dead card without panicking or giving up.

---

## 3. Glossary

- **GC (Garbage Collection):** Automatic cleanup of files that are no longer wanted or referenced.  
  *Example:* Deleting temporary download files that are no longer needed.
- **OCI (Open Container Initiative):** An industry standard for packaging and storing container images and artifacts.  
  *Example:* The standard file layout used by Docker and container registries.
- **Store (`Store`):** The Go object managing an OCI layout folder on disk.  
  *Example:* `s, err := oci.New("./my-store")`.
- **`index.json`:** The root catalog file in an OCI store listing all top-level entries.  
  *Example:* A JSON file containing `{"schemaVersion": 2, "manifests": [...]}`.
- **Manifest:** A small JSON file that lists the recipe of an image (its config and layers).  
  *Example:* A text recipe saying "this image uses Config A and Layers B and C".
- **Blob:** An arbitrary binary data file stored by its cryptographic hash.  
  *Example:* A layer `.tar.gz` file or a config JSON file stored under `blobs/sha256/<hash>`.
- **Config:** A specific blob describing how to run a container (environment variables, entrypoint).  
  *Example:* `{}` (an empty JSON object used as minimal container configuration).
- **Layer:** A blob containing filesystem changes (files and directories).  
  *Example:* A blob containing the text `"orphan"`.
- **Digest:** A cryptographic fingerprint (hash) that uniquely identifies content.  
  *Example:* `sha256:ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad`.
- **Descriptor:** A small metadata receipt containing a blob's media type, digest, and size in bytes.  
  *Example:* `{mediaType: "application/vnd.oci.image.manifest.v1+json", digest: "sha256:...", size: 120}`.
- **Tag / Untagged:** A tag is a human-friendly name (like `"latest"` or `"kept"`). An untagged item is known only by its raw digest.  
  *Example:* Tagged = `my-app:v1`; Untagged = `sha256:1234abcd...`.
- **Dangling Entry:** An entry in `index.json` whose matching blob cannot be found in the store.  
  *Example:* A catalog card pointing to a file that was deleted or never copied.
- **Orphan Blob:** A real file sitting on disk in the `blobs/` directory that nothing references.  
  *Example:* An old layer left behind when an operation was cancelled.
- **Reachable:** Content that can be traced starting from any tagged manifest.  
  *Example:* If tag `"kept"` points to Manifest M, and M lists Config C, both M and C are reachable.
- **Atomic Write:** Saving a file by writing to a temporary file first, then instantly renaming it, so readers never see half-written files.  
  *Example:* Writing `index.json.tmp` and replacing `index.json` in one instant step.

---

## 4. How `Store.GC` Works (Step by Step)

When you call `s.GC(ctx)` in [`content/oci/oci.go`](file:///D:/VIBEXIO/LEARNING/issue/oras-go/content/oci/oci.go), here is what happens:

```
[Store.GC]
   │
   ├── 1. Lock store (s.sync.Lock())
   │
   ├── 2. Re-read index & build reachability graph (s.gcIndex -> gcReachable)
   │        ├── Keep all tagged manifests and their children
   │        ├── Walk subjects of untagged manifests (skip if not found)
   │        └── Drop unreachable & missing entries
   │
   ├── 3. Save cleaned index to disk (s.saveIndexFrom -> writeIndexFile)
   │
   ├── 4. Sweep blobs folder on disk (os.ReadDir -> os.Remove)
   │        └── Delete any blob file not in reachable graph
   │
   └── 5. Opportunistically clean temp leftovers (s.gcLeftovers)
```

1. **Locking (`s.sync.Lock`):**  
   The store locks itself so no other operation writes files while cleanup is running.

2. **Re-evaluating Reachability (`s.gcIndex` & `gcReachable`):**  
   The cleaner inspects all references:
   - It traverses every tagged manifest and indexes all referenced child blobs into a memory graph (`graph.IndexAll`).
   - For untagged entries, it checks whether they refer to something that reaches a tagged manifest via `manifestutil.Subject`. If an untagged entry's file is missing (`errdef.ErrNotFound`), it breaks the loop without error and discards it.

3. **Writing the Updated Index (`s.saveIndexFrom` & `s.writeIndexFile`):**  
   If `AutoSaveIndex` is true, the store converts the surviving items into a new `index.json` and writes it atomically to disk. The dead untagged entry is now gone from `index.json`.

4. **Purging Garbage Blobs (`os.ReadDir` & `os.Remove`):**  
   The store inspects each directory inside `blobs/sha256/`. If a blob file on disk is not in the set of reachable nodes, the store deletes it using `os.Remove`. This is where orphan blobs get deleted.

5. **Cleaning Stale Temp Files (`s.gcLeftovers`):**  
   It sweeps up interrupted write artifacts and temporary files older than one hour.

---

## 5. What Was Missing and Why It Matters

Before this task, the codebase contained a test named `TestGCReachable_MissingUntaggedEntry`. However:
- That test only called the internal helper `gcReachable(...)` in memory.
- It passed an artificial in-memory map of descriptors (`map[string]ocispec.Descriptor`).
- It **did not** test:
  - Reading a corrupt or dangling `index.json` from the real filesystem via `New(root)`.
  - The high-level `Store.GC(ctx)` method.
  - The serialization and atomic writing of `index.json` back to disk.
  - The physical deletion of orphan blobs from disk while handling the missing entry.

In real life, bugs often happen at the boundary between disk I/O and business logic. Testing only the in-memory helper meant that if `New(root)` or `s.saveIndexFrom()` choked on a missing untagged entry, no test would catch it. The new end-to-end test fills this exact gap.

---

## 6. What the Test Does (Step by Step, Mapped to the Analogy)

The test function is [`TestStore_GC_MissingUntaggedEntry`](file:///D:/VIBEXIO/LEARNING/issue/oras-go/content/oci/oci_test.go#L4907-L4996).

| Step | Go Code in Test | Library Analogy |
| :--- | :--- | :--- |
| **1. Create valid tagged content** | `New(root)`. Push config `{}` and manifest referencing it. `Tag(ctx, taggedDesc, "kept")`. | Buy a new book, file its catalog card, and put a sticky note labelled `"kept"` on it. |
| **2. Create an orphan blob** | Push layer `orphan`. Do not reference or tag it. | Leave a random unlabeled box (`"orphan"`) sitting on a shelf. It is not listed on any card. |
| **3. Inject dangling untagged entry** | Read `s.indexPath`, unmarshal, append untagged descriptor for `"missing manifest"` (no blob pushed), write back to `index.json`. | Sneak a catalog card into the drawer for a book called `"missing manifest"`, but do not put any actual book on the shelf. No name tag on this card. |
| **4. Reload and run GC** | Reload with `New(root)`, call `s.GC(ctx)`. Assert error is `nil`. | Bring in the night cleaner (`GC`). The cleaner must not panic or abort when noticing the ghost card. |
| **5. Verify `index.json`** | Read `index.json` from disk. Assert `len(Manifests) == 1`, and it has tag `"kept"`. | Open the card drawer. Verify exactly one card remains: the `"kept"` book. The ghost card was thrown away. |
| **6. Verify disk blobs** | `Exists(tagged manifest)` is `true`; `Exists(orphan)` is `false`. | Check the shelves. The `"kept"` book is safe. The random `"orphan"` box was hauled off to the dumpster. |

---

## 7. Before vs. After: Proving the Test is Meaningful

To prove the test is genuine and not a "false pass," we ran a mutation check by commenting out `s.GC(ctx)`.

### Test Run A: With `s.GC(ctx)` Commented Out (Expected Failure)

When `s.GC(ctx)` does not run, the injected dangling card is never removed from `index.json`:

```
PS D:\VIBEXIO\LEARNING\issue\oras-go> go test ./content/oci -run '^TestStore_GC_MissingUntaggedEntry$' -count=1 -v
=== RUN   TestStore_GC_MissingUntaggedEntry
    oci_test.go:4977: len(index.Manifests) = 2, want 1
--- FAIL: TestStore_GC_MissingUntaggedEntry (0.03s)
FAIL
FAIL    github.com/oras-project/oras-go/v3/content/oci  1.321s
FAIL
```
*Meaning:* The test caught the problem! Because GC was skipped, `index.json` still had 2 manifests instead of 1.

### Test Run B: With `s.GC(ctx)` Active (Passing)

When `s.GC(ctx)` runs as intended:

```
PS D:\VIBEXIO\LEARNING\issue\oras-go> go test ./content/oci -run '^TestStore_GC_MissingUntaggedEntry$' -count=1 -v
=== RUN   TestStore_GC_MissingUntaggedEntry
--- PASS: TestStore_GC_MissingUntaggedEntry (0.03s)
PASS
ok      github.com/oras-project/oras-go/v3/content/oci  1.390s
```
*Meaning:* `Store.GC` successfully pruned the missing untagged entry, saved the clean index with only 1 manifest, deleted the orphan blob, and preserved the tagged manifest.

---

## 8. Likely Interview Questions & Short Answers

### Q1: Why did this issue ask for a test rather than a code fix?
**Answer:** The underlying fix inside `gcReachable` was already written, but only unit-tested against an in-memory map. Issue #1621 was created specifically because the project needed regression test coverage of the complete `Store.GC` user-facing API interacting with actual disk files.

### Q2: How do you know this test doesn't pass by accident?
**Answer:** We performed negative verification (fault injection) by commenting out `s.GC(ctx)`. The test immediately failed with `len(index.Manifests) = 2, want 1`. This proves the test directly measures the effects of `s.GC`.

### Q3: Why does `New(root)` not fail when loading an index with a missing blob?
**Answer:** In `readonlyoci.go`, `loadIndex` calls `graph.IndexAll`. When indexing successors, if a node's blob is missing on disk, `IndexAll` checks `errors.Is(err, errdef.ErrNotFound)` and skips it. The store is designed to tolerate incomplete content on load so recovery and GC can fix it.

### Q4: Why did we test both a missing untagged manifest and an orphan blob?
**Answer:** To verify both responsibilities of `Store.GC` in a single run: cleaning up metadata in `index.json` (removing the dangling manifest descriptor) and cleaning up unreferenced files in `blobs/` (deleting the orphan layer).

### Q5: Why do 5 other tests in `content/oci` fail on Windows on a clean clone?
**Answer:** Those 5 tests (`TestStore_Success`, `TestStore_GCRemoveError`, `TestStorage_UnsafePush_FileErrors`, `TestVerify_Layout_TempDir_ReadOnly`, `TestVerify_TempDir_ReadOnly`) rely on Unix-specific file permissions (`os.Chmod` with `0555` or read-only directories) or POSIX file replacement semantics. Windows treats file locks and permission bits differently, causing those specific tests to fail in Windows environments without WSL.

### Q6: What is the difference between an untagged manifest and an orphan blob?
**Answer:** An untagged manifest is listed inside `index.json` (the catalog), but only identified by its digest rather than a friendly tag name. An orphan blob is physically stored in the `blobs/` folder on disk, but is not referenced by any manifest or catalog entry.

### Q7: Why did the test write directly to `s.indexPath` using `os.WriteFile` in step 3?
**Answer:** Normal store methods like `Push` or `Tag` enforce integrity and will not write an entry for a blob that does not exist. To simulate a corrupted store or an interrupted operation leaving a dangling entry, the test had to directly manipulate `index.json`.
