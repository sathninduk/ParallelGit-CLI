# ParallelGit — Complete Implementation Plan

A single document covering everything from `mkdir parallelgit` to first public release. Built for Go + Claude Code, optimized for actually shipping in 2-3 weeks of evening work.

---

## Part 1: Foundations

### 1.1 Tech stack (locked in)

| Concern | Choice | Why |
|---|---|---|
| Language | Go 1.22+ | Single binary, fast startup, trivial cross-compile |
| Git access | `github.com/go-git/go-git/v5` | Pure Go, no CGO, no libgit2 |
| CLI framework | `github.com/spf13/cobra` | Standard for Go CLIs |
| Diff algorithm | `github.com/sergi/go-diff/diffmatchpatch` | Mature line-mode LCS |
| Colors | `github.com/fatih/color` | TTY-aware ANSI |
| Terminal width | `golang.org/x/term` | Width detection |
| Testing | stdlib + `github.com/stretchr/testify` | Idiomatic Go testing |
| Release automation | GoReleaser | The Go CLI distribution standard |

### 1.2 Project structure

```
parallelgit/
├── cmd/
│   └── parallelgit/
│       └── main.go              # binary entry point
├── internal/
│   ├── core/                    # PURE — no I/O
│   │   ├── types.go
│   │   ├── diff.go
│   │   ├── diff_test.go
│   │   └── load.go              # orchestrator: calls git + core
│   ├── git/                     # all Git I/O
│   │   ├── repo.go
│   │   └── repo_test.go
│   └── render/                  # output formatting
│       ├── compact.go
│       ├── compact_test.go
│       ├── json.go
│       └── wide.go
├── docs/
│   ├── demo.tape                # VHS recording script
│   └── demo.gif                 # generated demo
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── .goreleaser.yaml
├── go.mod
├── go.sum
├── CLAUDE.md                    # Claude Code persistent context
├── README.md
├── LICENSE
├── notes.md                     # YOUR dogfooding log (gitignored)
└── .gitignore
```

The `internal/` layout enforces a critical discipline: `core/` is pure (no Git calls), `git/` is the only place that touches the repo, and `render/` is the only place that produces output. This separation makes every layer independently testable and keeps the JSON schema as the contract between layers.

### 1.3 Day 0 setup (45 minutes)

```bash
# Create and initialize
mkdir parallelgit && cd parallelgit
git init
go mod init github.com/sathnindu/parallelgit

# Directory scaffolding
mkdir -p cmd/parallelgit internal/core internal/git internal/render docs .github/workflows

# Dependencies
go get github.com/go-git/go-git/v5
go get github.com/spf13/cobra
go get github.com/sergi/go-diff/diffmatchpatch
go get github.com/fatih/color
go get golang.org/x/term
go get github.com/stretchr/testify

# .gitignore
cat > .gitignore <<EOF
dist/
parallelgit
parallelgit.exe
notes.md
.DS_Store
*.log
coverage.out
EOF

# Initial commit
git add -A && git commit -m "chore: project scaffolding"
```

### 1.4 The CLAUDE.md file

This goes at the repo root. Claude Code reads it every session — it's how you keep context across many sessions without re-explaining the project each time.

```markdown
# ParallelGit
CLI tool that shows a single file across N Git branches simultaneously,
collapsing identical regions and stacking divergent ones.

## Architecture
- `cmd/parallelgit/main.go` — binary entry, wires up cobra commands
- `internal/git/` — go-git wrapper, the ONLY place that touches Git
- `internal/core/` — pure N-way diff algorithm, no I/O ever
- `internal/render/` — output formatters (compact, json, wide)

The JSON schema in `internal/core/types.go` is the contract between layers.
All renderers consume LineageReport; future extensions/web/Electron all consume
the JSON output from the CLI.

## Conventions
- Go 1.22+, generics OK when they pay off
- No `interface{}` or `any` — always typed
- All errors wrapped: `fmt.Errorf("loading file at %s: %w", branch, err)`
- `internal/core/` is pure: takes file contents and branch names as inputs,
  no Git calls inside this package, ever
- Tests as `*_test.go` in same package, table-driven, using testify/assert
- Run `go test ./...` after any change to internal/

## Current focus
Phase 1: `parallelgit diff <file> --branches <list>` with JSON and compact output.

## Commands quick reference
- `go run ./cmd/parallelgit diff <file> -b master,uat,dev` — run during dev
- `go test ./...` — run all tests
- `go build -o pg ./cmd/parallelgit && ./pg diff ...` — build and test binary
```

---

## Part 2: The JSON schema (the most important file)

Everything else depends on this. Build it first, get it right, then never break it.

### 2.1 The types

`internal/core/types.go`:

