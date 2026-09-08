<!---

This file is used to generate your project datasheet. Please fill in the information below and delete any unused
sections.

You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

## How it works

This project implements an SPI-controlled PWM peripheral consisting of two main modules: an SPI Peripheral for register management and a PWM Peripheral for signal generation. The design runs at a 10 MHz system clock and is configured over an SPI interface running at approximately 100 kHz.

### SPI Peripheral

The SPI peripheral operates in Mode 0 (data sampled on the rising edge of SCLK, valid on the falling edge) and supports write-only transactions — CIPO is not implemented, so read operations are ignored. All SPI inputs (nCS, SCLK, COPI) are synchronized through a 2-stage flip-flop chain to prevent metastability from crossing clock domains.

Each SPI transaction is a fixed length of 16 clock cycles, broken down as follows:

| Field       | Width | Description                        |
|-------------|-------|-------------------------------------|
| R/W bit     | 1 bit | 1 = Write, 0 = Read (ignored)       |
| Address     | 7 bits| Register address, valid range 0x00–0x04 |
| Data        | 8 bits| Data byte to write                  |

A transaction begins on the falling edge of nCS, and data is captured on each rising edge of SCLK. Invalid addresses are ignored.

**Example transaction** — write `0xF0` to address `0x00`:
Bitstream: `1` (write) + `0000000` (address 0x00) + `11110000` (data 0xF0)

### Register Map

| Address | Register           | Description                          | Reset Value |
|---------|---------------------|---------------------------------------|-------------|
| 0x00    | `en_reg_out_7_0`     | Enable outputs on `uo_out[7:0]`       | 0x00        |
| 0x01    | `en_reg_out_15_8`    | Enable outputs on `uio_out[7:0]`      | 0x00        |
| 0x02    | `en_reg_pwm_7_0`     | Enable PWM on `uo_out[7:0]`           | 0x00        |
| 0x03    | `en_reg_pwm_15_8`    | Enable PWM on `uio_out[7:0]`          | 0x00        |
| 0x04    | `pwm_duty_cycle`     | PWM duty cycle (0x00 = 0%, 0xFF = 100%)| 0x00       |

### Output Behavior

Each of the 16 outputs (`{uio_out[7:0], uo_out[7:0]}`) is independently controlled by a pair of bits — one from the output-enable registers, one from the PWM-enable registers:

| Output Enable Bit | PWM Mode Bit | Result       |
|--------------------|--------------|---------------|
| 0                   | X            | Output driven low (0) |
| 1                   | 0            | Output driven high (1)|
| 1                   | 1            | Output driven by PWM  |

Output Enable takes precedence over PWM Mode — if a bit's output enable is 0, it is forced low regardless of its PWM enable setting.

### PWM Peripheral

The PWM peripheral generates a 3 kHz PWM signal, derived from the 10 MHz system clock via a clock divider. The duty cycle is set by the `pwm_duty_cycle` register (address 0x04):

- Duty cycle (%) = `(pwm_duty_cycle / 256) * 100%`
- Special case: `pwm_duty_cycle == 0xFF` forces the output permanently high (100%), rather than 255/256.

Any output bit with its PWM-enable bit set will reflect this PWM waveform (subject to the output-enable precedence rule above); all PWM-enabled bits share the same frequency and duty cycle, since there is a single duty cycle register.

## How to test

The design is verified using Cocotb testbenches:

- **SPI testbench (provided)** — covers valid and invalid address handling, register writes, and checks that `uo_out` / `uio_out` reflect the correct values after each write. A helper function is provided to execute SPI transactions.
- **PWM testbench (to be written)** — sweeps `pwm_duty_cycle` from 0x00 to 0xFF, verifies the interaction between the output-enable and PWM-enable registers, and checks:
  - PWM frequency is ~3 kHz (±1%)
  - PWM duty cycle accuracy (±1%)
  - Edge cases: duty cycle = 0x00, duty cycle = 0xFF, and mid-range values

Waveforms can be inspected via the generated `tb.vcd` file to debug register writes and PWM output during development.

## External hardware

None — this design only uses the standard Tiny Tapeout SPI-style pin mapping (COPI on `ui_in[1]`, nCS on `ui_in[2]`, SCLK on `ui_in[0]`) and drives its outputs directly on `uo_out` / `uio_out`.