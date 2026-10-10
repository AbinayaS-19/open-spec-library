# SPEC-004: AXI4-Lite Slave

| Field | Value |
|-------|-------|
| Spec ID | SPEC-004 |
| Title | AXI4-Lite slave |
| Version | 0.1 (Draft) |
| Author | Abinaya Senthil |
| Date | 2026-10-10 |
| Status | Draft for review |

## 1. Overview

An AXI4-Lite slave with a small register file. It implements the full five channel VALID/READY handshake and is a good first protocol block: the rules are strict, the state space is small, and every transaction is checkable against this spec.

## 2. Features

* 32 bit data, 12 bit address (4 KB address space)
* Eight read write registers plus one read only ID register
* Full VALID/READY handshake on all five channels
* Byte strobes (WSTRB) on writes
* Independent read and write channels
* SLVERR responses for illegal accesses

## 3. Parameters

| Name | Default | Description |
|------|---------|-------------|
| ADDR_WIDTH | 12 | Address width in bits. Covers 0x000 to 0xFFF |
| DATA_WIDTH | 32 | Data width in bits |

## 4. Interface

All signals are synchronous to the rising edge of clk. Reset is synchronous and active low.

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| clk | 1 | input | Clock |
| rst_n | 1 | input | Synchronous reset, active low |
| awaddr | 12 | input | Write address |
| awvalid | 1 | input | Write address valid |
| awready | 1 | output | Write address ready |
| wdata | 32 | input | Write data |
| wstrb | 4 | input | Write byte strobes, one bit per byte |
| wvalid | 1 | input | Write data valid |
| wready | 1 | output | Write data ready |
| bresp | 2 | output | Write response: 2'b00 OKAY, 2'b10 SLVERR |
| bvalid | 1 | output | Write response valid |
| bready | 1 | input | Write response ready |
| araddr | 12 | input | Read address |
| arvalid | 1 | input | Read address valid |
| arready | 1 | output | Read address ready |
| rdata | 32 | output | Read data |
| rresp | 2 | output | Read response: 2'b00 OKAY, 2'b10 SLVERR |
| rvalid | 1 | output | Read data valid |
| rready | 1 | input | Read data ready |

## 5. Functional description

1. Reset: when rst_n is low at a rising edge, all eight read write registers clear to 0x00000000. The ID register is hardwired to 0x00000001 and is not affected by reset.
2. Handshake rule: a transfer on any channel happens on a rising edge when VALID and READY are both high. The slave may hold READY low for any number of cycles to insert wait states.
3. The write address and write data channels are independent. The AW and W handshakes may complete in either order or in the same cycle.
4. The write response is issued only after both the address handshake and the data handshake have completed. bvalid waits for both. bvalid stays high until the bvalid/bready handshake completes.
5. Legal write addresses are 0x000, 0x004, 0x008, 0x00C, 0x010, 0x014, 0x018, and 0x01C, always with addr[1:0] == 2'b00. A write to the ID register (0x020), to any address from 0x024 to 0xFFF, or to an unaligned address returns SLVERR on bresp. The register file is unchanged by an errored write.
6. Byte strobes: on a successful write, only the bytes with wstrb[i] == 1 are updated. A write with wstrb == 4'b0000 leaves the register unchanged but still returns OKAY.
7. Read: the arvalid/arready handshake latches the address. rdata and rresp are then presented with rvalid and complete on the rvalid/rready handshake. rdata is stable from rvalid assertion until that handshake completes.
8. Legal read addresses are the eight read write registers plus the ID register at 0x020, which returns 0x00000001 with OKAY. Reads to 0x024 to 0xFFF or to unaligned addresses return SLVERR with rdata == 0x00000000.
9. Read and write channels are fully independent. A read and a write may be in flight at the same time, in any order.
10. Back to back transactions are supported on all channels. The slave may deassert its ready outputs between transactions.
11. Response codes: this slave returns only OKAY (2'b00) and SLVERR (2'b10). DECERR and EXOKAY are never used.
12. A read and a write to the same register must not be in flight at the same time. Read data in that case is undefined. The write still completes with the correct strobed value.

## 6. Timing requirements

* Single clock domain. All signals are synchronous to the rising edge of clk.
* The slave must not create a combinational path from an input VALID to its own READY in a way that forms a combinational loop with the master. Registered ready outputs meet this spec.

## 7. Register map

| Offset | Name | Access | Reset value | Description |
|--------|------|--------|-------------|-------------|
| 0x00 | REG0 | R/W | 0x00000000 | General purpose register 0 |
| 0x04 | REG1 | R/W | 0x00000000 | General purpose register 1 |
| 0x08 | REG2 | R/W | 0x00000000 | General purpose register 2 |
| 0x0C | REG3 | R/W | 0x00000000 | General purpose register 3 |
| 0x10 | REG4 | R/W | 0x00000000 | General purpose register 4 |
| 0x14 | REG5 | R/W | 0x00000000 | General purpose register 5 |
| 0x18 | REG6 | R/W | 0x00000000 | General purpose register 6 |
| 0x1C | REG7 | R/W | 0x00000000 | General purpose register 7 |
| 0x20 | ID_REG | R | 0x00000001 | Identification register, hardwired |

## 8. Verification objectives

1. Prove a write to each of the eight registers followed by a read returns the written value.
2. Prove WSTRB granularity: a write with wstrb == 4'b0001 changes only byte 0 while the other three bytes keep their values.
3. Prove a write to the ID register (0x020) returns SLVERR and the ID value still reads 0x00000001.
4. Prove a write to an out of range address (for example 0x100) returns SLVERR and changes no register.
5. Prove a read of an out of range address returns SLVERR with rdata == 0x00000000.
6. Prove bvalid is not asserted until both the AW and W handshakes have completed, checking both orders: address first and data first.
7. Prove read and write channels operate independently: a full write transaction completes correctly while a read transaction is in flight.
8. Prove back to back writes to different registers complete with correct data and no dropped transfers.
9. Prove the slave tolerates wait states: randomly deasserting ready never loses or duplicates a transfer on any channel.
10. Prove reset clears all eight registers to zero while the ID register still reads 0x00000001.
11. Prove an unaligned write address (for example 0x001) returns SLVERR.
12. Prove rdata is stable from rvalid assertion until the rvalid/rready handshake completes.
