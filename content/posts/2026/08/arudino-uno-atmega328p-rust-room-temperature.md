+++
date = "2026-08-28"

title = "Display Room Temperature on LCD with Arduino Uno and Rust"

description = "Build a temperature sensor with Embedded Rust on the Arduino Uno. Learn how to read an NTC thermistor with the ATmega328P ADC, calculate temperature using the Beta equation, and display the result on a 16x2 I2C LCD."

[taxonomies]
tags = [
"embedded-rust",
"arduino",
"arduino-uno",
"atmega328p",
"avr",
"avr-hal",
"thermistor",
"i2c",
"lcd", "hd44780"
]
+++

<figure>
  <img src="/img/2026/08/arduino-uno-thermistor-room-temperature-embedded-rust.jpg" alt="Room Temperature in Arduino Uno With Rust">
  <figcaption>Room Temperature in Arduino Uno With Rust</figcaption>
</figure>

This is the next experiment I did with the Arduino Uno and Embedded Rust. In the last post, I showed you [how to use the Arduino Uno with Rust](https://blog.implrust.com/posts/2026/08/arudino-uno-with-embedded-rust/) and how to check the final firmware size. The firmware size for the LED blinking program was just 262 bytes.

This time, we will create a project where we display the room temperature on an HD44780 LCD using an NTC thermistor.

{% admonition(type="tip") %}
I will not go into detail about how a thermistor works or what a voltage divider is in this post. If you are not familiar with these concepts, you can check out the thermistor chapters in either [Pico Book](https://pico.implrust.com/thermistor/index.html) or the [ESP32 book](https://esp32.implrust.com/thermistor/index.html). The Books are available for free.
{% end %}

A thermistor is a resistor whose resistance changes with temperature. We are using an NTC thermistor, whose resistance decreases as the temperature increases. By measuring this change in resistance, we can calculate the temperature.

## Components Required

For this project, we need the following components:

- NTC thermistor
- 10 kΩ resistor
- 16x2 HD44780 LCD with I2C interface
- Breadboard 
- Jumper wires

## Hardware Setup

We will connect the NTC thermistor and a `10 kΩ` resistor as a voltage divider. The output of the voltage divider is connected to the Arduino Uno's `A1` analog pin.

The 16x2 HD44780 LCD is connected through I2C. The `SDA` and `SCL` lines are connected to `A4` and `A5`, respectively.

The complete hardware setup is shown below:


<figure>
  <a href="/img/2026/08/arudino-uno-thermistor-lcd-rust.png" target="_blank">
    <img src="/img/2026/08/arudino-uno-thermistor-lcd-rust.png"
         alt="Arduino Uno thermistor and I2C LCD circuit">
  </a>
  <figcaption>Arduino Uno thermistor and I2C LCD circuit</figcaption>
</figure>

The Arduino Uno provides `5 V` power to both the thermistor circuit and the LCD. The Rust program will read the voltage divider through the ADC, calculate the temperature, and display it on the LCD.

**Thermistor Circuit**

<table>
  <thead>
    <tr>
      <th>From</th>
      <th style="width: 250px; margin: 0 auto;">Wire</th>
      <th>To</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Arduino Uno 5V</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire red" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>10 kΩ Resistor</td>
    </tr>
    <tr>
      <td>NTC Thermistor</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire black" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>Arduino Uno GND</td>
    </tr>
    <tr>
      <td>10 kΩ Resistor</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire yellow" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>NTC Thermistor</td>
    </tr>
    <tr>
      <td>Junction between resistor and thermistor</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire yellow" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>Arduino Uno A1</td>
    </tr>
  </tbody>
</table>

The junction between the resistor and thermistor is connected to the Arduino Uno's A1 pin, allowing us to measure the voltage from the voltage divider.

**LCD Circuit**

<table>
  <thead>
    <tr>
      <th>From</th>
      <th style="width: 250px; margin: 0 auto;">Wire</th>
      <th>To</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Arduino Uno 5V</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire red" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>LCD VCC</td>
    </tr>
    <tr>
      <td>Arduino Uno GND</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire black" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>LCD GND</td>
    </tr>
    <tr>
      <td>Arduino Uno A4 (SDA)</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire blue" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>LCD SDA</td>
    </tr>
    <tr>
      <td>Arduino Uno A5 (SCL)</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire green" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>LCD SCL</td>
    </tr>
  </tbody>
</table>

## From ADC Reading to Temperature

What we read from the ADC is a digital value representing the voltage at the junction of the resistor and thermistor. We need to convert this value into temperature. First, we calculate the thermistor's resistance from the ADC reading. We can then use that resistance to calculate the temperature.

For our voltage divider, we can calculate the thermistor resistance using:

$$
R_T = R_1 \frac{ADC}{ADC_{max} - ADC}
$$

where:

- $R_T$ is the thermistor resistance.
- $R_1$ is the fixed resistor resistance (`10 kΩ`).
- $ADC$ is the value read from the ADC.
- $ADC_{max}$ is the maximum ADC value (`1023` for the Arduino Uno).

Once we have the thermistor resistance, we can calculate the temperature using the Beta equation:

$$
T = \frac{1}{\frac{1}{T_0} + \frac{1}{B}\ln\left(\frac{R_T}{R_0}\right)}
$$

The formula uses the thermistor's current resistance ($R_T$), its reference resistance ($R_0$) at a reference temperature ($T_0$), and its Beta coefficient ($B$). Manufacturers usually provide these values.

For our thermistor, $R_0 = 10 kΩ$ at $T_0 = 25°C$, and $B = 3950$. We calculate $R_T$ from the ADC reading.

## Project Setup

We will use the same `avr-hal` project template that we used in the previous Arduino Uno project.

Run the following command:

```bash
cargo generate --git https://github.com/Rahix/avr-hal-template.git
```

The template will ask a few questions. Select **Arduino Uno** as the board and enter a name for your project. I will use `room-temperature` as the project name.

## Crates

Add the following crates to `Cargo.toml`:

```toml
hd44780-driver = "0.4.0"
libm = "0.2.16"
```

There are several Rust crates for communicating with an HD44780 LCD. I will use `hd44780-driver`. If you want custom character support, you can use a crate such as `liquid_crystal`.

We will need the natural logarithm (`log`) as part of the temperature calculation formula, so we use `libm` for this calculation in our `no_std` project.

We will also use `heapless` to format the temperature value as text before sending it to the LCD:

```toml
heapless = "0.9.3"
```

We can reduce the firmware size by avoiding the use of `heapless` with a different approach. However, we will first use `heapless` to keep the code simple and then optimize it later.

## Converting ADC Reading to Temperature

Let's convert the formulas into Rust functions and define the corresponding constants. These functions should be obvious, so we will not go into detail about them.

```rust
const fn kelvin_to_celsius(kelvin: f64) -> f64 {
    kelvin - 273.15
}

const fn celsius_to_kelvin(celsius: f64) -> f64 {
    celsius + 273.15
}

const B_VALUE: f64 = 3950.0;

const REF_TEMP: f64 = 25.0; // Reference temperature 25°C
const REF_RES: f64 = 10_000.0; // Thermistor resistance at the Reference Temperature(25°C)
const REF_TEMP_K: f64 = celsius_to_kelvin(REF_TEMP);

const R1_RES: f64 = REF_RES; // 10_000.0 ohms

const ADC_MAX: f64 = 1023.0; // 1023 for 10-bit ADC

fn adc_to_resistance(adc_value: f64) -> f64 {
    let x: f64 = adc_value / (ADC_MAX - adc_value);
    R1_RES * x
}

// B Equation to convert resistance to temperature
fn calculate_temperature(current_res: f64, b_val: f64) -> f64 {
    let ln_value = libm::log(current_res / REF_RES); // Use libm for `no_std`
    let inv_t = (1.0 / REF_TEMP_K) + ((1.0 / b_val) * ln_value);
    1.0 / inv_t
}
```

## Setting Up the ADC

Let's set up the ADC to read the thermistor connected to `A1`.

```rust
let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());
let a1 = pins.a1.into_analog_input(&mut adc);
```

## Setting Up the LCD

Next, let's set up the I2C interface and initialize the LCD.

```rust
let i2c = arduino_hal::I2c::new(
    dp.TWI,
    pins.a4.into_pull_up_input(),
    pins.a5.into_pull_up_input(),
    50000,
);

let mut text: String<32> = String::new();
let mut delay = Delay::new();

let i2c_address = 0x27;

let Ok(mut lcd) = HD44780::new_i2c(
    i2c,
    i2c_address,
    &mut delay,
) else {
    panic!("failed to initialize display");
};
```

## Reading and Displaying the Temperature

In the main loop, we will read the ADC value and calculate the temperature from it. Since `avr-hal` does not support formatting floating-point values, we split the temperature into whole and fractional parts as `i32` values. With the help of `heapless`, we format these values into a string and finally send the text to the LCD.

```rust
let mut text: String<32> = String::new();
let mut delay = Delay::new();

loop {
    let adc_value = a1.analog_read(&mut adc);

    let current_res = adc_to_resistance(adc_value as f64);
    let temperature_kelvin = calculate_temperature(current_res, B_VALUE);
    let temperature_celsius = kelvin_to_celsius(temperature_kelvin);

    // Clear the Heapless string
    text.clear();
    // Unshift display and set cursor to 0
    lcd.reset(&mut delay).unwrap();
    // Clear existing characters
    lcd.clear(&mut delay).unwrap();

    let whole = temperature_celsius as i32;
    let frac = ((temperature_celsius.abs() * 100.0) as i32) % 100;

    write!(&mut text, "{}.", whole).unwrap();
    if frac < 10 {
        write!(&mut text, "0").unwrap();
    }
    write!(&mut text, "{}", frac).unwrap();

    lcd.write_str("Temp: ", &mut delay).unwrap();
    lcd.write_str(text.as_str(), &mut delay).unwrap();
    lcd.write_byte(0xDF, &mut delay).unwrap(); // Degree symbol °
    lcd.write_char('C', &mut delay).unwrap();

    arduino_hal::delay_ms(5000);
}
```

## Alternative: Avoiding `heapless`

We can also format the temperature without using `heapless`, which helps reduce the firmware size. Instead of converting the temperature into a string, we can write each digit directly to the LCD.

```rust
let temp_abs = temperature_celsius.abs();
let whole = temp_abs as u8;
let frac = ((temp_abs * 100.0) as i32) % 100;

// Logic to send temperature value without heapless
lcd.write_str("Temp: ", &mut delay).unwrap();
if temperature_celsius.is_sign_negative() {
    lcd.write_str("-", &mut delay).unwrap();
}

lcd.write_byte((whole / 10) + b'0', &mut delay).unwrap();
lcd.write_byte((whole % 10) + b'0', &mut delay).unwrap();
lcd.write_char('.', &mut delay).unwrap();
lcd.write_byte((frac / 10) as u8 + b'0', &mut delay).unwrap();
lcd.write_byte((frac % 10) as u8 + b'0', &mut delay).unwrap();

lcd.write_byte(0xDF, &mut delay).unwrap(); // Degree symbol °
lcd.write_char('C', &mut delay).unwrap();
```

## Firmware Size

Let's check the firmware size using the `heapless` version:


```text
text    data     bss     dec     hex filename
23984     260       1   24245    5eb5 room-temperature.elf
```

Now, let's remove heapless and use the alternative approach we discussed earlier:
```text
text    data     bss     dec     hex filename
20784      30       1   20815    514f room-temperature.elf
```

By avoiding heapless, the firmware size decreases from 24,245 bytes to 20,815 bytes, saving 3,430 bytes.

This was a quick win in terms of firmware size. I did not want to spend too much time on further optimization and overcomplicate this article, so i will stop here.

## The Full code

```rust
#![no_std]
#![no_main]

use arduino_hal::prelude::*;
use arduino_hal::Delay;

use core::fmt::Write;
use heapless::String;
use panic_halt as _;

// HD44780 Driver
use hd44780_driver::HD44780;

const fn kelvin_to_celsius(kelvin: f64) -> f64 {
    kelvin - 273.15
}

const fn celsius_to_kelvin(celsius: f64) -> f64 {
    celsius + 273.15
}

const B_VALUE: f64 = 3950.0;

const REF_TEMP: f64 = 25.0; // Reference temperature 25°C
const REF_RES: f64 = 10_000.0; // Thermistor resistance at the Reference Temperature(25°C)
const REF_TEMP_K: f64 = celsius_to_kelvin(REF_TEMP);

const R1_RES: f64 = REF_RES; // 10_000.0 ohms

const ADC_MAX: f64 = 1023.0; // 1023 for 10-bit ADC

fn adc_to_resistance(adc_value: f64) -> f64 {
    let x: f64 = adc_value / (ADC_MAX - adc_value);
    R1_RES * x
}

// B Equation to convert resistance to temperature
fn calculate_temperature(current_res: f64, b_val: f64) -> f64 {
    let ln_value = libm::log(current_res / REF_RES); // Use libm for `no_std`
    let inv_t = (1.0 / REF_TEMP_K) + ((1.0 / b_val) * ln_value);
    1.0 / inv_t
}

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);

    let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());
    let a1 = pins.a1.into_analog_input(&mut adc);
    let i2c = arduino_hal::I2c::new(
        dp.TWI,
        pins.a4.into_pull_up_input(),
        pins.a5.into_pull_up_input(),
        50000,
    );

    let mut text: String<32> = String::new();
    let mut delay = Delay::new();

    let i2c_address = 0x27;

    let Ok(mut lcd) = HD44780::new_i2c(i2c, i2c_address, &mut delay) else {
        panic!("failed to initialize display");
    };

    lcd.set_cursor_visibility(hd44780_driver::Cursor::Invisible, &mut delay)
        .unwrap();
    lcd.set_cursor_blink(hd44780_driver::CursorBlink::Off, &mut delay)
        .unwrap();

    loop {
        let adc_value = a1.analog_read(&mut adc);
        ufmt::uwriteln!(&mut serial, "ADC: {}", adc_value).unwrap_infallible();

        let current_res = adc_to_resistance(adc_value as f64);

        let temperature_kelvin = calculate_temperature(current_res, B_VALUE);
        let temperature_celsius = kelvin_to_celsius(temperature_kelvin);

        // Clear the Heapless string
        text.clear();
        // Unshift display and set cursor to 0
        lcd.reset(&mut delay).unwrap();
        // Clear existing characters
        lcd.clear(&mut delay).unwrap();

        let whole = temperature_celsius as i32;
        let frac = ((temperature_celsius.abs() * 100.0) as i32) % 100;

        // Alternative: Avoiding Heapless
        // let temp_abs = temperature_celsius.abs();
        // let whole = temp_abs as u8;
        // let frac = ((temp_abs * 100.0) as i32) % 100;

        ufmt::uwrite!(&mut serial, "Temperature: {}.", whole).unwrap_infallible();

        write!(&mut text, "{}.", whole).unwrap();
        if frac < 10 {
            ufmt::uwrite!(&mut serial, "0").unwrap_infallible();
            write!(&mut text, "0").unwrap();
        }
        write!(&mut text, "{}", frac).unwrap();

        ufmt::uwriteln!(&mut serial, "{} °C", frac).unwrap_infallible();

        lcd.write_str("Temp: ", &mut delay).unwrap();
        lcd.write_str(text.as_str(), &mut delay).unwrap();
        lcd.write_byte(0xDF, &mut delay).unwrap(); // Degree symbol °
        lcd.write_char('C', &mut delay).unwrap();

        // Alternative: Avoiding Heapless
        // Logic to send temperature value without heapless
        // lcd.write_str("Temp: ", &mut delay).unwrap();
        // if temperature_celsius.is_sign_negative() {
        //     lcd.write_str("-", &mut delay).unwrap();
        // }
        //
        // lcd.write_byte((whole / 10) + b'0', &mut delay).unwrap();
        // lcd.write_byte((whole % 10) + b'0', &mut delay).unwrap();
        // lcd.write_char('.', &mut delay).unwrap();
        // lcd.write_byte((frac / 10) as u8 + b'0', &mut delay)
        //     .unwrap();
        // lcd.write_byte((frac % 10) as u8 + b'0', &mut delay)
        //     .unwrap();
        //
        // lcd.write_byte(0xDF, &mut delay).unwrap(); // Degree symbol °
        // lcd.write_char('C', &mut delay).unwrap();

        arduino_hal::delay_ms(5000);
    }
}
```

## Final Thoughts

Another Arduino Uno experiment is done. This time, we went a little beyond blinking an LED and built a simple room temperature monitor with a thermistor and I2C LCD.

You can find the complete source code for this experiment on [GitHub](https://github.com/ImplFerris/uno-rust-projects).

You can navigate to the `room-temperature` folder to find the code for this project.
