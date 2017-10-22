---
title: Rust vs Swift
category: Programming
tags:
    - programming
    - rust
    - swift
toc: true
author_profile: false
---

A thorough comparison between Rust and Swift: two very similar languages aimed
towards system and applications programming.

## Primitives

Both languages ensure that primitive sizes and signedness are explicitly known.

| Swift     | Rust    | Size   | Signed | Notes |
| :-------: | :-----: | :----: | :----: | :---- |
| `Int`     | `isize` | 4 or 8 | Y      | Pointer-sized
| `Int8`    | `i8`    | 1      | Y      |
| `Int16`   | `i16`   | 2      | Y      |
| `Int32`   | `i32`   | 4      | Y      |
| `Int64`   | `i64`   | 8      | Y      |
| n/a       | `i128`  | 16     | Y      | [Pending][i128]
| `UInt`    | `usize` | 4 or 8 | N      | Pointer-sized
| `Unt8`    | `u8`    | 1      | N      |
| `UInt16`  | `u16`   | 2      | N      |
| `UInt32`  | `u32`   | 4      | N      |
| `UInt64`  | `u64`   | 8      | N      |
| n/a       | `u128`  | 16     | N      | [Pending][i128]
| `Float`   | `f32`   | 4      | Y      |
| `Double`  | `f64`   | 8      | Y      |
| `Float80` | n/a     | 16     | Y      | [x86 extended precision]

[i128]: https://github.com/rust-lang/rust/issues/35118
[x86 extended precision]: https://en.wikipedia.org/wiki/Extended_precision#x86_extended_precision_format

## References

## Strings

## Sum Types

## Pattern Matching

## Constants and Statics

## Error Handling
