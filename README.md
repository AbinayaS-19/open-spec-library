# Open Spec Library

Hi, I am Abinaya Senthil. I am a design verification engineer, and I started this library to fix a small but real problem.

## What this is

This is a free library of hardware specification documents written for verification practice. Every spec describes a small hardware block completely: its interface, its exact behavior, its corner cases, and a list of verification objectives you can check your work against.

It started from a real classroom moment. I was giving a guest lecture on AI in verification when a professor asked a simple question: where do students get specs to practice on? There was no good answer. Datasheets for real chips are long, often under NDA, and rarely written with a student in mind. So I wrote the specs I wished existed.

## Who it is for

* Students learning digital design verification who want realistic practice material
* Engineers preparing for DV roles who want to sharpen their SystemVerilog and assertion skills
* Professors and mentors who need homework or lab material with clear pass and fail criteria

## How to use it

1. Pick a spec from the table below. Start with a Beginner one if this is new to you.
2. Write a SystemVerilog testbench and a set of assertions against the spec, treating it as the only source of truth.
3. Check your work against the verification objectives at the end of each spec. If you can prove every objective, your testbench is in good shape.

Each spec is self contained. You do not need the others to use any one of them.

## The specs

| ID | Block | Difficulty | What you practice |
|----|-------|-----------|-------------------|
| SPEC-001 | Synchronous FIFO | Beginner | Counters, flags, full and empty corner cases, pipelined reads |
| SPEC-002 | Round robin arbiter | Beginner | Arbitration, fairness, one hot grants, rotation pointers |
| SPEC-003 | UART | Intermediate | Serial timing, oversampling, baud rate math, error flags |
| SPEC-004 | AXI4-Lite slave | Intermediate | Protocol handshakes, byte strobes, error responses, independent channels |
| SPEC-005 | Countdown timer | Beginner | Prescalers, one shot vs periodic modes, interrupt pulses, load timing |

## License

This library is released under Creative Commons Attribution 4.0 (CC-BY-4.0). You are free to share and adapt the material for any purpose, including commercial use, as long as you give credit.

## Contributing

New specs are welcome. To keep the library consistent, every spec must include:

* A header block with ID, version, author, and date
* A parameter table and a full interface table (signal, width, direction, description)
* A functional description precise enough to verify against, including reset behavior and corner cases
* A verification objectives section with 8 to 12 provable objectives

I review every submission for accuracy before it merges. A spec with a wrong detail is worse than no spec, so please check your numbers twice.

More about me: https://abinayas-19.github.io/
