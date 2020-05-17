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

# Solutions

## Slice literal

This is a reference to data that is known at compile time and baked into the binary. Therefore the size and content of the slice must be known at compile time.

You can change the content, but there is no way to change the size.

```rust
fn main() {
    let my_data = &mut [1, 2, 3];
    my_data[1] = 4;
    assert_eq!(my_data[1], 4);
}
```

## Array literal

When you create an array in Rust, you need to specify the size at compile time.

You can change the content, but there is no way to change the size.

```rust
fn main() {
    {
        let mut my_data = [1, 2, 3];
        my_data[1] = 4;
        assert_eq!(my_data[1], 4);
    }
}
```

## `std::vec::Vec`

[`Vec`](https://doc.rust-lang.org/std/vec/struct.Vec.html) is the `std` way to do variable-length contiguous data. Both the content and size can be changed. When you try to add data and there's not enough room, the data structure will allocate more memory for the data.

```rust
fn main() {
    let mut vec = Vec::new();
    vec.push(1);
    vec.push(2);
    
    assert_eq!(vec.len(), 2);
    assert_eq!(vec[0], 1);
}
``` 

## `smallvec`

The [`smallvec`](https://github.com/servo/rust-smallvec) provides an interface that is very close to Vec, but it will store small numbers of items on the stack instead of allocating on the heap. This is good if you suspect your data structure will normally be quite small but it may need to grow occasionally.

```rust
use smallvec::{SmallVec, smallvec};
    
fn main() {
    // This SmallVec can hold up to 4 items on the stack:
    let mut v: SmallVec<[i32; 4]> = smallvec![1, 2, 3, 4];
    
    // It will automatically move its contents to the heap if
    // contains more than four items:
    v.push(5);
    
    // SmallVec points to a slice, so you can use normal slice
    // indexing and other methods to access its contents:
    v[0] = v[1] + v[2];
    v.sort();
}
```

## `arrayvec`

This crate will let you store a Vec inside of an array, but it won't let you exceed the size of the array.

[arrayvec](https://docs.rs/arrayvec)

```rust
use arrayvec::ArrayVec;

fn main() {
    let mut array = ArrayVec::<[_; 2]>::new();
    assert_eq!(array.try_push(1), Ok(()));
    assert_eq!(array.try_push(2), Ok(()));
    assert!(array.try_push(3).is_err());
    assert_eq!(&array[..], &[1, 2]);
}
```