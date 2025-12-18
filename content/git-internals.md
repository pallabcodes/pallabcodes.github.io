+++
title = "Git Internals: How Git Really Works"
date = 2024-12-09
description = "Understanding Git's object model and how it enables powerful version control."
[taxonomies]
tags = ["git", "version-control", "internals"]
+++

Most developers use Git daily without understanding how it works internally. Let's peek under the hood.

<!-- more -->

## Git is a Content-Addressable Filesystem

At its core, Git is a key-value store where keys are SHA-1 hashes:

```bash
$ echo "Hello, Git!" | git hash-object --stdin
af5626b4a114abcb82d63db7c8082c3c4756e51b
```

The same content always produces the same hash. This is fundamental.

## The Four Object Types

Git stores everything as one of four object types:

```
┌─────────────────────────────────────────────────────┐
│                   Git Object Types                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│  blob   - File contents (no filename!)              │
│  tree   - Directory listing (names + blob refs)     │
│  commit - Snapshot + metadata + parent ref          │
│  tag    - Named reference to a commit               │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## Blob: Storing File Contents

A blob is just content, no metadata:

```bash
# Create a blob
$ echo "Hello" | git hash-object -w --stdin
ce013625030ba8dba906f756967f9e9ca394464a

# View the blob
$ git cat-file -p ce01362
Hello
```

## Tree: Directory Structure

A tree references blobs and other trees:

```bash
$ git cat-file -p HEAD^{tree}
100644 blob 3b18e512...   README.md
100644 blob f5512345...   index.js
040000 tree a1b2c3d4...   src/
```

Notice: trees contain names, blobs don't. The same file content in different locations is stored once.

## Commit: The Snapshot

A commit ties it all together:

```bash
$ git cat-file -p HEAD
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent 8a3b5c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b
author Alice <alice@example.com> 1702234567 +0000
committer Alice <alice@example.com> 1702234567 +0000

Add initial implementation
```

## How Branches Work

A branch is just a file containing a commit hash:

```bash
$ cat .git/refs/heads/main
8a3b5c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b
```

That's it. When you "create a branch," Git creates a 41-byte file.

## The Commit Graph

```
     A ← B ← C ← D  ← main
              ↖
               E ← F  ← feature
```

Each commit points to its parent(s). Merge commits have multiple parents.

## What Actually Happens During Common Operations

### git add

```bash
# 1. Hash file content → create blob
# 2. Update index (.git/index) with blob reference
$ git add file.txt
```

### git commit

```bash
# 1. Create tree from index
# 2. Create commit pointing to tree + parent
# 3. Update branch ref to new commit
$ git commit -m "message"
```

### git merge

```bash
# 1. Find common ancestor (merge base)
# 2. Create combined tree
# 3. Create commit with TWO parents
$ git merge feature
```

## Practical Implications

### Why rebasing "rewrites history"

```
Before rebase:
A ← B ← C ← D  (main)
     ↖
      E ← F   (feature)

After rebase:
A ← B ← C ← D ← E' ← F'  (main + feature)
```

E' and F' are NEW commits (new hashes) because their parent changed.

### Why Git is fast

```bash
# Same content = same hash = stored once
$ find .git/objects -type f | wc -l
147  # Even with thousands of files
```

Git deduplicates automatically via content-addressing.

### Recovery: The reflog

```bash
# Git tracks all ref changes
$ git reflog
8a3b5c8 HEAD@{0}: commit: Add feature
f1e2d3c HEAD@{1}: checkout: moving from main to feature
...

# Recover "lost" commits
$ git checkout HEAD@{5}
```

## Key Takeaways

1. **Git is a content-addressed store** - Hashes are keys
2. **Blobs store content, trees store names** - Deduplication
3. **Branches are just pointers** - 41-byte files
4. **Commits are immutable** - "Modifying" creates new commits
5. **The reflog is your safety net** - Hard to permanently lose work

Understanding Git's internals makes complex operations intuitive.
