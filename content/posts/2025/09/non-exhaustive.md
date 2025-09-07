+++
date = "2025-09-07"

title = "How #[non_exhaustive] Helps Your Rust Library"
description = "In this post, i will try to explain what blanket implementation means in Rust"

[taxonomies]
tags = ["blanket", "impl", "generic"]
+++

The #[non_exhaustive] attribute tells Rust that a type might get new fields or variants in the future. You can use it on enums, structs, and even individual enum variants. When you mark something with this attribute, Rust enforces certain restrictions on how external crates can use that type.

> [Official doc explanation](https://doc.rust-lang.org/reference/attributes/type_system.html): The non_exhaustive attribute indicates that a type or variant may have more fields or variants added in the future.

For enums, it means users must include a wildcard pattern (_) in match statements. For structs, it prevents direct construction and requires wildcards in pattern matching. Think of it as a "future-proofing" attribute that says "hey, I might add more to this later, so don't assume this is complete". However, these restrictions only apply to downstream crates that depend on your library. Within the defining crate itself, these rules don't apply.

## Example: Marking enum with non_exhaustive attribute

I'll start with an enum example since we used this attribute for the ErrorKind enum in the RTC HAL chapter of the Rust Embedded Drivers (RED) book. That's what made me want to write this post. Instead of explaining it in detail in the book, I decided to cover it separately here.

Let's say we're creating a library called menu with an enum called `Menu` that other developers can use in their apps. Let's see what happens when you try to add a new variant to the enum.

### Creating the Menu Library
First, create a new library:

```sh
cargo new --lib menu
cd menu
```

In src/lib.rs, define a simple menu enum:

```rust
// menu/src/lib.rs
pub enum Menu {
    Pizza,
    Burger,
    Tacos,
}

impl Menu {
    pub fn price(&self) -> f32 {
        match self {
            Menu::Pizza => 100.0,
            Menu::Burger => 180.0,
            Menu::Tacos => 60.0,
        }
    }
}
```

### Creating an App That Uses the Library

Now create an app that uses your menu library:

```sh
cd ..
cargo new restaurant_app
cd restaurant_app
```

Add your local menu library to Cargo.toml:
```toml
# restaurant_app/Cargo.toml
[dependencies]
menu = { path = "../menu" }
```

Use the library in the app:

```rust
// restaurant_app/src/main.rs
use menu::Menu;

fn main() {
    let order = Menu::Pizza;
    
    match order {
        Menu::Pizza => println!("You ordered pizza for ${}", order.price()),
        Menu::Burger => println!("You ordered a burger for ${}", order.price()),
        Menu::Tacos => println!("You ordered tacos for ${}", order.price()),
    }
}
```

When you run the app, you should get output like "You ordered pizza for $12.99".  Everything works great!

### The Breaking Change Problem


A month later, you want to add burritos to your menu. Update the library:

```rust
// menu/src/lib.rs
pub enum Menu {
    Pizza,
    Burger,
    Tacos,
    Burrito,  // New item!
}

impl Menu {
    pub fn price(&self) -> f32 {
        match self {
            Menu::Pizza => 12.99,
            Menu::Burger => 9.99,
            Menu::Tacos => 8.99,
            Menu::Burrito => 10.99,  // Add price for burrito
        }
    }
}
```

Now try to run the restaurant app, you will get the following compile error:

```sh
error[E0004]: non-exhaustive patterns: `Menu::Burrito` not covered
 --> src/main.rs:6:11
  |
6 |     match order {
  |           ^^^^^ pattern `Menu::Burrito` not covered
```

The app is broken now! Anyone using your library will have to update their code before they can use your new version. This is a breaking change. According to the [Cargo docs](https://doc.rust-lang.org/cargo/reference/semver.html#enum-variant-new), this should be a major change. So as a library developer, you should bump your library version to the next major version (for example: 1.0.0 to 2.0.0) to maintain SemVer compatibility.

### The Solution: Using #[non_exhaustive]

Go back to your menu library and add the #[non_exhaustive] attribute:

```rust
// menu/src/lib.rs
#[non_exhaustive]
pub enum Menu {
    Pizza,
    Burger,
    Tacos,
}

impl Menu {
    pub fn price(&self) -> f32 {
        match self {
            Menu::Pizza => 12.99,
            Menu::Burger => 9.99,
            Menu::Tacos => 8.99,
        }
    }
}
```

> NOTE: adding non_exhaustive attribute itself is a breaking change because without the wildcard pattern, it will give compilation error.

Now you have to update the restaurant app to handle the non-exhaustive enum:

```rust
// restaurant_app/src/main.rs
use menu::Menu;

fn main() {
    let order = Menu::Pizza;
    
    match order {
        Menu::Pizza => println!("You ordered pizza for ${}", order.price()),
        Menu::Burger => println!("You ordered a burger for ${}", order.price()),
        Menu::Tacos => println!("You ordered tacos for ${}", order.price()),
        _ => println!("You ordered something for ${}", order.price()),
    }
}
```

The _ wildcard is now required. Without it, you get this error: "non-exhaustive patterns: `_` not covered"

### Adding New Variants Without Breaking Changes

Now you can safely add the burrito:

```rust
// menu/src/lib.rs
#[non_exhaustive]
pub enum Menu {
    Pizza,
    Burger,
    Tacos,
    Burrito,  // New item added safely!
}

impl Menu {
    pub fn price(&self) -> f32 {
        match self {
            Menu::Pizza => 12.99,
            Menu::Burger => 9.99,
            Menu::Tacos => 8.99,
            Menu::Burrito => 10.99,
        }
    }
}
```

With this change, you can add more variants. No breaking change, no need to update the major version number.

## Example: Marking struct with non_exhaustive attribute

When you mark a struct with #[non_exhaustive], it prevents other crates from creating instances of that struct directly using the `MyStruct { field: value }` syntax. They also can't pattern match on it without using ".." to indicate there might be more fields in the future.

Inside your own crate, you can still create and use the struct normally. But external crates must use constructor functions you provide and include ".." when pattern matching.

This way, you can safely add new fields to your struct in future versions without breaking anyone's code. 

Let's add a configuration struct to our menu library:

```rust
// menu/src/lib.rs
#[non_exhaustive]
pub struct MenuConfig {
    pub theme: String,
    pub show_prices: bool,
}

impl MenuConfig {
    pub fn new(theme: String) -> Self {
        Self {
            theme,
            show_prices: true,
        }
    }
}
```

We've marked MenuConfig with the #[non_exhaustive] attribute. If someone tries to create an instance of MenuConfig using struct literal syntax in the restaurant app, they'll get the error "cannot create non-exhaustive struct using struct expression":

```rust
// This fails to compile! (in the app crate, not in the library)
let config = MenuConfig {
    theme: "dark".to_string(),
    show_prices: false,
};
```

Instead, external crates must use the constructor function you provide:

```rust
let config = MenuConfig::new("dark".to_string());
```

You can still access the fields since we marked them as pub:

```rust
println!("{}", config.theme);
```

This way, you can safely add new fields like "font_size" or "language" to MenuConfig in future versions without breaking anyone's code.


## References

- [Official Doc Reference](https://doc.rust-lang.org/reference/attributes/type_system.html)
- [#\[non_exhaustive\] and private fields for extensibility ](https://rust-unofficial.github.io/patterns/idioms/priv-extend.html)
