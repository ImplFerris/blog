+++
date = "2026-09-17"

title = "Accessing STM32 AHB-AP with ESP32 and Embedded Rust"

description = "Use an ESP32 and Embedded Rust to communicate with an STM32 through SWD. Learn how to power up the debug system, select the AHB Access Port, and verify the connection by reading the AP IDR."

[taxonomies]
tags = [
"embedded-rust",
"esp32",
"swd",
"debug-probe",
"stm32",
"stm32f103",
"swd-programmer"
]
+++


<figure>
  <img src="/img/2026/09/SWD (Serial Wire Debug) interface Access Port AHB-AP.jpg" alt="SWD architecture: Debug Port (SW-DP) connected to the AHB-AP and Cortex memory map">
  <figcaption>SWD architecture: Debug Port (SW-DP) connected to the AHB-AP and Cortex memory map</figcaption>
</figure>

In the [previous article](https://blog.implrust.com/posts/2026/09/swd-protocol-programmer-embedded-rust/), I explained the SWD protocol used by debug probes and programmers. I then showed how we can read something called the IDCode from the DP register of the STM32 using an ESP32. It was a simple demonstration that we can make an ESP32 communicate with the STM32 core using SWD.

This article is a continuation of that. This time, we will go one step further and access the Access Port (AHB-AP).

Once we can access the AHB-AP, we will have the foundation needed to access the STM32 memory. In the next article, we will use this to flash firmware to the STM32.

I have already successfully used the ESP32 as a programmer to flash firmware into an STM32F103C8T6 (Blue Pill) board and run a simple LED blinky program. However, I decided to split the remaining steps into another article because the complete implementation was becoming too dense to fit into a single article.

## Plan

The overall flow is:

1. Power up the debug and system domains.
2. Select the Access Port and its register bank.
3. Read and verify the AHB-AP IDR.

## Structure and other modifications

Before moving further, I have restructured the code to keep things organized. We have separate modules for the STM32F1-specific code and the SWD code, while `main` will contain only the code needed to put everything together. Any registers and functions specific to the STM32F1 will be placed in the `stm32f1` module.

The project structure looks like this:

```text
├── src
│   ├── bin
│   │   └── main.rs
│   ├── lib.rs
│   ├── stm32f1.rs
│   └── swd.rs
```

## Writing to a Debug Port Register

In the last article, we only read from the registers. However, this time we will also need to write to registers.

The `write_register` function is almost similar to the `read_register` function, except that we send a Write access request and then send the data bits along with the calculated parity bit.

```rust
fn write_register(&mut self, port: Port, address: u8, value: u32) -> Result<(), SwdError> {
    let request = Self::make_request(port, Access::Write, address);

    self.swdio_output();

    // Request.
    self.write_bits(request as u32, 8);

    // Turnaround.
    self.swdio_input();
    self.clock();

    // ACK.
    self.read_ack()?;

    // Target -> host turnaround.
    self.clock();

    // Host now owns SWDIO.
    self.swdio_output();

    // Data.
    self.write_bits(value, 32);

    // Parity.
    let parity = (value.count_ones() & 1) != 0;
    self.write_bit(parity);

    self.idle_cycle();

    Ok(())
}
```

I have created an enum for the Debug Port registers since we will be working with multiple registers.

```rust

/// Debug Port Read Registers
#[derive(Clone, Copy)]
pub enum DpReadRegister {
    Idcode = 0x00,
    CtrlStat = 0x04,
    Resend = 0x08,
    Rdbuff = 0x0C,
}

/// Debug Port Write Registers
#[derive(Clone, Copy)]
pub enum DpWriteRegister {
    Abort = 0x00,
    CtrlStat = 0x04,
    Select = 0x08,
}
```

We can then create small wrapper functions to read from and write to the Debug Port registers.

```rust
/// Read DP Register
pub fn read_dp(&mut self, address: DpReadRegister) -> Result<u32, SwdError> {
    self.read_register(Port::Dp, address as u8)
}

/// Write to DP Register
pub fn write_dp(&mut self, address: DpWriteRegister, value: u32) -> Result<(), SwdError> {
    self.write_register(Port::Dp, address as u8, value)
}
```

## Power Up the Debug System

So far, we have not accessed the Access Port (AHB-AP in our case). We have only talked to the Debug Port. In order to access the AHB-AP and use it to access the STM32 memory, we first need to power up the debug and system domains.

{% admonition(type="tip") %}
A power domain is a part of the chip whose power can be controlled separately. Here, we are simply making sure that the debug and system parts needed for our SWD communication are powered up.
{% end %}

To achieve that, we will modify the `CTRL/STAT` register in the Debug Port. The register has two power-up request bits: `CDBGPWRUPREQ` (28th bit of the register) and `CSYSPWRUPREQ` (30th bit of the register). `CDBGPWRUPREQ` requests power for the debug domain, while `CSYSPWRUPREQ` requests power for the system domain. We set both bits to power up these domains.

```rust
pub fn power_up_debug(&mut self) -> Result<(), SwdError> {
    const CDBGPWRUPREQ: u32 = 1 << 28;
    const CDBGPWRUPACK: u32 = 1 << 29;

    const CSYSPWRUPREQ: u32 = 1 << 30;
    const CSYSPWRUPACK: u32 = 1 << 31;

    defmt::info!("Writing CTRL/STAT power-up request...");
    self.write_dp(DpWriteRegister::CtrlStat, CDBGPWRUPREQ | CSYSPWRUPREQ)?;

    defmt::info!("Reading CTRL/STAT...");
    for _ in 0..100 {
        let status = self.read_dp(DpReadRegister::CtrlStat)?;

        if status & (CDBGPWRUPACK | CSYSPWRUPACK) == (CDBGPWRUPACK | CSYSPWRUPACK) {
            return Ok(());
        }

        self.delay.delay_micros(100);
    }

    Err(SwdError::PowerUpTimeout)
}
```

After setting the request bits, we read the `CTRL/STAT` register and wait for the corresponding `CDBGPWRUPACK` and `CSYSPWRUPACK` bits. These bits indicate that the debug and system domains have acknowledged the power-up requests.

## Selecting Access Port

As you already know, the Debug Port can be connected to multiple Access Ports (APs). We need to select which AP we want to work with. We can do this using the `SELECT` register in the Debug Port. The `APSEL` field helps us choose which AP we want to access. This field occupies bits 24 through 31 of the `SELECT` register.

Since we are working with the AHB-AP, we will keep `APSEL` set to 0.

## Accessing AP Registers

In the previous article, we saw how to read registers from the Debug Port (DP). This time, we need to access registers from the Access Port (AP).

The SWD request packet only provides two address bits, `A[3:2]`, for selecting an AP register. The lower two address bits, `A[1:0]`, are always 0. Therefore, the address bits in the SWD request can select only four register locations at a time.

To access more registers, the AP register space is divided into banks. Each bank contains four register locations, selected by the `A[3:2]` bits in the SWD request.

The bank is selected using the `APBANKSEL` field in the Debug Port's `SELECT` register. This field provides the upper address bits, `A[7:4]`, while `A[3:2]` come from the SWD request. Together, these fields determine which AP register we access.

The following are the AHB-AP registers that we will use:

| Bank  | Address | Read | Write |
| ----- | ------- | ---- | ----- |
| `0x0` | `0x00`  | CSW  | CSW   |
| `0x0` | `0x04`  | TAR  | TAR   |
| `0x0` | `0x0C`  | DRW  | DRW   |
| `0xF` | `0xFC`  | IDR  | N/A   |

Let's say we want to read the AHB-AP `IDR` register at address `0xFC`.

The upper part, `0xF`, identifies the register bank we want to access. We select this bank by setting the `APBANKSEL` field of the `SELECT` register to `0xF`.

The `SELECT` register is therefore:

<figure>
  <img src="/img/2026/09/SELECT-Register-in-SWD-Debug-Port.jpg" alt="APSEL and APBANKSEL value in SELECT Register">
  <figcaption>APSEL and APBANKSEL value in SELECT Register</figcaption>
</figure>

The lower part of the address is `0xC`, which is `0b1100`. As we discussed in the previous article, the SWD request only contains the `A3` and `A2` bits of the register address. The lower two bits, `A1` and `A0`, are not included in the request and are always 0.

For `0xC` (`0b1100`), `A3` and `A2` are both 1. Therefore, the SWD request will look like this:

<figure>
  <img src="/img/2026/09/SWD-Request-For-Access-Port-IDR.jpg" alt="SWD Request for Accessing the Access Port(AP) IDR">
  <figcaption>SWD Request for Accessing the AP IDR</figcaption>
</figure>

We will represent these AHB-AP registers as a simple enum:

```rust
/// AHB-AP registers
#[derive(Clone, Copy)]
pub enum AhbApRegister {
    Csw = 0x00,
    Tar = 0x04,
    Drw = 0x0C,
    Idr = 0xFC,
}
```

## Function to Select Access Port and Bank

We will create a simple wrapper function to select the Access Port and its register bank. Since we will only work with the AHB-AP, we will keep `APSEL` hardcoded to 0.

```rust
#[derive(Clone, Copy)]
pub enum ApBank {
    Bank0 = 0x0,
    BankF = 0xF,
}

/// Select AHB-AP and Register Bank
pub fn select_ap(&mut self, bank: ApBank) -> Result<(), SwdError> {
    // APSEL = 0
    // APBANKSEL = bank
    self.write_dp(DpWriteRegister::Select, (bank as u32) << 4)
}
```

We will access two banks: `BankF`, which we use to read the `IDR`, and `Bank0`, which contains the `CSW`, `TAR`, and `DRW` registers that we will use later for memory access and Flash programming.

## Reading and Writing to AP Registers

Next, we will create helper functions to read from and write to registers of the Access Port.

The AP read operation is slightly different from reading a DP register because AP reads are "posted". This means that the result of an AP read is returned on the next transfer. We can retrieve this result by reading the `RDBUFF` register of the Debug Port.

```rust
pub fn read_ap(&mut self, address: AhbApRegister) -> Result<u32, SwdError> {
    self.read_register(Port::Ap, address as u8)?;

    self.read_dp(DpReadRegister::Rdbuff)
}
```
 
For writing to an AP register:

```rust
/// Write to AP Register
pub fn write_ap(&mut self, address: AhbApRegister, value: u32) -> Result<(), SwdError> {
    self.write_register(Port::Ap, address as u8, value)
}
```


## Verify the AHB-AP

The AHB-AP Identification Register value for my STM32 Blue Pill is `0x24770011`. You can also check the Arm AHB-AP programmer's model for the Cortex-M3 [here](https://support.arm.com/documentation/ddi0337/h/debug/about-the-ahb-ap/ahb-ap-programmers-model?lang=en). So, we will read the IDR and check if it matches this value.

```rust
const AHB_AP_IDR: u32 = 0x2477_0011;

// Btw, this function goes into Stm32F1 struct
 pub fn verify_ahb_ap(&mut self) -> Result<(), SwdError> {
    self.swd.select_ap(ApBank::BankF)?;

    let idr = self.swd.read_ap(AhbApRegister::Idr)?;

    defmt::info!("AHB-AP IDR = {:#010x}", idr);

    if idr != AHB_AP_IDR {
        return Err(SwdError::InvalidApIdr(idr));
    }

    Ok(())
}
```

## Putting It All Together

Now we can put everything together in the `connect()` function.  

```rust
// part of the Stm32F1 struct
 pub fn connect(&mut self) -> Result<(), SwdError> {
    self.swd.switch_to_swd();
    let dp_idcode = self.swd.read_dp(DpReadRegister::Idcode)?;
    defmt::info!("DP IDCODE = {:#010x}", dp_idcode);

    self.swd.power_up_debug()?;

    self.verify_ahb_ap()?;

    self.swd.select_ap(ApBank::Bank0)?;

    Ok(())
}
```

In main, we create the `Swd` instance, pass it to `Stm32F1`, and then call `connect()`.

```rust
let swd = Swd::new(swclk, swdio, delay);
let mut stm32 = Stm32F1::new(swd);

// --------------------------------------------------------
// Connect to STM32
// --------------------------------------------------------

defmt::info!("Connecting to STM32...");

stm32
    .connect()
    .unwrap_or_else(|e| fail("STM32 connection failed", e));

defmt::info!("SWD connected!");
```

## Final Thoughts

In the first article, we saw how to access the Debug Port registers. In this article, we accessed the Access Port registers. With this, we now have the foundation needed to access the STM32 memory through the AHB-AP.

In the next article, we will use this to access the STM32 Flash controller and program our firmware.

You can find the complete code for this project on [GitHub](https://github.com/implferris/swd-access-port).
