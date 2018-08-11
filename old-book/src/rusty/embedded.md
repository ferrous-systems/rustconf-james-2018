# Why Rust for Embedded?

> Why not?

> * LLVM already supports microcontrollers, thanks to its' C and C++ usage
> * Rust as a language has defined and is stabilizing all of the language items needed in an embedded context, mostly around the wierdness that is running code outside of an operating system, like what to do if your code panics, or how to start your code after you first boot up
> * We don't have an operating system, or anything we can pretend is one, so it would be hard to use the standard library, but we can throw it all out if we need, and add back just the parts we want
> * Oh, and as of the 2018 edition of Rust, you can pretty much just type `cargo build --target thumbv7em-none-eabihf`, and get a microcontroller binary out, while still keeping all of the things that are nice about Rust like helpful compiler warnings, cargo for managing packages, the borrow checker, the type systems, etc.

>