```go
package core

import "time"

// LineageReport is the complete N-way diff output for one file.
// This is the stable contract consumed by all renderers and external tools.
type LineageReport struct {
	File         string       `json:"file"`
	AnchorBranch string       `json:"anchor_branch"`
	Branches     []BranchMeta `json:"branches"`
	Segments     []Segment    `json:"segments"`
}

type BranchMeta struct {
	Name         string `json:"name"`
	HeadSHA      string `json:"head_sha"`
	MergeBaseSHA string `json:"merge_base_sha"`
}

// SegmentKind discriminates the two variants.
type SegmentKind string

const (
	SegmentShared    SegmentKind = "shared"
	SegmentDivergent SegmentKind = "divergent"
)

// Segment is a tagged union. Exactly one of Shared or Divergent
// is populated, determined by Kind. Done this way (vs interfaces)
// because the output JSON-serializes to a stable schema for tools.
type Segment struct {
	Kind      SegmentKind        `json:"kind"`
	Shared    *SharedSegment     `json:"shared,omitempty"`
	Divergent *DivergentSegment  `json:"divergent,omitempty"`
}

type SharedSegment struct {
	Lines []string `json:"lines"`
	Range [2]int   `json:"range"` // [startLine, endLine] in anchor, 1-indexed inclusive
}

type DivergentSegment struct {
	// Variants keyed by branch name. Every selected branch appears.
	Variants map[string]Variant `json:"variants"`
}

type Variant struct {
	Lines        []string  `json:"lines"`
	LastModified time.Time `json:"last_modified"`
	LastAuthor   string    `json:"last_author"`
}
```

### 2.2 Why this shape

- **Tagged union via `Kind` + pointer fields** — Go has no sum types. This is the cleanest workaround; it serializes cleanly to JSON and TypeScript consumers (future VS Code extension) can use a discriminated union with `kind` as the discriminator.
- **`Range [2]int`** — easier than separate `Start`/`End` and 1-indexed to match how editors and humans count lines.
- **`Variants map[string]Variant`** — keyed by branch name. Predictable for JSON consumers, easy iteration order can be guaranteed by passing the branch list separately.
- **`time.Time` for dates** — serializes to RFC3339 by default, parseable everywhere.

---

## Part 3: Phase 1 — the diff engine (Days 1-5)

### 3.1 The Git wrapper

`internal/git/repo.go`:

```go
package git

import (
	"fmt"
	"strings"
	"time"

	"github.com/go-git/go-git/v5"
	"github.com/go-git/go-git/v5/plumbing"
	"github.com/go-git/go-git/v5/plumbing/object"
)

type Repo struct {
	r    *git.Repository
	path string
}

func Open(path string) (*Repo, error) {
	r, err := git.PlainOpen(path)
	if err != nil {
		return nil, fmt.Errorf("opening repo at %s: %w", path, err)
	}
	return &Repo{r: r, path: path}, nil
}

// FileAtBranch returns the file at the tip of branch as a slice of lines.
// Lines do NOT include trailing newlines.
func (r *Repo) FileAtBranch(branch, path string) ([]string, error) {
	ref, err := r.r.Reference(plumbing.NewBranchReferenceName(branch), true)
	if err != nil {
		return nil, fmt.Errorf("resolving branch %s: %w", branch, err)
	}
	commit, err := r.r.CommitObject(ref.Hash())
	if err != nil {
		return nil, fmt.Errorf("loading commit for %s: %w", branch, err)
	}
	tree, err := commit.Tree()
	if err != nil {
		return nil, fmt.Errorf("loading tree for %s: %w", branch, err)
	}
	file, err := tree.File(path)
	if err != nil {
		return nil, fmt.Errorf("file %s not found on %s: %w", path, branch, err)
	}
	content, err := file.Contents()
	if err != nil {
		return nil, fmt.Errorf("reading file contents: %w", err)
	}

	// Split, preserving empty trailing line semantics
	lines := strings.Split(content, "\n")
	// Drop trailing empty line if file ended with \n
	if len(lines) > 0 && lines[len(lines)-1] == "" {
		lines = lines[:len(lines)-1]
	}
	return lines, nil
}

// HeadSHA returns the commit SHA at the tip of branch.
func (r *Repo) HeadSHA(branch string) (string, error) {
	ref, err := r.r.Reference(plumbing.NewBranchReferenceName(branch), true)
	if err != nil {
		return "", fmt.Errorf("resolving branch %s: %w", branch, err)
	}
	return ref.Hash().String(), nil
}

// MergeBase returns the common ancestor of N branches.
// Computed as left-fold: mb(a, b, c) = mb(mb(a, b), c).
func (r *Repo) MergeBase(branches []string) (string, error) {
	if len(branches) == 0 {
		return "", fmt.Errorf("merge base requires at least one branch")
	}
	if len(branches) == 1 {
		return r.HeadSHA(branches[0])
	}

	current, err := r.commitOfBranch(branches[0])
	if err != nil {
		return "", err
	}

	for _, b := range branches[1:] {
		next, err := r.commitOfBranch(b)
		if err != nil {
			return "", err
		}
		bases, err := current.MergeBase(next)
		if err != nil {
			return "", fmt.Errorf("computing merge-base between %s and %s: %w",
				current.Hash, next.Hash, err)
		}
		if len(bases) == 0 {
			return "", fmt.Errorf("no common ancestor between %s and %s",
				current.Hash, next.Hash)
		}
		current = bases[0]
	}
	return current.Hash.String(), nil
}

func (r *Repo) commitOfBranch(branch string) (*object.Commit, error) {
	ref, err := r.r.Reference(plumbing.NewBranchReferenceName(branch), true)
	if err != nil {
		return nil, fmt.Errorf("resolving branch %s: %w", branch, err)
	}
	return r.r.CommitObject(ref.Hash())
}

// LastModifiedForLines returns the most recent commit info touching any line
// in [startLine, endLine] (1-indexed inclusive) on the given branch.
func (r *Repo) LastModifiedForLines(branch, path string, startLine, endLine int) (time.Time, string, error) {
	commit, err := r.commitOfBranch(branch)
	if err != nil {
		return time.Time{}, "", err
	}

	blame, err := git.Blame(commit, path)
	if err != nil {
		return time.Time{}, "", fmt.Errorf("blaming %s on %s: %w", path, branch, err)
	}

	var latest time.Time
	var author string
	for i := startLine - 1; i < endLine && i < len(blame.Lines); i++ {
		line := blame.Lines[i]
		if line.Date.After(latest) {
			latest = line.Date
			author = line.AuthorName
		}
	}
	return latest, author, nil
}
```

