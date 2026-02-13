# Reducing Entropy in Go

---
description: >-
  Reducing-entropy principles adapted specifically to Go idioms. Less code,
  fewer types, fewer abstractions. Every line of code is a liability.
  Measure the end state, not the effort.
---

## Core Principle

More code begets more code. Entropy accumulates. The goal is **less total code
in the final codebase** -- not less code to write right now.

- Writing 50 lines that delete 200 lines = net win.
- Keeping 14 functions to avoid writing 2 = net loss.
- "No churn" is not a goal. Less code is the goal.

**Measure the end state, not the effort.**

For the full philosophy, see the **reducing-entropy** skill.

## Interface Bloat

An interface with 5+ methods is probably a concrete type in disguise.

**The gold standard is `io.Reader`:** one method, universally useful, composes
trivially.

- 1-2 methods: probably a good interface.
- 3-4 methods: suspicious. Are all methods needed by all consumers?
- 5+ methods: almost certainly wrong. Split it or use a concrete type.

If an interface has only one implementation and no test double, delete the
interface and use the concrete type. You can always extract an interface later
when a second implementation actually appears.

```go
// BAD: interface with one implementation
type UserStore interface {
    Get(id string) (*User, error)
    Create(u *User) error
    Update(u *User) error
    Delete(id string) error
    List(filter Filter) ([]*User, error)
}

// GOOD: just use the concrete type
type UserStore struct { db *sql.DB }
func (s *UserStore) Get(id string) (*User, error) { ... }
```

If a caller only needs one method, accept a one-method interface at the call
site:

```go
type UserGetter interface {
    Get(id string) (*User, error)
}

func handleGetUser(store UserGetter, id string) (*User, error) { ... }
```

## Package Proliferation

Do not create a package for 1-2 files. `internal/` is fine for hiding
implementation details, but ask whether you even need the boundary.

Signs a package should be folded into its parent:

- Only one file.
- Only one exported symbol.
- No other package imports it except the parent.
- The package name just echoes the parent (e.g., `server/serverutil`).

## Custom Error Types

Do you need `type FooError struct{...}` or is `fmt.Errorf("foo: %w", err)`
enough?

Create a custom error type **only** when callers need to match on it
programmatically:

```go
// Worth it: callers check error type to decide retry behavior
if connErr, ok := errors.AsType[*ConnError](err); ok {
    if connErr.Retryable {
        retry()
    }
}

// Not worth it: callers just log the error
// fmt.Errorf("connect to %s: %w", addr, err) is sufficient
```

With `errors.AsType` (Go 1.26), typed errors are cheaper to check. But the
type itself is still code that must be maintained. Only create it if callers
actually branch on it.

## Builder Patterns

In Go, struct literals + functional options cover 95% of use cases. A builder
type is almost always more code for zero benefit.

```go
// BAD: 40+ lines of builder boilerplate
server := NewServerBuilder().
    WithPort(8080).
    WithTimeout(30 * time.Second).
    WithLogger(logger).
    Build()

// GOOD: 5 lines, same result, zero indirection
server := &Server{
    Port:    8080,
    Timeout: 30 * time.Second,
    Logger:  logger,
}
```

Functional options (`func(*Server)`) are warranted when defaults are complex
or when the API is a public library with backward compatibility concerns. For
internal code, struct literals are almost always sufficient.

## Getters and Setters

If a field can be public, make it public. A getter that returns a field with no
logic is pure entropy.

```go
// BAD: 6 lines of code that do nothing
func (s *Server) Port() int { return s.port }
func (s *Server) SetPort(p int) { s.port = p }

// GOOD: 0 lines -- just make it a public field
type Server struct {
    Port int
}
```

Add a getter/setter only when there is validation, side effects, or the field
needs to be computed.

## Thin Wrappers

If a function just calls another function with the same arguments and return
values, delete it:

```go
// BAD: adds indirection, no logic
func CloseConn(c *Conn) error {
    return c.Close()
}

// GOOD: callers call c.Close() directly
```

## Premature Generics

If a generic function is only used with one type today, make it concrete.
Generics should delete duplicate code, not prepare for hypothetical future
types.

```go
// BAD: generic for no reason (only used with string)
func First[T any](s []T) T { return s[0] }

// GOOD: concrete, less cognitive overhead
func FirstString(s []string) string { return s[0] }

// ACTUALLY GOOD: generic because it replaced 4 duplicated functions
func First[T any](s []T) T { return s[0] }
```

The litmus test: does the generic version delete more code than it adds? If
yes, use generics. If no, stay concrete.

## The Deletion Checklist

Every time you add code, ask:

1. What does this new code make obsolete? Delete it.
2. What was only needed because of what we are replacing? Delete it.
3. Is there a function/type that is now unreachable? Delete it.
4. Are there TODO comments older than 6 months? Delete them (and the code they
   reference if it was never done).
5. Is there an abstraction layer that no longer earns its keep? Inline it.

## Red Flags

- **"Keep what exists"** -- Status quo bias. The question is total code, not
  churn.
- **"This adds flexibility"** -- For what? YAGNI.
- **"Better separation of concerns"** -- More files/packages = more code.
  Separation is not free.
- **"Type safety"** -- Worth how many lines? Sometimes a runtime check in less
  code wins.
- **"Easier to understand"** -- 14 things are not easier than 2 things.

## Further Reading

- **reducing-entropy** skill -- The full entropy-reduction philosophy.
- [Rich Hickey, "Simple Made Easy"](https://www.infoq.com/presentations/Simple-Made-Easy/) -- Simple != easy. Complecting things creates entropy.
- [Rob Pike, "Simplicity is Complicated"](https://www.youtube.com/watch?v=rFejpH_tAHM) -- Go's design philosophy of deliberate simplicity.
