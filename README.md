# A Guide to Slice Data in Rust

This is an opinionated guide that explains commonly-used crates and techniques for storing "slice data" in Rust. When I say slice data, I really mean contiguous data. Following the form of [a guide to global data in Rust](https://github.com/paulkernfeld/global-data-in-rust).

# TODO

- [`std::collections::VecDeque`](https://doc.rust-lang.org/std/collections/vec_deque/struct.VecDeque.html)
- https://crates.io/crates/set_slice ?
- https://crates.io/crates/fixed-slice-vec ?

# Tradeoffs

These are things to think about when choosing a technique for storing slice data.

## Lifetime

You may want your data to have the `'static` lifetime so that it'll be available anywhere within your program. This provides a lot of flexibility in how and where you can use the data but less flexibility in how you can modify it.

## Mutability

There are a few broad levels of mutability for contiguous data in Rust:

- Completely immutable
- Fixed size, mutable contents
- Mutable size and contents

## Extend vs. refuse

When you run out of memory in your contiguous slice, should the data structure figure out a way to get more memory or should it refuse to give you more memory? If you want to intentionally use a capped amount of memory, it might be better to refuse to extend the memory. This could be useful in embedded programming where you might not have access to an allocator to give you more memory.

## Splitting

There are several ways that you can split contiguous data in Rust. Reviewing [References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) from TRPL might be helpful to understanding this better.

### Splitting with shared references

You can always create as many overlapping shared slices (`&[T]`) as you want. The restriction is that you can't mutate them.

```rust
const MY_DATA: [i8; 3] = [2; 3];

fn main() {
    // Compare two overlapping slices
    assert_eq!(&MY_DATA[..2], &MY_DATA[1..]);
}
```

### Splitting with mutable references

You can often divide a data structure into multiple non-overlapping mutable slices.

```rust
fn main() {
    let my_data = &mut [0, 2, 3, 0];
    let (left, right) = my_data.split_at_mut(2);
    left[0] = 1;
    right[1] = 4;
    assert_eq!(my_data, &[1, 2, 3, 4]);
}
```

### Splitting with owned data

It is less common to be able to divide the ownership of a contiguous block of data. See the `bytes` crate for a way to do this.

# Solutions

The solutions are roughly in order of increasing power, according to the [Principle of Least Power](https://www.lihaoyi.com/post/StrategicScalaStylePrincipleofLeastPower.html#dependency-injection). So, consider using the first solution that will solve your problem.

## Fixed-size array: `[T; N]`

When you create a [fixed-size array](https://doc.rust-lang.org/std/primitive.array.html) in Rust, you must specify the size (`N`) at compile time.

Given a mutable array, you can change the content, but there is no way to change the size.

```rust
const MY_DATA: [i8; 3] = [1, 2, 3];

fn main() {
    let mut my_data = [2; 3];
    my_data[0] = 1;
    my_data[2] = 3;
    assert_eq!(MY_DATA, my_data);
}
```

## Slice: `&[T]`

In Rust, a slice is "a dynamically-sized view into a contiguous sequence" (primitive type [slice](https://doc.rust-lang.org/std/primitive.slice.html), [TRPL on slice data](https://doc.rust-lang.org/book/ch04-03-slices.html)).

Given a mutable slice, you can change the content, but there is no way to change the size. When we take a mutable slice to an array, we're actually modifying the data in the array itself. That's `my_array` needs to be declared as `mut` below.

```rust
fn main() {
    let mut my_array: [i8; 3] = [1, 2, 0];
    let my_slice: &mut[i8] = &mut my_array[1..];
    my_slice[1] = 3;
    assert_eq!(my_array, [1, 2, 3]);
}
```

Note that the following code would not compile if `my_slice` were an array, i.e. with type `&[i8; 3]`; in that case, the compiler would say: ``can't compare `[i8; 3]` with `[i8; 4]``. However, the compiler is happy to let us compare slices of two different sizes.

```rust
fn main() {
    let my_slice: &[i8] = &[1, 2, 3];
    let my_array: [i8; 4] = [1, 2, 3, 4];
    assert_ne!(my_slice, &my_array);
}
```


## Boxed array: `Box<[T; N]>`

With a boxed array, you can create arrays at run time without knowing the size in advance. In most cases, you could probably just use `Vec` instead. However, boxed arrays do have a few use cases:

- Writing a custom buffer (e.g. [`std::io::BufReader`](https://doc.rust-lang.org/std/io/struct.BufReader.html))
- If you want to save a few bytes relative to a `Vec`

The code below won't compile; you can't collect an iterator into a fixed-size array because the number of elements in an iterator isn't generally known at compile time.

```
fn main() {
    let my_data: [i8; 3] = [1, 2, 3].iter().cloned().collect();
}
```

...but you can collect into a boxed array:

```rust
const MY_DATA: [i8; 3] = [1, 2, 3];


fn main() {
    let my_data: Box<[i8]> = MY_DATA.iter().cloned().collect();

    // We need to take a slice b/c we can't compare boxed and fixed-size arrays directly
    assert_eq!(&my_data[..], MY_DATA);
}
```

## `Vec`

[`std::vec::Vec`](https://doc.rust-lang.org/std/vec/struct.Vec.html) is the `std` way to do variable-length contiguous data. Both the content and size can be changed. When you try to add data and there's not enough room, the data structure will allocate more memory for the data.

```rust
fn main() {
    let mut vec = Vec::new();
    vec.push(1);
    vec.push(2);
    vec.push(3);
    
    assert_eq!(vec.len(), 3);
    assert_eq!(vec, [1, 2, 3]);
}
``` 

We can split a `Vec` into two separate `Vec`s using the `split_off` method. However, doing this (surprisingly?) copies the data. [According to user hanna-kruppe on the Rustlang forum,](https://users.rust-lang.org/t/split-owned-vec-t-without-reallocation/6346/2):

> But as far as I know, there’s currently no way to tell the allocator “Hey I got this piece of memory from you, now please pretend you gave it to me as two separate, contiguous allocations”

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

# `tinyvec`

`tinyvec` provides 100%-safe alternatives to both `arrayvec` and `smallvec`. It works pretty much the same way except that the types must implement `Default`.

```rust
use tinyvec::ArrayVec;

fn main() {
    let mut array = ArrayVec::<[_; 2]>::new();
    array.push(1);
    array.push(2);
}
```

```rust,should_panic
use tinyvec::ArrayVec;

fn main() {
    let mut array = ArrayVec::<[_; 2]>::new();
    array.push(1);
    array.push(2);
    array.push(3);
}
```

## The `bytes` crate 

[bytes](https://docs.rs/bytes) provides `Bytes`, "an efficient container for storing and operating on contiguous slices of memory." One of its signature features is that, unlike `Vec`, it allows you to split ownership of data without copying.

```rust
use bytes::{BytesMut, BufMut};

fn main() {
    let mut left = BytesMut::with_capacity(1024);
    left.put(&[1u8, 2, 3, 4] as &[u8]);
    
    let mut right = left.split_off(2);
    
    std::thread::spawn(move || {
        right[1] = 6;
        assert_eq!(&right[..], [3, 6]);
    });
    
    left[0] = 5;
    assert_eq!(&left[..], [5, 2]);
}
```