### 3.2 The N-way diff algorithm

This is the heart of the tool. The algorithm:

1. For each branch, run a 2-way diff against the anchor.
2. Build an alignment map per branch: `anchorLineIndex → branchLineIndex or -1`.
3. Walk anchor lines once. At each line, check all alignment maps. If every branch matches the anchor, it's shared. Otherwise, divergent.
4. Group consecutive lines of the same kind into segments.
5. For divergent segments, populate variants by collecting each branch's actual content for the corresponding anchor span.

`internal/core/diff.go`:

```go
package core

import (
	"github.com/sergi/go-diff/diffmatchpatch"
)

// Compute is the pure N-way diff. Takes pre-loaded file contents and
// returns the LineageReport structure. No I/O.
//
// anchorLines: the file's content at the anchor (usually merge-base)
// branchLines: map from branch name to that branch's file content
// orderedBranches: the canonical ordering (for stable output)
func Compute(
	file string,
	anchorBranch string,
	anchorLines []string,
	branchLines map[string][]string,
	orderedBranches []string,
) LineageReport {
	// Step 1: per-branch diff against anchor.
	// Step 2: build per-branch alignment from anchor index → branch slice.
	alignments := make(map[string]alignment, len(orderedBranches))
	for _, b := range orderedBranches {
		alignments[b] = align(anchorLines, branchLines[b])
	}

	// Step 3: classify each anchor line.
	// kinds[i] is true if anchor line i is shared across ALL branches.
	kinds := make([]bool, len(anchorLines))
	for i := range anchorLines {
		shared := true
		for _, b := range orderedBranches {
			if !alignments[b].matchesAnchor(i) {
				shared = false
				break
			}
		}
		kinds[i] = shared
	}

	// Step 4: group consecutive lines of same kind into segments.
	segments := groupIntoSegments(anchorLines, kinds, orderedBranches, branchLines, alignments)

	return LineageReport{
		File:         file,
		AnchorBranch: anchorBranch,
		Segments:     segments,
		// Branches filled in by the Load orchestrator
	}
}

// alignment records, for each anchor line index, whether the branch
// has the SAME content at the equivalent position. If a branch added/
// removed lines, the anchor positions affected are marked as not-matching.
type alignment struct {
	// matches[anchorIdx] = true if branch matches anchor at this line.
	matches []bool
	// For divergent regions, this maps an anchor line range to the
	// branch's variant content for that range. Populated by extractVariant.
	branchLines []string
}

func (a alignment) matchesAnchor(anchorIdx int) bool {
	if anchorIdx >= len(a.matches) {
		return false
	}
	return a.matches[anchorIdx]
}

// align computes the alignment of branch lines against anchor lines using
// line-mode diffmatchpatch.
func align(anchor, branch []string) alignment {
	dmp := diffmatchpatch.New()
	a := joinLines(anchor)
	b := joinLines(branch)
	chars1, chars2, lineArr := dmp.DiffLinesToChars(a, b)
	diffs := dmp.DiffMain(chars1, chars2, false)
	diffs = dmp.DiffCharsToLines(diffs, lineArr)

	matches := make([]bool, len(anchor))
	anchorIdx := 0

	for _, d := range diffs {
		dlines := splitLines(d.Text)
		switch d.Type {
		case diffmatchpatch.DiffEqual:
			for range dlines {
				if anchorIdx < len(anchor) {
					matches[anchorIdx] = true
					anchorIdx++
				}
			}
		case diffmatchpatch.DiffDelete:
			// Lines in anchor that are not in branch — mark not-matching.
			for range dlines {
				if anchorIdx < len(anchor) {
					matches[anchorIdx] = false
					anchorIdx++
				}
			}
		case diffmatchpatch.DiffInsert:
			// Lines in branch not in anchor — these don't advance anchorIdx
			// but they mean the surrounding region is divergent. We mark
			// the adjacent anchor line as not-matching to catch this.
			if anchorIdx > 0 {
				matches[anchorIdx-1] = false
			}
			if anchorIdx < len(matches) {
				matches[anchorIdx] = false
			}
		}
	}

	return alignment{matches: matches, branchLines: branch}
}

func groupIntoSegments(
	anchorLines []string,
	kinds []bool,
	orderedBranches []string,
	branchLines map[string][]string,
	alignments map[string]alignment,
) []Segment {
	var segments []Segment
	i := 0

	for i < len(kinds) {
		j := i
		for j < len(kinds) && kinds[j] == kinds[i] {
			j++
		}

		if kinds[i] {
			// Shared segment from i..j-1
			segments = append(segments, Segment{
				Kind: SegmentShared,
				Shared: &SharedSegment{
					Lines: append([]string(nil), anchorLines[i:j]...),
					Range: [2]int{i + 1, j}, // 1-indexed inclusive
				},
			})
		} else {
			// Divergent segment from i..j-1
			variants := make(map[string]Variant, len(orderedBranches))
			for _, b := range orderedBranches {
				variants[b] = Variant{
					Lines: extractVariant(branchLines[b], alignments[b], i, j),
					// LastModified + LastAuthor filled by Load orchestrator
				}
			}
			segments = append(segments, Segment{
				Kind: SegmentDivergent,
				Divergent: &DivergentSegment{
					Variants: variants,
				},
			})
		}

		i = j
	}

	return segments
}

// extractVariant returns the branch's content corresponding to anchor lines
// [start, end). For v0.1 this is a naive cut from the branch's full content
// at the same anchor offsets, with bounds checking. v0.2 will be smarter.
func extractVariant(branchLines []string, _ alignment, start, end int) []string {
	if start >= len(branchLines) {
		return []string{}
	}
	endIdx := end
	if endIdx > len(branchLines) {
		endIdx = len(branchLines)
	}
	out := make([]string, endIdx-start)
	copy(out, branchLines[start:endIdx])
	return out
}

func joinLines(lines []string) string {
	s := ""
	for _, l := range lines {
		s += l + "\n"
	}
	return s
}

func splitLines(s string) []string {
	if s == "" {
		return nil
	}
	var out []string
	start := 0
	for i := 0; i < len(s); i++ {
		if s[i] == '\n' {
			out = append(out, s[start:i])
			start = i + 1
		}
	}
	if start < len(s) {
		out = append(out, s[start:])
	}
	return out
}
```

