# Phase 0: Rust Fundamentals

## Overview

**Priority:** P1
**Status:** pending
**Duration:** 2 weeks
**Goal:** Master Rust core concepts before building the chat client

## Key Insights

- Ownership/borrowing is THE core concept - everything else builds on this
- .NET developers struggle most with ownership and lifetimes
- Rust Book + Rustlings is the proven learning path
- Don't skip fundamentals - they bite back later

## Learning Topics

### Week 1: Core Concepts

| Topic | Rust Book | Rustlings | Priority |
|-------|-----------|-----------|----------|
| Variables & Mutability | Ch 3.1 | variables1-6 | High |
| Data Types | Ch 3.2 | primitive_types1-6 | High |
| Functions | Ch 3.3 | functions1-5 | High |
| Control Flow | Ch 3.5 | if1-3, quiz1 | High |
| Ownership | Ch 4.1 | ownership1-5 | **Critical** |
| References & Borrowing | Ch 4.2 | borrow1-3 | **Critical** |
| Slices | Ch 4.3 | slice1-2 | High |

### Week 2: Advanced Concepts

| Topic | Rust Book | Rustlings | Priority |
|-------|-----------|-----------|----------|
| Structs | Ch 5 | structs1-3 | High |
| Enums & Pattern Matching | Ch 6 | enums1-5, match1-2 | **Critical** |
| Packages, Crates, Modules | Ch 7 | modules1-3 | High |
| Collections (Vec, String, HashMap) | Ch 8 | vecs1-3, strings1-4, hashmaps1-3 | High |
| Error Handling | Ch 9 | option1-3, result1-4 | **Critical** |
| Generics | Ch 10.1 | generics1-2 | High |
| Traits | Ch 10.2 | traits1-5 | **Critical** |
| Lifetimes | Ch 10.3 | lifetimes1-3 | **Critical** |

## Resources

### Primary
- [The Rust Book](https://doc.rust-lang.org/book/) - Official, excellent quality
- [Rustlings](https://github.com/rust-lang/rustlings) - Interactive exercises

### Supplementary
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) - Code-first learning
- [Rust Language Cheat Sheet](https://cheats.rs/) - Quick reference

## Practice Exercises

### Exercise 1: Calculator with Error Handling
```rust
// Goal: Practice Result, enums, pattern matching
// Build a calculator that handles division by zero, invalid input

fn calculate(a: i32, b: i32, op: Operation) -> Result<i32, CalcError> {
    todo!()
}

#[derive(Debug)]
enum Operation { Add, Sub, Mul, Div }

#[derive(Debug)]
enum CalcError { DivisionByZero, Overflow }
```

### Exercise 2: Mini Key-Value Store
```rust
// Goal: Practice HashMap, ownership, borrowing
// Build a simple in-memory key-value store with:
// - insert(key, value)
// - get(key) -> Option<&str>
// - delete(key)

struct KeyValueStore {
    data: HashMap<String, String>
}
```

### Exercise 3: CLI Argument Parser
```rust
// Goal: Practice String handling, Vec, Option
// Parse command line arguments manually (no clap yet)
// ./myapp --help
// ./myapp --name "hello"
// ./myapp --count 5

fn parse_args(args: &[String]) -> Result<Config, ParseError> {
    todo!()
}
```

## .NET Mental Model Shifts

### Ownership vs GC
```
// C# - GC handles everything
var list = new List<int> { 1, 2, 3 };
var list2 = list; // Both point to same object
// No worries about who owns what

// Rust - You track ownership
let list = vec![1, 2, 3];
let list2 = list; // list is MOVED, can't use it anymore
// Or:
let list2 = &list; // Borrow, list still valid
let list2 = &mut list; // Mutable borrow, exclusive access
```

### Null vs Option
```
// C#
string name = null; // Valid, runtime NullReferenceException
string? name = null; // Nullable reference type (C# 8+)

// Rust - No null, must use Option
let name: Option<&str> = None;
let name: Option<&str> = Some("Alice");

// Pattern matching required
match name {
    Some(n) => println!("Hello, {}", n),
    None => println!("No name"),
}
```

### Exceptions vs Result
```
// C#
try {
    var result = DoSomething();
} catch (Exception ex) {
    // Handle
}

// Rust - Errors are values
match do_something() {
    Ok(result) => println!("{}", result),
    Err(e) => eprintln!("Error: {}", e),
}

// Or use ? operator
fn my_function() -> Result<(), Error> {
    let result = do_something()?; // Propagates error
    Ok(())
}
```

## Todo Checklist

### Week 1
- [ ] Install Rust: `rustup install stable`
- [ ] Install Rustlings: `cargo install rustlings`
- [ ] Read Rust Book Chapters 1-4
- [ ] Complete Rustlings: variables, functions, if, primitive_types
- [ ] Complete Rustlings: ownership, borrow, slice
- [ ] Practice: Write a simple guessing game

### Week 2
- [ ] Read Rust Book Chapters 5-10
- [ ] Complete Rustlings: structs, enums, match
- [ ] Complete Rustlings: modules, vecs, strings, hashmaps
- [ ] Complete Rustlings: option, result, generics, traits, lifetimes
- [ ] Practice: Build mini key-value store
- [ ] Practice: Build CLI argument parser

## Success Criteria

- [ ] All Rustlings exercises completed
- [ ] Understand ownership vs borrowing vs cloning
- [ ] Comfortable with `Option` and `Result`
- [ ] Can read Rust code and understand lifetimes
- [ ] Understand trait-based polymorphism

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Overwhelmed by ownership | High | Use Rust Book visualizations, practice with small examples |
| Lifetimes confusion | High | Start with simple cases, compiler hints are helpful |
| Giving up on Rustlings | Medium | Set daily goals (5-10 exercises), celebrate progress |

## Next Steps

After completing Phase 0:
1. Move to [Phase 1: Minimal Chat Client](./phase-01-minimal-chat-client.md)
2. Start with HTTP client implementation
3. Reference ZeroClaw's `glm.rs` for patterns