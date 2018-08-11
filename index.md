---
title: Getting Something for Nothing
theme: black
---

## RustConf 2018

James Munns

@bitshiftmask

james.munns@ferrous-systems.com

---

## Embedded Systems

::: notes

So, today's talk is sort of about embedded systems, but really the cool thing we're going to talk about is how to get the compiler to understand more about what you are trying to do with your program, and enforce your rules at compile time, rather than writing code that checks itself at run time.

In Rust, this isn't a new concept, "Zero Cost Abstractions" are the first feature listed on the home page:

:::

---

![](./assets/rust-home-page.png)

::: notes

This means that we are able to make some kind of abstraction, for zero cost. The cost we are talking about here is overhead at runtime, either in memory used or cpu cycles when you are running your program.

For embedded systems, you are always trying to reduce your costs, typically by using hardware with lower features, while still operating as it is supposed to. The lower the number of CPU cycles needed per second, or the lower the maximum amount of RAM or Flash storage is needed, the lower the parts necessary to build the system will cost.

CPU cycles also have another cost for embedded systems. The more the CPU does, the more battery it uses, making battery life shorter.

:::

---

## Crash Course in Embedded Systems

::: notes

So, embedded systems is a pretty broad term, but it generally covers any system containing hardware and software that is built for one primary task. This is different to devices like phones, laptops, desktops, and servers, which are General Purpose Computers. They are built to run any number of different applications. Unfortunately, the term "embedded systems" covers everything from a TV remote to a rocket engine.

:::

---

## Microcontrollers

::: notes

We'll narrow our scope down the the lower chunk of that range, focusing down on Microcontrollers systems, where even the smallest amount of overhead can be pretty painful. Microcontrollers typicallly have:

* No operating system, or only a minimal one that provides threads and timers
* Single Core, 32 bit CPUs, with 16-600 MHz clock speeds
* 20K-512M of SRAM memory
* 20K-512M of Flash storage

> This covers a huge chunk of devices you might not even think of as individual computers. This could be things like the Thermostat on your wall, a fitness tracker, a Yubikey, or a washing machine.

:::

---

## Peripherals

::: notes

Most Microcontrollers have more than just a CPU and RAM, they also come with a bunch of stuff called Peripherals which are useful for interacting with other hardware, like sensors, bluetooth radios, screens, or touch pads. These peripherals are great because you can offload a lot of the processing to them, so you don't have to handle everything in software. Kind of like offloading graphics processing to a video card, so your CPU can spend it's time doing something else important, or doing nothing so it can save power.

:::

---

## Hardware API

::: notes

However, unlike graphics cards, which typically have a Software API like Vulkan, Metal, OpenGL, or DirectX, peripherals are exposed to our CPU with a hardware interface, which is mapped to a chunk of the memory. Because of this, we call these Memmory Mapped Peripherals.

:::

---

## Memory Mapped Peripherals

::: notes

On a microcontroller, when you write some data to a certain address, like `0x2000_0000`, or even `0x0000_0000`, you're really writing to that address. There isn't anything like an MMU which is abstracting one chunk of memory to some other virtual address.

Because 32 bit microcontrollers have this real and linear memory space, from `0x0000_0000`, and `0xFFFF_FFFF`, and they only generally use a few hundred kilobytes of it for actual memory, there is lots of room left over. Instead of ignoring that space, Microcontroller designers instead put the interface for parts of the hardware, like peripherals, in certain memory locations. This ends up looking something like this:

:::

---

![](./assets/nrf52-memory-map.png)

::: notes

So for example, if you want to send 32 bits of data over a serial port, you write to the address of the serial port output buffer, and the Serial Port Peripheral takes over and sends out the data for you automatically. If you want to turn an LED on? You write one bit in a special memory address, and the LED turns on.

Configuration of these peripherals works the same. Instead of calling a function to configure some peripheral, they just have a chunk of memory which serves as the hardware API. Write `0x8000_0000` to this address, and the serial port will send data at 1 Megabit per second. Write `0x0800_0000` to this address, and the serial port will send data at 512 Kilobits per second. Write `0x0000_0000` to another address, and the serial port gets disabled. You get the idea. These configuration registers look a little bit like this:

:::

---

![](./assets/nrf52-spi-frequency-register.png)

::: notes

This interface is how you interact with the hardware, no matter what language you are talking about. Assembly, C, and also Rust.

:::

---

