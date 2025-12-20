+++
date = "2025-12-20"

title = "Working with Fixed-Point Numbers in Rust Using the fixed Crate"
description = "In this post, we take a quick look at fixed-point numbers and how to use them in Rust. We also briefly explain why they are useful in embedded and other scenarios, with examples using the fixed crate."

[taxonomies]
tags = ["fixed-point", "embedded", "floating"]
+++

<img src="/img/2025/12/fixed-point-number-illustration.png"  alt="Fixed Point number illustration" style="width:400px; height:auto; display:block; margin:auto;"/>


While revising the Pico Pico book (an embedded Rust book for the Raspberry Pi Pico 2), I was working on the PWM chapter for controlling servo motors. In that chapter, embassy-rp uses a fixed-point number to store the PWM clock divider. Both the integer and fractional parts are stored together in a single field called divider, using a type from the "fixed" crate.

Instead of explaining fixed-point numbers briefly there, I decided it made more sense to write a separate blog post. That is how this post came to be. Here, I will give a short introduction to fixed-point numbers and explain how to use them in Rust.

## What is Fixed Point?

A fixed-point number is a way to represent fractional (non-integer) values using only integer arithmetic. Unlike floating-point numbers (like `f32` or `f64`), fixed-point numbers reserve a predetermined number of bits for the fractional part. This makes them predictable, fast, and ideal for embedded systems where floating-point hardware might be unavailable or too slow.


## Understanding Fixed-Point Notation

