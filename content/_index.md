---
title: go-lines
---

**Line-oriented text processing for Go: a handful of pure per-line primitives plus a buffered `Process` that scans an `io.Reader`, applies a transform to each line, honors context cancellation, and reports line counts.**

- **Source:** [gomatic/go-lines](https://github.com/gomatic/go-lines)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-lines](https://pkg.go.dev/github.com/gomatic/go-lines)

## Install

```sh
go get github.com/gomatic/go-lines
```

```go
import lines "github.com/gomatic/go-lines"
```

## Usage

The per-line primitives are small, pure functions over the `Line` type. `Process` owns the scanning, counting, and joining; you supply a `Transform` that maps each numbered line to its processed form and reports whether to keep it.

```go
package main

import (
	"context"
	"fmt"
	"strings"

	lines "github.com/gomatic/go-lines"
)

func main() {
	input := strings.NewReader("alpha\nskip\nbeta")

	// Keep lines that contain "a", uppercase and number the survivors.
	transform := func(line lines.Line, n lines.LineNumber) (lines.Line, bool) {
		if !lines.Contains(line, "a") {
			return "", false
		}
		return lines.Numbered(lines.Uppercase(line), n), true
	}

	output, stats, err := lines.Process(context.Background(), input, transform)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(output))
	fmt.Printf("total=%d kept=%d\n", stats.Total, stats.Kept)
}
```

Output:

```
   1 | ALPHA
   3 | BETA
total=3 kept=2
```

### Primitives

- `Uppercase(line Line) Line` — the line converted to uppercase.
- `WithPrefix(line Line, prefix Prefix) Line` — the line with `prefix` prepended.
- `Numbered(line Line, number LineNumber) Line` — the line prefixed with its right-aligned line number (`%4d | `).
- `Contains(line Line, filter Filter) bool` — whether the line contains the `filter` substring.

### Process

```go
func Process(ctx context.Context, reader io.Reader, transform Transform) (Output, Stats, error)
```

`Process` scans `reader` line by line, applies `transform`, and joins the kept lines with `"\n"`. It stops early if `ctx` is cancelled and returns a `Stats{Total, Kept}` reporting how many lines were seen and kept.

## Design

- **Buffered, not streaming.** Every kept line is retained in memory and joined into a single `Output`, so peak memory is O(input).
- **Trailing newline is not round-tripped.** Lines are joined with `"\n"` and no terminator, so `"a\nb\nc\n"` and `"a\nb\nc"` both yield `"a\nb\nc"`. CRLF is normalized to LF because the scanner strips a trailing `"\r"` from each line; a lone CR and embedded NUL bytes are preserved as ordinary content.
- **MaxLine ceiling.** The scan buffer is raised from bufio's default 64 KiB to `MaxLine` (1 MiB). A single line longer than `MaxLine` fails with `ErrReadInput` wrapping `bufio.ErrTooLong`.
- **Sentinel errors.** The only value the package emits is `const ErrReadInput errs.Const`, from [gomatic/go-error](https://github.com/gomatic/go-error). The underlying cause is wrapped via `ErrReadInput.With(cause)`, so the result matches both `ErrReadInput` and the cause under `errors.Is`. On any read failure, `Process` discards partial output and returns a zero `Stats`.
