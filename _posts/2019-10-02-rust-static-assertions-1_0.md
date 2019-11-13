---
title: 'Static Assertions 1.0'
category: Programming
tags:
    - programming
    - rust
toc: true
author_profile: true
header:
    image: /assets/images/posts/rust-static-assertions-1_0-header.png
    teaser: assets/images/posts/rust-static-assertions-1_0-teaser.png
---

The [`static_assertions`] Rust library is now 1.0!

This means quite a lot of things. Check out [`CHANGELOG.md`] and the
[1.0 docs](https://docs.rs/static_assertions/1.0.0)
for detailed info.

## Thank you

First of all, I would like to thank you (yes, you). Thank you for being part of
the community that's used this library and given me feedback and suggestions for
it.

At RustConf 2019, it wasn't uncommon for me to introduce myself as the creator
of `static_assertions` and subsequently be told how productive and confident
it's made people. Hearing that over the course of the conference brought me
immense joy.

This community is the reason that I will be able to attend
[RustFest Barcelona](https://barcelona.rustfest.eu) this year through
[my GoFundMe](https://www.gofundme.com/f/nikolai-rustfest-barcelona).

❤️

## What is Static Assertions?

`static_assertions` is a library designed to enable users to perform various
checks at [compile-time]. It allows for finding errors quickly and early when it
comes to ensuring certain features or aspects of a codebase. The macros it
provides are especially important when exposing a public API that requires types
to be the same size or implement certain traits.

## No more labels

With Rust 1.37 introducing [`const _`], the [labeling requirement][#1] has been
completely removed! 🎉

Prior to this, every global macro needed a label to avoid conflicts:

```rust
const_assert!(val1; true);
const_assert!(val2; 1 < 2);
```

Now it's as simple as:

```rust
const_assert!(true);
const_assert!(42 < 9000);
```

Many thanks to [@joshlf] for
[the RFC](https://github.com/rust-lang/rfcs/blob/master/text/2526-const-wildcard.md)
and to [@Centril] for pushing this feature forward.

## Asserting equal type alignment

As of 1.0, it is now possible to assert that two types are equal in alignment
via [`assert_eq_align!`].

```rust
struct Foo(*const u8);

assert_eq_align!(Foo, *mut u8, usize);
```

The macro is defined as:

```rust
macro_rules! assert_eq_align {
    ($x:ty, $($xs:ty),+ $(,)?) => {
        const _: fn() = || {
            use $crate::_core::mem::align_of;
            $(let _: [(); align_of::<$x>()] = [(); align_of::<$xs>()];)+
        };
    };
}
```

This ensures correct alignment by making sure that the declared array type
(`[(); align_of::<$x>()]`) and the array instances (`[(); align_of::<$xs>()]`)
match in size.

## New syntax for some macros

The [`assert_fields!`] and `assert_impl_*` family of macros are now called with
a colon instead of a comma to separate out the type.

```rust
struct Foo {
    bar: // ...
    baz: // ...
}

assert_fields!(Foo: bar, baz);

trait Bar {}
trait Baz {}

impl Bar for Foo {}
impl Baz for Foo {}

assert_impl_all!(Foo: Bar, Baz);
```

## Asserting `!Trait`, a.k.a. negative trait bounds

As of 0.3.4, it is possible to assert that a given type _does not_ implement
_all_ or _any_ in a given set of traits via [`assert_not_impl_all!`] and
[`assert_not_impl_any!`] respectively. This is thanks to [@HeroicKatora] via
[#17].

The following example fails to compile since [`u32`] can be converted into
[`u64`]:

```rust
assert_not_impl_any!(u32: Into<u64>);
```

The following compiles because `Cell<u32>` is neither [`Sync`] nor [`Send`]:

```rust
use std::cell::Cell;

assert_not_impl_all!(Cell<u32>: Sync, Send);
```

## Going Forward

There are a number of features that are not yet implemented that I think would
be possible.

### Advanced constant evaluation

Today, by using the `-Zunleach-the-miri-inside-of-you` flag, you can do `if`,
loops, and other control flow in a `const` context. My friend [@oli-obk]
mentioned the possibilities enabled with this during
[his RustConf 2019 talk @ 22:54](https://youtu.be/wkXNm_qo8aY?t=1374).

Once `if` in `const` is stabilized (see [issue][const_if]),
[`const_assert!`] will go from having its awkward "subtraction with overflow"
error message to simply [`panic!`]-ing with a proper error message.

### `assert_impl_any!`

There should exist a macro that asserts a given type implements _any_ of a given
set of traits. This is being tracked at [#19].

```rust
struct Foo;

trait Bar {}
trait Baz {}

assert_impl_any!(Foo: Bar, Baz);
```

Until `Foo` implements `Bar` or `Baz`, this program would fail to compile.

### Mutually exclusive trait implementations

By leveraging `assert_impl_any!` in combination with [`assert_not_impl_all!`],
you can ensure that a type implements one of two traits but not both. This is
being tracked at [#20].

```rust
struct Foo;

trait Bar {}
trait Baz {}

impl Bar for Foo {}

// Uncommenting the following line results in a compile failure
// impl Baz for Foo {}

assert_impl_any!(Foo: Bar, Baz);
assert_not_impl_all!(Foo: Bar, Baz);
```

[`static_assertions`]: https://github.com/nvzqz/static-assertions-rs

[compile-time]: https://en.wikipedia.org/wiki/Compile_time

[#1]:  https://github.com/nvzqz/static-assertions-rs/issues/1
[#17]: https://github.com/nvzqz/static-assertions-rs/pull/17
[#19]: https://github.com/nvzqz/static-assertions-rs/pull/19
[#20]: https://github.com/nvzqz/static-assertions-rs/pull/20

[`CHANGELOG.md`]: https://github.com/nvzqz/static-assertions-rs/blob/master/CHANGELOG.md

[@joshlf]:       https://github.com/joshlf
[@Centril]:      https://github.com/Centril
[@HeroicKatora]: https://github.com/HeroicKatora
[@oli-obk]:      https://github.com/oli-obk

[const_if]:  https://github.com/rust-lang/rust/issues/49146
[`const _`]: https://github.com/rust-lang/rfcs/blob/master/text/2526-const-wildcard.md

[`panic!`]: https://doc.rust-lang.org/std/macro.panic.html
[`Send`]:   https://doc.rust-lang.org/std/marker/trait.Send.html
[`Sync`]:   https://doc.rust-lang.org/std/marker/trait.Sync.html
[`u32`]:    https://doc.rust-lang.org/std/primitive.u32.html
[`u64`]:    https://doc.rust-lang.org/std/primitive.u64.html

[`const_assert!`]:        https://docs.rs/static_assertions/1.0.0/static_assertions/macro.const_assert.html
[`assert_fields!`]:       https://docs.rs/static_assertions/1.0.0/static_assertions/macro.assert_fields.html
[`assert_eq_align!`]:     https://docs.rs/static_assertions/1.0.0/static_assertions/macro.assert_eq_align.html
[`assert_not_impl_all!`]: https://docs.rs/static_assertions/1.0.0/static_assertions/macro.assert_not_impl_all.html
[`assert_not_impl_any!`]: https://docs.rs/static_assertions/1.0.0/static_assertions/macro.assert_not_impl_any.html
