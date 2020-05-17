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

# Tradeoffs

These are things to think about when choosing a data structure for storing slice data.

## Lifetime

You may want your data to have the `'static` lifetime so that it'll be available anywhere within your program. This provides a lot of flexibility in how and where you can use the data but not a lot in how you can modify it.

## Extend vs. refuse

When you run out of memory in your contiguous slice, should the data structure figure out a way to get more memory or should it refuse to give you more memory? If you want to intentionally use a capped amount of memory, it might be better to refuse to extend the memory. This could also be useful in embedded programming where you might not have access to an allocator to give you more memory.