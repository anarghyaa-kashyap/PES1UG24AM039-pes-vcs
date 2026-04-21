# PES-VCS Lab Report

## Screenshots

- 1A: test_objects output — all tests passing
- 1B: find .pes/objects -type f — sharded directory structure
- 2A: test_tree output — all tests passing
- 2B: xxd of raw tree object
- 3A: pes init → pes add → pes status sequence
- 3B: cat .pes/index — text format index
- 4A: pes log — three commits
- 4B: find .pes -type f | sort — object store growth
- 4C: cat .pes/refs/heads/main and cat .pes/HEAD
- Final: make test-integration — all tests completed

## Phase 5: Branching and Checkout

### Q5.1
To implement pes checkout, two things must change in .pes/: HEAD must be updated to point to the new branch name, and the working directory must be updated to match the target branch tree. The complexity comes from having to recursively read the target commit tree and overwrite every tracked file in the working directory to match, while also creating or deleting files that differ between branches.

### Q5.2
To detect a dirty working directory conflict, for every file in the index, run stat() on the actual file and compare mtime and size to the stored index values. If they differ, the file has been modified since staging. Then check if that same file exists in the target branch tree with a different blob hash. If both conditions are true, refuse the checkout because the uncommitted change would be overwritten.

### Q5.3
In detached HEAD state, HEAD contains a raw commit hash instead of a branch reference like ref: refs/heads/main. Any new commits are written to the object store and HEAD is updated to point to the new commit hash directly, but no branch pointer advances. Once you move HEAD elsewhere, those commits become unreachable and invisible to pes log. To recover them, while still in detached HEAD state run pes log to note the commit hash, then create a new branch file manually: echo HASH > .pes/refs/heads/recovery, then update HEAD to point to it.

## Phase 6: Garbage Collection

### Q6.1
The algorithm is mark-and-sweep. Start from every file in .pes/refs/heads/ and collect all commit hashes. For each commit, add its hash to a reachable set, then follow its tree hash and recursively add all tree and blob hashes reachable from it, then follow the parent pointer and repeat. After full traversal, scan every file in .pes/objects/ and delete any whose hash is not in the reachable set. A hash set is the right data structure for O(1) lookup. For 100,000 commits and 50 branches, you would visit roughly 100,000 commits plus their trees and blobs — likely 500,000 to 2,000,000 objects total depending on how many files each commit touches.

### Q6.2
A race condition exists between GC and commit. A commit operation proceeds in stages: write blobs, write tree, write commit object, update HEAD. If GC runs between the blob/tree write and the HEAD update, it sees those new objects but HEAD has not moved yet, so they appear unreachable and gets deleted. The subsequent HEAD update then points to a commit whose tree and blobs no longer exist — silent corruption. Git avoids this by never collecting objects newer than a configurable grace period (default 2 weeks), so any object written recently is safe even if no ref points to it yet. GC also uses a lock file to prevent two GC processes from running simultaneously.