## Why Rust for Embedded?

---

## LLVM

::: notes

LLVM already supports microcontrollers, thanks to its' C and C++ usage

:::

---

## Rust in 2018

::: notes

Rust as a language has defined and is stabilizing all of the language items needed in an embedded context, mostly around the wierdness that is running code outside of an operating system, like what to do if your code panics, or how to start your code after you first boot up

:::

---

`#![no_std]`

::: notes

We don't have an operating system, or anything we can pretend is one, so it would be hard to use the standard library, but we can throw it all out if we need, and add back just the parts we want

:::

---

```
$ rustup default stable
$ cargo build --target thumbv7em-none-eabihf
```

::: notes

Oh, and as of the 2018 edition of Rust, you can pretty much just type `cargo build --target thumbv7em-none-eabihf`, and get a microcontroller binary out, while still keeping all of the things that are nice about Rust like helpful compiler warnings, cargo for managing packages, the borrow checker, the type systems, etc.

:::

---

## Now what?

::: notes

Great! We have all of the superpowers of Rust, but how do we interact with those peripherals? They are just arbitrary memory locations, and dereferencing those is `unsafe`! Do we need to do something like this every time we want to use a peripheral?

:::

---

```rust
const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

fn read_serial_port_speed() -> u32 {
    unsafe {
        *SER_PORT_SPEED_REG
    }
}

fn write_serial_port_speed(val: u32) {
    unsafe {
        *SER_PORT_SPEED_REG = val;
    }
}
```

::: notes

Actually, its a little worse. Since these peripherals can change at any time (if the hardware decides to change it without the CPU knowing), we need to mark these fields as volatile, so the compiler won't optimize away reads or writes. That looks like this:

:::

---

```rust
use core::ptr;
const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

fn read_serial_port_speed() -> u32 {
    unsafe {
        ptr::read_volatile(SER_PORT_SPEED_REG)
    }
}

fn write_serial_port_speed(val: u32) {
    unsafe {
        ptr::write_volatile(SER_PORT_SPEED_REG, val);
    }
}
```

::: notes

This is a little messy, so the first reaction might be to wrap these related things up in to a `struct` to organize them better. Maybe you would come up with something like this:

:::

---

```rust
use core::ptr;

struct SerialPort;

impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;
    pub const SER_PORT_SPEED_HI: u32 = 0x8000_0000;
    pub const SER_PORT_SPEED_LO: u32 = 0x0800_0000;

    fn new() -> SerialPort {
        SerialPort
    }

    fn read_speed(&self) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }

    fn write_speed(&mut self, val: u32) {
        unsafe {
            ptr::write_volatile(Self::SER_PORT_SPEED_REG, val);
        }
    }
}
```

::: notes

And this is a little better! We've hidden that random looking memory address, and presented something that feels a little more rusty. We can even use our new interface:

:::

---

```rust
fn do_something() {
    let mut serial = SerialPort::new();

    let speed = serial.read_speed();
    // Do some work
    serial.write_speed(speed * 2);
}
```

::: notes

But the problem with this is that you can create one of these structs anywhere! Imagine this:

:::

---

```rust
fn do_something() {
    let mut serial = SerialPort::new();
    let speed = serial.read_speed();

    // Be careful, we have to go slow!
    if speed != SerialPort::SER_PORT_SPEED_LO {
        serial.write_speed(SerialPort::SER_PORT_SPEED_LO)
    }
    // First, send some pre-data
    something_else();
    // Okay, lets send some slow data
    // ...
}

fn something_else() {
    // We gotta go fast for this!
    serial.write_speed(SerialPort::SER_PORT_SPEED_HI);
    // send some data...
}
```

::: notes

In this case, if we were only looking at the code in `do_something()`, we would think, we are definitely sending our serial data slowly, why isn't that thing working?

In this example, it is easy to see. However, once this code is spread out over multiple modules, drivers, developers, and days, it gets easier and easier to make these kinds of mistakes.

:::

---

## What are our rules?

---

1. We should be able to share any number of read-only accesses to these peripherals
2. If something has read-write access to a peripheral, it should be the only reference

::: notes

Which, sounds suspiciously exactly like what the Borrow Checker does already!

But for the Borrow Checker, we need to have exactly one instance of each peripheral, so Rust can handle ownerships and borrows correctly. Well, luckliy in the hardware, there is only one instance of this specific serial port, but how can we expose that in code?

:::