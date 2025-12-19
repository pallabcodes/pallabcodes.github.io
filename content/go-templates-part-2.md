+++
title = "Go Templates Part 2: Patterns That Scale"
date = 2025-12-19
description = "Context rebinding with 'with', effective range patterns, custom functions, and template composition"
[taxonomies]
tags = ["go", "templates", "patterns"]
+++

Part 1 covered the mental model. Now let's look at the patterns you'll actually use in production: context control, iteration, functions, and composition.

<!-- more -->

## Context Rebinding with `with`

`with` does two things at once:
1. Checks if a value exists (non-empty/non-nil)
2. Rebinds `.` to that value

```go
{{with .User}}
  Name: {{.Name}}
  Email: {{.Email}}
{{else}}
  No user found
{{end}}
```

Inside the `with` block, `.` is `.User`, not the original data. You've "zoomed in."

### Why This Matters

Without `with`, you repeat yourself:

```go
// Verbose
Name: {{.User.Name}}
Email: {{.User.Email}}
Phone: {{.User.Phone}}

// Clean with context rebinding
{{with .User}}
Name: {{.Name}}
Email: {{.Email}}
Phone: {{.Phone}}
{{end}}
```

### Nil Safety

`with` also guards against nil:

```go
// Panics if .User is nil
{{.User.Name}}

// Safe - else branch runs if .User is nil
{{with .User}}{{.Name}}{{else}}unknown{{end}}
```

This is why you'll see `with` throughout production Go templates—it's both cleaner and safer.

## Range Patterns

### Basic Iteration

```go
{{range .Items}}
  - {{.}}
{{end}}
```

Inside range, `.` becomes each element.

### With Index

```go
{{range $index, $item := .Items}}
  {{$index}}: {{$item}}
{{end}}
```

### Empty Collection Handling

```go
{{range .Items}}
  - {{.}}
{{else}}
  No items found
{{end}}
```

The `else` block runs when the collection is empty.

### Maps (Use With Caution)

```go
{{range $key, $value := .Config}}
  {{$key}} = {{$value}}
{{end}}
```

**Warning**: Map iteration order is random. If you need deterministic output (for diffs, tests, or reproducibility), sort keys in Go before rendering:

```go
type KV struct { Key, Value string }

var items []KV
for k, v := range config {
    items = append(items, KV{k, v})
}
sort.Slice(items, func(i, j int) bool {
    return items[i].Key < items[j].Key
})

tmpl.Execute(w, items)
```

### Preserving Outer Context

Inside `range`, you lose access to the original `.`. Save it first:

```go
{{$root := .}}
{{range .Users}}
  {{.Name}} works at {{$root.Company}}
{{end}}
```

## Custom Template Functions

Templates support registered functions for formatting:

```go
tmpl := template.Must(
    template.New("t").
        Funcs(template.FuncMap{
            "upper": strings.ToUpper,
            "formatDate": func(t time.Time) string {
                return t.Format("2006-01-02")
            },
        }).
        Parse(`{{upper .Name}}, joined {{formatDate .JoinedAt}}`),
)
```

### Function Rules

Template functions must be:
- **Pure**: same input → same output
- **Side-effect free**: no mutations, no I/O
- **Fast**: called potentially many times
- **Panic-safe**: handle edge cases gracefully

```go
// ✅ Good: pure formatting
"formatMoney": func(cents int) string {
    return fmt.Sprintf("$%.2f", float64(cents)/100)
}

// ❌ Bad: side effects
"saveToDb": func(data Data) error {
    return db.Save(data)  // Never do this
}
```

The rule: **functions format data, they don't fetch or mutate it**.

### Common Functions Worth Having

```go
template.FuncMap{
    "upper":    strings.ToUpper,
    "lower":    strings.ToLower,
    "join":     strings.Join,
    "contains": strings.Contains,
    "default": func(def, val any) any {
        if val == nil || val == "" {
            return def
        }
        return val
    },
}
```

## Template Composition

Go templates compose through `define`, `template`, and `block`.

### Defining Reusable Templates

```go
{{define "header"}}
<header>{{.Title}}</header>
{{end}}

{{define "footer"}}
<footer>© {{.Year}}</footer>
{{end}}

<html>
{{template "header" .}}
<main>{{.Content}}</main>
{{template "footer" .}}
</html>
```

Note: you must pass context explicitly with `{{template "name" .}}`. Without the `.`, the sub-template gets nil.

### Base Layout Pattern

```go
// base.html
{{define "base"}}
<!DOCTYPE html>
<html>
<head><title>{{template "title" .}}</title></head>
<body>
  {{template "content" .}}
</body>
</html>
{{end}}

// page.html
{{define "title"}}My Page{{end}}
{{define "content"}}
<h1>Hello {{.Name}}</h1>
{{end}}
```

Execute with:

```go
tmpl.ExecuteTemplate(w, "base", data)
```

### Block: Define with Default

`block` combines definition with a default:

```go
{{define "base"}}
<html>
<body>
  {{block "content" .}}
    <p>Default content</p>
  {{end}}
</body>
</html>
{{end}}
```

If "content" isn't redefined elsewhere, the default renders. If it is, the override wins.

## Whitespace Control

Templates preserve whitespace, which can mess up your output:

```go
{{range .Items}}
  {{.}}
{{end}}
```

This produces extra newlines. Use `-` to trim:

```go
{{range .Items -}}
  {{.}}
{{- end}}
```

`{{-` trims whitespace before, `-}}` trims after.

## Putting It Together

Real templates combine these patterns:

```go
{{define "user-list"}}
{{with .Users}}
<ul>
{{range . -}}
  <li>{{.Name | upper}} - {{.Email}}</li>
{{- end}}
</ul>
{{else}}
<p>No users found</p>
{{end}}
{{end}}
```

This template:
- Guards against nil `.Users` with `with`
- Handles empty list with `else`
- Uses range for iteration
- Pipes through `upper` function
- Controls whitespace with `-`

## Key Takeaways

1. **`with` zooms in** and guards nil in one construct
2. **Range changes context**—save `$root` if you need outer data
3. **Functions must be pure**—formatting only
4. **Composition via define/template**—always pass context explicitly
5. **Control whitespace** with `{{-` and `-}}`

Next: [Part 3](/go-templates-part-3) covers production concerns—debugging, caching, security, and performance.
