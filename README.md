<h1 align="center">ParallelGit</h1>

<p align="center">
  <strong>See a single file across every branch — at once.</strong>
</p>

<p align="center">
  <a href="https://github.com/sathnindu/parallelgit/releases"><img src="https://img.shields.io/github/v/release/sathnindu/parallelgit?style=flat-square" alt="Release"></a>
  <a href="https://github.com/sathnindu/parallelgit/actions"><img src="https://img.shields.io/github/actions/workflow/status/sathnindu/parallelgit/release.yml?style=flat-square" alt="Build"></a>
  <a href="https://github.com/sathnindu/parallelgit/blob/main/LICENSE"><img src="https://img.shields.io/github/license/sathnindu/parallelgit?style=flat-square" alt="License"></a>
  <a href="https://goreportcard.com/report/github.com/sathnindu/parallelgit"><img src="https://goreportcard.com/badge/github.com/sathnindu/parallelgit?style=flat-square" alt="Go Report"></a>
</p>

---

## What it does

`git diff` shows you two branches. ParallelGit shows you **N**.

Pick a file. Pick any number of branches. ParallelGit produces a single unified view: lines that are identical across every branch collapse into one block, and lines that diverge fan out — one variant per branch, labeled, color-coded, with the last author and date that touched each one.

```
$ parallelgit diff src/main/java/UserService.java -b master,uat,dev

UserService.java  |  master (a3f9c12)  |  uat (b7e2d05)  |  dev (c4f1e88)

--- lines 1-40 identical across all branches ---

+----------------------------------------------------------------------+
| master   2025-03-12   sathnindu                                      |
|   public User findById(Long id) {                                    |
|       return repo.findById(id).orElseThrow();                        |
|   }                                                                  |
+----------------------------------------------------------------------+
| uat      2025-04-22   sathnindu                                      |
|   public User findById(Long id) {                                    |
|       return repo.findById(id)                                       |
|           .orElseThrow(() -> new UserNotFoundException(id));         |
|   }                                                                  |
+----------------------------------------------------------------------+
| dev      2025-05-18   sathnindu                                      |
|   public Optional<User> findById(Long id) {                          |
|       return repo.findById(id);                                      |
|   }                                                                  |
+----------------------------------------------------------------------+

--- lines 56-112 identical across all branches ---
```

That's the entire pitch. If you've ever maintained release branches, run multi-tenant forks, or backported fixes across versions — you already know why this view should have existed years ago.

## Why this exists

Every Git tool today is **branch-at-a-time**. You check out `dev`, you see `dev`. You diff two branches, you see a two-column wall of red and green. Nobody actually understands how a single function diverged across five branches without juggling tabs in their head.

ParallelGit treats a file the way Git actually treats it: as something that exists in many parallel states simultaneously. The shared regions collapse. The divergent regions stack. You see the truth of the file across your whole repo, in one view.

## Install

### macOS & Linux (Homebrew)

```bash
brew install sathnindu/parallelgit/parallelgit
```

### Windows (Scoop)

```powershell
scoop bucket add parallelgit https://github.com/sathnindu/scoop-parallelgit
scoop install parallelgit
```

### Debian / Ubuntu

```bash
curl -LO https://github.com/sathnindu/parallelgit/releases/latest/download/parallelgit_linux_amd64.deb
sudo dpkg -i parallelgit_linux_amd64.deb
```

### Fedora / RHEL

```bash
curl -LO https://github.com/sathnindu/parallelgit/releases/latest/download/parallelgit_linux_amd64.rpm
sudo rpm -i parallelgit_linux_amd64.rpm
```

### Go

```bash
go install github.com/sathnindu/parallelgit/cmd/parallelgit@latest
```

### Anywhere else

