# Rust pattern: Display adapter

Sometimes, you want to display a value in a specific way. Convert a `String` to a different format, display a `i32` in a particular way. What is the most ergonomic way to do that in Rust?

## The obvious solution

Let's start with what anyone would do in most languages: Write a function that returns a string:

The first example converts a camel-case identifier to a lower-snake-case one:

```rust
fn to_snake_case(ident: &str) -> String {
    let mut result = String::new();

    let mut first = true;
    for c in ident.chars() {
        if c.is_uppercase() {
            if !first {
                result.push('_');
            }

            for c in c.to_lowercase() {
                result.push(c);
            }
        } else {
            result.push(c);
        }

        first = false;
    }

    result
}

assert_eq!(to_snake_case("SomeLongIdentifier"), "some_long_identifier");
```

The second example converts an `i32` to a string that says "*N* above zero" for positive *N*, "*N* below zero" for negative *N* and "zero" if the value is 0.

```rust
fn to_wordy_number(n: i32) -> String {
    match n {
        n if n < 0 => format!("{} below zero", -n),
        n if n > 0 => format!("{} above zero", n),
        0 => "zero".into(),
    }
}

assert_eq!(to_wordy_number(-5), "5 below zero");
assert_eq!(to_wordy_number(5), "5 above zero");
assert_eq!(to_wordy_number(0), "zero");
```

What's wrong with this pattern? The purpose of such conversions is usually to use the result in a larger string, or for writing it to a file or the standard output. You're probably doing unnecessary allocations for intermediate strings, only to make them parts of something bigger and drop them. And this has no chance of working in no_std without liballoc.

## A more rusty solution: Display adapters

I am sure I am not the first one to come up with this pattern, as it's rather obvious. Still, I could not find anyone discussing it explicitly.

By implementing the `Display` trait on a tuple struct with a single `pub` field, we can create an adapter that transforms how a type is printed. Let us transform our examples:

```rust
struct SnakeCase<'a>(&'a str);

impl<'a> Display for SnakeCase<'a> {
    fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
        let mut first = true;
        for c in self.0.chars() {
            if c.is_uppercase() {
                if !first {
                    fmt.write_char('_')?;
                }

                write!(fmt, "{}", c.to_lowercase())?;
            } else {
                fmt.write_char(c)?;
            }

            first = false;
        }

        Ok(())
    }
}

assert_eq!(
    SnakeCase("SomeLongIdentifier").to_string(),
    "some_long_identifier"
);
let identifier = format!("{}_mut", SnakeCase("SomeLongIdentifier"));
```

```rust
struct WordyNumber(i32);

impl Display for WordyNumber {
    fn fmt(&self, fmt: &mut fmt::Formatter) -> fmt::Result {
        match self.0 {
            n if n < 0 => write!(fmt, "{} below zero", -n),
            n if n > 0 => write!(fmt, "{} above zero", n),
            _ => fmt.write_str("zero"),
        }
    }
}

assert_eq!(WordyNumber(-5).to_string(), "5 below zero");
assert_eq!(WordyNumber(5).to_string(), "5 above zero");
assert_eq!(WordyNumber(0).to_string(), "zero");
println!("The temperature is {}.", WordyNumber(5));
```

This has several advantages over creating a string:

* You only allocate what you really need by delaying formatting (or you might not allocate at all).
* There is no runtime overhead from creating or using the adapter.
* It works with everything that supports Rust's formatting machinery (including `ToString::to_string()`, `println!`, `format!` and friends).
* It works with no_std.

I want to encourage everyone who currently publishes custom string conversion functions to use this pattern instead.

If you think any part of this article is confusing, misleading or even incorrect, please [file an issue](https://github.com/brain0/articles/issues) or [open a pull request](https://github.com/brain0/articles/pulls) on GitHub.

Thanks for reading!
