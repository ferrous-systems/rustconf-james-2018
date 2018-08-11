# Hook

> So, today's talk is sort of about embedded systems, but really the cool thing we're going to talk about is how to get the compiler to understand more about what you are trying to do with your program, and enforce your rules at compile time, rather than writing code that checks itself at run time.
>
> In Rust, this isn't a new concept, "Zero Cost Abstractions" are the first feature listed on the home page:

![Rust Language Home Page](./assets/rust-home-page.png)

> This means that we are able to make some kind of abstraction, for zero cost. The cost we are talking about here is overhead at runtime, either in memory used or cpu cycles when you are running your program.

> For embedded systems, you are always trying to reduce your costs, typically by using hardware with lower features, while still operating as it is supposed to. The lower the number of CPU cycles needed per second, or the lower the maximum amount of RAM or Flash storage is needed, the lower the parts necessary to build the system will cost.

> CPU cycles also have another cost for embedded systems. The more the CPU does, the more battery it uses, making battery life shorter.