**Known v0.1 limitations** (document these and move on):

- `extractVariant` is naive — for branches with very different line counts than the anchor, the cut may not align perfectly with branch semantics. Real fix involves tracking branch-line offsets through the alignment phase. Acceptable for v0.1 because divergent segments tend to be small.
- The "insert" handling marks adjacent anchor lines as not-matching, which slightly over-counts divergent regions. Conservative direction (over-report divergence rather than miss it).

These are exactly the kind of issues you'll catch when you use it on `payment-orchestrator`.

### 3.3 The orchestrator

`internal/core/load.go`:

```go
package core

import (
	"fmt"

	"github.com/sathnindu/parallelgit/internal/git"
)

// Load is the orchestrator: it does the Git I/O and calls Compute.
func Load(repo *git.Repo, file string, branches []string) (LineageReport, error) {
	if len(branches) == 0 {
		return LineageReport{}, fmt.Errorf("at least one branch required")
	}

	// Compute merge-base as anchor.
	mergeBaseSHA, err := repo.MergeBase(branches)
	if err != nil {
		return LineageReport{}, fmt.Errorf("computing merge-base: %w", err)
	}

	// Load file at each branch.
	branchLines := make(map[string][]string, len(branches))
	meta := make([]BranchMeta, 0, len(branches))

	for _, b := range branches {
		lines, err := repo.FileAtBranch(b, file)
		if err != nil {
			return LineageReport{}, fmt.Errorf("loading file at %s: %w", b, err)
		}
		branchLines[b] = lines

		headSHA, err := repo.HeadSHA(b)
		if err != nil {
			return LineageReport{}, err
		}

		meta = append(meta, BranchMeta{
			Name:         b,
			HeadSHA:      headSHA,
			MergeBaseSHA: mergeBaseSHA,
		})
	}

	// Anchor = first branch's file content as the spine. For v0.1, use the
	// first branch as anchor instead of loading the merge-base tree.
	// Acceptable because the lockstep walk is anchor-relative either way.
	anchorBranch := branches[0]
	anchorLines := branchLines[anchorBranch]

	report := Compute(file, anchorBranch, anchorLines, branchLines, branches)
	report.Branches = meta
	report.File = file

	// Fill in LastModified / LastAuthor per variant.
	if err := enrichVariants(repo, file, &report); err != nil {
		return LineageReport{}, err
	}

	return report, nil
}

func enrichVariants(repo *git.Repo, file string, report *LineageReport) error {
	lineCursor := 1

	for _, seg := range report.Segments {
		switch seg.Kind {
		case SegmentShared:
			lineCursor = seg.Shared.Range[1] + 1

		case SegmentDivergent:
			// For each variant, blame the corresponding range on that branch.
			start := lineCursor
			end := lineCursor
			for _, v := range seg.Divergent.Variants {
				if len(v.Lines) > end-start {
					end = start + len(v.Lines)
				}
			}

			for branchName, v := range seg.Divergent.Variants {
				if len(v.Lines) == 0 {
					continue
				}
				ts, author, err := repo.LastModifiedForLines(branchName, file, start, start+len(v.Lines)-1)
				if err != nil {
					// Non-fatal: blame can fail on deleted lines, etc.
					continue
				}
				v.LastModified = ts
				v.LastAuthor = author
				seg.Divergent.Variants[branchName] = v
			}
			lineCursor = end
		}
	}

	return nil
}
```

### 3.4 Tests for the core

`internal/core/diff_test.go`:

