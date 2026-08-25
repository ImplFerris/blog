+++
date = "2026-08-25"

title = "Make the micro:bit's LED Matrix React to Sound with Rust"

description = "Embedded Rust code using Embassy and microbit-bsp to read the micro:bit V2's built-in microphone and make its onboard LED matrix react to sound."

[taxonomies]
tags = [
    "rust",
    "embedded-rust",
    "microbit",
    "nrf52833",
    "embassy",
]
+++

<video autoplay muted loop playsinline>
  <source src="/videos/2026/08/microbit-v2-embedded-rust-led-matrix-sound-meter.mp4" type="video/mp4">
</video>

My first dev board, the micro:bit, has been sitting in my box and staring at me every time I open my electronics box. So I thought it was time to give it a little attention and have some fun with it.

In this project, I used the micro:bit's built-in microphone to make its LED matrix react to sound. Try shouting at it and watch the LEDs light up!

Your little one (or you) will love it ;)

## Project Setup

I will be using [`microbit-bsp`](https://github.com/lulf/microbit-bsp), an async Board Support Package (BSP) for the BBC micro:bit V2.

The `microbit-bsp` repository already has a nice [microphone example](https://github.com/lulf/microbit-bsp/tree/main/examples/microphone) that reads the sound level and lights up the LED matrix accordingly.

I also implemented a similar project, [Clap to Smile](https://mb2.implrust.com/microphone/clap-to-smile.html), in my [Impl Rust for micro:bit](https://mb2.implrust.com/) book. There, we detect a loud sound such as a clap and display a smiley on the LED matrix.

So this time, instead of reacting to a single loud sound, we will keep track of the recent sound levels and turn them into a small moving wave on the LED matrix.

I generated the project using the same template I use for the book:

```bash
cargo generate --git https://github.com/ImplFerris/mb2-template.git --rev 3d07b56
```

When it asks, select Embassy and enabled the BSP.

## Reading the Microphone

Let's initialize the microphone using the SAADC peripheral.

```rust
let mut microphone = Microphone::new(board.saadc, irqs, board.microphone, board.micen);
```

## Collecting Sound Levels

Now let's continuously read the sound level and keep track of the five most recent readings. We smooth the sound level by averaging the previous value with the new reading. This makes the LED display less jumpy.

{% admonition(type="tip") %}
There are many ways to process the sound level and display it on the LED matrix. The approach I used here is just one of them. Feel free to experiment and make your own version more fun and interesting, like the Clap to Smile project I mentioned earlier.
{% end %}

```rust
let mut smooth_level = 0usize;
let mut levels = [0usize; 5];

loop {
    let sound_level = microphone.sound_level().await as usize;

    smooth_level = (smooth_level + sound_level) / 2;

    levels.copy_within(1..5, 0);
    levels[4] = smooth_level;

    display_sound_wave(&mut display, DISPLAY_DURATION, &levels).await;
}
```

We use `copy_within` to move the previous readings one position to the left and add the latest reading at the end. This makes the sound wave appear to move from right to left.

## Displaying the Sound Wave

Now let's display the sound levels on the LED matrix.

We use one column for each sound level. We convert each sound level into a value from 0 to 5, which tells us how many LEDs to light up in that column.

The LEDs are lit from the bottom row upwards. So a low sound level produces a short column, while a loud sound produces a taller one.

```rust
async fn display_sound_wave(display: &mut LedMatrix, length: Duration, levels: &[usize; 5]) {
    let mut frame = Frame::<5, 5>::empty();

    const MAX_ROWS: usize = 5;

    for (column, level) in levels.iter().enumerate().take(5) {
        let height = match level {
            0..=10 => 0,
            11..=24 => 1,
            25..=38 => 2,
            39..=52 => 3,
            53..=66 => 4,
            _ => 5,
        };

        for row in (MAX_ROWS - height)..MAX_ROWS {
            frame.set(column, row);
        }
    }

    display.display(frame, length).await;
}
```

## Completed Project

You can check out the [completed project](https://github.com/ImplFerris/microbit-sound-meter) if you want to clone it and build on top of it.

## Run the Program

Flash the program to the micro:bit and make some noise. As you speak or shout near the microphone, the LED matrix reacts to the sound and the wave moves across the display.

```bash
cargo run
```

## Full Code

```rust
#![no_std]
#![no_main]

use defmt::info;
use embassy_executor::Spawner;
use embassy_time::Duration;
use microbit_bsp::{
    LedMatrix, Microbit,
    display::{Brightness, Frame},
    embassy_nrf::{bind_interrupts, saadc::InterruptHandler},
    mic::Microphone,
};

use {defmt_rtt as _, panic_probe as _};

const DISPLAY_DURATION: Duration = Duration::from_millis(50);

bind_interrupts!(
    struct InterruptRequests {
        SAADC => InterruptHandler;
    }
);

#[embassy_executor::main]
async fn main(_spawner: Spawner) -> ! {
    let board = Microbit::default();

    let mut display = board.display;
    display.set_brightness(Brightness::MAX);

    let irqs = InterruptRequests {};

    let mut microphone = Microphone::new(board.saadc, irqs, board.microphone, board.micen);

    let mut smooth_level = 0usize;
    let mut levels = [0usize; 5];

    loop {
        let sound_level = microphone.sound_level().await as usize;

        smooth_level = (smooth_level + sound_level) / 2;

        info!("Sound level: {}", smooth_level);

        levels.copy_within(1..5, 0);
        levels[4] = smooth_level;

        display_sound_wave(&mut display, DISPLAY_DURATION, &levels).await;
    }
}

async fn display_sound_wave(display: &mut LedMatrix, length: Duration, levels: &[usize; 5]) {
    let mut frame = Frame::<5, 5>::empty();

    const MAX_ROWS: usize = 5;

    for (column, level) in levels.iter().enumerate().take(5) {
        let height = match level {
            0..=10 => 0,
            11..=24 => 1,
            25..=38 => 2,
            39..=52 => 3,
            53..=66 => 4,
            _ => 5,
        };

        for row in (MAX_ROWS - height)..MAX_ROWS {
            frame.set(column, row);
        }
    }

    display.display(frame, length).await;
}
```


