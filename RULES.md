# An error-style guide

This document tries to provide a set of rules and guidelines to help with error-handling best-practices[^1].

[^1]: Or more honestly, perform a lot of *bikeshedding*.

## Preqrequisites

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
* ["Annoucing failure"](https://boats.gitlab.io/blog/post/2017-11-16-announcing-failure/),
* ["Failure 1.0"](https://boats.gitlab.io/blog/post/2018-02-22-failure-1.0/).
* [Bay Area Rust Meetup November 2017](https://air.mozilla.org/bay-area-rust-meetup-november-2017/)

## Definitions

Error handling is dependant on context, different environments have different needs for error handling. The following guidelines assume three different contexts, named and defined as follows:

<dl>
<dt>**application**</dt><dd>Application is the most common context for executable Rust programs, being ran on an operating system with plenty of memory and computing power.</dd>
<dt>**embedded**</dt><dd>"Heavily resource constrained" would be a better, but less catchy name for this. A low-powered MCU or high-performance code with high performance requirements. The term *embedded* is a bit misleading here; e.g. a Raspberry Pi is much closer to an *application* environment.</dd>
<dt>**library**</dt><dd>Libraries have to serve both contexts well, as they often cannot anticipate which context they are going to be used in. Occasionally one can make assumptions based on what the library does, which may be out of scope for either context.</dd>
</dl>

## Rules

### R1: Use panics when the caller violates the API, return `Err` otherwise.

Every function has invariants that need to hold true about its inputs; some of which cannot be expressed through the type system. Providing correct inputs to a function is the callers job, to avoid conflating the API with error handling for programming errors, panics (through `assert!`, `panic!` or similar) should be used instead, as precedented by the standard library.

Example: [`Vec::split_off()`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.split_off).

Variant **R1A**: It is possible to move some or all of the error handling back into the return value (`Result` or `Option`) if doing reduces the burden on the caller significantly or enables common use cases.

Example: [`Vec::pop()`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.pop).

### R2: Use `assert!` instead of `if .. { panic! }`

Instead of using

```rust
if user_id <= 0 {
    panic!("user_id must be greater than zero");
}
```

use `assert!`:

```rust
assert!(user_id <= 0, "user_id must be greater than zero");
```

### R3: Use newtype to avoid having to handle err

When having to restrict input domains for a value in a non-trivial way, newtypes should be used to restrict it. This allows turning a panic into an error by moving the error into the appropriate domain:

```rust
#[derive(Debug)]
struct UserId(usize);

// Error/Fail impl omitted.
struct UserIdInvalid {}

impl Into<usize> for UserId {
    fn into(self) -> usize {
        self.0
    }
}

impl TryFrom<usize> for UserId {
    type Error;

    fn try_from(value: usize) -> Result<Self, Self::Error> {
        if value <= 0 {
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

`.expect()` is the same as `.unwrap()`, except it allows adding a custom message. This is vastly preferrable over a comment:

```rust
value.expect("operation did not return meaningful value");
```

Variant **R4A**: In some embedded cases such as microcontrollers with mere kilobytes or bytes of memory, it is sometimes preferrable to `unwrap()` to avoid having to include the error message string into the program binary.

### R5: Use lower-case, simple messages for `.expect()`.

When used, `.expect()` messages should be short, start lowercase and report what went wrong. Contrary to the name, it should *not* state what was expected.

```rust
fs::File::open("foo.txt").expect("could not open data file");
```

### R6: Avoid unwrapping.

As often as possible, avoid the use of `.expect()` or `.unwrap()`, but use destructuring instead:

```rust
if let Some(value) = operation() {
    //
} else {
    // `panic!()` can be used here, but there are other options.
}
```

### R7: Avoid unwraps, unless mandated by R1.

A panic will cause the currently running thread to end, which might abort the program or put it in a state where a thread is dead. This makes it impossible for callers to recover.

Never use `.expect()` (see R4) in libraries unless unavoidable due to borrow-checker limitations or expensive design flaws. When doing so, add a comment (in addition to the message from R5) explaining why `.expect()` was used in the first place.

In general, code will be inspected and every potential panic should have a good, described reason for being there.

### R8: Bubble errors upwards when not handled.

When errors cannot be handled in a function, they should be passed up the call stack. Ultimately, any error not handled should arrive (possibly wrapped multiple times) in the `main()` function.

### R9: Use error returns in `main`.

In applications, defining main with a (see [RFC 1937](https://github.com/rust-lang/rust/issues/43301)) `Result` return type like

```rust
fn main() -> Result<(), Box<Error>>
```

is slightly better than using `.unwrap()` in `main` (see R8):

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

Variant **R9A**: When error handling is not set up properly (during prototyping, when dealing with legacy code), using explicit `.expect()`s instead can be necessary due to R4.

### R10: Avoid boxed errors.

Especially in embedded contexts, using `Box<Error>` or `failure::Error` must be avoided to not introduce unnecessary heap allocations. In general, no library should return boxed errors.

Exceptions to this guideline are `main` (see `R8`, there is little gain in introducing an error type for main only) and complex functions outside the hot-path in applications where introducing a massive error type bring few gains.
