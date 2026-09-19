# {{crate_name}}

{{description}}

## Usage

```rust
use {{crate_name}}::{DefaultGreeter, Greeter};

let g = DefaultGreeter;
let msg = g.greet("world").unwrap();
assert_eq!(msg, "hello, world");
```

## Logging

This crate emits [`tracing`](https://docs.rs/tracing) events on error and lifecycle paths (never per message). Install any subscriber to see them, e.g. `tracing_subscriber::fmt::init()`. Filter per crate with `RUST_LOG=<crate_name>=debug`. Disable at compile time with `tracing`'s `release_max_level_off` feature in your binary.

## License

Dual-licensed under either [MIT](LICENSE-MIT) or [Apache 2.0](LICENSE-APACHE), at your option.
