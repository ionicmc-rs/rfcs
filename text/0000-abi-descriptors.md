- Feature Name: `abi_descriptors`
- Start Date: `1/22/2026`
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Allows users to create custom **A**pplication **B**inary **I**nterfaces (ABIs) inside of Rust, allowing them to set the input registers, output registers, alignment requirements, etc.

## Motivation
[motivation]: #motivation

Currently, the selection of ABIs available in rust is very suffecient. There is also a currently on-going implementation of `extern "custom"` functions (and possibly other items - see [#140829](https://github.com/rust-lang/rust/issues/140829))

however, those are not general ABIs for all usages, this RFC refers more to an ABI that:
- can be used for regular, non-naked functions
- can be used for alignment, stacks, etc.

Additionally, some low-level projects need custom ABIs such as the `x86-interrupt` abi before it was added; however, there was no possible way to create a custom ABI. Resultingly, many of these projects link external libraries to be able to use ABIs foreign to Rust. This causes a lot of `unsafe` code in these projects - which is not ideal.

As an example, I (personally) was developing a project where a function was passed an argument via the `r12` register. However, due to the lack of a suitable ABI, I was forced to use external code to move the registers to their correct places; causing extra, unneccesary code.

In general, custom ABIs could lower the usage of external code fixing ABI requirements. it also gives the additional information that `abi_custom` does not (align as an example)

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

there are 2 possible ways this freature may be implemented.

### Trait
Example for C ABI
```rust
use core::abi::{Abi, Register}; // in core

struct CAbi;

unsafe impl Abi for CAbi { // unsafe impl
    // might use slices instead.
    const REGISTER_COUNT: u8 = 6;
    const FLOAT_REGISTER_COUNT: u8 = 2;

    const INPUT_REGISTERS: [Register; Self::REGISTER_COUNT] = [
         Register::Rdi,
         // ...
    ];
    const INPUT_FLOATS: [Register; Self::FLOAT_REGISTER_COUNT] = [
        Register::Xmm0,
        // ...
    ]

    const STACK_ALIGN: usize = 16;

    // more ABI fields here.
}
```
Usage:
```rust
extern CAbi fn c_func(arg: u64, float: f32) {
    // ...
}
```
Pros:
- Simple, easy to understand.
- extern is not a string - does not shadow builtin ABIs.
Cons:
- Lots of boilerplate.
- Arch-specific.
- Requires parser changes.

### `#[abi(name)]`
This would be simmilar to `#[global_allocator]`
```
use core::abi::{CustomAbi, Register};

#[unsafe(abi("bad-c"))] // unsafe attr
const C_ABI: CustomAbi = CustomAbi::new()
    .set_inputs([Register::Rdi, ...]);
```
Usage:
```
extern "bad-c" fn c_func(arg: u64, float: f32) {}
```
Pros:
- Less boilerplate than traits.
- string names allow for more variety.
- better docs, due to the `CustomAbi` type.
Cons:
- String names may be confusing with Rust's builtins.
- Amiguouty if all field of `CustomAbi` must be set.
- Requires a new lang item.
### Disscussion
This feature allows for better code maintainability. Instead of reading online documentation for an ABI, or worse, scouting undocumented code for the used ABI, the ABI lives directly in Rust, allowing for easier changing.

Additionally, from the core/std-side of things, we can define an ABI for general use - not stabilizing the `"Rust"`, though.

This feature fits nicely with [#140829](https://github.com/rust-lang/rust/issues/140829)), with a new macro.
```
use core::abi::call_with_abi;

#[unsafe(naked)]
extern "custom" unsafe fn ret() {
    core::arch::naked_asm!(
        "ret"
    )
}

fn foo() {}

// SomeAbi defined here...
unsafe {
    call_with_abi(ret, "C"); // Calls with C abi
    call_with_abi(ret, SomeAbi); // Calls with SomeAbi
    // call_with_abi(foo, "C"); // ERROR: Only function with the `custom` abi can be called.
}
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

For the first implementation method, the defenition would likely look like this:
```
// #[rustc_deny_creation] // we may also deny creation, I'm not sure if there is already an attr for this.
pub unsafe trait CustomAbi {
    // constants, no runtime-variables
}
```

this feature would require changing:
- the parser to allow non-strings after extern (first impl-method)
- the compiler to allow custom ABIs. (specifically anything that stores its ABI).

Also, creation of ABI that is invalid is considered undefined behaviour, hence the `unsafe` keyword. The exact defenition for an "Invalid ABI" is not clear, as certain ABIs may have different forms. Basic things to check include all alignment consts being powers of 2. (Note: We still check what can be checked)

Additionally, calling a "custom" function with an incorrect ABI is UB, and this applies to both the call and creation of the ABI.

### Internal Implementation
During code generation, the compiler load the evaluated constants to ensure type-safety, and use the correct registers for the ABI. I'm not exactly sure if the order of the compiler's operations allows for this.

## Drawbacks
[drawbacks]: #drawbacks

Custom ABIs are just unsafe. At the end of the day, users are creating these ABIs, and there is no way to check the validity of such ABI.

Additionally, these may be a little too arch-specific, and this increases the chance of users accedeintly causing UB.

There are likely other reasons.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design allows for the most customisability.

Other designs include the `"custom"` abi, and simply the `"C"` abi. But these only define a set ruleset, if a user neeeds specific functionallity, they must wait until we add that abi.

Not doing this wont have much of an impact - it may even be the better choice to avoid refactoring the compiler too much.

## Prior art
[prior-art]: #prior-art

The `"custom"` abi is one solution to this same problem, it:

- allows for ABIs that are not supported
- allows users to specify their items are not in a supported ABI to prevent confusion.

However, it:

- does not support customisability.
- is builtin, and not accessible to users as a type.

This RFC isn't aiming to *replace* custom, but rather enhance it by adding what was missing.

Also, Other programming languages dont support custom ABIs, which is one reason why Rust should.
## Unresolved questions
[unresolved-questions]: #unresolved-questions
> What parts of the design do you expect to resolve through the RFC process before this gets merged?
- How would the compiler implement this?
- Would we implement this via trait or global constant?
> What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- How do we ensure as much safety as we can?
- How can we use this feature in the standard library?

> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
N/A

## Future possibilities
[future-possibilities]: #future-possibilities

We could possibly stabilize the previously mentioned ABI in the standard library for general use, not neccessarily for Rust, howevver, but having a general ABI wouldn't hurt
