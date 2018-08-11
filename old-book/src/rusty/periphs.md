# Rust and Peripherals

> Great! We have all of the superpowers of Rust, but how do we interact with those peripherals? They are just arbitrary memory locations, and dereferencing those is `unsafe`! Do we need to do something like this every time we want to use a peripheral?

```rust
const SERIAL_PORT_SPEED_REGISTER: *mut u32 = 0x4000_1000 as _;

fn read_serial_port_speed() -> u32 {
    unsafe {
        *SERIAL_PORT_SPEED_REGISTER
    }
}

fn write_serial_port_speed(val: u32) {
    unsafe {
        *SERIAL_PORT_SPEED_REGISTER = val;
    }
}
```

Actually, its a little worse. Since these peripherals can change at any time (if the hardware decides to change it without the CPU knowing), we need to mark these fields as volatile, so the compiler won't optimize away reads or writes. That looks like this:

```rust
use core::ptr;

const SERIAL_PORT_SPEED_REGISTER: *mut u32 = 0x4000_1000 as _;

fn read_serial_port_speed() -> u32 {
    unsafe {
        ptr::read_volatile(SERIAL_PORT_SPEED_REGISTER)
    }
}

fn write_serial_port_speed(val: u32) {
    unsafe {
        ptr::write_volatile(SERIAL_PORT_SPEED_REGISTER, val);
    }
}
```

This is a little messy, so the first reaction might be to wrap these related things up in to a `struct` to organize them better. Maybe you would come up with something like this:

```rust
use core::ptr;

struct SerialPort;

impl SerialPort {
    const SERIAL_PORT_SPEED_REGISTER: *mut u32 = 0x4000_1000 as _;
    pub const SERIAL_PORT_SPEED_HI: u32 = 0x8000_0000;
    pub const SERIAL_PORT_SPEED_LO: u32 = 0x0800_0000;

    fn new() -> SerialPort {
        SerialPort
    }

    fn read_speed(&self) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SERIAL_PORT_SPEED_REGISTER)
        }
    }

    fn write_speed(&mut self, val: u32) {
        unsafe {
            ptr::write_volatile(Self::SERIAL_PORT_SPEED_REGISTER, val);
        }
    }
}
```

And this is a little better! We've hidden that random looking memory address, and presented something that feels a little more rusty. We can even use our new interface:

```rust
fn do_something() {
    let mut serial = SerialPort::new();

    let speed = serial.read_speed();
    // Do some work
    serial.write_speed(speed * 2);
}
```

But the problem with this is that you can create one of these structs anywhere! Imagine this:

```rust
fn do_something() {
    let mut serial = SerialPort::new();
    let speed = serial.read_speed();

    // Be careful, we have to go slow!
    if speed != SerialPort::SERIAL_PORT_SPEED_LO {
        serial.write_speed(SerialPort::SERIAL_PORT_SPEED_LO)
    }

    // First, send some pre-data
    something_else();

    // Okay, lets send some slow data
    // ...
}

fn something_else() {
    // We gotta go fast for this!
    serial.write_speed(SerialPort::SERIAL_PORT_SPEED_HI);
    // send some data...
}
```

In this case, if we were only looking at the code in `do_something()`, we would think, we are definitely sending our serial data slowly, why isn't that thing working?

In this example, it is easy to see. However, once this code is spread out over multiple modules, drivers, developers, and days, it gets easier and easier to make these kinds of mistakes.

What we really want to enforce is:

1. We should be able to share any number of read-only accesses to these peripherals
2. If something has read-write access to a peripheral, it should be the only reference

Which, sounds suspiciously exactly like what the Borrow Checker does already!

But for the Borrow Checker, we need to have exactly one instance of each peripheral, so Rust can handle ownerships and borrows correctly. Well, luckliy in the hardware, there is only one instance of this specific serial port, but how can we expose that in code?

<!--

So what should we do? Well, in the hardware, there is only one instance of this specific serial port, but how can we enforce that?

We could make everything a public static, like this:

```rust
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    }
}
```

But this has two problems:

1. We have to use `unsafe` every time we touch a mutable static value
2. Everyone still has access


    // TODO: Work this in somewhere
    This is a little verbose, but luckily we have a tool called `svd2rust`. It is kind of like `bindgen` or `c2rust`, but instead of turning C code or headers into Rust code, it takes an XML file containing all the register addresses and valid register values, which is provided by the company making the chip, and generates a lot of this code for you. It even does a couple convenient things, like `TALK ABOUT COOL SVD2RUST STUFF`

-->