```go
package core

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestCompute_AllIdentical(t *testing.T) {
	anchor := []string{"a", "b", "c"}
	branches := map[string][]string{
		"master": {"a", "b", "c"},
		"dev":    {"a", "b", "c"},
	}
	r := Compute("f", "master", anchor, branches, []string{"master", "dev"})
	assert.Len(t, r.Segments, 1)
	assert.Equal(t, SegmentShared, r.Segments[0].Kind)
	assert.Equal(t, []string{"a", "b", "c"}, r.Segments[0].Shared.Lines)
	assert.Equal(t, [2]int{1, 3}, r.Segments[0].Shared.Range)
}

func TestCompute_MiddleDivergence(t *testing.T) {
	anchor := []string{"a", "b", "c"}
	branches := map[string][]string{
		"master": {"a", "b", "c"},
		"dev":    {"a", "B", "c"},
	}
	r := Compute("f", "master", anchor, branches, []string{"master", "dev"})
	// Expect: shared[a], divergent[b vs B], shared[c]
	assert.Len(t, r.Segments, 3)
	assert.Equal(t, SegmentShared, r.Segments[0].Kind)
	assert.Equal(t, SegmentDivergent, r.Segments[1].Kind)
	assert.Equal(t, SegmentShared, r.Segments[2].Kind)
}

func TestCompute_TwoAgreeOneDiffers(t *testing.T) {
	anchor := []string{"a", "b", "c"}
	branches := map[string][]string{
		"master": {"a", "b", "c"},
		"uat":    {"a", "b", "c"},
		"dev":    {"a", "X", "c"},
	}
	r := Compute("f", "master", anchor, branches, []string{"master", "uat", "dev"})
	assert.Len(t, r.Segments, 3)
	div := r.Segments[1]
	assert.Equal(t, SegmentDivergent, div.Kind)
	assert.Equal(t, "b", div.Divergent.Variants["master"].Lines[0])
	assert.Equal(t, "b", div.Divergent.Variants["uat"].Lines[0])
	assert.Equal(t, "X", div.Divergent.Variants["dev"].Lines[0])
}

func TestCompute_BranchAddsLines(t *testing.T) {
	anchor := []string{"a", "c"}
	branches := map[string][]string{
		"master": {"a", "c"},
		"dev":    {"a", "b", "c"},
	}
	r := Compute("f", "master", anchor, branches, []string{"master", "dev"})
	// dev adds a line — should appear as divergent region
	hasDivergent := false
	for _, seg := range r.Segments {
		if seg.Kind == SegmentDivergent {
			hasDivergent = true
		}
	}
	assert.True(t, hasDivergent, "expected divergent segment for inserted line")
}

func TestCompute_AllCompletelyDifferent(t *testing.T) {
	anchor := []string{"a", "b", "c"}
	branches := map[string][]string{
		"master": {"a", "b", "c"},
		"dev":    {"x", "y", "z"},
	}
	r := Compute("f", "master", anchor, branches, []string{"master", "dev"})
	assert.Len(t, r.Segments, 1)
	assert.Equal(t, SegmentDivergent, r.Segments[0].Kind)
}
```

After this, you have a working diff engine. `go test ./internal/core/...` should be all green.

---

## Part 4: Phase 2 — CLI and renderers (Days 6-9)

### 4.1 The JSON renderer (trivial)

`internal/render/json.go`:

```go
package render

import (
	"encoding/json"
	"io"

	"github.com/sathnindu/parallelgit/internal/core"
)

func RenderJSON(report core.LineageReport, w io.Writer) error {
	enc := json.NewEncoder(w)
	enc.SetIndent("", "  ")
	return enc.Encode(report)
}
```

### 4.2 The compact renderer

`internal/render/compact.go`:

```go
package render

import (
	"fmt"
	"io"
	"os"
	"strings"

	"github.com/fatih/color"
	"github.com/sathnindu/parallelgit/internal/core"
	"golang.org/x/term"
)

var branchColors = []*color.Color{
	color.New(color.FgCyan, color.Bold),
	color.New(color.FgMagenta, color.Bold),
	color.New(color.FgYellow, color.Bold),
	color.New(color.FgGreen, color.Bold),
	color.New(color.FgBlue, color.Bold),
	color.New(color.FgRed, color.Bold),
}

var (
	dim   = color.New(color.Faint)
	plain = color.New()
)

func RenderCompact(report core.LineageReport, w io.Writer) error {
	width := detectWidth()
	showAuthor := width > 100

	// Header
	branchTags := make([]string, len(report.Branches))
	for i, b := range report.Branches {
		c := branchColors[i%len(branchColors)]
		shortSHA := b.HeadSHA
		if len(shortSHA) > 7 {
			shortSHA = shortSHA[:7]
		}
		branchTags[i] = c.Sprintf("%s", b.Name) + dim.Sprintf(" (%s)", shortSHA)
	}
	fmt.Fprintf(w, "%s  |  %s\n\n",
		color.New(color.Bold).Sprint(report.File),
		strings.Join(branchTags, "  |  "))

	// Body
	for _, seg := range report.Segments {
		switch seg.Kind {
		case core.SegmentShared:
			rng := seg.Shared.Range
			dim.Fprintf(w, "--- lines %d-%d identical across all branches ---\n\n",
				rng[0], rng[1])
		case core.SegmentDivergent:
			renderDivergent(w, seg.Divergent, report.Branches, width, showAuthor)
			fmt.Fprintln(w)
		}
	}

	return nil
}

func renderDivergent(w io.Writer, div *core.DivergentSegment, branches []core.BranchMeta, width int, showAuthor bool) {
	bar := strings.Repeat("-", min(width-2, 70))
	fmt.Fprintf(w, "+%s+\n", bar)

	for i, bm := range branches {
		c := branchColors[i%len(branchColors)]
		v := div.Variants[bm.Name]

		header := c.Sprintf("| %-8s", bm.Name)
		if !v.LastModified.IsZero() {
			header += dim.Sprintf("  %s", v.LastModified.Format("2006-01-02"))
		}
		if showAuthor && v.LastAuthor != "" {
			header += dim.Sprintf("  %s", v.LastAuthor)
		}
		fmt.Fprintln(w, header)

		for _, line := range v.Lines {
			fmt.Fprintf(w, "|   %s\n", line)
		}

		if i < len(branches)-1 {
			fmt.Fprintf(w, "+%s+\n", bar)
		}
	}

	fmt.Fprintf(w, "+%s+\n", bar)
}

func detectWidth() int {
	if w, _, err := term.GetSize(int(os.Stdout.Fd())); err == nil && w > 0 {
		return w
	}
	return 80
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

### 4.3 The CLI

`cmd/parallelgit/main.go`:

```go
package main

