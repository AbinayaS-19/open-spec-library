# SPEC-003: UART

| Field | Value |
|-------|-------|
| Spec ID | SPEC-003 |
| Title | UART |
| Version | 0.1 (Draft) |
| Author | Abinaya Senthil |
| Date | 2026-10-10 |
| Status | Draft for review |

## 1. Overview

A single clock domain universal asynchronous receiver transmitter. It sends and receives standard serial frames and is the block students usually meet first when they learn about timing, sampling, and baud rate math. Typical use is a debug console or a slow control link.

## 2. Features

* 8 data bits, LSB first, 1 start bit, 1 stop bit
* Optional parity bit (even or odd), enabled at run time
* 16x oversampling receiver with mid bit sampling
* Run time programmable baud divisor
* Frame error and parity error reporting
* Simple signal level configuration, no bus interface

## 3. Parameters

| Name | Default | Description |
|------|---------|-------------|
| CLK_FREQ | 50_000_000 | System clock frequency in Hz. Used with BAUD to compute the recommended divisor |
| BAUD | 115200 | Target baud rate in bits per second. Used with CLK_FREQ to compute the recommended divisor |

## 4. Interface

All signals are synchronous to the rising edge of clk. Reset is synchronous and active low.

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| clk | 1 | input | System clock |
| rst_n | 1 | input | Synchronous reset, active low |
| cfg_divisor | 16 | input | Baud divisor. One bit period lasts 16 * (cfg_divisor + 1) clock cycles. Program 26 for 115200 baud at 50 MHz. Values below 1 are illegal |
| cfg_parity_en | 1 | input | Enable the parity bit. Sampled at the start of each frame |
| cfg_parity_odd | 1 | input | 0 = even parity, 1 = odd parity. Sampled at the start of each frame |
| tx_start | 1 | input | Single cycle pulse. Starts a transmit frame using tx_data |
| tx_data | 8 | input | Byte to transmit. Must be stable while tx_start is high |
| tx_serial | 1 | output | Transmit line, idle high |
| tx_busy | 1 | output | High from tx_start until the stop bit completes |
| rx_serial | 1 | input | Receive line, idle high |
| rx_data | 8 | output | Received byte, valid when rx_done pulses |
| rx_done | 1 | output | Single cycle pulse when a frame completes |
| rx_frame_error | 1 | output | High if the stop bit of the last frame read low. Updated at rx_done, held until the next frame |
| rx_parity_error | 1 | output | High if the parity check of the last frame failed. Updated at rx_done, held until the next frame |

Parity is controlled at run time by cfg_parity_en and cfg_parity_odd. There are no compile time parity parameters.

## 5. Functional description

1. Reset: when rst_n is low at a rising edge, tx_serial goes to 1, tx_busy clears to 0, rx_done clears to 0, both error flags clear to 0, and all internal state returns to idle. The cfg inputs are driven by the user and are not affected by reset.
2. Frame format: start bit (low, one bit period), 8 data bits LSB first, optional parity bit, stop bit (high, one bit period). Bit order on the wire: start, d0, d1, d2, d3, d4, d5, d6, d7, parity if enabled, stop.
3. Parity: when cfg_parity_en is high, the transmitter appends a parity bit after d7. Even parity (cfg_parity_odd = 0) sets the bit so the total number of 1s in data plus parity is even. Odd parity sets it so the total is odd.
4. TX start: a single cycle tx_start pulse with tx_busy low begins a frame. tx_data is captured on that rising edge.
5. TX busy: tx_start pulses while tx_busy is high are ignored. tx_busy stays high until the stop bit has fully elapsed, then drops.
6. TX timing: each bit is held for exactly 16 * (cfg_divisor + 1) clock cycles. tx_serial idles high between frames.
7. RX start: a falling edge on rx_serial moves the receiver out of idle. The start bit is confirmed by sampling rx_serial at the middle of the bit period (8 oversample ticks after the edge). If it reads high, the edge was a glitch and the receiver returns to idle with no rx_done.
8. RX sampling: data bits, and the parity bit when enabled, are sampled at the middle of each bit period. The receiver counts 16 oversample ticks per bit starting from the confirmed start edge.
9. RX done: after the stop bit period elapses, rx_done pulses for one cycle and rx_data holds the received byte until the next frame completes.
10. Frame error: if the stop bit samples low, rx_frame_error is set when rx_done pulses. rx_data still holds the received bits.
11. Parity error: if parity is enabled and the received parity bit does not match the expected value for the received data, rx_parity_error is set when rx_done pulses.
12. Error flags hold their values until the next frame completes, at which point they are updated again.
13. Configuration sampling: cfg_divisor, cfg_parity_en, and cfg_parity_odd are sampled at the start of each transmit and receive frame. Changing them in the middle of a frame gives undefined behavior for that frame.
14. Loopback: connecting tx_serial to rx_serial with matching divisor settings transfers bytes correctly. This is the simplest way to build a testbench for this block.

Divisor math, worked example: at 50 MHz with cfg_divisor = 26, one bit period is 16 * 27 = 432 clocks, giving 50,000,000 / 432 = 115740.7 baud, which is 0.47 percent above 115200. Program 26 for 115200 baud at 50 MHz.

## 6. Timing requirements

* Single clock domain. tx_serial changes only on clock edges.
* rx_serial is asynchronous to clk in real systems. This spec assumes the user synchronizes it before it reaches this block. The receiver samples it on clk edges.
* The minimum legal bit period is 32 clocks (divisor = 1). Divisor 0 is illegal.

## 7. Register map

None. This block has no memory mapped registers. All configuration uses the dedicated signals in the interface table.

## 8. Verification objectives

1. Prove the transmit frame format on tx_serial: low start bit, 8 data bits LSB first, high stop bit, each with the correct width in clocks.
2. Prove each bit lasts exactly 16 * (cfg_divisor + 1) clock cycles for at least three different divisor values.
3. Prove tx_busy rises with tx_start and falls exactly when the stop bit completes.
4. Prove a tx_start pulse while tx_busy is high is ignored and does not corrupt the ongoing frame.
5. Prove tx_serial idles high between frames.
6. Prove the receiver rejects a start glitch: a low pulse shorter than half a bit period returns the receiver to idle with no rx_done.
7. Prove a clean transmitted frame is received with the correct byte on rx_data and a single cycle rx_done.
8. Prove rx_frame_error is set when the stop bit is forced low, and that rx_data still updates.
9. Prove even parity end to end: a frame transmitted with parity enabled arrives with rx_parity_error low, and flipping one data bit on the wire sets rx_parity_error.
10. Prove odd parity behaves the same way with the opposite parity sense.
11. Prove both error flags hold their values until the next frame completes and then update.
12. Prove a serial loopback (tx_serial tied to rx_serial) transfers 256 distinct byte values with no errors at the default divisor.
