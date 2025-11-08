# Reducing Error Boilerplate with thiserror in Rust

## Introduction

Error handling is a crucial aspect of Rust programming, but implementing custom error types can be verbose and repetitive. The standard library's `std::error::Error` trait requires implementing `Display`, handling error sources, and writing `From` implementations for error conversions—all of which adds significant boilerplate code.

The `thiserror` crate provides a powerful derive macro that automatically generates much of this boilerplate, making error types more concise and maintainable. This article demonstrates how refactoring from manual `std::error::Error` implementations to `thiserror` can dramatically reduce code while maintaining the same functionality.

---

## The Problem: Error Handling Boilerplate

### Standard Library Approach

When creating custom error types in Rust, you typically need to:

1. **Implement `Display`** - Format the error message
2. **Implement `Error`** - Provide error source information
3. **Implement `From`** - Convert from other error types
4. **Handle error variants** - Match on different error cases

This results in a lot of repetitive code, especially for enums with multiple variants.

### Example: Manual Error Implementation

Here's what a typical error type looks like using only the standard library:

```rust
use std::error::Error;
use std::fmt::{Display, Formatter, Result};
use std::net::AddrParseError;
use std::str::Utf8Error;

#[derive(Debug)]
pub enum NetworkError {
    /// Error parsing network address
    AddressParse(AddrParseError),
    /// Invalid UTF-8 encoding
    InvalidEncoding(Utf8Error),
    /// Generic error message
    Other(String),
}

impl Display for NetworkError {
    fn fmt(&self, f: &mut Formatter) -> Result {
        match self {
            NetworkError::AddressParse(e) => {
                write!(f, "address parse error: {}", e)
            }
            NetworkError::InvalidEncoding(e) => {
                write!(f, "invalid encoding: {}", e)
            }
            NetworkError::Other(msg) => {
                write!(f, "{}", msg)
            }
        }
    }
}

impl Error for NetworkError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            NetworkError::AddressParse(e) => Some(e),
            NetworkError::InvalidEncoding(e) => Some(e),
            NetworkError::Other(_) => None,
        }
    }
}

impl From<AddrParseError> for NetworkError {
    fn from(err: AddrParseError) -> Self {
        NetworkError::AddressParse(err)
    }
}

impl From<Utf8Error> for NetworkError {
    fn from(err: Utf8Error) -> Self {
        NetworkError::InvalidEncoding(err)
    }
}
```

**Lines of code: ~50 lines**

---

## The Solution: Using thiserror

### What is thiserror?

`thiserror` is a derive macro crate that automatically generates `Display`, `Error`, and `From` implementations for your error types. It uses attributes to configure how errors are displayed and converted.

### Same Example with thiserror

Here's the same error type using `thiserror`:

```rust
use std::net::AddrParseError;
use std::str::Utf8Error;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum NetworkError {
    #[error("address parse error: {0}")]
    AddressParse(#[from] AddrParseError),

    #[error("invalid encoding: {0}")]
    InvalidEncoding(#[from] Utf8Error),

    #[error("{0}")]
    Other(String),
}
```

**Lines of code: ~10 lines**

That's an **80% reduction** in boilerplate code! 🎉

---

## Real-World Refactoring Example

Let's examine a more complex real-world example inspired by an actual refactoring. This example shows an API client error type that was refactored from manual implementations to `thiserror`.

### Before: Manual Implementation