This idea is explained well in the [fixed crate](https://docs.rs/fixed/) documentation, but let me give a quick overview.  

The key idea is this. If you have n total bits and you choose f bits for the fractional part, then you automatically have `n - f` bits left for the integer part. Once this split is chosen, the position of the decimal point is fixed.

Fixed-point numbers maintain uniform spacing between consecutive values throughout their range. The step size formula is `1/2^f`.

For example, consider a 16-bit fixed-point number where 4 bits are used for the fraction. Here, f = 4, so the remaining 16 - 4 = 12 bits are used for the integer part. With 4 fractional bits, the smallest step size is 1 / 2⁴ = 0.0625. This means fractional values are represented as multiples of 0.0625. After 0.0625, the next value is 0.125, and the numbers continue to increase in fixed, evenly spaced steps.   


## Binary Representation

Let's use a 16-bit unsigned number as an example. Normally, to store an integer like 5, the binary representation would be `0000_0000_0000_0101`. With fixed-point representation, we dedicate a predetermined number of bits for the fractional part, depending on our precision needs. The remaining upper bits store the integer part.

For instance, if we allocate the last 4 bits for the fraction, we have 12 bits left for the integer.

### Example: 4 Fractional bits

Here's how the bits map to their values:

| Bit Position | 15   | 14   | 13  | 12  | 11  | 10 | 9  | 8  | 7  | 6  | 5  | 4  | . | 3   | 2    | 1     | 0      |
| ------------ | ---- | ---- | --- | --- | --- | -- | -- | -- | -- | -- | -- | -- | - | --- | ---- | ----- | ------ |
| Weight       | 2¹¹  | 2¹⁰  | 2⁹  | 2⁸  | 2⁷  | 2⁶ | 2⁵ | 2⁴ | 2³ | 2² | 2¹ | 2⁰ |   | 2⁻¹ | 2⁻²  | 2⁻³   | 2⁻⁴    |

The decimal point (shown as `.` in the table) is just for our understanding, it doesn't actually exist in memory. With 4 fractional bits, the smallest value we can represent is 2⁻⁴ = 0.0625, giving us a precision of 1/16.

### Example: 8 Fractional bits

If we need more precision, we can allocate more bits to the fractional part. With 8 fractional bits, we get:

| Bit Position | 15  | 14 | 13 | 12 | 11 | 10 | 9  | 8  | . | 7   | 6    | 5     | 4      | 3       | 2        | 1         | 0          |
| ------------ | --- | -- | -- | -- | -- | -- | -- | -- | - | --- | ---- | ----- | ------ | ------- | -------- | --------- | ---------- |
| Weight       | 2⁷  | 2⁶ | 2⁵ | 2⁴ | 2³ | 2² | 2¹ | 2⁰ |   | 2⁻¹ | 2⁻²  | 2⁻³   | 2⁻⁴    | 2⁻⁵     | 2⁻⁶      | 2⁻⁷       | 2⁻⁸        |

With 8 fractional bits, our smallest representable value is 2⁻⁸ = 0.00390625 (1/256), giving us finer precision but reducing the maximum integer we can represent.
 
## Fixed Crate Types

The fixed crate provides fixed-point number types in different sizes. The number in the type name tells you how many bits are used in total.

For example, FixedI8 and FixedU8 are 8-bit fixed-point numbers. FixedI16 and FixedU16 are 16-bit types. In the same way, there are 32-bit, 64-bit, and even 128-bit fixed-point types, both signed (FixedI*) and unsigned (FixedU*).

What makes these types flexible is the extra generic parameter that controls how many bits are used for the fractional part. For example, `FixedU16<extra::U4>` is a 16-bit unsigned fixed-point number where 4 bits are used for the fraction. That leaves 16 - 4 = 12 bits for the integer part. If you use `FixedU16<U0>`, there are no fractional bits, so it behaves just like a normal u16. 

The crate also provides convenient type aliases so you do not always have to write the generic form. For example, U12F4 represents a unsigned fixed-point number with 12 integer bits and 4 fractional bits. You can use it directly like this:

```rust
use fixed::types::U12F4;
```

## Example 1

Create a new Rust project, add the `fixed` crate as a dependency, and replace main.rs with the following code.

To make things easier to understand, the example prints the same value in several different forms.

First, it prints the fixed-point value itself. The :.4 formatting is only for display and shows four digits after the decimal point.

Next, it uses helper methods from the fixed crate to print the fractional part and the integer part separately.
 
Then it prints the binary representation in two different ways.

The first binary print ({:b}) is a human-friendly representation provided by the fixed crate. It inserts a binary point to show where the fractional bits are. This can feel a little confusing at first glance, at least it did for me. But once you get used to reading it, it actually becomes easier to reason about fixed-point values.

The second binary print uses to_bits(), which returns the raw 16-bit value stored internally. For clarity, the integer bits and the fractional bits are manually separated with an underscore when printing.

```rust
use fixed::{FixedU16, types::extra};
use fixed::types;

fn main() {
    // let f_u16: FixedU16<extra::U4> = FixedU16::from_num(5.75); // you can use this approach
    let f_u16: FixedU16<extra::U4> = types::U12F4::from_num(5.75); // or this approach
    assert_eq!(f_u16, 5.75);

    println!("Fixed Point: {:.4}", f_u16);
    println!("Fraction: {}", f_u16.frac());
    println!("Integer: {}", f_u16.int());
    println!("Binary with Fraction repr: {:b}", f_u16);

    // println!("Actual binary: {:016b}", f_u16.to_bits());
    let bits = format!("{:016b}", f_u16.to_bits());
    let formatted = format!(
        "{}_{}",
        &bits[..12],
        &bits[12..],
    );
    println!("Actual binary: {}", formatted);

    println!("F32: {}", f_u16.to_num::<f32>());
}
```

If you run this code, you will see output like this:

```sh
Fixed Point: 5.7500
Frac: 0.75
Int: 5
Binary with Fraction repr: 101.11
Actual binary: 000000000101_1100
F32: 5.75
```

When I first saw Binary with Fraction repr: 101.11, it was confusing. My brain read the 11 in the fractional part as the number three. That is not how fixed-point fractions work.


To understand what this value really represents, let's look at the binary representation.

| Bit Position | 15  | 14  | 13 | 12 | 11 | 10 | 9  | 8  | 7  | 6  | 5  | 4  | . | 3   | 2    | 1   | 0   |
| ------------ | --- | --- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | - | --- | ---- | --- | --- |
| Stored Bit   | 0   | 0   | 0  | 0  | 0  | 0  | 0  | 0  | 0  | 1  | 0  | 1  |   | 1   | 1    | 0   | 0   |
| Weight       | 2¹¹ | 2¹⁰ | 2⁹ | 2⁸ | 2⁷ | 2⁶ | 2⁵ | 2⁴ | 2³ | 2² | 2¹ | 2⁰ |   | 2⁻¹ | 2⁻²  | 2⁻³ | 2⁻⁴ |

The integer part is straightforward. The bits 101 represent the value 5 using normal binary rules.

The fractional part is where the confusion usually comes from. Each fractional bit represents a negative power of two.

```sh
fraction = 1 × 0.5 + 1 × 0.25 + 0 × 0.125 + 0 × 0.0625
```

That adds up to 0.75.

## Example 2

Let’s look at one more example. This time, I try to represent the value 20.3. I keep the same fixed-point configuration as before, with 4 fractional bits.

```rust
use fixed::{FixedU16, types::extra};
use fixed::types;

fn main() {
    let f_u16: FixedU16<extra::U4> = FixedU16::from_num(20.3);
    assert_eq!(f_u16, 20.3125);

    println!("Fixed Point: {:.4}", f_u16);
    println!("Fraction: {}", f_u16.frac());
    println!("Integer: {}", f_u16.int());
    println!("Binary with Fraction repr: {:b}", f_u16);

    // println!("Actual binary: {:016b}", f_u16.to_bits());
    let bits = format!("{:016b}", f_u16.to_bits());
    let formatted = format!(
        "{}_{}",
        &bits[..12],
        &bits[12..],
    );
    println!("Actual binary: {}", formatted);

    println!("F32: {}", f_u16.to_num::<f32>());
}
```

If you expect the line assert_eq!(f_u16, 20.3125) to fail, you might be surprised. It does not fail. The code compiles successfully and produces the following output:

```sh
Fixed Point: 20.3125
Fraction: 0.3
Integer: 20
Binary with Fraction repr: 10100.0101
Actual binary: 000000010100_0101
F32: 20.3125
```

This happens because 20.3 cannot be represented exactly using this fixed-point format. As stated earlier, the fractional part can only increase in steps of 1 / 2^f. With 4 fractional bits, the step size is: `1 / 2^4 = 0.0625`

So the fraction 0.3 must be rounded to the nearest multiple of 0.0625. The value 0.3125 is a valid multiple:  `0.0625 × 5 = 0.3125`

In the binary, it is represented like this:

| Bit Position | 15  | 14  | 13 | 12 | 11 | 10 | 9  | 8  | 7  | 6  | 5  | 4  | . | 3   | 2    | 1   | 0   |
| ------------ | --- | --- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | - | --- | ---- | --- | --- |
| Stored Bit   | 0   | 0   | 0  | 0  | 0  | 0  | 0  | 0  | 0  | 1  | 0  | 1  |   | 0   | 1    | 0   | 1   |
| Weight       | 2¹¹ | 2¹⁰ | 2⁹ | 2⁸ | 2⁷ | 2⁶ | 2⁵ | 2⁴ | 2³ | 2² | 2¹ | 2⁰ |   | 2⁻¹ | 2⁻²  | 2⁻³ | 2⁻⁴ |


If we calculate the fractional part:

```sh
fraction = 0 × 0.5 + 1 × 0.25 + 0 × 0.125 + 1 × 0.0625 = 0.3125
```

## In Raspberry Pi Pico 2 (RP2350)

On the Raspberry Pi Pico 2 (RP2350), the PWM clock divider controls how fast the PWM counter runs. The PWM counting rate is calculated by dividing the system clock frequency by this divider value.

The divider register stores the value using a fixed point format. It is split into an integer part and a fractional part. The upper 8 bits represent the integer portion of the divider, and 4 bits represent the fractional portion. Together, these bits form a fixed point number, allowing finer control over the PWM frequency.


The following code snippet is from the [rp-pac](https://github.com/embassy-rs/rp-pac/blob/86fd095ca5056d24a61f0235ed6b07bbcec572d5/src/rp235x/pwm/regs.rs#L220) crate and shows how the divider register is defined.

```rust
#[doc = "INT and FRAC form a fixed-point fractional number. Counting rate is system clock frequency divided by this number. Fractional division uses simple 1st-order sigma-delta."]
#[repr(transparent)]
#[derive(Copy, Clone, Eq, PartialEq)]
pub struct ChDiv(pub u32);
impl ChDiv {
    #[inline(always)]
    pub const fn frac(&self) -> u8 {
        let val = (self.0 >> 0usize) & 0x0f;
        val as u8
    }

    ...

    #[inline(always)]
    pub const fn int(&self) -> u8 {
        let val = (self.0 >> 4usize) & 0xff;
        val as u8
    }

    ...
}
```

Although the register is defined as a 32 bit unsigned integer, only 12 bits are actually used. Of these, 8 bits are used for the integer part and 4 bits for the fractional part.

In the embassy-rp crate, the divider is represented using a type from the fixed crate. When configuring the PWM hardware, this fixed point value is converted into a u32 and written directly to the register.

```rust
if config.divider > FixedU16::<fixed::types::extra::U4>::from_bits(0xFFF) {
    panic!("Requested divider is too large");
}

p.div().write_value(ChDiv(config.divider.to_bits() as u32));
```
