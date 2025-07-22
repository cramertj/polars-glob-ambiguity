# Reproducer for polars_plan 0.48.1 import error

Rust version: `rustc 1.90.0-nightly (9748d87dc 2025-07-21)`
Error message:

```
error[E0603]: function `date_range` is private
  --> src/main.rs:2:27
   |
2  |     use polars_plan::dsl::date_range;
   |                           ^^^^^^^^^^ private function
   |
note: the function `date_range` is defined here
  --> /usr/local/google/home/cramertj/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/polars-plan-0.48.1/src/dsl/mod.rs:87:5
   |
87 | use crate::prelude::*;
   |     ^^^^^^^^^^^^^^
help: import `date_range` directly
   |
2  -     use polars_plan::dsl::date_range;
2  +     use polars_time::date_range::date_range;
   |

error[E0603]: function `time_range` is private
  --> src/main.rs:3:27
   |
3  |     use polars_plan::dsl::time_range;
   |                           ^^^^^^^^^^ private function
   |
note: the function `time_range` is defined here
  --> /usr/local/google/home/cramertj/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/polars-plan-0.48.1/src/dsl/mod.rs:87:5
   |
87 | use crate::prelude::*;
   |     ^^^^^^^^^^^^^^
help: import `time_range` directly
   |
3  -     use polars_plan::dsl::time_range;
3  +     use polars_time::date_range::time_range;
   |

For more information about this error, try `rustc --explain E0603`.
```

[`date_range`](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/dsl/functions/range.rs#L25-L31)
and
[`time_range`](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/dsl/functions/range.rs#L85-L91)
are both defined as `pub` and exported under `polars_plan::dsl` via the following chain of `pub use` statements:
1. [`pub mod dsl` in `lib.rs`](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/lib.rs#L10)
2. [`pub use functions::*;` in `dsl/mod.rs`](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/dsl/mod.rs#L56)
3. [`pub use range::date_range` and `pub use range::time_range` in `dsl/functions/mod.rs`](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/dsl/functions/mod.rs#L36-L39)

However, `rustc` provides an error message pointing at `polars_time::date_range::{date_range, time_range}`, despite the fact that
the current crate has no direct dependency on `polars_time`.

Furthermore, `rustc` thinks that the `date_range` and `time_range` types were defined in `src/dsl/mod.rs:87`:

[```use crate::prelude::*;```](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/dsl/mod.rs#L87)

The prelude does include a [`pub(crate)` reexport of the types from `polars_time`](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/prelude.rs#L10).
However, it *also* includes a (circular) [`pub` reexport of the dsl::* types](https://github.com/pola-rs/polars/blob/5e1b4b793a7c65aa6e2d76414c1b239086a9bab0/crates/polars-plan/src/prelude.rs#L13)

Based on the (incomplete) [Rust reference section on `use`](https://doc.rust-lang.org/reference/items/use-declarations.html#ambiguities),
I would expect an ambiguity error rather than a privacy error.