import (
	"fmt"
	"os"
	"strings"

	"github.com/spf13/cobra"
	"github.com/sathnindu/parallelgit/internal/core"
	pgit "github.com/sathnindu/parallelgit/internal/git"
	"github.com/sathnindu/parallelgit/internal/render"
)

// Populated by GoReleaser at build time via ldflags.
var (
	version = "dev"
	commit  = "none"
	date    = "unknown"
)

func main() {
	root := &cobra.Command{
		Use:     "parallelgit",
		Short:   "See a single file across every branch at once.",
		Version: fmt.Sprintf("%s (commit %s, built %s)", version, commit, date),
	}
	root.AddCommand(diffCmd())
	if err := root.Execute(); err != nil {
		os.Exit(1)
	}
}

func diffCmd() *cobra.Command {
	var (
		branchesFlag string
		format       string
		repoPath     string
	)

	cmd := &cobra.Command{
		Use:   "diff <file>",
		Short: "Show a file across N branches",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			file := args[0]

			if branchesFlag == "" {
				return fmt.Errorf("--branches is required (comma-separated list)")
			}
			branches := splitAndTrim(branchesFlag)

			if repoPath == "" {
				cwd, err := os.Getwd()
				if err != nil {
					return err
				}
				repoPath = cwd
			}

			repo, err := pgit.Open(repoPath)
			if err != nil {
				return err
			}

			report, err := core.Load(repo, file, branches)
			if err != nil {
				return err
			}

			switch format {
			case "json":
				return render.RenderJSON(report, os.Stdout)
			case "compact":
				return render.RenderCompact(report, os.Stdout)
			default:
				return fmt.Errorf("unknown format: %s", format)
			}
		},
	}

	cmd.Flags().StringVarP(&branchesFlag, "branches", "b", "", "Comma-separated branches")
	cmd.Flags().StringVarP(&format, "format", "f", "compact", "Output format: compact, json")
	cmd.Flags().StringVarP(&repoPath, "repo", "r", "", "Path to repo (defaults to cwd)")

	return cmd
}

func splitAndTrim(s string) []string {
	parts := strings.Split(s, ",")
	out := make([]string, 0, len(parts))
	for _, p := range parts {
		if t := strings.TrimSpace(p); t != "" {
			out = append(out, t)
		}
	}
	return out
}
```

### 4.4 First run

```bash
# Build and test
go build -o pg ./cmd/parallelgit

# Use it on payment-orchestrator
cd ~/work/payment-orchestrator
~/projects/parallelgit/pg diff src/main/java/.../UserService.java -b master,uat,dev

# Get JSON for inspection
~/projects/parallelgit/pg diff src/main/java/.../UserService.java -b master,uat,dev -f json | jq
```

If this shows useful output on a real file, **you have the MVP.** Stop and breathe.

---

## Part 5: Phase 3 — dogfood for a week (Days 10-16)

This is the phase most likely to be skipped, and the most important.

**Rules:**
1. Use the tool on every multi-branch question you have at work.
2. **Do not add features.** Just fix actual bugs that block use.
3. Keep `notes.md` open in your editor. Every annoyance, every "I wish this could…", every confusing output — log it.
4. Take screenshots of weird output.

**What to look for:**
- Is the compact format readable on real files?
- Are divergent variants accurate when branches have very different line counts?
- Does the merge-base behave right when branches have diverged for months?
- Is the CLI ergonomic? Do you reach for it naturally?
- Does it crash on edge cases (empty files, binary files, files that don't exist on one branch)?

End-of-week `notes.md` is your v0.2 roadmap, and it'll be far more honest than anything you could plan upfront.

### Likely issues to expect

Based on the design, here are the most likely real-world problems and their fixes:

1. **"File doesn't exist on this branch" errors crash the whole command.** Fix: handle gracefully, mark that branch's variant as "(file does not exist on this branch)".
2. **Branches with very different line counts produce misaligned variants.** Fix in v0.2: improve `extractVariant` to track branch-line offsets properly.
3. **`--branches master,uat,dev` is repetitive when used daily.** Fix: support a `.parallelgit.yaml` config file with named branch sets.
4. **JSON output for piping to other tools needs stable ordering.** Already handled by `orderedBranches`, but watch for map iteration in renderers.

---

## Part 6: Phase 4 — distribution (Days 17-18)

This is where Go pays off.

### 6.1 `.goreleaser.yaml`

```yaml
version: 2