```rust
use std::error;
use std::fmt;
use hyper::Error as HyperError;
use http::Error as HttpError;

#[derive(Debug)]
pub enum ApiClientError {
    /// Error performing HTTP request
    Request(HyperError),
    /// 401 Not Authorized response
    NotAuthorized,
    /// Server returned 500 Internal Server Error
    ServerError(String),
    /// Unexpected HTTP status code
    UnexpectedStatus(u16),
    /// Error parsing URL
    UrlParse(http::uri::InvalidUri),
    /// Error converting response body from UTF-8
    InvalidEncoding(std::str::Utf8Error),
    /// HTTP protocol error
    HttpError(HttpError),
}

impl fmt::Display for ApiClientError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ApiClientError::UnexpectedStatus(code) => {
                write!(f, "unexpected HTTP status code: {}", code)
            }
            ApiClientError::NotAuthorized => {
                write!(f, "server returned 401 not authorized")
            }
            ApiClientError::ServerError(msg) => {
                write!(f, "{}", msg)
            }
            ApiClientError::Request(ref e) => {
                write!(f, "error making request: {}", e)
            }
            ApiClientError::UrlParse(e) => {
                write!(f, "error parsing URL: {}", e)
            }
            ApiClientError::InvalidEncoding(e) => {
                write!(f, "invalid encoding on response body: {}", e)
            }
            ApiClientError::HttpError(e) => {
                write!(f, "HTTP error: {}", e)
            }
        }
    }
}

impl error::Error for ApiClientError {
    fn source(&self) -> Option<&(dyn error::Error + 'static)> {
        match *self {
            ApiClientError::UnexpectedStatus(_) => None,
            ApiClientError::NotAuthorized => None,
            ApiClientError::ServerError(_) => None,
            ApiClientError::UrlParse(ref e) => Some(e),
            ApiClientError::Request(ref e) => Some(e),
            ApiClientError::InvalidEncoding(ref e) => Some(e),
            ApiClientError::HttpError(ref e) => Some(e),
        }
    }
}

// Error conversions
impl From<http::uri::InvalidUri> for ApiClientError {
    fn from(e: http::uri::InvalidUri) -> ApiClientError {
        ApiClientError::UrlParse(e)
    }
}

impl From<HyperError> for ApiClientError {
    fn from(e: HyperError) -> ApiClientError {
        ApiClientError::Request(e)
    }
}

impl From<http::Error> for ApiClientError {
    fn from(e: http::Error) -> ApiClientError {
        ApiClientError::HttpError(e)
    }
}

impl From<std::str::Utf8Error> for ApiClientError {
    fn from(e: std::str::Utf8Error) -> ApiClientError {
        ApiClientError::InvalidEncoding(e)
    }
}
```

**Lines of code: ~80 lines**

### After: Using thiserror

```rust
use hyper::Error as HyperError;
use http::Error as HttpError;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiClientError {
    #[error("error making request: {0}")]
    Request(#[from] HyperError),

    #[error("server returned 401 not authorized")]
    NotAuthorized,

    #[error("{0}")]
    ServerError(String),

    #[error("unexpected HTTP status code: {0}")]
    UnexpectedStatus(u16),

    #[error("error parsing URL: {0}")]
    UrlParse(#[from] http::uri::InvalidUri),

    #[error("invalid encoding on response body: {0}")]
    InvalidEncoding(#[from] std::str::Utf8Error),

    #[error("HTTP error: {0}")]
    HttpError(#[from] HttpError),
}
```

**Lines of code: ~20 lines**

That's a **75% reduction** in code while maintaining identical functionality!

---

## Key Features of thiserror

### 1. Automatic `Display` Implementation

The `#[error(...)]` attribute automatically generates `Display` implementation:

```rust
#[derive(Error)]
pub enum MyError {
    #[error("failed to connect: {0}")]
    ConnectionFailed(String),

    #[error("timeout after {0} seconds")]
    Timeout(u64),
}
```

### 2. Automatic `From` Implementations

The `#[from]` attribute automatically generates `From` implementations:

```rust
#[derive(Error)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),
}
```

Now you can use `?` operator directly:

```rust
fn read_file() -> Result<String, MyError> {
    let content = std::fs::read_to_string("file.txt")?; // Automatically converts!
    Ok(content)
}
```

### 3. Automatic Error Source

`thiserror` automatically implements `Error::source()` for variants with `#[from]`:

```rust
let err = MyError::Io(std::io::Error::new(
    std::io::ErrorKind::NotFound,
    "file not found"
));

// Automatically provides source
if let Some(source) = err.source() {
    println!("Source error: {}", source);
}
```

### 4. Custom Formatting

You can customize error messages with format strings:

```rust
#[derive(Error)]
pub enum ValidationError {
    #[error("field '{field}' is invalid: {reason}")]
    InvalidField { field: String, reason: String },

    #[error("value {0} is out of range [1, 100]")]
    OutOfRange(u64),
}
```

