---
title: Rust Static Assertions
category: Programming
tags:
    - programming
    - rust
toc: true
author_profile: true
header:
    image: /assets/images/posts/rust-static-assertions-header.png
    teaser: assets/images/posts/rust-static-assertions-teaser.png
---

Four months ago I published the [static_assertions] crate. To understand how it works, we need to learn how to creatively use compile errors.

Here I'll be explaining how one would go about defining macros to
[assert constant conditions](#asserting-constant-conditions) and
[assert equal sized types](#asserting-equal-sized-types) **at compile-time**.

## Setup

This post assumes some basic knowledge of [Rust] and it'll be using version
1.21.0. This code should work with some previous and all later versions.

Rust 1.21.0 can be installed and enabled via:

```bash
rustup install 1.21.0 && rustup default 1.21.0
```

If you'd rather not install Rust, you can follow along just as easily with
[a Rust playground][play].

## Asserting Constant Conditions

The final result can be used [here](https://play.rust-lang.org/?gist=a981e147c536884c974014311fdd0d39).

### Reasoning

The main reason I wrote [static_assertions] was to have something similar to
`static_assert` in C++ but available in Rust.

One common error in programming is integer overflow/underflow. What does Rust
tell us when this happens within a constant?

```rust
const ASSERT: usize = 0 - 1;
```

```
warning: constant evaluation error: attempt to subtract with overflow.
This will become a HARD ERROR in the future
 --> src/main.rs:1:18
  |
1 | const ASSERT: usize = 0 - 1;
  |                       ^^^^^
  |
  = note: #[warn(const_err)] on by default
```

Well, we want it to be a hard error _now_. This can be achieved with:

```rust
#[deny(const_err)]
const ASSERT: usize = 0 - 1;
```

```
error: constant evaluation error: attempt to subtract with overflow.
This will become a HARD ERROR in the future
 --> src/main.rs:2:18
  |
2 | const ASSERT: usize = 0 - 1;
  |                       ^^^^^
  |
note: lint level defined here
 --> src/main.rs:1:8
  |
1 | #[deny(const_err)]
  |        ^^^^^^^^^

error: aborting due to previous error
```

**Note:** You may be getting warnings about unused code. We can use
`#[allow(dead_code)]` to disable these warnings.
{: .notice--info}

Better!

Now how can we turn this into an assertion? Recall that `bool` can be converted
into an integer primitive via an `as` cast.

In Rust, `false as usize == 0` and `true as usize == 1`. Thus we can easily
update our code to accept a boolean:

```rust
const CONDITION: bool = false;

#[deny(const_err)]
#[allow(dead_code)]
const ASSERT: usize = 0 - !CONDITION as usize;
```

```
error: constant evaluation error: attempt to subtract with overflow.
This will become a HARD ERROR in the future
 --> src/main.rs:4:23
  |
5 | const ASSERT: usize = 0 - !CONDITION as usize;
  |                       ^^^^^^^^^^^^^^^^^^^^^^^
  |
note: lint level defined here
 --> src/main.rs:3:8
  |
3 | #[deny(const_err)]
  |        ^^^^^^^^^

error: aborting due to previous error
```

### The Macro

We can define our initial macro as such:

```rust
macro_rules! const_assert {
    ($condition:expr) => {
        #[deny(const_err)]
        #[allow(dead_code)]
        const ASSERT: usize = 0 - !$condition as usize;
    }
}
```

Here `$condition` is our input and we let Rust know we're accepting it as any
expression as denoted by `:expr`.

Let's put our new macro to use with a simple boolean:

```rust
const_assert!(false);
```

```
error: constant evaluation error: attempt to subtract with overflow.
This will become a HARD ERROR in the future
 --> src/main.rs:5:31
  |
5 |         const ASSERT: usize = 0 - !$condition as usize;
  |                               ^^^^^^^^^^^^^^^^^^^^^^^^
...
9 | const_assert!(false);
  | --------------------- in this macro invocation
  |
note: lint level defined here
 --> src/main.rs:3:16
  |
3 |         #[deny(const_err)]
  |                ^^^^^^^^^
...
9 | const_assert!(false);
  | --------------------- in this macro invocation

error: aborting due to previous error
```

Awesome, it works! Now let's try asserting a more useful condition.

```rust
const_assert!(2 + 2 == 4);
```

This compiles perfectly. If we change `==` to `!=` or make the expression
`false` in any way, it'll give us a compile error just like before.

Let's try asserting multiple conditions.

```rust
const_assert!(2 + 2 == 4);

const_assert!(5 * 5 == 25);
```

```
error[E0428]: the name `ASSERT` is defined multiple times
 --> src/main.rs:5:9
  |
5 |         const ASSERT: usize = 0 - !$condition as usize;
  |         -----------------------------------------------
  |         |
  |         `ASSERT` redefined here
  |         previous definition of the value `ASSERT` here
...
9 | const_assert!(2 + 2 == 4);
  | -------------------------- in this macro invocation
  |
  = note: `ASSERT` must be defined only once in the value namespace of this module

error: aborting due to previous error
```

Uh-oh.

Well Rust allows us to use `_` to create anonymous bindings with `let`. What if
we used that here? Within the macro, rename `ASSERT` to `_` as such:

```rust
const _: usize = 0 - !$condition as usize;
```

```
error: expected identifier, found `_`
 --> src/main.rs:5:15
  |
5 |         const _: usize = 0 - !$condition as usize;
  |               ^
...
9 | const_assert!(2 + 2 == 4);
  | -------------------------- in this macro invocation
  |
  = note: `_` is a wildcard pattern, not an identifier
```

Hmmm. So it seems that Rust doesn't support anonymous constants. We could try
using a `let` binding instead of `const`.

```rust
let _: usize = 0 - !$condition as usize;
```

```
error: expected item after attributes
 --> src/main.rs:4:27
  |
4 |         #[allow(dead_code)]
  |                           ^
...
9 | const_assert!(2 + 2 == 4);
  | -------------------------- in this macro invocation

error: macro expansion ignores token `let` and any following
 --> src/main.rs:5:9
  |
5 |         let _: usize = 0 - !$condition as usize;
  |         ^^^
  |
note: caused by the macro expansion here; the usage of `const_assert!`
is likely invalid in item context
 --> src/main.rs:9:1
  |
9 | const_assert!(2 + 2 == 4);
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Aaahh!! What does this mean??

If you've been doing what I've been doing and have been using `const_assert!`
within a global scope you'll get this error. What if we put it into `main` (or
some other function)?

```rust
fn main() {
    const_assert!(2 + 2 == 4);
    const_assert!(5 * 5 == 25);
}
```

Wow, it compiles! Sweet!

And because we're using `_`, we can have multiple assertions at once.

### Improving Usability

What if we don't want to define a function every time we need to use this macro?
Fortunately, there's a solution. I call it "labeling".

Macros can accept expressions, but they can also accept other items such as
identifiers. We can take advantage of this and have our macro accept an optional
identifier to "label" a global assertion.

```rust
macro_rules! const_assert {
    ($condition:expr) => {
        #[deny(const_err)]
        #[allow(dead_code)]
        let _: usize = 0 - !$condition as usize;
    }; // <-- Notice the semicolon!
    ($label:ident; $condition:expr) => {
        fn $label() {
            const_assert!($condition);
        }
    };
}
```

We have now defined a macro with two sets of input options. The second set will
create a function with the given label as its name. It'll then recursively call
`const_assert!` with the provided expression.

Let's put it to use in both global and function contexts:

```rust
const_assert!(simple_math; 2 * 2 == 2 + 2);

fn main() {
    const_assert!(2 + 2 == 4);
    const_assert!(5 * 5 == 25);
}
```

It compiles! However, we get this warning:

```
warning: function is never used: `simple_math`
  --> src/main.rs:8:9
   |
8  | /         fn $label() {
9  | |             const_assert!($condition);
10 | |         }
   | |_________^
...
14 |   const_assert!(simple_math; 2 * 2 == 2 + 2);
   |   ------------------------------------------- in this macro invocation
   |
   = note: #[warn(dead_code)] on by default
```

We can disable the warning by annotating our `fn` with `#[allow(dead_code)]`.
Now it'll successfully build without any warnings or errors.

What if we want to use a different convention for our labels? For example:

```rust
const_assert!(SimpleMath; 2 * 2 == 2 + 2);
```

We can annotate our label function with `#[allow(non_snake_case)]`. We can
actually combine our annotations into one line:

```rust
($label:ident; $condition:expr) => {
    #[allow(non_snake_case, dead_code)]
    fn $label() {
        const_assert!($condition);
    }
};
```

### Ensuring Constants Only

But wait, because we're using a `let` binding, our macro can also accept
non-constants. Fortunately there's a trick to ensure `const_assert!` _only_
accepts constants. We can do so by making our value be an array size.

In Rust, arrays are declared as `[T; N]` where `N` is some constant `usize`.

We can update our `let` binding accordingly:

```rust
let _ = [(); 0 - !$condition as usize];
```

Here our array element type is the unit type: a zero sized type. The array's
size is exactly the same as our previous value.

We can now remove the `#[deny(const_err)]` annotation since array sizes are
never allowed to overflow. We may also remove `#[allow(dead_code)]` while we're
at it since our variable is `_` which can't be used anyway.

Now our macro has finally grown up and looks like this:

```rust
macro_rules! const_assert {
    ($condition:expr) => {
        let _ = [(); 0 - !$condition as usize];
    };
    ($label:ident; $condition:expr) => {
        #[allow(non_snake_case, dead_code)]
        fn $label() {
            const_assert!($condition);
        }
    };
}
```

### Better Usability (Advanced)

What if we want to assert multiple conditions? Sure we can use `&&` but maybe we
want to make our conditions be independent yet required.

My approach is to make the conditions comma-separated.

```rust
macro_rules! const_assert {
    ($($condition:expr),+ $(,)*) => {
        let _ = [(); 0 - !($($condition)&&+) as usize];
    };
    ($label:ident; $($rest:tt)+) => {
        #[allow(non_snake_case, dead_code)]
        fn $label() {
            const_assert!($($rest)+);
        }
    };
}
```

Here the first branch takes one or more comma-separated expressions as well as
zero or more trailing commas. The second branch takes all tokens passed after
`$label` and forwards them to the first branch.

When repeating `$condition` we can place `&&` between each repetition since it's
viewed as a single token.

This new definition can be used as such:

```rust
const_assert! { simple_math;
    2 * 2 == 2 + 2,
    5 * 2 == 10,
}
```

## Asserting Equal Sized Types

The final result can be used [here](https://play.rust-lang.org/?gist=f3de36b61b277ab789314710c1f0b150).

### Reasoning

Rust is a language that's suitable for working on systems at a low level. This
may involve pointer casts between different types.

If we change our types' internal structures, their sizes may also change. This
will lead to strange behavior if we're accidentally casting between
differently-sized types.

What if we want to ensure `usize` has the same size as `u64` within somewhere
deep within our code? Maybe we can use something in conjunction with
`#[cfg(target_pointer_width = "64")]`.

One way of performing type casts is via [`std::mem::transmute`][transmute]. It
takes the bits of a value of a given type and returns the same bits but with a
different type.

When the output type is not the same size as the input, we get a compile error:

```rust
use std::mem::transmute;

fn main() {
    let a: u32 = 42;
    let b: u16 = unsafe { transmute(a) };
}
```

```
error[E0512]: transmute called with types of different sizes
 --> src/main.rs:5:27
  |
5 |     let b: u16 = unsafe { transmute(a) };
  |                           ^^^^^^^^^
  |
  = note: source type: u32 (32 bits)
  = note: target type: u16 (16 bits)

error: aborting due to previous error
```

Can we use this to our advantage in a general and ergonomic way? Of course!

This is Rust; we shouldn't have to be `unsafe` to get what we want. As it turns
out, getting the function pointer will emit the same error.

```rust
let f = transmute::<u32, u16>;
```

You may have noticed the odd `::<>` after `transmute`. This is  unofficially
referred to as [turbofish] syntax. It tells the compiler what generic parameters
to use for `transmute`, which takes an input and output type.

### The Macro

With these tools we can write our basic macro:

```rust
use std::mem::transmute;

macro_rules! assert_eq_size {
    ($x:ty, $y:ty) => {
        let _ = transmute::<$x, $y>;
    }
}
```

Here `$x` and `$y` are annotated with `:ty`, which means that we can accept
**any** type. Yes, Rust's macros are this powerful.

Let's try this out:

```rust
assert_eq_size!(u16, u32);
```

```
error[E0512]: transmute called with types of different sizes
  --> src/main.rs:5:17
   |
5  |         let _ = transmute::<$x, $y>;
   |                 ^^^^^^^^^^^^^^^^^^^
...
10 |     assert_eq_size!(u16, u32);
   |     -------------------------- in this macro invocation
   |
   = note: source type: u16 (16 bits)
   = note: target type: u32 (32 bits)

error: aborting due to previous error
```

Excellent, it works!

### Improving Usability

Similar to with how we initially defined `const_assert!`, we can only use this
macro within the context of a function. We can apply the same labeling technique
here!

```rust
macro_rules! assert_eq_size {
    ($x:ty, $y:ty) => {
        let _ = transmute::<$x, $y>;
    }; // <-- Notice the semicolon!
    ($label:ident; $x:ty, $y:ty) => {
        #[allow(non_snake_case)] // Allows any naming convention
        #[allow(dead_code)]      // Can be left unused
        fn $label() {
            assert_eq_size!($x, $y);
        }
    };
}
```

This new definition can take a "label" and create a function that then
recursively calls `assert_eq_size!` with the given types.

We can now use our macro in both global and function contexts:

```rust
assert_eq_size!(string_size; &str, (usize, usize));

fn main() {
    assert_eq_size!([u8; 4], u32);
}
```

### Better Usability (Advanced)

We're not done yet; we can make our macro even more awesome!

What if we want to check more than two types against one another? Yeah, that's
very much possible.

```rust
macro_rules! assert_eq_size {
    ($x:ty, $($xs:ty),+ $(,)*) => {
        $(let _ = transmute::<$x, $xs>;)+
    };
    ($label:ident; $($rest:tt)+) => {
        #[allow(dead_code, non_snake_case)]
        fn $label() {
            assert_eq_size!($($rest)+);
        }
    };
}
```

Here the first branch takes some first type `$x` followed by one or more
comma-separated types denoted by `$xs`. It can also have zero or more trailing
commas. The second branch takes all tokens passed after `$label` and forwards
them to the first branch.

Notice that we're repeating our statement over each type matched by `$xs` but we
maintain the same initial `$x` type.

Our final macro can now be used as such:

```rust
assert_eq_size! { some_sizes;
    [u8; 4],
    (u16, u16),
    u32,
}
```

## Final Comments

Evidently we can utilize Rust's powerful compile-time checks in a very elegant
way.

This post was inspired from speaking to other Rustaceans at the Boston Rust
meetup last night. Apparently there's members of the community who are just as
interested in these cool little hacks as I am. 😁

Currently, `assert_eq_size!` can't be implemented via `const_assert!` because
`std::mem::size_of` is not a constant function (yet).

Check out [static_assertions] for more functionality just like this! If you have
any suggestions feel free to open an issue or pull request.

[static_assertions]: https://crates.io/crates/static_assertions
[Rust]: https://www.rust-lang.org
[play]: https://play.rust-lang.org/
[turbofish]: https://github.com/steveklabnik/rust/commit/4f22b4d1dbaa14da92be77434d9c94035f24ca5d#commitcomment-14014176

[transmute]:       https://doc.rust-lang.org/std/mem/fn.transmute.html
[`uninitialized`]: https://doc.rust-lang.org/std/mem/fn.uninitialized.html
[`forget`]:        https://doc.rust-lang.org/std/mem/fn.forget.html

