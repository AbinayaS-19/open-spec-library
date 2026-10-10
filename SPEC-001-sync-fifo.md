# SPEC-001: Synchronous FIFO

| Field | Value |
|-------|-------|
| Spec ID | SPEC-001 |
| Title | Synchronous FIFO |
| Version | 0.1 (Draft) |
| Author | Abinaya Senthil |
| Date | 2026-10-10 |
| Status | Done|

## 1. Overview

A synchronous first in first out buffer with a single clock for reads and writes. FIFOs like this sit between producer and consumer logic running on the same clock, for example between a DMA engine and a packet formatter, smoothing out bursts and rate differences.

## 2. Features

* Single clock domain, fully synchronous
* Configurable data width and depth (depth must be a power of 2)
* Pipelined read: data appears one cycle after rd_en
* Programmable almost full and almost empty thresholds
* Write ignored when full, read ignored when empty, no data loss or corruption

## 3. Parameters

| Name | Default | Description |
|------|---------|-------------|
| WIDTH | 32 | Data width in bits |
| DEPTH | 16 | Number of entries. Must be a power of 2 |
| AF_THRESH | 14 | almost_full asserts when count >= AF_THRESH. Default is DEPTH - 2 |
| AE_THRESH | 1 | almost_empty asserts when count <= AE_THRESH |

## 4. Interface

All signals are synchronous to the rising edge of clk. Reset is synchronous and active low.

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| clk | 1 | input | Clock |
| rst_n | 1 | input | Synchronous reset, active low |
| wr_en | 1 | input | Write enable, sampled on the rising edge |
| wr_data | WIDTH | input | Write data, sampled when wr_en is high and the FIFO is not full |
| rd_en | 1 | input | Read enable, sampled on the rising edge |
| rd_data | WIDTH | output | Read data, valid one cycle after rd_en |
| full | 1 | output | High when count == DEPTH |
| empty | 1 | output | High when count == 0 |
| almost_full | 1 | output | High when count >= AF_THRESH |
| almost_empty | 1 | output | High when count <= AE_THRESH |
| data_count | $clog2(DEPTH+1) | output | Number of entries stored. 5 bits when DEPTH is 16 |

## 5. Functional description

1. Reset: when rst_n is low at a rising clock edge, the stored count clears to 0, empty asserts, full and almost_full deassert, and almost_empty asserts (because 0 <= AE_THRESH).
2. Write: on a rising edge, if wr_en is high and full is low, wr_data is stored and the count increments by 1.
3. Write ignored: if wr_en is high while full is high, the write is dropped and the count is unchanged.
4. Read: on a rising edge, if rd_en is high and empty is low, the oldest entry is scheduled for output and the count decrements by 1. rd_data carries that entry starting at the next rising edge. This is a pipelined read with exactly one cycle of latency, not first word fall through.
5. Read ignored: if rd_en is high while empty is high, the read is dropped and the count is unchanged.
6. Simultaneous read and write: if rd_en and wr_en are both high while the FIFO is neither full nor empty, both succeed, the count is unchanged, and rd_data returns the oldest entry one cycle later.
7. Simultaneous read and write at full: the read succeeds and the write is ignored, because the FIFO was full at the sampling edge. The count goes from DEPTH to DEPTH - 1.
8. Simultaneous read and write at empty: the write succeeds and the read is ignored. The count goes from 0 to 1.
9. Ordering: entries are read out in the order they were written.
10. Flags: full is high when count == DEPTH, empty when count == 0, almost_full when count >= AF_THRESH, almost_empty when count <= AE_THRESH. Flags are computed from the registered count and update on the rising clock edge together with the count.
11. rd_data holds its last value when no read is in progress.
12. DEPTH must be a power of 2. Behavior for any other value is undefined.

## 6. Timing requirements

* Single clock domain. All inputs are sampled on the rising edge of clk and must meet setup and hold.
* Read latency is exactly one cycle: data for a read requested in cycle N is valid on rd_data during cycle N + 1.
* Status flags are valid in the cycle after the count change that caused them.

## 7. Register map

None. This block is controlled through the dedicated signals in the interface table.

## 8. Verification objectives

1. Prove that after a synchronous reset, empty is high, the count is 0, full and almost_full are low, and almost_empty is high.
2. Prove that full asserts exactly when the DEPTHth write completes, and not before.
3. Prove that a write issued while full is dropped and the stored count does not change.
4. Prove that a read issued while empty is dropped and the stored count does not change.
5. Prove first in first out ordering across a full fill and drain sequence using distinct data values.
6. Prove that simultaneous read and write on a partially filled FIFO leaves the count unchanged and returns the oldest entry.
7. Prove the boundary corner cases: read plus write at full (count goes DEPTH to DEPTH - 1, write dropped) and at empty (count goes 0 to 1, read dropped).
8. Prove rd_data is valid exactly one cycle after rd_en and holds its value when no read is in progress.
9. Prove almost_full asserts when the count reaches AF_THRESH and deasserts when the count falls below it.
10. Prove almost_empty asserts when the count reaches AE_THRESH and deasserts when the count rises above it.
11. Prove the count always equals writes minus reads under randomized traffic with random stalls on both interfaces.
12. Prove empty and full are never high at the same time.