---

## Advanced Patterns

### Pattern 1: Struct Variants

For complex errors with multiple fields:

```rust
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("missing required field: {field}")]
    MissingField { field: String },

    #[error("invalid value for {field}: {value} (expected: {expected})")]
    InvalidValue {
        field: String,
        value: String,
        expected: String,
    },
}
```

### Pattern 2: Transparent Errors

For wrapping errors without changing the message:

```rust
#[derive(Debug, Error)]
pub enum MyError {
    #[error(transparent)]
    Io(#[from] std::io::Error),

    #[error(transparent)]
    Json(#[from] serde_json::Error),
}
```

### Pattern 3: Context and Backtraces

You can combine `thiserror` with `anyhow` for context:

```rust
use anyhow::Context;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ParseError {
    #[error("invalid format")]
    InvalidFormat,
}

fn parse_config() -> anyhow::Result<Config> {
    let content = std::fs::read_to_string("config.toml")
        .context("failed to read config file")?;

    toml::from_str(&content)
        .map_err(|e| ParseError::InvalidFormat)
        .context("failed to parse config")?;

    Ok(config)
}
```

### Pattern 4: Nested Error Types

You can nest error types for better organization:

```rust
#[derive(Debug, Error)]
pub enum AppError {
    #[error("network error: {0}")]
    Network(#[from] NetworkError),

    #[error("parse error: {0}")]
    Parse(#[from] ParseError),
}

#[derive(Debug, Error)]
pub enum NetworkError {
    #[error("connection failed: {0}")]
    ConnectionFailed(String),

    #[error("timeout after {0}s")]
    Timeout(u64),
}
```

---

## Comparison: Before and After

### Code Reduction

| Metric | Manual Implementation | With thiserror | Reduction |
|--------|---------------------|---------------|-----------|
| Lines of code | ~80 lines | ~20 lines | 75% |
| Display impl | ~20 lines | 0 lines | 100% |
| Error impl | ~15 lines | 0 lines | 100% |
| From impls | ~20 lines | 0 lines | 100% |
| Match statements | Multiple | 0 | 100% |

### Maintainability Benefits

1. **Less Code to Maintain**: Fewer lines mean fewer bugs
2. **Single Source of Truth**: Error messages defined in one place
3. **Type Safety**: Compile-time checking of error variants
4. **Easier Refactoring**: Change error structure without updating multiple implementations
5. **Better Documentation**: Error messages are co-located with variants

---

## Migration Guide

### Step 1: Add thiserror Dependency

Add to your `Cargo.toml`:

```toml
[dependencies]
thiserror = "1.0"
```

### Step 2: Replace Manual Implementations

**Before:**
```rust
#[derive(Debug)]
pub enum MyError {
    Io(std::io::Error),
}

impl Display for MyError { /* ... */ }
impl Error for MyError { /* ... */ }
impl From<std::io::Error> for MyError { /* ... */ }
```

**After:**
```rust
#[derive(Debug, Error)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

### Step 3: Update Error Messages

Move error messages from `Display::fmt` to `#[error(...)]` attributes:

```rust
// Before
impl Display for MyError {
    fn fmt(&self, f: &mut Formatter) -> Result {
        match self {
            MyError::Io(e) => write!(f, "IO error: {}", e),
        }
    }
}

// After
#[derive(Error)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

### Step 4: Remove Manual From Implementations

Replace manual `From` implementations with `#[from]`:

```rust
// Before
impl From<std::io::Error> for MyError {
    fn from(err: std::io::Error) -> Self {
        MyError::Io(err)
    }
}

// After
#[derive(Error)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),  // Automatic From impl!
}
```

---

## Complete Example

Here's a complete example showing a refactored error type:

```rust
use std::io;
use std::num::ParseIntError;
use std::str::Utf8Error;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DataProcessingError {
    #[error("failed to read file: {0}")]
    ReadError(#[from] io::Error),

    #[error("invalid UTF-8 encoding: {0}")]
    EncodingError(#[from] Utf8Error),

    #[error("failed to parse integer: {0}")]
    ParseError(#[from] ParseIntError),

    #[error("validation failed: {field} - {reason}")]
    ValidationError {
        field: String,
        reason: String,
    },

    #[error("processing timeout after {0} seconds")]
    Timeout(u64),

    #[error("unknown error: {0}")]
    Unknown(String),
}

// Usage example
fn process_data(file_path: &str) -> Result<Vec<i32>, DataProcessingError> {
    let content = std::fs::read_to_string(file_path)?; // Auto-converts io::Error

    let numbers: Vec<i32> = content
        .lines()
        .map(|line| line.parse()) // Returns Result<i32, ParseIntError>
        .collect::<Result<_, _>>()?; // Auto-converts ParseIntError

    if numbers.is_empty() {
        return Err(DataProcessingError::ValidationError {
            field: "numbers".to_string(),
            reason: "empty data set".to_string(),
        });
    }

    Ok(numbers)
}

#[tokio::main]
async fn main() {
    match process_data("data.txt") {
        Ok(numbers) => println!("Processed {} numbers", numbers.len()),
        Err(e) => {
            eprintln!("Error: {}", e);
            if let Some(source) = e.source() {
                eprintln!("Source: {}", source);
            }
        }
    }
}
```

---

## Best Practices

### 1. **Use Descriptive Error Messages**

```rust
// Good ✅
#[error("failed to connect to database at {host}:{port}: {reason}")]
ConnectionFailed { host: String, port: u16, reason: String }

// Less ideal ❌
#[error("error")]
ConnectionFailed { host: String, port: u16, reason: String }
```

### 2. **Use `#[from]` for Automatic Conversions**

```rust
// Good ✅
#[error("IO error: {0}")]
Io(#[from] std::io::Error)

// Less ideal ❌
#[error("IO error")]
Io(std::io::Error)  // Requires manual From impl
```

### 3. **Use `transparent` for Wrapper Errors**

```rust
// Good ✅ - Preserves original error message
#[error(transparent)]
Io(#[from] std::io::Error)

// Alternative - Custom message
#[error("file operation failed: {0}")]
Io(#[from] std::io::Error)
```

### 4. **Group Related Errors**

```rust
#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("connection error: {0}")]
    Connection(#[from] ConnectionError),

    #[error("query error: {0}")]
    Query(#[from] QueryError),
}
```

### 5. **Provide Context in Error Messages**

```rust
#[error("failed to process user {user_id}: {reason}")]
ProcessingFailed { user_id: String, reason: String }
```

---

## When to Use thiserror vs Alternatives

### Use thiserror when:
- ✅ You want type-safe error handling
- ✅ You need custom error types with variants
- ✅ You want automatic `From` implementations
- ✅ You need structured error information
- ✅ You're building a library with public error types

### Consider alternatives when:
- ❌ You just need simple error messages (use `anyhow::Context`)
- ❌ You're prototyping and don't need structured errors
- ❌ You want to minimize dependencies (use manual impls)

---

## Performance Considerations

`thiserror` has **zero runtime cost**. All code generation happens at compile time:

- No runtime overhead
- No dynamic dispatch
- Same performance as manual implementations
- Generated code is optimized by the compiler

---

## Conclusion

The `thiserror` crate dramatically reduces boilerplate code when implementing custom error types in Rust. By using derive macros, you can:

- **Reduce code by 70-80%** compared to manual implementations
- **Maintain type safety** and compile-time guarantees
- **Improve maintainability** with co-located error messages
- **Simplify refactoring** by centralizing error definitions

The refactoring examples shown in this article demonstrate real-world improvements: going from 80 lines of boilerplate to just 20 lines while maintaining identical functionality. This makes error handling in Rust more ergonomic and less error-prone.

### Key Takeaways

1. `thiserror` eliminates most error handling boilerplate
2. Use `#[error(...)]` for custom error messages
3. Use `#[from]` for automatic error conversions
4. Use `transparent` to preserve original error messages
5. Error messages are co-located with variants for better maintainability

### Further Reading

- [thiserror documentation](https://docs.rs/thiserror/)
- [Rust Error Handling Book](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
- [anyhow crate](https://docs.rs/anyhow/) - For application error handling
- [Error Handling in Rust](https://blog.burntsushi.net/rust-error-handling/)

---
