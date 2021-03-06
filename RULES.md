# An error-style guide

This document tries to provide a set of rules and guidelines to help with error-handling best-practices[^1].

[^1]: Or more honestly, perform a lot of *bikeshedding*.

## Prerequisites

It it assumed that the reader is a reasonably proficient Rust programmer and aware of at least the following concepts:

* [Algebraic data types (in Rust)](https://doc.rust-lang.org/book/second-edition/ch06-00-enums.html),
* [`From`/`Into` traits](https://doc.rust-lang.org/rust-by-example/conversion/from_into.html),
* [the `try!` macro and its replacement](https://doc.rust-lang.org/std/macro.try.html) and
* [how all these tie together](https://doc.rust-lang.org/book/second-edition/ch09-00-error-handling.html).
* [RFC 1937: `?` in `main`](https://github.com/rust-lang/rust/issues/43301)

### Other sources

Documents that inspired this, some of which might be more of a historical read, are

* ["Error Handling in Rust" (Andrew Gallant)](https://blog.burntsushi.net/rust-error-handling/),
* [failure docs](https://boats.gitlab.io/failure/),
* ["Announcing failure"](https://boats.gitlab.io/blog/post/2017-11-16-announcing-failure/),
* ["Failure 1.0"](https://boats.gitlab.io/blog/post/2018-02-22-failure-1.0/).
* [Bay Area Rust Meetup November 2017](https://air.mozilla.org/bay-area-rust-meetup-november-2017/)
* [Redefining failure (Ed Page)](https://epage.github.io/blog/2018/03/redefining-failure/)

## Definitions

Error handling is dependent on context, different environments have different needs for error handling. The following guidelines assume three different contexts, named and defined as follows:

<dl>
<dt>**application**</dt><dd>Application is the most common context for executable Rust programs, being run on an operating system with plenty of memory and computing power.</dd>
<dt>**embedded**</dt><dd>"Heavily resource constrained" would be a better, but less catchy name for this. A low-powered MCU or high-performance code with high performance requirements. The term *embedded* is a bit misleading here; e.g. a Raspberry Pi is much closer to an *application* environment.</dd>
<dt>**library**</dt><dd>Libraries have to serve both contexts well, as they often cannot anticipate which context they are going to be used in. Occasionally one can make assumptions based on what the library does, which may be out of scope for either context.</dd>
</dl>

## Rules

### R1: Use panics when the caller violates the API, return `Err` otherwise.

Every function has invariants that need to hold true for its inputs; some of which cannot be expressed through the type system. Providing correct inputs to a function is the callers job, to avoid conflating the API with error handling for programming errors, panics (through `assert!`, `panic!` or similar) should be used instead, as precedented by the standard library.

Example: [`Vec::split_off()`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.split_off).

Variant **R1A**: It is possible to move some or all of the error handling back into the return value (`Result` or `Option`) if doing so reduces the burden on the caller significantly or enables common use cases.

Example: [`Vec::pop()`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.pop).

### R2: Use `assert!` instead of `if .. { panic! }`

Instead of using

```rust
if user_id == 0 {
    panic!("user_id must be greater than zero");
}
```

use `assert!`:

```rust
assert!(user_id == 0, "user_id must be greater than zero");
```

### R3: Use newtypes to avoid having to panic on API violations.

When having to restrict input domains for a value in a non-trivial way, newtypes should be used to restrict it. This allows turning a panic into an error by moving the conversion outside the function call:

```rust
#[derive(Debug)]
struct UserId(usize);

// Error/Fail impl omitted.
struct UserIdInvalid {}

impl TryFrom<usize> for UserId {
    type Error;

    // `try_from` is a conversion function, a failing input value is within the
    // realm of possible outcomes and not an API violation. This avoids a panic
    // in `get_user_from_db`.
    fn try_from(value: usize) -> Result<Self, Self::Error> {
        if value == 0 {
            return Err(UserIdInvalid);
        }

        UserId(value)
    }
}

fn get_user_from_db(user_id: UserId) -> Result<User, DbError> {
    // ...
}
```

### R4: Prefer `.expect()` over `.unwrap()`.

`.expect()` is the same as `.unwrap()`, except it allows adding a custom message, which in turn is vastly preferable over a comment:

```rust
value.expect("operation did not return meaningful value");
```

Variant **R4A**: In some embedded cases such as microcontrollers with mere kilobytes or bytes of memory, it is sometimes preferable to `unwrap()` to avoid having to include the error message string into the program binary.

### R5: Use lower-case, simple messages for `.expect()`.

When used, `.expect()` messages should be short, start lowercase and report what went wrong. Contrary to the name, it should *not* state what was expected.

```rust
fs::File::open("foo.txt").expect("could not open data file");
```

### R6: Avoid unwraps, unless mandated by R1.

A panic will cause the currently running thread to end, which might abort the program or put it in a state where a thread is dead. This makes it impossible for callers to recover.

Never use `.expect()` (see R4) in libraries unless unavoidable due to borrow-checker limitations or expensive design flaws. When doing so, add a comment (in addition to the message from R5) explaining why `.expect()` was used in the first place and why it will (ideally) never be triggered.

In general, code will be inspected and every potential panic should have a good, well described reason for being there.

### R7: Bubble errors upwards when not handled.

When errors cannot be handled in a function, they should be passed up the call stack. Ultimately, any error not handled should arrive (possibly wrapped multiple times) in the `main()` function.

### R8: Use error returns in `main` instead of unwrapping.

In applications, defining main with a (see [RFC 1937](https://github.com/rust-lang/rust/issues/43301)) `Result` return type like

```rust
fn main() -> Result<(), Box<Error>>
```

is slightly better than using `.unwrap()` in `main` (see R7):

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err`
value: (DEBUG IMPL OF ERROR TYPE)', libcore/result.rs:945:5
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

becomes

```
Error: (DEBUG IMPL OF ERROR TYPE)
```

when using the `?` instead of `.unwrap()`. The extra text cover very little usable information and should be omitted.

Variant **R8A**: When error handling is not set up properly (during prototyping, when dealing with legacy code), using explicit `.expect()`s instead can be necessary due to R4.

### R9: Avoid boxed errors.

Especially in embedded contexts, using `Box<Error>` or `failure::Error` must be avoided to not introduce unnecessary heap allocations. In general, no library should return boxed errors.

Exceptions to this guideline are `main` (see `R7`, there is little gain in introducing an error type for main only) and complex functions outside the hot-path in applications where introducing a massive error type would yield little gains.

### R10: Always use `failure` and `failure_derive` when possible.

The [failure](https://docs.rs/failure/) crate replaces "manual" error management as well as [error-chain](https://docs.rs/error-chain/). It greatly reduces the amount of boilerplate code and is more flexible than either of the old solutions. Due to its `no_std` compatibility, it is even feasible to use in an embedded environment.

The `fmt::Debug` trait implementation of `Fail` types will, if derived, typically print a backtrace and causes.

Currently, community consensus seems be heading towards the failure crate as well.

## Open Questions

Some patterns present themselves repeatedly and can be addressed in a different way, although most of the time there is no obvious best pattern yet.

### Q1: How to distinguish between the same error on multiple call-sites?

The same error variant can occur multiple times in a single function:

```rust
use std::{fs, io, path};

#[derive(Debug, Fail)]
enum LoadError {
    ParseError,
    Io(io::Error),
}

impl From<io::Error> for LoadError {
    fn from(e: io::Error) -> LoadError {
        LoadError::Io(e))
    }
}

fn load_from_file<P: AsRef<path::Path>>) -> Result<Data, LoadError> {
    // Early-returns `LoadError::Io` if opening the file fails.
    let mut input_file = fs::File::open(p)?;

    let mut buf = String::new();

    // Early-returns `LoadError::Io` if reading the file fails.
    input_file.read_to_string(buf)?;

    // ...
}
```

Here, `LoadError::Io` is returned in two different places, with no distinction possible by the consumer of said error.

#### P1A: Bottom-up error variant definition

Solve the problem by defining one error variant per call-site:

```rust
#[derive(Debug, Fail)]
enum LoadError {
    ParseError,
    OpenDataFile(io::Error),
    ReadDataFile(io::Error),
}

// No more `From<io::Error>` impl.

fn load_from_file<P: AsRef<path::Path>>) -> Result<Data, LoadError> {
    let mut input_file = fs::File::open(p).map_err(LoadError::OpenDataFile)?;

    let mut buf = String::new();
    input_file.read_to_string(buf).map_err(LoadError::ReadDataFile)?;

    // ...
}
```

Now selecting the appropriate error and returning it is straightforward.

There are drawbacks: No more `From<io::Error>` conversions due to ambiguity, resulting in more verbose code. A risk of coupling the interface of the function with the implementation also exist, considering the following refactoring:

```rust
fn load_from_file<P: AsRef<path::Path>>) -> Result<Data, LoadError> {
    let buf = fs::read_to_string(p).map_err(/* which error type to map to? */)?;

    // ...
}
```

For this reason, after introducing one error type per call site, these variants should be coalesced into fewer that are sufficiently fine-grained but abstract away implementation details. Example:

```rust
#[derive(Debug, Fail)]
enum LoadError {
    ParseError,
    DataFileRead(io::Error),
}
```

Note that `LoadError::DataFileRead` has the same signature as `LoadError::Io` earlier, but has no direct `From<io::Error>` implementation, leaving the class extensible in case another-but-different error occurs. This should also be done to simplify the error type, deciding which kinds of errors the client is allowed to distinctly care about is a design decision.

#### P1B: Using backtraces

Another way of handling these problems that incurs some runtime cost is adding backtraces to errors. While `failure` offers facilities for handling backtraces, they are cumbersome to use: Either a specially constructed error type is required which will, in the worst case, require a lot of boilerplate; or the boxed error type `failure::Error` must be used.

Adding backtraces is usually very valuable when P1A-style errors have been coalesced to a point where it is impossible to directly find the call site easily. This is sometimes unavoidable, but more valuable for the library implementer than it is for its consumer.

### Q2: How to include contextual information into errors?

Another common problem is wanting to include more contextual information in an error:

```rust
#[derive(Debug, Fail)]
enum LoadError {
    Parse(ParseError),
    DataFileRead(io::Error),
}

impl From<ParseError> for LoadError {
    fn from(e: ParseError) -> LoadError {
        LoadError::Parse(e)
    }
}

fn load_from_file<P: AsRef<path::Path>>) -> Result<Data, LoadError> {
    let buf = fs::read_to_string(p).map_err(LoadError::DataFileRead)?;

    for line in buf.lines() {
        let data = parse_line(line)?;
        // ...
    }
}
```

If an error occurs, additional context is required for the user to pinpoint the cause of the error, such as the line in the file.

#### P2A: **Inline contextual information**

Additional information can be added directly into the error:

```rust
#[derive(Debug, Fail)]
enum LoadError {
    Parse {
        line: usize,
        err: ParseError,
    },
    DataFileRead(io::Error),
}

// ... [inside `load_from_file()`]:

    for (idx, line) in buf.lines().enumerate() {
        let data = parse_line(line).map_err(|err| LoadError::Parse{ line: idx+1, err })?;
        // ...
    }
```

Convenience functions *may* be added (thanks to `impl Trait`), be aware that they are almost equivalent to adding a `From<ParseError>` implementation:

```rust
impl LoadError {
    fn parse_line(line: usize) -> impl Fn(ParseError) -> LoadError {
        move |err| LoadError::Parse { line, err }
    }
}

// ...

        let data = parse_line(line).map_err(parse_line(idx+1))?;
```

Note that `LoadError` does not carry any information about the filename of the file the read failed from; this is because it only ever touches a single file that is passed to it via the API. An error type further up the chain should annotate the filename as contextual information in the same vein, if it is handling multiple files.

#### P2B: Using `failure::Context`

The `failure::Context` type is an alternative for handling contextual info, it comes with significant drawbacks. For one, it only works smooth with `failure::Error` types or when a lot of the machinery to include a context is implemented manually. Otherwise simply writing `.context()` does not work well.

Other than that it is fairly cumbersome to extract information beyond descriptive strings manually from this context, which makes it only suitable for adding additional information for end-users or developers debugging.

However, it adds backtraces by default, which is very convenient in both cases.