Download a prebuilt binary from the [Releases page](https://github.com/sathnindu/parallelgit/releases) — macOS (Intel & ARM), Linux (amd64 & arm64), Windows (amd64 & arm64). No runtime, no dependencies.

## Quick start

```bash
# Compare a file across three branches
parallelgit diff UserService.java -b master,uat,dev

# Use the current branch + master as default
parallelgit diff UserService.java

# Compare across every branch that has touched this file
parallelgit diff UserService.java --all

# Output JSON for piping into other tools
parallelgit diff UserService.java -b master,dev -f json

# List every branch that has touched this file, with last-modified dates
parallelgit branches UserService.java
```

## Commands

### `parallelgit diff <file>`

The hero command. Shows a file across N branches.

| Flag | Description |
|---|---|
| `-b, --branches <list>` | Comma-separated branch names. Defaults to current branch + `main`/`master`. |
| `--all` | Include every branch that has touched the file. |
| `-a, --anchor <branch>` | Anchor branch for diff alignment. Defaults to the merge-base of selected branches. |
| `-f, --format <fmt>` | Output format: `compact` (default), `json`, `wide`. |
| `-r, --repo <path>` | Path to the Git repository. Defaults to current directory. |
| `--no-color` | Disable ANSI colors. Automatic when output is not a terminal. |

### `parallelgit branches <file>`

Lists every local branch that has ever touched the given file, with the date and author of the last modification on each branch. Useful for answering "where did this code go?"

## Output formats

### `compact` (default)

Human-optimized terminal output. Shared regions collapse to a single dim line. Divergent regions render as bordered blocks, one labeled variant per branch.

### `json`

Structured output for tooling. The schema is stable across versions:

```json
{
  "file": "UserService.java",
  "anchor_branch": "master",
  "branches": [
    { "name": "master", "head_sha": "a3f9c12...", "merge_base_sha": "0fe1d22..." },
    { "name": "uat",    "head_sha": "b7e2d05...", "merge_base_sha": "0fe1d22..." },
    { "name": "dev",    "head_sha": "c4f1e88...", "merge_base_sha": "0fe1d22..." }
  ],
  "segments": [
    {
      "kind": "shared",
      "range": [1, 40],
      "lines": ["package com.arimac.paylater;", "..."]
    },
    {
      "kind": "divergent",
      "variants": {
        "master": { "lines": ["..."], "last_modified": "2025-03-12T10:14:00Z", "last_author": "sathnindu" },
        "uat":    { "lines": ["..."], "last_modified": "2025-04-22T09:01:00Z", "last_author": "sathnindu" },
        "dev":    { "lines": ["..."], "last_modified": "2025-05-18T16:33:00Z", "last_author": "sathnindu" }
      }
    }
  ]
}
```

Pipe it into `jq`, `fx`, or build your own renderer.

### `wide`

Side-by-side columns, one per branch. Best on a wide terminal (>140 cols).

## When ParallelGit is useful

- **Maintaining release branches.** Did the hotfix make it into `release/1.4`, `release/1.5`, *and* `master`?
- **Multi-tenant forks.** Your codebase has `client-a`, `client-b`, `client-c` branches that each customize the same modules. ParallelGit shows you how they've drifted.
- **Long-lived feature branches.** Before merging a 3-month-old branch, see exactly which lines diverge from main *now*, not when the branch was created.
- **Incident review.** When prod is broken and staging is fine, ParallelGit shows you every line of the suspect file across both branches in one pass.
- **Code review.** Reviewing a PR? Run ParallelGit on the touched files across `pr-branch`, `main`, and the previous release to see the full picture.

## When it's not for you

- **Trunk-based teams.** If you merge to `main` daily and never have long-lived branches, you don't need this. Plain `git diff` works fine.
- **Massive files.** Files >5000 lines render slowly. Compact mode helps, but very large files are not the design target.
- **Binary files.** ParallelGit is line-oriented. Binary diffs aren't supported.

## How it works

ParallelGit is a single static Go binary built on [`go-git`](https://github.com/go-git/go-git). For each invocation:

1. Resolve the selected branches and compute their merge-base (the anchor).
2. Load the file's content at each branch tip.
3. Run a line-level diff of each branch against the anchor.
4. Walk the anchor file once, classifying each line as **shared** (every branch agrees) or **divergent** (at least one branch differs).
5. Group consecutive lines of the same kind into segments.
6. Render the segments in the chosen format.

No daemon, no index, no server. Just the binary and your repo.

## Roadmap

ParallelGit is in active development. Honest status of what's shipped and what isn't:

**Shipped (v0.1)**
- N-way diff across any number of branches
- Compact, JSON, and wide output formats
- Branch listing per file
- Cross-platform binaries (macOS, Linux, Windows)
- Homebrew, Scoop, deb, and rpm distribution

**On the roadmap**
- `parallelgit drift` — surface lines that have diverged from a chosen "source of truth" branch and for how long
- `parallelgit blame` — multi-branch blame, showing per-branch authorship per line
- Editor integration — VS Code & JetBrains extensions calling the binary
- Web view — shareable links with `parallelgit serve`
- Multi-branch edit — write a fix once, generate patches for N branches with conflict preview

**Not planned**
- Becoming a full Git client. Tower, GitKraken, and Fork exist; ParallelGit stays focused on the one thing they don't do.
- Mercurial/Jujutsu support. Maybe one day. Not soon.

## Building from source

```bash
git clone https://github.com/sathnindu/parallelgit
cd parallelgit
go build -o parallelgit ./cmd/parallelgit
./parallelgit --version
```

Requires Go 1.22 or later.

## Contributing

Issues and PRs welcome — but please open an issue describing the change before submitting a large PR. ParallelGit's value comes from staying focused; feature creep is the main risk.

If you've used the tool on a real codebase and have feedback on the output format, the CLI ergonomics, or the JSON schema, that feedback is the single most valuable thing you can contribute right now.

## License

MIT. See [LICENSE](LICENSE).

## Acknowledgments

Built with [`go-git`](https://github.com/go-git/go-git), [`cobra`](https://github.com/spf13/cobra), and [`go-diff`](https://github.com/sergi/go-diff). Inspired by every developer who has ever had to ask "which branch is this fix actually on?"

Unrelated to [`beijunyi/parallelgit`](https://github.com/beijunyi/parallelgit), a Java NIO Git filesystem library.

---

<p align="center">
  Built by <a href="https://github.com/sathnindu">Sathnindu</a> · <a href="https://parallelgit.dev">parallelgit.dev</a>
</p>
