+++
title = "Let's understand what is Blanket Implementation in Rust"
date = "2025-09-07"
description = "In this post, i will try to explain what blanket implementation means in Rust"

[taxonomies]
tags = ["blanket", "impl", "generic"]
+++
 
While writing the RTC HAL chapter for the [Rust Embedded Drivers (RED) book](https://red.implrust.com), I needed to use blanket implementations as a design choice. Instead of explaining this concept in the book and cluttering the book, I thought it would be better to write a blog post instead. That's how i end up with my first blog post in implRust :)

# What is a Blanket Implementation?

The name comes from the English idiom where "blanket" means something that covers or applies to multiple things at once.  A blanket implementation lets you implement a trait for all types that meet certain conditions. Instead of writing the same trait implementation for different types one by one, you write it once and it applies to all matching types.

Here's the basic syntax:

```rust
impl<T> MyTrait for T 
where 
    T: SomeCondition 
{
    // implementation
}
```

You can read it as "For any type T that meets SomeCondition, implement MyTrait for T"

## Example

Let's say we create a trait called `Loggable` with a log method. Instead of implementing this trait for each individual type, we can use a blanket implementation for all types that implement the Debug trait:

```rust
use core::fmt::Debug;

trait Loggable {
    fn log(&self);
}

// Blanket implementation
impl<T: Debug> Loggable for T {
    fn log(&self) {
        println!("[log] {:?}", self);
    }
}

#[allow(dead_code)]
#[derive(Debug)]
struct Cat {
    name: String,
}

fn main() {
    let numbers = vec![1, 2, 3];
    numbers.log();

    let s = String::from("Blanket");
    s.log();

    let c = Cat { name: "Astra".to_string() };
    c.log();
}
```

Now any type that implements Debug automatically gets the Loggable trait. We didn't have to write separate implementations for Vec<i32>, String, or our custom Cat struct.

## Rust Standard Library

Blanket implementations are used extensively in the Rust standard library. A great example is how the ToString trait is implemented for any type that implements the Display trait:

```rust
impl<T: fmt::Display + ?Sized> ToString for T {
    #[inline]
    fn to_string(&self) -> String {
        <Self as SpecToString>::spec_to_string(self)
    }
}
```

This means you don't need to manually implement ToString for your types. If your type implements Display, it automatically gets the to_string() method for free.  You can read more about this in the [Rust book](https://doc.rust-lang.org/book/ch10-02-traits.html#listing-10-15).

## Example Use Case: Embedded HAL's I2C Blanket Implementation

Here's a simplified version of how embedded-hal uses blanket implementations for I2C. It implements the I2c trait for mutable references (&mut T) to all types that already implement I2c:

```rust
mod hal {
    pub trait I2c {
        fn write(&mut self, data: &[u8]);
    }

    impl<T: I2c> I2c for &mut T { // Blanket Implementation
        #[inline]
        fn write(&mut self, data: &[u8]) {
            T::write(self, data);
        }
    }
}

use hal::I2c;

pub struct I2cBus {}

impl hal::I2c for I2cBus {
    fn write(&mut self, data: &[u8]) {
        dbg!(data);
    }
}

fn write_to_any_i2c<T: I2c>(mut i: T) {  // assume this is a driver 
    i.write(b"Rust");
}

fn main() {
    let bus1 = I2cBus {};
    write_to_any_i2c(bus1); // Takes ownership

    let mut bus2 = I2cBus {};
    write_to_any_i2c(&mut bus2); // Takes reference
}
```

### Why is this useful?
This allows drivers to be flexible - they can take either ownership of the I2C device or just borrow it. Without this blanket implementation, you would need separate functions or the driver would be forced to choose between taking ownership or taking a reference, limiting how it can be used.

Let's comment out the blanket implementation and see what happens:

```rust
mod hal {
    pub trait I2c {
        fn write(&mut self, data: &[u8]);
    }

    // impl<T: I2c> I2c for &mut T { // Blanket Implementation
    //     #[inline]
    //     fn write(&mut self, data: &[u8]) {
    //         T::write(self, data);
    //     }
    // }
}

use hal::I2c;

pub struct I2cBus {}

impl hal::I2c for I2cBus {
    fn write(&mut self, data: &[u8]) {
        dbg!(data);
    }
}

// Function that takes ownership
fn write_to_any_i2c<T: I2c>(mut i: T) {
    i.write(b"Rust");
}

// Function that takes a mutable reference
fn write_to_any_i2c_ref<T: I2c>(i: &mut T) {
    i.write(b"Rust");
}


fn main() {
    let bus1 = I2cBus {};
    write_to_any_i2c(bus1); // Takes ownership

    // Compilation error: error[E0277]: the trait bound `&mut I2cBus: I2c` is not satisfied
    // let mut bus2 = I2cBus {};
    // write_to_any_i2c(&mut bus2); // Takes reference
    
    let mut bus2 = I2cBus {};
    write_to_any_i2c_ref(&mut bus2); // Takes reference
}
```

Without the blanket implementation, you need two separate functions - one for owned values and one for references. Your generic functions can't work with both, so users have to pick the right function based on how they're using their data.

## With Great Power Comes Great Responsibility

Blanket implementations are powerful, but they come with an important limitation: once you create a blanket implementation, you cannot create specific implementations for individual types that meet the same condition.

For example, let's say we want to add a Human struct that also implements Debug, but we want it to log differently than other types. We can't do this:

```rust
use core::fmt::Debug;

trait Loggable {
    fn log(&self);
}

// Blanket implementation
impl<T: Debug> Loggable for T {
    fn log(&self) {
        println!("[log] {:?}", self);
    }
}

#[allow(dead_code)]
#[derive(Debug)]
struct Cat {
    name: String,
}

#[allow(dead_code)]
#[derive(Debug)]
struct Human {
    first_name: String,
    last_name: String,
}

impl Loggable for Human{
    fn log(&self){
        println!("[log] This is Human {}",  self.first_name)
    }
}

fn main() {
    let numbers = vec![1, 2, 3];
    numbers.log();

    let s = String::from("Blanket");
    s.log();

    let c = Cat {
        name: "Astra".to_string(),
    };
    c.log();

    let h = Human {
        first_name: "Astra".to_string(),
        last_name: "Kernel".to_string(),
    };
    h.log();
}
```

You'll get a "conflicting implementations of trait" error because Rust can't decide which implementation to use for Human - the blanket one or the specific one.

This is why you need to carefully consider whether a blanket implementation is the right choice for your use case.
