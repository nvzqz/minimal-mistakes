---
title: Rust Static Assertions
category: Programming
tags:
    - programming
    - rust
toc: true
author_profile: false
---

Four months ago I published the [static_assertions] crate. To understand how it
works, we need to learn how to creatively use compile errors.

Here I'll be explaining how one would go about defining a macro to
[assert constant conditions](#asserting-constant-conditions).

## Setup

This post assumes some basic knowledge of [Rust] and it'll be using version
1.21.0. This code should work with some previous and all later versions.

Rust 1.21.0 can be installed and enabled via:

```bash
rustup install 1.21.0 && rustup default 1.21.0
```

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

We've now defined a macro with two sets of input options. The second set will
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

Here, our array element type is the unit type: a zero sized type. The array's
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

[static_assertions]: https://crates.io/crates/static_assertions
[Rust]: rust-lang.org
