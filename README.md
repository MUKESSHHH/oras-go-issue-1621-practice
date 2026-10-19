# oras-go Issue #1621 Practice: End-to-End Test for Store.GC

A hands-on practice repository exploring oras-go issue [#1621](https://github.com/oras-project/oras-go/issues/1621). This project adds an end-to-end regression test for `Store.GC` to verify how an OCI store handles dangling untagged entries in `index.json`. All test steps, outputs, and explanations are documented here for study and review.

## What I Did

1. **Found the Issue:** Explored open issues in `oras-project/oras-go` using ChatGPT's GitHub plugin and selected issue #1621.
2. **Wrote the Test:** Implemented `TestStore_GC_MissingUntaggedEntry` in `content/oci/oci_test.go` with Antigravity pairing.
3. **Verified Behavior:** Validated that the test passes with garbage collection enabled, and verified it fails (`len(index.Manifests) = 2, want 1`) when GC is bypassed.

## Repository Map & Suggested Reading Order

| Reading Order | Path | What's Inside |
| :---: | :--- | :--- |
| **1** | [`issue/ISSUE.md`](issue/ISSUE.md) | The issue background, link, and why end-to-end testing was needed. |
| **2** | [`docs/ISSUE_1621_EXPLAINED.md`](docs/ISSUE_1621_EXPLAINED.md) | Beginner-friendly explanation using a library & card-catalog analogy, glossary, and interview Q&A. |
| **3** | [`code/before/oci_test_gc_disabled.go.txt`](code/before/oci_test_gc_disabled.go.txt) | Mutation test ("bug demo") with GC disabled, demonstrating why the test catches failures. |
| **4** | [`code/after/oci_test_gc_fix.go.txt`](code/after/oci_test_gc_fix.go.txt) | The complete test function with GC active. |
| **5** | [`patch/1621-test.diff`](patch/1621-test.diff) | Clean patch diff applicable directly to `oras-go`. |
| **6** | [`outputs/`](outputs/) | Exact terminal outputs from both test runs and baseline failure documentation. |

## Before vs. After Results

| Run Scenario | Command | Result | Details |
| :--- | :--- | :---: | :--- |
| **GC Disabled (Mutation)** | `go test ./content/oci -run '^TestStore_GC_MissingUntaggedEntry$' -count=1 -v` | **FAIL** | `len(index.Manifests) = 2, want 1` ([`01_before_fail.txt`](outputs/01_before_fail.txt)) |
| **GC Enabled (Restored)** | `go test ./content/oci -run '^TestStore_GC_MissingUntaggedEntry$' -count=1 -v` | **PASS** | Clean run in 0.03s ([`02_after_pass.txt`](outputs/02_after_pass.txt)) |

## Honest Limits

- This is a local practice run; no commits, branches, or PRs were pushed upstream to `oras-project/oras-go`.
- Issue #1621 is closely related to duplicate issue #1603 and PR #1611.
- This was an exercise in understanding OCI layout internals, garbage collection graphs, and writing robust regression tests.

## Windows Baseline Failures

On a clean clone of `oras-go` on Windows without WSL, 5 existing tests fail due to POSIX vs. Windows permission handling and file locking:
- `TestStore_Success` (`index.json rename: Access is denied`)
- `TestStore_GCRemoveError`
- `TestStorage_UnsafePush_FileErrors`
- `TestVerify_Layout_TempDir_ReadOnly`
- `TestVerify_TempDir_ReadOnly`

These 5 failures are unrelated to issue #1621 and occur on unmodified upstream code on Windows.

## How to Re-Run This Test

1. Clone `oras-go`:
   ```powershell
   git clone https://github.com/oras-project/oras-go.git
   cd oras-go
   ```
2. Apply the patch:
   ```powershell
   git apply /path/to/patch/1621-test.diff
   ```
3. Run the targeted test:
   ```powershell
   go test ./content/oci -run '^TestStore_GC_MissingUntaggedEntry$' -count=1 -v
   ```