before:
  hooks:
    - go mod tidy

builds:
  - id: parallelgit
    main: ./cmd/parallelgit
    binary: parallelgit
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.commit={{.Commit}}
      - -X main.date={{.Date}}

archives:
  - id: default
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    formats: [tar.gz]
    format_overrides:
      - goos: windows
        formats: [zip]
    files:
      - README.md
      - LICENSE

checksum:
  name_template: "checksums.txt"

snapshot:
  version_template: "{{ incpatch .Version }}-next"

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^chore:"
      - "Merge pull request"
      - "Merge branch"

brews:
  - name: parallelgit
    repository:
      owner: sathnindu
      name: homebrew-parallelgit
    homepage: "https://github.com/sathnindu/parallelgit"
    description: "Visualize a single file across N Git branches simultaneously."
    license: MIT
    test: |
      system "#{bin}/parallelgit --version"
    install: |
      bin.install "parallelgit"

scoops:
  - repository:
      owner: sathnindu
      name: scoop-parallelgit
    homepage: "https://github.com/sathnindu/parallelgit"
    description: "Visualize a single file across N Git branches simultaneously."
    license: MIT

nfpms:
  - id: linux-packages
    package_name: parallelgit
    vendor: Sathnindu
    homepage: "https://github.com/sathnindu/parallelgit"
    maintainer: Sathnindu <your-email@example.com>
    description: "Visualize a single file across N Git branches simultaneously."
    license: MIT
    formats:
      - deb
      - rpm
      - apk
```

### 6.2 Two extra GitHub repos

Create these as empty public repos:
1. `sathnindu/homebrew-parallelgit` (the `homebrew-` prefix is mandatory for Homebrew taps)
2. `sathnindu/scoop-parallelgit`

GoReleaser will commit the formula/manifest into these repos on every release.

### 6.3 GitHub Actions

`.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - run: go vet ./...
      - run: go test -race -coverprofile=coverage.out ./...
      - run: go build ./...
```

`.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags: ['v*']

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - uses: goreleaser/goreleaser-action@v6
        with:
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_GITHUB_TOKEN: ${{ secrets.TAP_TOKEN }}
          SCOOP_TAP_GITHUB_TOKEN: ${{ secrets.TAP_TOKEN }}
```

### 6.4 TAP_TOKEN setup (one-time)

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token with `repo` scope (full)
3. Copy the token
4. In `sathnindu/parallelgit` repo → Settings → Secrets and variables → Actions
5. Add new secret named `TAP_TOKEN`, paste the value

This is the only secret needed; `GITHUB_TOKEN` is provided by Actions automatically.

### 6.5 First release

```bash
# Make sure everything works
go test ./...
go build ./cmd/parallelgit

# Tag and push
git tag v0.1.0
git push origin v0.1.0
```

GitHub Actions runs GoReleaser, which builds for all platforms, creates a GitHub Release with all artifacts, pushes the formula to `homebrew-parallelgit`, pushes the manifest to `scoop-parallelgit`. About 3 minutes end-to-end.

### 6.6 Verify install works

```bash
# macOS / Linux
brew install sathnindu/parallelgit/parallelgit
parallelgit --version

# Test on Windows
scoop bucket add parallelgit https://github.com/sathnindu/scoop-parallelgit
scoop install parallelgit
parallelgit --version
```

---

## Part 7: Phase 5 — public launch (Day 19+)

Don't launch publicly until you've:
1. Used the tool yourself for a week.
2. Used it on at least one repo other than your own (maybe a teammate's at Arimac).
3. Fixed every issue in `notes.md` marked "blocker" or "embarrassing".
4. Recorded the VHS demo GIF.
5. Polished the README based on real usage.

### Launch sequence

**Day -1: Capture the demo.**

`docs/demo.tape`:
```
Output docs/demo.gif
Set FontSize 14
Set Width 1100
Set Height 600
Set Theme "Catppuccin Mocha"
Set TypingSpeed 50ms

