# Crash Course In Embedded Systems

> So, embedded systems is a pretty broad term, but it generally covers any system containing hardware and software that is built for one primary task. This is different to devices like phones, laptops, desktops, and servers, which are General Purpose Computers. They are built to run any number of different applications. Unfortunately, the term "embedded systems" covers everything from a TV remote to a rocket engine.

> We'll narrow our scope down the the lower chunk of that range, focusing down on Microcontrollers systems, where even the smallest amount of overhead can be pretty painful. Microcontrollers typicallly have:
>
> * No operating system, or only a minimal one that provides threads and timers
> * Single Core, 32 bit CPUs, with 16-600 MHz clock speeds
> * 20K-512M of SRAM memory
> * 20K-512M of Flash storage

> This covers a huge chunk of devices you might not even think of as individual computers. This could be things like the Thermostat on your wall, a fitness tracker, a Yubikey, or a washing machine.

> Most Microcontrollers have more than just a CPU and RAM, they also come with a bunch of stuff called Peripherals which are useful for interacting with other hardware, like sensors, bluetooth radios, screens, or touch pads. These peripherals are great because you can offload a lot of the processing to them, so you don't have to handle everything in software. Kind of like offloading graphics processing to a video card, so your CPU can spend it's time doing something else important, or doing nothing so it can save power.

> However, unlike graphics cards, which typically have a Software API like Vulkan, Metal, OpenGL, or DirectX, peripherals are exposed to our CPU with a hardware interface, which is mapped to a chunk of the memory. Because of this, we call these Memmory Mapped Peripherals.