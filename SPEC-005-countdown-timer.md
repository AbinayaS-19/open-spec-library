# SPEC-005: Countdown Timer

| Field | Value |
|-------|-------|
| Spec ID | SPEC-005 |
| Title | Countdown timer |
| Version | 0.1 (Draft) |
| Author | Abinaya Senthil |
| Date | 2026-10-10 |
| Status | Draft for review |

## 1. Overview

A 32 bit countdown timer with a prescaler, one shot and periodic modes, and a single cycle interrupt pulse. Timers like this are in every microcontroller, which makes this a friendly block with real world behavior: load a value, count down, get an interrupt.

## 2. Features

* 32 bit down counter with a readable count value
* Programmable prescaler dividing the clock by 1 to 256
* One shot mode: counts down once and stops
* Periodic mode: reloads and repeats without stopping
* Single cycle interrupt pulse at zero
* Pause by deasserting enable, resume from the held value
* New value can be loaded at any time

## 3. Parameters

| Name | Default | Description |
|------|---------|-------------|
| WIDTH | 32 | Counter width in bits |

## 4. Interface

All signals are synchronous to the rising edge of clk. Reset is synchronous and active low.

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| clk | 1 | input | Clock |
| rst_n | 1 | input | Synchronous reset, active low |
| load_en | 1 | input | Single cycle pulse. Loads load_value (see functional description) |
| load_value | 32 | input | Value to load. Sampled on the rising edge where load_en is high |
| enable | 1 | input | High: counter runs. Low: counter pauses |
| mode | 1 | input | 0 = one shot, 1 = periodic. Sampled on load and at each zero event |
| prescale | 8 | input | Clock divider. The counter decrements once every (prescale + 1) clock cycles |
| irq | 1 | output | Interrupt. Pulses high for exactly one cycle when the counter reaches zero |
| count | 32 | output | Current counter value, updated every clock |

## 5. Functional description

1. Reset: at a rising edge with rst_n low, count clears to 0, irq clears to 0, the internal run latch clears, the reload register clears to 0, and the prescaler counter clears.
2. Prescaler: the counter decrements once every (prescale + 1) clk cycles. prescale = 0 means one decrement per clock, prescale = 255 means one decrement every 256 clocks. Changing prescale restarts the internal prescaler count.
3. Run latch: load_en sets the internal run latch to the current value of enable. The one shot zero event clears it (rule 7). Reset clears it. Driving enable low pauses counting but does not clear the latch, so counting resumes when enable goes high again.
4. Running: the counter decrements by 1 on each prescaled tick where enable is high and the run latch is set.
5. Pause: when enable is low, count holds its value and irq is forced low. Counting resumes from the held value when enable goes high again.
6. Zero event: the terminal event is the decrement that takes count from 1 to 0. On that tick, irq pulses high for exactly one clock cycle. Sitting at zero does not retrigger irq.
7. One shot mode (mode = 0): after the zero event, the run latch clears and the timer stops with count at 0. A later load_en pulse re-arms the timer: if enable is high, counting restarts from the loaded value.
8. Periodic mode (mode = 1): after the zero event, count reloads from the reload register on the next prescaled tick and counting continues without stopping. The run latch stays set.
9. Load: a load_en pulse captures load_value into the reload register and loads it into count immediately, except while the timer is running in periodic mode. In that case count is left alone and the new value takes effect at the next reload, so the current countdown is not disturbed.
10. Mode sampling: mode is sampled on load_en and at each zero event. Changing mode in the middle of a countdown takes effect at the next zero event.
11. Loading 0 arms the timer at zero. No zero event fires until a nonzero value counts down to zero.
12. count output reflects the registered counter value on every cycle.

## 6. Timing requirements

* Single clock domain. All control inputs are sampled on the rising edge of clk.
* irq is high for exactly one clk cycle per zero event.
* count updates on the rising edge.

## 7. Register map

None. This block has no memory mapped registers. All control uses the dedicated signals in the interface table.

## 8. Verification objectives

1. Prove the prescaler: with prescale = N, the count decrements exactly once every N + 1 clock cycles.
2. Prove prescale = 0 decrements the count every clock cycle.
3. Prove load_en loads load_value into count on the next rising edge.
4. Prove one shot mode: after the zero event, irq pulses for exactly one cycle, counting stops, and count stays 0.
5. Prove one shot re-arm: a second load_en restarts the countdown without touching reset.
6. Prove periodic mode: the timer reloads from the reload register and irq pulses on every terminal count, with no gap in the reload.
7. Prove a load while running in periodic mode does not disturb the current countdown and takes effect at the next reload.
8. Prove irq is exactly one cycle wide even with enable held high through the zero event.
9. Prove pausing: driving enable low holds the count and forces irq low, and reasserting enable resumes from the held value.
10. Prove loading 0 produces no irq and the timer sits at zero.
11. Prove reset clears count and irq, clears the run latch, and stops the timer.
12. Prove the count output matches the internal counter value cycle by cycle under randomized load, enable, and prescale changes.
