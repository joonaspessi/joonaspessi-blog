---
title: "Custom Error Handling in Rust"
date: 2026-02-02
description: "Learn patterns for building custom error types in Rust."
author: "Joonas Pessi"
---

If you've ever stared at an error message that just said "parse error" with no indication of what went wrong or where, you know the frustration. Generic errors leave you guessing, adding println! statements everywhere, and wasting time on problems that should be immediately obvious.

Rust's type system gives us the tools to do better. By defining custom error enums, we can make our errors as specific and helpful as our function signatures. Each variant can carry exactly the context needed to understand and fix the problem.

In this post, I'll walk through five patterns for structuring error enum variants, using a simple CSV parser as a running example. By the end, you'll have a toolkit for designing error types that make debugging a pleasure instead of a chore.

This example is based on the one seen in this [video](https://www.youtube.com/watch?v=KrZ0nmpNVOw), so all credit goes there.

## The Running Example: A CSV Parser

To make these patterns concrete, There's a small CSV parser that reads a file and parses its contents into a typed data structure. You can find the full source code at [github.com/joonaspessi/rust-custom-error-kata](https://github.com/joonaspessi/rust-custom-error-kata).

The main function has this signature:

```rust
pub fn read_csv<T: Copy + Default + FromStr>(filename: &str) -> Result<CsvData<T>>
```

It takes a filename and returns either parsed CSV data or an error. The `CsvData` struct holds a header row and a vector of typed data rows:

```rust
#[derive(Debug)]
pub struct CsvData<T: Copy + Default + FromStr> {
    pub header: Vec<String>,
    pub data: Vec<Vec<T>>,
}
```

What can go wrong when parsing a CSV file? Quite a lot:

- The file might not exist
- The file might exist but be unreadable
- The file might be empty
- A line might fail to parse
- A value might not convert to the expected type
- A row might have too few columns
- A row might have too many columns

Each of these failures needs different information to be useful. Let's look at how to model them.

## Five Patterns for Error Variants

### Pattern 1: Unit Variants

The simplest error variants carry no data at all:

```rust
FileNonExistent,
FileIsEmpty,
```

These are appropriate when the error type itself tells the whole story. If the file doesn't exist, you know exactly what happened. There's no additional context that would help—the filename is already in the calling code.

Use unit variants when:

- The error is completely described by its name
- No additional context would help debugging
- You want the simplest possible representation

Here's how these errors get created:

```rust
pub fn read_to_lines(filename: &str) -> Result<Vec<String>> {
    let path = std::path::Path::new(filename);
    if !path.exists() {
        return Err(CsvError::FileNonExistent);
    }
    // ...
}
```

### Pattern 2: Wrapping Standard Library Errors

When something goes wrong in a standard library call, you often want to preserve that error while adding domain context:

```rust
CouldNotOpenFile(io::Error),
```

This pattern wraps an `io::Error` inside your custom error type. The inner error contains details like permission denied or disk full, while the outer type indicates this happened while opening a file specifically.

The power of this pattern becomes clear when you implement `From`:

```rust
impl From<io::Error> for CsvError {
    fn from(value: io::Error) -> Self {
        Self::CouldNotOpenFile(value)
    }
}
```

Now you can use the `?` operator directly:

```rust
let file = OpenOptions::new().read(true).open(path)?;
```

If `open` fails, the `io::Error` automatically converts to `CsvError::CouldNotOpenFile`. No explicit error mapping needed.

Use wrapped errors when:

- You want to preserve the original error's details
- The standard library error needs domain context
- You want ergonomic `?` operator support

### Pattern 3: Trait Objects for Flexibility

Sometimes you don't know the concrete error type at compile time, or you want to handle multiple error types uniformly:

```rust
CouldNotParseLine(Box<dyn Error>),
```

This uses a trait object to store any error that implements the `Error` trait. It's flexible but loses type information—you can't pattern match on the specific error type later.

In the CSV parser, this handles errors from the `BufReader`:

```rust
let lines: Vec<_> = BufReader::new(file).lines().collect();
lines
    .into_iter()
    .map(|line| line.map_err(|e| CsvError::CouldNotParseLine(Box::new(e))))
    .collect()
```

Use trait objects when:

- Multiple error types could occur in the same place
- You don't need to handle specific error types differently
- Flexibility matters more than type precision

### Pattern 4: String Context

When the error message needs to include dynamic information like a specific value that failed to parse:

```rust
CouldNotParseValue(String),
```

This captures the actual string that couldn't be converted, making the error message immediately actionable:

```rust
let entries: Vec<Result<T>> = lines[i]
    .split(",")
    .map(|e| {
        let res = e.parse::<T>();
        res.map_err(|_| CsvError::CouldNotParseValue(e.into()))
    })
    .collect();
```

If you see `CouldNotParseValue("abc")` when parsing integers, you know exactly which value was the problem.

Use string context when:

- You need to capture a specific value that caused the error
- The problematic value is naturally a string (or converts to one)
- A single piece of context is sufficient

### Pattern 5: Custom Structs for Rich Context

When you need multiple pieces of information, define a dedicated struct:

```rust
#[derive(Debug)]
pub struct CsvLineLen {
    pub line_num: usize,
    pub num_entries: usize,
}

#[derive(Debug)]
pub enum CsvError {
    // ...
    LineTooShort(CsvLineLen),
    LineTooLong(CsvLineLen),
}
```

Now errors carry structured data that can be programmatically accessed:

```rust
if entries.len() < header.len() {
    return Err(CsvError::LineTooShort(CsvLineLen {
        line_num: i,
        num_entries: entries.len(),
    }));
} else if entries.len() > header.len() {
    return Err(CsvError::LineTooLong(CsvLineLen {
        line_num: i,
        num_entries: entries.len(),
    }));
}
```

An error like `LineTooShort(CsvLineLen { line_num: 5, num_entries: 2 })` tells you exactly where to look: line 5 only had 2 entries when more were expected.

Use custom structs when:

- Multiple pieces of context are needed together
- The context has a logical structure worth naming
- You want to access individual fields programmatically

## Supporting Infrastructure

Beyond the enum variants themselves, you need a few pieces of infrastructure to make custom errors work smoothly in Rust.

### Type Alias for Convenience

Define a type alias so you don't have to repeat the error type everywhere:

```rust
type Result<T> = std::result::Result<T, CsvError>;
```

Now your function signatures can just say `Result<CsvData<T>>` instead of `std::result::Result<CsvData<T>, CsvError>`.

### Display and Error Traits

Rust's error handling ecosystem expects errors to implement two traits:

```rust
impl std::fmt::Display for CsvError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:?}", self)
    }
}

impl Error for CsvError {}
```

The `Display` implementation here just delegates to `Debug`, which is quick but not ideal for production. In a real application, you'd want human-readable messages for each variant.

The `Error` trait implementation is empty because we don't need to override any default methods. Just implementing it marks our type as an error type.

### From Trait for Ergonomic Conversion

As shown earlier, implementing `From` enables the `?` operator:

```rust
impl From<io::Error> for CsvError {
    fn from(value: io::Error) -> Self {
        Self::CouldNotOpenFile(value)
    }
}
```

You can implement `From` for any error type you want to convert automatically. This keeps your parsing logic clean and focused on the happy path.

## The Complete Error Type

Here's the full `CsvError` enum with all five patterns:

```rust
#[derive(Debug)]
pub enum CsvError {
    // Pattern 1: Unit variants
    FileNonExistent,
    FileIsEmpty,

    // Pattern 2: Wrapped standard library error
    CouldNotOpenFile(io::Error),

    // Pattern 3: Trait object for flexibility
    CouldNotParseLine(Box<dyn Error>),

    // Pattern 4: String context
    CouldNotParseValue(String),

    // Pattern 5: Custom struct for rich context
    LineTooShort(CsvLineLen),
    LineTooLong(CsvLineLen),
}
```

## Takeaways

Start with the simplest variant that captures enough information. Unit variants are often sufficient. Add complexity only when debugging genuinely requires more context.

For larger projects, consider the [thiserror](https://docs.rs/thiserror/latest/thiserror/) crate, which generates `Display` and `Error` implementations from attributes. For applications where you just need to propagate errors without handling them, [anyhow](https://docs.rs/anyhow/latest/anyhow/) provides a convenient catch-all error type.

But understanding these five patterns gives you the foundation to make those choices deliberately. When an error occurs, you'll have exactly the information you need to fix it.
