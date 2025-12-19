+++
title = "Go Templates Part 3: Production Hardening"
date = 2025-12-19
description = "Debugging, caching, security pitfalls, and performance considerations for production Go templates"
[taxonomies]
tags = ["go", "templates", "production", "security"]
+++

Templates look harmless until they start panicking in production at 3 AM. This post covers the real-world concerns: debugging execution errors, caching safely, avoiding security mistakes, and keeping things fast.

<!-- more -->

## Debugging Template Failures

Template errors fall into two categories: parse-time and execution-time.

### Parse-Time Errors

These happen when you call `Parse()`:

```go
template: page.html:12: unexpected "{{end}}"
```

Parse errors mean your template syntax is broken. Find them early:

```go
// Crash at startup, not at request time
var tmpl = template.Must(template.ParseFiles("page.html"))
```

`Must` panics on parse error. In production, you want to fail fast.

### Execution-Time Errors

These happen during `Execute()`:

```go
template: page:5:18: executing "page" at <.User.Name>: 
    can't evaluate field Name in type *main.User (nil pointer)
```

This means `.User` was nil when the template tried to access `.Name`.

### Debugging Techniques

**1. Dump the context**

```go
{{printf "%#v" .}}
```

This prints the Go representation of `.`. Useful when you're confused about what data you actually have.

**2. Use `with` for nil safety**

```go
// Panics on nil
{{.User.Name}}

// Handles nil gracefully
{{with .User}}{{.Name}}{{else}}no user{{end}}
```

**3. Always check `Execute` errors**

```go
if err := tmpl.Execute(w, data); err != nil {
    log.Printf("template error: %v", err)
    http.Error(w, "render failed", 500)
    return
}
```

Never ignore the error from `Execute`. It tells you exactly what went wrong.

## Template Caching Strategy

### The Wrong Way

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Parsing on every request
    tmpl := template.Must(template.ParseFiles("page.html"))
    tmpl.Execute(w, data)
}
```

Problems:
- Slow (parsing is expensive)
- Memory churn (allocations every request)
- Late failure (parse errors at request time)

### The Right Way

```go
var templates = template.Must(
    template.ParseGlob("templates/*.html"),
)

func handler(w http.ResponseWriter, r *http.Request) {
    templates.ExecuteTemplate(w, "page.html", data)
}
```

Templates are parsed once at startup, then executed many times.

### Production Pattern: Template Registry

```go
type TemplateStore struct {
    templates *template.Template
}

func NewTemplateStore(pattern string) (*TemplateStore, error) {
    t, err := template.ParseGlob(pattern)
    if err != nil {
        return nil, err
    }
    return &TemplateStore{templates: t}, nil
}

func (s *TemplateStore) Render(w io.Writer, name string, data any) error {
    return s.templates.ExecuteTemplate(w, name, data)
}
```

Centralize template access. One place to add logging, metrics, or error handling.

### Hot Reload in Development

During development, you might want templates to reload without restart:

```go
func (s *TemplateStore) Render(w io.Writer, name string, data any) error {
    if devMode {
        // Re-parse on every render (dev only)
        t := template.Must(template.ParseGlob("templates/*.html"))
        return t.ExecuteTemplate(w, name, data)
    }
    return s.templates.ExecuteTemplate(w, name, data)
}
```

Never enable this in production.

## Security Pitfalls

### XSS via Wrong Package

```go
import "text/template"  // ❌ for HTML

tmpl.Execute(w, "<script>alert('xss')</script>")
// Output: <script>alert('xss')</script>
```

This is an XSS vulnerability. Use `html/template` for any HTML output:

```go
import "html/template"  // ✅

tmpl.Execute(w, "<script>alert('xss')</script>")
// Output: &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
```

Auto-escaping saves you.

### Data Leakage

Never pass raw domain objects to templates:

```go
// ❌ Dangerous: exposes everything
tmpl.Execute(w, user)

// ✅ Safe: explicit view model
tmpl.Execute(w, UserView{
    Name:  user.Name,
    Email: user.Email,
    // Deliberately omit: PasswordHash, InternalID, etc.
})
```

Templates can access any exported field. Control what's visible.

### Unsafe Functions

Template functions can be attack vectors:

```go
// ❌ Never do this
Funcs(template.FuncMap{
    "readFile": ioutil.ReadFile,  // File system access
    "env":      os.Getenv,        // Environment secrets
    "eval":     dangerousEval,    // Code execution
})
```

Functions should only format data:

```go
// ✅ Safe
Funcs(template.FuncMap{
    "upper":      strings.ToUpper,
    "formatDate": formatDate,
    "currency":   formatCurrency,
})
```

### Multi-Tenant Template Safety

If different tenants share a template service:

```go
type TenantView struct {
    // Only tenant-specific, sanitized data
    Name    string
    Items   []ItemView
    // Never: OtherTenantData, AdminConfig, etc.
}
```

Never pass cross-tenant data or system internals.

## Performance Considerations

### Templates Are Usually Not the Bottleneck

Before optimizing templates, measure. Most apps spend more time in:
- Database queries
- Network calls
- JSON serialization

Profile first: `go tool pprof`.

### When Templates Are Slow

Red flags:
- Very deep nesting
- Large loops with complex bodies
- Many function calls per element
- Parsing per request (the biggest one)

### Output Size Limits

Protect against runaway templates:

```go
var buf bytes.Buffer
if err := tmpl.Execute(&buf, data); err != nil {
    return err
}

if buf.Len() > 1<<20 {  // 1MB limit
    return errors.New("template output too large")
}

w.Write(buf.Bytes())
```

This prevents memory exhaustion from malformed data.

### Streaming Large Outputs

Templates stream by default—they write incrementally to `io.Writer`:

```go
// Writes directly to response, low memory
tmpl.Execute(w, data)
```

No intermediate buffer needed for normal cases.

## Comparison: Go vs Jinja vs Handlebars

| Aspect | Go | Jinja | Handlebars |
|--------|-----|-------|------------|
| Logic in template | Minimal | Extensive | Minimal |
| XSS protection | Built-in | Opt-in | Partial |
| Performance | Fast | Slow | Medium |
| Debugging | Harder | Easier | Medium |
| Philosophy | Templates render | Templates compute | Templates render |

Go templates force you to prepare data in Go. This seems limiting until you realize it makes templates predictable, testable, and secure.

## Key Takeaways

1. **Parse once at startup**—never per request
2. **Use `html/template` for HTML**—always
3. **Create view models**—don't pass raw domain objects
4. **Check `Execute` errors**—they're informative
5. **Guard nil access with `with`**—or suffer panics
6. **Functions must be pure**—no I/O, no mutations

Next: [Part 4](/go-templates-part-4) covers scaling—template services, versioning, observability, and operating templates as infrastructure.
