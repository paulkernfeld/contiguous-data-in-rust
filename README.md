# A Guide to Slice Data in Rust

This is an opinionated guide that explains commonly-used crates for storing slice data in Rust. (contiguous data?). Following the form of [a guide to global data in Rust](https://github.com/paulkernfeld/global-data-in-rust).

A cool thing about Rust is that it's possible to write many different data structures that can store [slice data](https://doc.rust-lang.org/book/ch04-03-slices.html). (primitive type [slice](https://doc.rust-lang.org/std/primitive.slice.html))
(`&[_]`).

Should include:

- `&[_]` (slice)
- arrays
- `Vec`
- `arrayvec`
- `smallvec`
- `bytes`
- `tinyvec`?