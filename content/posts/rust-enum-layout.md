+++
title = 'The optimization of enum layout in Rust'
date = 2022-01-23T13:05:08+08:00
+++

Before you start reading, you can guess what the output of the following Rust code will be on a common 64-bit machine:

```rust
struct A (i64, i8);
struct B (i64, i8, bool);
fn main() {
    dbg!(std::mem::size_of::<A>());
    dbg!(std::mem::size_of::<Option<A>>());
    dbg!(std::mem::size_of::<B>());
    dbg!(std::mem::size_of::<Option<B>>());
}
```

You can see the results in [Rust Playground](https://play.rust-lang.org/?version=nightly&mode=release&edition=2021&gist=bab644e35609b5475978821378d3560f).

---

Essentially, rust enum is the implementatio of [tagged union][tagged union], which corresponds to the sum type in algebraic data types, but we will talk too much about it here. In a concrete implementation, a byte is used to store the type tag in most cases. If the enum is complex and needs to represent more than 256 possibilities, more bytes will be used.

Without any optimization, the following two structures are equivalent. Enum is not zero-overhead, which violates the zen of Rust.

```rust
enum Attr {
    Color(u8, u8, u8),
    Shape(u16, u16),
}
```

```c
struct Color { uint8_t r, g, b };
struct Shape { uint16_t w, h };
struct Attr {
    uint8_t tag;
    union {
        Color color;
        Shape shape;
    }
}
```

Fortunately, Rust has never defined a stabilized ABI, and enums with data cannot be marked as `repr(C)`, which gives rustc the opportunity to optimize the enum layout. We will present some optimizations over `rustc 1.60.0-nightly` in the post.

Before exploring it, we need a helper function, which helps us to inspect the memory layout of variables.

```rust
fn print_memory<T>(v: &T) {
    println!("{:?}", unsafe {
        core::slice::from_raw_parts(v as *const _ as *const u8, std::mem::size_of_val(v))
    })
}
```

Take the above `Attr` as an example:

```rust
print_memory(&Attr::Color(1, 2, 3));
// [0, 1, 2, 3, 0, 0]
//  ^  ^  ^  ^  ^^^^
//  |  |  |  |    |
//  |  |  |  |    --- padding
//  |  |  |  -------- b
//  |  |  |---------- g
//  |  |------------- r
//  ----------------- tag

print_memory(&Attr::Shape(257, 258));
// [1, 0, 1, 1, 2, 1]
//  ^  ^  ^^^^  ^^^^
//  |  |    |     |
//  |  |    |     --- h, 256 + 2
//  |  |    --------- w, 256 + 1
//  |  |------------- padding
//  ----------------- tag
```

[tagged union]: https://en.wikipedia.org/wiki/Tagged_union

## Option\<P\<T\>\>

Assume P is a well-known smart pointer type, such as `&`/`&mut`/`Box`. Use `Option<P<T>>` to represent nullable pointer is a good pattern in Rust, which gives us [null safety][null safety wiki].

[null safety wiki]: https://en.wikipedia.org/wiki/Void_safety

`Option<T>` is implemented as an enum in Rust:

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

Without any optimization, `Option<i64>` takes 16 bytes (8 bytes of data, 1 byte of tag and 7 bytes of padding), resulting in a huge waste of memory space. In C++, we can always use `std::nullptr` to represent `None`. The case is so common that Rust has not only optimized it, but also written it to spec:

> If T is an FFI-safe non-nullable pointer type, `Option<T>` is guaranteed to have the same layout and ABI as `T` and is therefore also FFI-safe. As of this writing, this covers `&`, `&mut`, and function pointers, all of which can never be null.

```rust
let p = Box::new(0u64);
print_memory(&p);
// [208, 185, 38, 40, 162, 85, 0, 0]
print_memory(&Some(p));
// [208, 185, 38, 40, 162, 85, 0, 0]
```

The optimization is not a compiler hack for the concrete builtin `Option`, but also for all enums similar to this:

```rust
enum MyOption<T> {
    MySome(T),
    MyNone,
}

let p = Box::new(0u64);
print_memory(&MyOption::MySome(p));
// [208, 185, 38, 40, 162, 85, 0, 0]
// The address of `p`.
print_memory(&MyOption::<Box<u64>>::MyNone);
// [0, 0, 0, 0, 0, 0, 0, 0]
// Use nullptr to represent `MyNone`.
```

The real reason why `Option<P<T>>` can be optimized is that there is an always illegal value under the memory representation of `P`, and the enum needs to represents only an extra value. The optimization will fail if the constraint is not satisfied.

```rust
// 16 bytes is required for `MyOption2<u64>`.
enum MyOption2<T> {
    MySome(T),
    MyNone,
    MyNone2,
}

let p = Box::new(0u64);
print_memory(&MyOption2::MySome(p));
// [0, 0, 0, 0, 0, 0, 0, 0, 208, 185, 38, 40, 162, 85, 0, 0]
print_memory(&MyOption2::<Box<u64>>::MyNone2);
// [2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

## bool

`bool` is represented by one byte in rust, but only has two valid values, `true` and `false`, corresponding to `1u8` and `0u8`, so there are 254 invalid holes available for storing the type tag.

e.g., for `Option<bool>`:

```rust
print_memory(&Some(false));
// [0]
print_memory(&Some(true));
// [1]
print_memory(&(None as Option<bool>));
// [2]
```

Even more, `Option<Option<bool>>`:

```rust
print_memory(&Some(Some(false)));
// [0]
print_memory(&Some(Some(true)));
// [1]
print_memory(&(Some(None) as Option<Option<bool>>));
// [2]
print_memory(&(None as Option<Option<bool>>));
// [3]
```

## Enum

We can take a look at `std::cmp::Ordering`:

```rust
#[repr(i8)]
pub enum Ordering {
    Less,
    Equal,
    Greater,
}
```

Similar to `bool`, there is 253 holes in `Ordering`'s memory layout:

```rust
print_memory(&Some(std::cmp::Ordering::Less));
// [255]
print_memory(&Some(std::cmp::Ordering::Greater));
// [1]
print_memory(&Some(std::cmp::Ordering::Equal));
// [0]
print_memory(&(None as Option<std::cmp::Ordering>));
// [2]
```

In fact, `Ordering` is not special in rustc's perspective. All enums that contains invalid holes can be optimized, e.g.:

```rust
enum ShapeKind { Square, Circle }
print_memory(&Some(ShapeKind::Square));
// [0]

print_memory(&MyOption::MySome(MyOption::MySome(1u8)));
// [0, 1]
print_memory(&(MyOption::MyNone as MyOption<MyOption<u8>>));
// [2, 0]
// There is no holes in `u8`, but 254 holes in `MyOption<u8>`.
```

## Struct, Tuple

Struct and Tuple are the implementations of product type. In the concrete implementation, fields are always placed one after another with **appropriate padding**. If a field with invalid holes exists, then it can be used to store the type tag.

```rust
struct A1 (i8, bool);
print_memory(&Some(A1(1, false)));
// [1, 0]
print_memory(&Some(A1(1, true)));
// [1, 1]
print_memory(&(None as Option<A1>));
// [0, 2]
```

Let's go back to the example at the beginning of the article:

```rust
struct A (i64, i8);
struct B (i64, i8, bool);

dbg!(std::mem::size_of::<A>());
// 16
dbg!(std::mem::size_of::<Option<A>>());
// 24
dbg!(std::mem::size_of::<B>());
// 16
dbg!(std::mem::size_of::<Option<B>>());
// 16

print_memory(&Some(A(1, 1)));
// [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0]
//  ^  ^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^ ^  ^^^^^^^^^^^^^^^^^^^
//  |           |                      |            |           |
//  |           |                      |            |           ------------ padding
//  |           |                      |            ------------------------ .1
//  |           |                      ------------------------------------- .0
//  |           ------------------------------------------------------------ padding
//  ------------------------------------------------------------------------ tag
```

Both `A` and `B` take 16 bytes due to padding, and `B` has a `bool` field, so `Option<B>` can store the type tag in `B.2`, then `Option<B>` also takes 16 bytes, but `Option<A>` take 24 bytes without any optimizations. **Wait, why can't padding be used as invalid holes?**

Unfortunately, A's memory layout is determined while A is defined, and storing data in padding of another struct is UB (undefined behaviour). Some methods such as `memcpy` never promised that they will move its padding correctly when they operates on `A`. So it creates a counter-intuitive result where we store an extra meaningless field and instead take up less memory.

Of course, it's still possible to optimize the layout in padding with inline records:

```rust
use std::mem::size_of;
struct A(i64, i8);
// There is no UB because there is not a sub struct.
enum OptionA { Some(i64, i8), None }
dbg!(size_of::<A>()); // 16
dbg!(size_of::<Option<A>>()); // 24
dbg!(size_of::<OptionA>()); // 16
```

## Possibility of optimizing custom types

Currently, all layouts about enum layout are hacked by the compiler. We cannot control the layout of our custom type. The std library provides [`NonZero*`][NonZeroU8] to represent the integer that cannot be zero, which is implemented by internal attribute `rustc_layout_scalar_valid_range_start`, but it cannot be used by third-party library such as [`Option<ordered_float::NotNan<f64>>`][NotNan]. The attribute is also not good enough as a new feature. Consider we want to implement a new type `OddI64`, then it's impossible to represent all invalid holes by a range. How about providing a const iterator to produce all invalid holes? I'm not sure that's a good idea.

[NonZeroU8]: https://doc.rust-lang.org/std/num/struct.NonZeroI8.html
[NotNan]: https://docs.rs/ordered-float/2.10.0/ordered_float/struct.NotNan.html#

---

All examples in the post can be found at [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=2cecbb4523443229d32a943bea2e48aa).