Type "parallelgit diff UserService.java -b master,uat,dev"
Sleep 500ms
Enter
Sleep 4s
```

```bash
brew install vhs
vhs docs/demo.tape
```

**Day 0: Launch.**
- Post to Hacker News: "Show HN: ParallelGit — see a file across every branch at once"
- Post to r/programming and r/golang
- Tweet/X thread with the demo GIF
- Submit to Console.dev newsletter
- Submit to Awesome Go list

The demo GIF is the entire launch. If the GIF doesn't make people say "wait, I need this" in 5 seconds, no amount of explanation will save it.

---

## Part 8: Working with Claude Code

### 8.1 First prompts (paste these in order)

After Day 0 setup, open Claude Code in the repo and paste these one at a time:

**Prompt 1 (types):**
> Read CLAUDE.md for project context. Phase 1, Step 1: define the JSON schema as Go types in internal/core/types.go. Requirements: LineageReport struct, BranchMeta, Segment as tagged union with Kind discriminator and optional Shared/Divergent pointer fields, SharedSegment, DivergentSegment, Variant. All structs need JSON tags in snake_case. No interface{}, no any. Use time.Time for dates. Show me the file when done.

**Prompt 2 (git wrapper):**
> Phase 1, Step 2: implement internal/git/repo.go. Create a Repo struct wrapping go-git. Methods: Open, FileAtBranch (returns []string, no trailing newline), HeadSHA, MergeBase (n-way via left-fold), LastModifiedForLines (using go-git blame). All errors wrapped with context. Write table-driven tests in repo_test.go using a tmpdir fixture repo.

**Prompt 3 (diff algorithm):**
> Phase 1, Step 3: implement the N-way diff in internal/core/diff.go. The algorithm: (1) for each branch, run a line-mode diff against the anchor using diffmatchpatch (DiffLinesToChars → DiffMain → DiffCharsToLines), (2) build an alignment[anchorLineIdx]→bool matches array per branch, (3) walk anchor lines, marking each shared if all branches match, else divergent, (4) group consecutive same-kind lines into segments, (5) for divergent segments, extract each branch's variant. Pure function, no I/O. The signature is Compute(file, anchorBranch string, anchorLines []string, branchLines map[string][]string, orderedBranches []string) LineageReport.

**Prompt 4 (tests):**
> Phase 1, Step 4: write table-driven tests in internal/core/diff_test.go covering: all branches identical, middle divergence, three branches where two agree and one differs, branch adds lines, branch deletes lines, all completely different. Use testify/assert. Then run `go test ./internal/core/...` and confirm all pass.

**Prompt 5 (orchestrator):**
> Phase 1, Step 5: implement internal/core/load.go with the Load function — it takes a Repo, file, and branches list, loads file content from each branch, computes merge-base, calls Compute, enriches divergent variants with blame info. Returns a fully-populated LineageReport.

**Prompt 6 (renderers):**
> Phase 2: implement internal/render/json.go (trivial json.MarshalIndent) and internal/render/compact.go. Compact requirements: header with file path and branch list (each branch with short SHA in dim), shared segments shown as "--- lines X-Y identical across all branches ---" in dim, divergent segments as ASCII-bordered blocks (use + and | and -, NOT Unicode box-drawing) with branch name, date, optional author per variant. Width-aware via golang.org/x/term. Use fatih/color.

**Prompt 7 (CLI):**
> Phase 2 final: implement cmd/parallelgit/main.go with cobra. Root command "parallelgit" with --version. Subcommand "diff <file>" with flags --branches/-b (required, comma-separated), --format/-f (compact|json, default compact), --repo/-r (default cwd). Wire everything together. Make sure ldflags variables (version, commit, date) are at package level so GoReleaser can set them.

**Prompt 8 (release config):**
> Phase 4: create .goreleaser.yaml configured to build for macOS (amd64+arm64), Linux (amd64+arm64), Windows (amd64+arm64). Include brew tap config (homebrew-parallelgit), scoop bucket (scoop-parallelgit), and nfpm for deb/rpm/apk. Set ldflags to populate main.version/commit/date. Also create .github/workflows/ci.yml and .github/workflows/release.yml.

After each prompt: review the code, run `go test ./...`, commit if green.

### 8.2 Workflow tips

**One session per phase.** Don't try to do all of Phase 1 in one session — context gets too long. Phase 1 = one session, Phase 2 = new session. CLAUDE.md carries the across-session context.

**Update CLAUDE.md as you go.** End of each session, prompt: *"Update CLAUDE.md to reflect what's now complete and what's next."*

**Never accept code you don't understand.** If Claude generates something dense, ask: *"Walk me through this function line by line."* You're going to maintain this for years.

**Push back on `interface{}` and `any`.** Go has generics. If you see these in generated code, ask Claude to rewrite with proper types.

**Don't let Claude one-shot the diff algorithm.** It's the heart of the tool. Break into types → wrapper → algorithm → tests. Each step reviewable separately.

---

## Part 9: Realistic timeline

| Phase | Days | Output |
|---|---|---|
| Phase 0 — setup | 0.5 day | Repo, CLAUDE.md, dependencies |
| Phase 1 — diff engine | 4-5 days | `internal/core` and `internal/git` working with tests |
| Phase 2 — CLI + render | 3 days | `parallelgit diff` works locally on real repos |
| Phase 3 — dogfood | 7 days | `notes.md` full of real observations, bug fixes |
| Phase 4 — distribution | 1-2 days | First release via GoReleaser, installs work on all platforms |
| Phase 5 — public launch | 1 day | Demo GIF, HN/Reddit/Twitter posts |
| **Total to public** | **~3 weeks of evening work** | |

This is honest. It will probably take longer. That's fine — the dogfooding week is the most important phase and shouldn't be rushed.

---

## Part 10: What success looks like at v0.1

You'll know v0.1 succeeded if, three weeks in:

1. You reach for `parallelgit` without thinking, at least once a day, on real Arimac work.
2. At least one colleague says "wait, can you send me that thing again?"
3. The HN/Reddit post gets enough engagement (>200 upvotes, >50 comments) to suggest a real audience exists.
4. You're already writing the v0.2 roadmap in your head based on actual use, not speculation.

If those don't happen — the idea may not be a product, no matter how good the visualization is. Better to find that out in three weeks of evenings than three months of full-time work.
