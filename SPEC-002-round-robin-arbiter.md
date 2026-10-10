# SPEC-002: Round Robin Arbiter

| Field | Value |
|-------|-------|
| Spec ID | SPEC-002 |
| Title | Round robin arbiter |
| Version | 0.1 (Draft) |
| Author | Abinaya Senthil |
| Date | 2026-10-10 |
| Status | Done |

## 1. Overview

A round robin arbiter that grants shared resource access to NUM_REQ requesters in rotating order. It is used wherever several requesters share one bus, one memory port, or one pipeline stage, and where no requester may be starved by the others.

## 2. Features

* Configurable number of requesters (NUM_REQ >= 2)
* One hot grant output
* Fair rotation: grants move in circular order
* A grant holds while the requester holds its request
* Parks on the last grant when idle, no default winner

## 3. Parameters

| Name | Default | Description |
|------|---------|-------------|
| NUM_REQ | 4 | Number of request inputs. Must be 2 or more |

## 4. Interface

All signals are synchronous to the rising edge of clk. Reset is synchronous and active low.

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| clk | 1 | input | Clock |
| rst_n | 1 | input | Synchronous reset, active low |
| req | NUM_REQ | input | Request vector, one bit per requester. Level sensitive |
| gnt | NUM_REQ | output | Grant vector. One hot when any request is active, all zero when idle |

## 5. Functional description

1. Reset: at a rising edge with rst_n low, gnt clears to all zero and the rotation pointer sets to 0.
2. The rotation pointer always indicates the requester with the highest priority for the next grant. After reset it is 0.
3. If no grant is currently held and any req bit is high, gnt is issued to the requesting input found by scanning upward from the pointer, wrapping from NUM_REQ - 1 back to 0. Exactly one gnt bit is high.
4. When a grant is issued to input i, the pointer updates to (i + 1) mod NUM_REQ.
5. Grant hold: while the granted input keeps its req high, gnt stays on that input, even if other inputs assert req.
6. Release: on the first rising edge where the granted input's req is low, the arbiter scans from the pointer and grants the next requesting input in rotation order. If no input is requesting, gnt goes to all zero and the pointer holds its value.
7. Idle parking: with no requests, gnt is all zero and the pointer keeps the value it had after the last grant. There is no default grant to input 0.
8. New requests while idle: the scan starts from the parked pointer, so the winner depends on rotation history, not on fixed priority.
9. Fairness: provided every granted requester eventually drops its request, a continuously requesting input is never starved. It is granted at least once every NUM_REQ grant events.
10. req is level sensitive and sampled on the rising edge. gnt updates on the rising edge.

## 6. Timing requirements

* All inputs are sampled on the rising edge of clk. gnt updates on the rising edge.
* No combinational paths from req to gnt are required. A fully registered implementation meets this spec.

## 7. Register map

None. This block is controlled through the dedicated signals in the interface table.

## 8. Verification objectives

1. Prove gnt is never more than one hot: at most one bit is high in any cycle.
2. Prove gnt is all zero whenever req is all zero.
3. Prove a grant is never issued to an input whose req is low.
4. Prove a grant holds while its req stays high, even with all other req bits high.
5. Prove rotation order: with all req bits high and each grant released after one cycle, the grants visit the inputs in circular order 0, 1, 2, 3, 0, and so on.
6. Prove that after reset, the first grant with all req bits high goes to input 0.
7. Prove there is no default winner: after a grant to input 1 is released, simultaneous requests from inputs 0 and 2 grant input 2, because the scan resumes from the parked pointer.
8. Prove pointer hold while idle: after a grant is released with no other requests pending, gnt stays all zero and the next arbitration still starts from the parked pointer.
9. Prove the fairness bound: with all four req bits continuously high and each grant held exactly one cycle, each input is granted exactly once every four grants.
10. Prove no starvation: a continuously requesting input is granted within NUM_REQ grant events after the current grant releases.
11. Prove gnt changes only on clock edges, with no glitches between edges, under randomized req toggling.
