# System Prompt for Go Codebase

<!-- ref: https://github.com/KentBeck/BPlusTree3 -->

## Code Quality Standards

NEVER write production code that contains:

1. **panic() in normal operation paths** – always return `error`
2. **resource leaks** – every opened resource must be closed with `defer`
3. **data corruption potential** – all state transitions must preserve consistency
4. **inconsistent error handling patterns** – establish and follow a single pattern (e.g. `if err != nil { return fmt.Errorf("context: %w", err) }`)

ALWAYS:

1. **Write comprehensive tests BEFORE implementing features**
2. **Validate invariants in data structures and state machines**
3. **Check bounds for all numeric conversions and slice/index accesses**
4. **Document known bugs immediately and fix them before continuing**
5. **Implement proper separation of concerns (package-level responsibilities clear)**
6. **Use static analysis tools (`go vet`, `staticcheck`, `ineffassign`) before code is considered complete**

---

## Development Process Guards

### TESTING REQUIREMENTS:
- Write failing tests first (TDD), then implement to make them pass
- Never commit code that expects `panic` for a bug — fix the bug
- Use property-based testing for data structures when applicable
- Include benchmarks (`go test -bench .`) for performance-critical code
- Validate edge cases and boundary conditions with explicit tests

### ARCHITECTURE REQUIREMENTS:
- Explicit error handling — no silent ignores
- Resource safety — every `Close` must be paired with `defer`
- Performance conscious — avoid unnecessary allocations, reflection, and goroutine leaks
- API design — consistent, minimal, and idiomatic Go patterns across all exported interfaces

### REVIEW CHECKPOINTS:

Before marking any code complete, verify:

1. **No compilation or vet warnings**
2. **All tests pass (including benchmarks and race detector `-race`)**
3. **Memory usage and goroutine counts are bounded and predictable**
4. **No data corruption potential in any code path**
5. **Error handling is comprehensive and consistent**
6. **Code is modular and maintainable**
7. **Documentation matches implementation**
8. **Performance benchmarks show acceptable results**

---

## Go-Specific Quality Standards

### ERROR HANDLING:
- Return `(T, error)` for all fallible operations
- Wrap errors with context using `fmt.Errorf("context: %w", err)`
- Never ignore errors (use `_ =` only when justified and documented)
- Provide meaningful error messages (include operation and key parameters)

### RESOURCE MANAGEMENT:
- Always `defer resource.Close()` immediately after open
- Avoid goroutine leaks — exit goroutines via `context.Context`
- Use `sync.WaitGroup` / `errgroup.Group` for concurrent lifecycles
- Benchmark long-running scenarios for leaks with `pprof`

### DATA STRUCTURE INVARIANTS:
- Document all invariants in comments
- Test invariant preservation across operations
- Validate state consistency at package boundaries
- Use types to enforce invariants (e.g. `type PositiveInt int`)

### PACKAGE ORGANIZATION:
- One responsibility per package
- Clear public vs private API (capitalize exported identifiers)
- Package-level documentation with `doc.go`
- Logical dependency hierarchy, avoid cyclic deps

---

## Critical Patterns to Avoid

### DANGEROUS PATTERNS:
```go
// NEVER DO THIS – production panic
panic("This should never happen")

// NEVER DO THIS – unchecked conversion
id := int32(size) // may truncate

// NEVER DO THIS – ignoring errors
data, _ := os.ReadFile("config.json")

// NEVER DO THIS – leaking resources
f, _ := os.Open("file.txt")
// ... forgot f.Close()
````

### PREFERRED PATTERNS:

```go
// DO THIS – proper error handling
func operation() error {
    val, err := riskyOperation()
    if err != nil {
        return fmt.Errorf("operation failed: %w", err)
    }
    return process(val)
}

// DO THIS – safe conversion
id, err := safeConvert(size)
if err != nil {
    return fmt.Errorf("invalid size: %d", size)
}

// DO THIS – explicit error handling
if err := someOperation(); err != nil {
    return fmt.Errorf("failed someOperation: %w", err)
}

// DO THIS – defer resource cleanup
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close()
```

---

## Testing Standards

### COMPREHENSIVE TEST COVERAGE:

* Unit tests for all public functions
* Integration tests for cross-package flows
* Property-based tests using `testing/quick` or testify/quickcheck
* Stress tests for concurrent/goroutine-heavy code
* Leak detection tests (goroutines, file descriptors)
* Edge case and boundary condition tests

### TEST ORGANIZATION:

```go
func TestNormalOperation(t *testing.T) {
    // Typical usage patterns
}

func TestEdgeCases(t *testing.T) {
    // Boundary conditions
}

func TestErrorConditions(t *testing.T) {
    // All error paths
}

func TestInvariantsPreserved(t *testing.T) {
    // Verify data structure invariants
}
```

### MEMORY / CONCURRENCY TESTING:

```go
func TestNoGoroutineLeaks(t *testing.T) {
    before := runtime.NumGoroutine()
    runOperationsWithContext()
    after := runtime.NumGoroutine()
    if after > before {
        t.Fatalf("goroutine leak detected: before=%d after=%d", before, after)
    }
}
```

---

## Documentation Standards

### CODE DOCUMENTATION:

* Document all exported identifiers with GoDoc style
* Provide example usage in `_test.go` files
* Explain invariants and preconditions
* Include notes about concurrency safety
* Avoid repeating implementation detail in comments

### ERROR DOCUMENTATION:

```go
// Insert adds a key-value pair to the map.
//
// Arguments:
//   key   – must not be empty
//   value – arbitrary string
//
// Returns:
//   error if key already exists
//
// Example:
//   m := New()
//   if err := m.Insert("k", "v"); err != nil {
//       log.Fatal(err)
//   }
func (m *Map) Insert(key, value string) error {
    // Implementation
}
