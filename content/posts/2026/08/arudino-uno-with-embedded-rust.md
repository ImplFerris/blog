+++
date = "2026-08-15"

title = "Can I Use Rust with Arduino Uno? Getting Started with Embedded Rust"

description = "Learn how to use Embedded Rust with the Arduino Uno. Get started with Rust on the ATmega328P and discover the tools and libraries needed to build your first Rust program."

[taxonomies]
tags = [
    "embedded-rust",
    "arduino",
    "arduino-uno",
    "atmega328p",
    "avr",
    "avr-hal",
    "embedded"
]
+++

<figure>
  <img src="/img/2026/08/arduino-compatible-uno-blinky-with-rust.jpg" alt="Arduino Uno compatible board running a Rust blinky program">
  <figcaption>Arduino Uno compatible board</figcaption>
</figure>

Embedded Rust keeps getting more mature, and most people use boards like the ESP32, Raspberry Pi Pico or STM32 with Rust. At least, those are the boards I have used most for Embedded Rust.

But what if you already have an Arduino Uno? Can you use Rust with it?

The answer is **YES**.

The Arduino Uno uses the ATmega328P, an 8-bit AVR RISC microcontroller with 32 KB of flash memory, 2 KB of SRAM, and 23 general-purpose I/O pins. With such limited resources, you might wonder if a Rust program can actually fit on the board.

This question sometimes comes up on social media. Some people see the large binary size of Rust programs on desktop and assume Rust will also be too large for microcontrollers. But Embedded Rust has a different side. With `no_std`, we can write programs without the standard library and target small microcontrollers directly. So instead of guessing, let's try it on the Arduino Uno and see how much space our Rust program actually needs.

By the way, I didn't have an Arduino Uno before this, and I had never coded with the Arduino IDE either. I just bought one (a cheap clone) to experiment with Rust :)

{% admonition(type="tip") %}
Don't judge Embedded Rust based only on the Arduino Uno experience. The AVR platform and avr-hal have their own limitations, and the experience can be quite different on other microcontrollers. For example, I like Embassy, which makes it easier to work across different boards and use async programming.
{% end %}

## AVR HAL

For programming the Arduino Uno with Rust, we will use [`avr-hal`](https://github.com/Rahix/avr-hal). It is a Hardware Abstraction Layer for AVR microcontrollers and boards such as the Arduino Uno.

For the Arduino Uno specifically, we will use `arduino-hal`. It is a batteries-included HAL for Arduino and similar boards. It is designed to abstract away the differences between boards as much as possible, so we can use a consistent API when working with different Arduino boards.

## Development Setup

I am using an Ubuntu-based Linux distribution. For other operating systems, or if these steps change in the future, refer to the [`avr-hal` repository](https://github.com/Rahix/avr-hal) for the latest setup instructions.

First, install the AVR development tools and the required system packages:

```bash
sudo apt install avr-libc gcc-avr pkg-config avrdude libudev-dev build-essential
```

We also need [`ravedude`](https://github.com/Rahix/avr-hal/tree/main/ravedude), a CLI utility that makes Rust development for AVR microcontrollers easier. It wraps `avrdude` for flashing and also provides access to the board's serial console, similar to the Arduino IDE.

```bash
cargo +stable install ravedude
```


Finally, install [`cargo-generate`](https://github.com/cargo-generate/cargo-generate). It is a Cargo subcommand that allows us to create a new Rust project from an template. We will use it to create our Arduino Uno project from the `avr-hal` template:

```bash
cargo install cargo-generate
```

The `avr-hal` project currently uses the Rust nightly toolchain. Don't worry about setting this up manually. The project template that we are going to use includes the required configuration, so we can simply use the template and let it take care of the Rust toolchain for us.

With the required tools installed, we are ready to create our first Rust project for the Arduino Uno.

## Creating the Project

The `avr-hal` project provides a template that sets up the required configuration for our Arduino Uno project.

Run the following command:

```bash
cargo generate --git https://github.com/Rahix/avr-hal-template.git
```

The template will ask a few questions. Select **Arduino Uno** as the board and enter a name for your project. I will use "hello-blinky" as the project name.

Once the project is created, move into it:

```bash
# cd your-project-name
cd hello-blinky
```

## Our First Rust Program

The project template already gives us a simple LED blink program in `src/main.rs`. We don't need to change anything for now.

```rust
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    /*
     * For examples (and inspiration), head to
     *
     *     https://github.com/Rahix/avr-hal/tree/main/examples
     *
     * NOTE: Not all examples were ported to all boards!  There is a good chance though, that code
     * for a different board can be adapted for yours.  The Arduino Uno currently has the most
     * examples available.
     */

    let mut led = pins.d13.into_output();

    loop {
        led.toggle();
        arduino_hal::delay_ms(1000);
    }
}
```

The Arduino Uno has an onboard LED connected to pin 13. The program configures the pin 13 as an output and toggles it every second, making the LED blink.

## Running the Project

Let's build and upload the program to the Arduino Uno:

```bash
cargo run
```

If `ravedude` cannot automatically detect the serial port, we can find it manually. On Linux, run:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

For my Arduino Uno clone, this shows `/dev/ttyUSB0`. Your port may be different.

Pass the detected port to `ravedude` using the `-P` option:

```bash
cargo run -- -P "/dev/ttyUSB0"
```

After the program is uploaded, the onboard LED should start blinking.

## How Big Is the Rust Program?

Now, let's see how much space our Rust program actually uses.

If we look at the generated files:

```bash
ls -lh ./target/avr-none/debug/
```

We can see that `hello-blinky.elf` is around 70 KB:

```text
-rwxrwxr-x 2 user user  70K hello-blinky.elf
```

At first, this looks like it would not fit into the Arduino Uno's 32 KB flash memory. But the ELF file also contains debugging information and other data used during development, so it is not the actual firmware size.

We can use the `avr-size` command to check how much flash and SRAM our program uses:

```bash
avr-size -C --mcu=atmega328p target/avr-none/debug/hello-blinky.elf
```

For our program:

```text
AVR Memory Usage
----------------
Device: atmega328p

Program:     262 bytes (0.8% Full)
(.text + .data + .bootloader)

Data:          1 bytes (0.0% Full)
(.data + .bss + .noinit)
```

So our program uses only **262 bytes of flash** out of the Arduino Uno's 32 KB. The 70 KB ELF file is not the actual firmware size.

In Embedded Rust, `cargo size` is also commonly used to check the size of a program. We can install the required LLVM tools with:

```bash
rustup component add llvm-tools
```

Then:

```bash
cargo size
```

```text
   text    data     bss     dec     hex filename
    262       0       1     263     107 hello-blinky.elf
```

The release mode build can help reduce the firmware size. You can build it with `cargo build --release` (or run it with `cargo run --release`). However, for this program, it didn't make much difference.

## More Examples

The `avr-hal` project has a collection of examples showing how to work with different peripherals and features. There is a separate set of examples for the Arduino Uno.

You can find them in the [`avr-hal` examples](https://github.com/Rahix/avr-hal/tree/main/examples) directory.
