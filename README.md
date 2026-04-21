# PES-VCS

PES-VCS is a minimal, Git-inspired version control system implemented in C. It stores content-addressed objects (blobs, trees, commits), stages changes in a text index, and supports commit history traversal.

## Features Implemented

- Content-addressable object store with SHA-256 hashing, deduplication, and integrity checks
- Tree object serialization and reconstruction from the staging index
- Text-based staging index with atomic save and metadata tracking
- Commit creation, parent linking, and log traversal
- CLI commands: `init`, `add`, `status`, `commit -m`, `log`

## Build and Run

Prerequisites (Linux or WSL recommended):

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

Build:

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove build artifacts
```

Optional tests:

```bash
make test_objects
./test_objects

make test_tree
./test_tree

make test-integration
```

Author configuration (used in commits):

```bash
export PES_AUTHOR="Your Name <PESXUG24CS042>"
```

If unset, the default is `PES User <pes@localhost>`.

## Usage Examples

```bash
./pes init

echo "hello" > hello.txt
./pes add hello.txt
./pes status

./pes commit -m "Initial commit"
./pes log
```

## Implementation by Phase

Phase 1: Object storage
- Implemented `object_write` and `object_read` with SHA-256 hashing, deduplication, and integrity checks.

Phase 2: Tree objects
- Implemented `tree_from_index` to build hierarchical trees from the index and write tree objects.

Phase 3: Index (staging area)
- Implemented `index_load`, `index_save` (atomic write), and `index_add` for staging blobs.

Phase 4: Commits and history
- Implemented `commit_create` to build commits from the index and update HEAD/refs.

## Design Notes and Limitations

- Objects are stored as `<type> <size>\0<data>` and named by SHA-256 hash.
- Object storage is sharded by the first two hex characters and written atomically (temp + rename).
- The index is a simple text file; status checks use file size and mtime for fast change detection.
- Commits snapshot the staged index, not the working directory.
- `pes add` stages regular files only; directories are represented through tree construction.
- Not implemented: branching, checkout, merge, or garbage collection.

## Analysis Answers

Q5.1: Implementing `pes checkout <branch>`
Change `.pes/HEAD` to point to `refs/heads/<branch>` (or create it if missing), then update the working directory to match the target commit's tree. This is complex because files must be added, modified, or deleted to match the tree, and conflicts must be detected when the working directory is dirty.

Q5.2: Detecting dirty working directory conflicts
Compare each tracked file's index metadata (mtime, size, and hash) with the working directory. If a tracked file is modified in the working directory and the target branch's tree has a different version, refuse checkout. This can be done by loading the current index, re-hashing only the files that have changed metadata, and comparing against the tree object from the target commit.

Q5.3: Detached HEAD behavior and recovery
When HEAD points directly to a commit hash, new commits advance HEAD but do not move any branch ref. Those commits become unreachable by branch names. A user can recover by creating a branch pointing to the detached commit (e.g., `refs/heads/recover`) before it is garbage-collected.

Q6.1: Garbage collection algorithm
Start from all branch refs and HEAD, traverse reachable commits, then their trees and blobs. Use a hash set to mark reachable object IDs. After traversal, delete any objects not in the reachable set. For 100,000 commits and 50 branches, the traversal visits up to 100,000 commits plus all referenced trees and blobs; the exact count depends on project size and sharing, but all reachable objects must be visited once.

Q6.2: GC race condition and prevention
If GC runs while a commit is being written, it could delete a newly written object before the branch ref is updated, causing the commit to reference missing objects. Git avoids this with lock files, atomic renames, and grace periods (or marking recent objects) so that objects created during active operations are not collected.

## Storage Notes (Reference)

```python
# Pseudocode
def store_object(content):
    hash = sha256(content)
    path = f".pes/objects/{hash[0:2]}/{hash[2:]}"
    write_file(path, content)
    return hash
```

This gives us:
- **Deduplication:** Identical files stored once
- **Integrity:** Hash verifies data isn't corrupted
- **Immutability:** Changing content = different hash = different object

Objects are sharded by the first two hex characters to avoid huge directories:

```
.pes/objects/
├── 2f/
│   └── 8a3b5c7d9e...
├── a1/
│   ├── 9c4e6f8a0b...
│   └── b2d4f6a8c0...
└── ff/
    └── 1234567890...
```

## Exploring a Real Git Repository

You can inspect Git's internals yourself:

```bash
mkdir test-repo && cd test-repo && git init
echo "Hello" > hello.txt
git add hello.txt && git commit -m "First commit"

find .git/objects -type f          # See stored objects
git cat-file -t <hash>            # Show type: blob, tree, or commit
git cat-file -p <hash>            # Show contents
cat .git/HEAD                     # See what HEAD points to
cat .git/refs/heads/main          # See branch pointer
```

---

## What You'll Build

PES-VCS implements five commands across four phases:

```
pes init              Create .pes/ repository structure
pes add <file>...     Stage files (hash + update index)
pes status            Show modified/staged/untracked files
pes commit -m <msg>   Create commit from staged files
pes log               Walk and display commit history
```

The `.pes/` directory structure:

```
my_project/
├── .pes/
│   ├── objects/          # Content-addressable blob/tree/commit storage
│   │   ├── 2f/
│   │   │   └── 8a3b...   # Sharded by first 2 hex chars of hash
│   │   └── a1/
│   │       └── 9c4e...
│   ├── refs/
│   │   └── heads/
│   │       └── main      # Branch pointer (file containing commit hash)
│   ├── index             # Staging area (text file)
│   └── HEAD              # Current branch reference
└── (working directory files)
```

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                      WORKING DIRECTORY                        │
│                  (actual files you edit)                       │
└───────────────────────────────────────────────────────────────┘
                              │
                        pes add <file>
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           INDEX                               │
│                (staged changes, ready to commit)              │
│                100644 a1b2c3... src/main.c                    │
└───────────────────────────────────────────────────────────────┘
                              │
                       pes commit -m "msg"
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                       OBJECT STORE                            │
│  ┌───────┐    ┌───────┐    ┌────────┐                         │
│  │ BLOB  │◄───│ TREE  │◄───│ COMMIT │                         │
│  │(file) │    │(dir)  │    │(snap)  │                         │
│  └───────┘    └───────┘    └────────┘                         │
│  Stored at: .pes/objects/XX/YYY...                            │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           REFS                                │
│       .pes/refs/heads/main  →  commit hash                    │
│       .pes/HEAD             →  "ref: refs/heads/main"         │
└───────────────────────────────────────────────────────────────┘
```

## Further Reading

- **Git Internals** (Pro Git book): https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain
- **Git from the inside out**: https://codewords.recurse.com/issues/two/git-from-the-inside-out
- **The Git Parable**: https://tom.preston-werner.com/2009/05/19/the-git-parable.html
