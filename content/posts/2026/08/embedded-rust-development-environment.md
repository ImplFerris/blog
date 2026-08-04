+++
date = "2026-08-04"

title = "Setting Up an Embedded Rust Development Environment"

description = "Set up a complete Embedded Rust development environment. Install Rust, configure the ARM Cortex-M target, install probe-rs, and verify your setup before programming embedded microcontrollers."

[taxonomies]
tags = [
    "embedded-rust",
    "rust",
    "probe-rs",
    "cortex-m",
    "arm",
    "embedded"
]
+++

When developing embedded applications with Rust, you will need to install a few tools on your computer.

If you've already written Rust programs on your desktop, most of the setup is already done. We only need to add support for embedded targets and install a few additional tools for flashing and debugging firmware.

## Install Rust

If you do not already have Rust installed, download and install it from the official Rust website. I recommend following the installation instructions on the official Rust website:

[https://rust-lang.org/tools/install/](https://rust-lang.org/tools/install/)

On Linux and macOS, you can install Rust by running:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

On Windows, download and run the installer from the Rust website.

{% admonition(type="tip") %}
To be frank, if this is your first time learning Rust, I do not recommend jumping straight into Embedded Rust. Spend some time learning the basics of Rust first. It will make your Embedded Rust journey much easier.

After all, I assume you chose Rust not because you just want to copy and paste code. You want to understand how things work, enjoy the learning process, and benefit from the safety that Rust provides.
{% end %}

Once the installation is complete, verify it by running:

```sh
rustc --version
cargo --version
rustup --version
```

You should see the installed versions printed on the screen.

## Install the ARM Cortex-M Target

Rust supports many embedded architectures, including ARM Cortex-M, RISC-V, and Xtensa.

In this article, we'll install the ARM Cortex-M targets, as they are used by many popular development boards such as the Raspberry Pi Pico, Raspberry Pi Pico W, Pico 2, and STM32 Blue Pill.

{% admonition(type="note") %}
If you're developing for an ESP32 board, the setup process is slightly different because ESP32 uses a different architecture. You'll need to install the appropriate Rust target and development tools.

If you're looking for a step-by-step guide, I also have a dedicated free book on [Embedded Rust with ESP32](https://esp32.implrust.com/).
{% end %}

Different microcontrollers use different Cortex-M cores, so Rust provides a separate compilation target for each one.

Some common targets are:

| Cortex-M Core    | Rust Target                 |
| ---------------- | --------------------------- |
| Cortex-M0 / M0+  | `thumbv6m-none-eabi`        |
| Cortex-M3        | `thumbv7m-none-eabi`        |
| Cortex-M4 / M7   | `thumbv7em-none-eabi`       |
| Cortex-M4F / M7F | `thumbv7em-none-eabihf`     |
| Cortex-M33       | `thumbv8m.main-none-eabihf` |

Install the target for boards based on the ARM Cortex-M0+ core, such as the Raspberry Pi Pico and Pico W.

```sh
rustup target add thumbv6m-none-eabi
```

Install the target for boards based on the ARM Cortex-M3 core, such as the STM32 Blue Pill (STM32F103).

```sh
rustup target add thumbv7m-none-eabi
```

Install the target for boards based on the ARM Cortex-M33 core with hardware floating-point support, such as the Raspberry Pi Pico 2 (RP2350).

```sh
rustup target add thumbv8m.main-none-eabihf
```

## Install probe-rs

Once you've compiled your Embedded Rust application, you'll need a way to flash it onto your development board and debug it. `probe-rs`, a modern tool for flashing, debugging, and interacting with embedded microcontrollers.

I recommend following the installation instructions on the official probe-rs website:

[https://probe.rs/docs/getting-started/installation/](https://probe.rs/docs/getting-started/installation/)

Once the installation is complete, verify it by running:

```sh
probe-rs --version
```

If the version is printed successfully, probe-rs is installed correctly.

## Install cargo-generate

When starting a new project, you'll often need to create the same files and write the same configuration before you can begin writing code. `cargo-generate` is a tool that helps to create a new project from a template, so you don't have to set up everything manually.

I normally use `cargo-generate` to create new projects from templates. There are many templates available from the community, and once you're familiar with the Rust ecosystem, you can create your own templates, store them in a Git repository or local directory, and use them to generate future projects.

Install it by running:

```sh
cargo install cargo-generate
```

Verify the installation by running:

```sh
cargo generate --version
```

## Install cargo-binutils (Optional)

`cargo-binutils` provides additional commands for inspecting compiled binaries. It can display firmware size, disassemble machine code, and inspect symbols.

While it isn't required for Embedded Rust development, it can be useful when debugging or exploring what the compiler generated.

Install it by running:

```sh
cargo install cargo-binutils
rustup component add llvm-tools
```

That's it! Your Embedded Rust development environment is now ready.

