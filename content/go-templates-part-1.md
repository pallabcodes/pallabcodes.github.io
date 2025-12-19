+++
title = "Go Templates Part 1: The Mental Model"
date = 2025-12-19
description = "Understanding Go's template system from first principles - the dot, parsing, and execution model"
[taxonomies]
tags = ["go", "templates", "fundamentals"]
+++

Go's `text/template` package looks simple until you actually use it. Then you hit the dot, get confused by context, and wonder why your template panics on nil. This post builds the mental model you need.

<!-- more -->

## The Core Abstraction

A Go template is a **compiled text structure** that you execute with data. Two distinct phases:

```go
// Phase 1: Parse (compile)
tmpl := template.Must(template.New("t").Parse("Hello {{.}}"))

// Phase 2: Execute (render)
tmpl.Execute(os.Stdout, "World")
// Output: Hello World
```

`Parse` happens once. `Execute` happens many times. If you're parsing templates per request, you're doing it wrong.

## The Dot: Current Context

The single most important concept in Go templates is `.` (dot). It represents **whatever data is currently in scope**.

At the start of execution, `.` equals whatever you passed to `Execute`:

```go
tmpl.Execute(w, "Alice")  // . == "Alice"
tmpl.Execute(w, 42)       // . == 42
tmpl.Execute(w, user)     // . == user struct
```

### Accessing Fields

When `.` is a struct or map, you access fields with `.FieldName`:

```go
tmpl := template.Must(template.New("t").Parse(`
Name: {{.Name}}
Age: {{.Age}}
`))

tmpl.Execute(os.Stdout, struct {
    Name string
    Age  int
}{"Alice", 30})
```

This works identically for structs and maps:

```go
// Both work with .Name
tmpl.Execute(w, struct{ Name string }{"Alice"})
tmpl.Execute(w, map[string]string{"Name": "Alice"})
```

### Context Changes Inside `range`

Here's where people get confused. Inside `range`, the dot **becomes each element**:

```go
tmpl := template.Must(template.New("t").Parse(`
{{range .}}
  - {{.}}
{{end}}
`))

tmpl.Execute(os.Stdout, []string{"Go", "Rust", "Zig"})
```

Before `range`: `. == []string{"Go", "Rust", "Zig"}`
Inside `range`: `. == "Go"`, then `. == "Rust"`, then `. == "Zig"`

If you need the original context inside a range:

```go
{{$root := .}}
{{range .Items}}
  {{.Name}} belongs to {{$root.Owner}}
{{end}}
```

## Truthiness Rules

`if` checks whether a value is "truthy":

```go
{{if .}}yes{{else}}no{{end}}
```

Go templates consider these **falsy**:
- `false`
- `0`
- `nil`
- Empty string `""`
- Empty slice/map/channel

Everything else is truthy.

## Why Templates Are Logic-Light

Go templates deliberately limit what you can do:

```go
// ✅ Allowed
{{if .IsAdmin}}...{{end}}
{{range .Items}}...{{end}}
{{.User.Name}}

// ❌ Not allowed
{{if .Count > 10}}...{{end}}     // No operators
{{.Count + 1}}                    // No arithmetic
{{$x := computeValue .}}          // No arbitrary function calls
```

This isn't a limitation—it's the point. Templates render data; they don't compute it. All logic belongs in Go code.

The philosophy: if you need to transform data, do it before calling `Execute`. Your templates stay readable, testable, and secure.

## text/template vs html/template

These are **almost identical APIs** with one critical difference:

```go
import "text/template"  // No escaping
import "html/template"  // Auto-escapes for HTML

tmpl.Execute(w, "<script>alert(1)</script>")
// text/template: <script>alert(1)</script>
// html/template: &lt;script&gt;alert(1)&lt;/script&gt;
```

**Rule**: HTML output → `html/template`. Everything else → `text/template`.

Using `text/template` for HTML is an XSS vulnerability. Don't do it.

## Parse Once, Execute Many

This pattern matters for performance and safety:

```go
// ❌ Bad: parsing per request
func handler(w http.ResponseWriter, r *http.Request) {
    tmpl := template.Must(template.ParseFiles("page.html"))
    tmpl.Execute(w, data)
}

// ✅ Good: parse at startup
var tmpl = template.Must(template.ParseFiles("page.html"))

func handler(w http.ResponseWriter, r *http.Request) {
    tmpl.Execute(w, data)
}
```

Why this matters:
- Parsing is expensive (string processing, memory allocation)
- Parse errors should crash at startup, not at request time
- Templates are goroutine-safe for execution

## Practical Debugging

When a template doesn't render what you expect, dump the context:

```go
{{printf "%#v" .}}
```

This prints the full Go representation of whatever `.` currently is. Remove it before shipping.

## Key Takeaways

1. **`.` = current context**, not "the data you passed in"
2. **Parse once** at startup, execute many times
3. **Templates render, they don't compute**—logic goes in Go
4. **`html/template` for HTML**, always

Next up: [Part 2](/go-templates-part-2) covers `with`, `range` patterns, custom functions, and template composition.
