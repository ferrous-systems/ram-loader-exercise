# `ram-loader-exercise`

The goal of this exercise is to implement a "RAM loader", firmware that receives a program (new
firmware), loads it to memory (RAM) and then executes it.

This repository consists of 3 main packages:

- `ramloader`, this is the "RAM loader" firmware
- `elfloader`, this is a host-side tool that sends programs to the target (the nRF52840 microcontroller)
- `app`, a firmware project configured to produce binaries with an 'only RAM' memory layout (all
  linker sections are placed in RAM)

## Existing code

### `elfloader`

The `elfloader` tool is provided in a working state. This tool exchanges messages, encoded using the
`postcard` library, with a device running the `ramloader` firmware.

This command-line tools take a single argument: a path to an ELF file. This ELF file is a compiled
Rust program (Cargo places this artifact in the `target` directory). When executed, the tool will
send the 'loadable sections' of the ELF as messages to the target.

The `elfloader` can send 2 types of `Host2TargetMessages` to the target:

- `Write { data, start_address }`, this message instructs the target to write the given `data`
  (bytes) at the specified `start_address`.
- `Execute`, this message instructs the target to execute the previously written program

`elfloader` expects the target to respond to the `Write` message with a `Target2HostMessage`.

### `ramloader`

The firmware contains starter code that receives and decodes `postcard` messages sent by `elfloader`
but it's missing the handling of those messages: the starter code will panic when a message is
received.

### `elf`

This project contains a few examples. These are covered in more detail in the "Test ELFs" section.

## Memory layout

``` text
+-----------+ 0x2004_0000
|           |   (stack)
|    APP    |   (static variables)
|           |   (constants)
|           |   (functions)
+-----------+ 0x2002_0000
|           |   (stack)
| RAMLOADER |
|           |   (static variables)
+-----------+ 0x2000_0000
```

The `ramloader` firmware is configured to place functions and constants in Flash, and static
variables and the call stack in RAM -- the usual memory layout provided by `cortex-m-rt`. But it's
configured to not use the entire RAM available but only half of it (128 KiB). See `RAM.LENGTH` in
`ramloader/memory.x`.

The `app` project has been configured to produce binaries that are fully contained in RAM.
See `memory.x`, the `FLASH` region is placed in the RAM addresss space.

## What needs to be implemented

`handle_request` needs to be implemented as follows

- `Write { data, start_address }` variant
  - verify that the requested address is valid (e.g. it's RAM and doesn't overlap with `ramloader`
    memory)
  - write the given `data` into `start_address`

- `Execute` variant: launch the program that was previously written to RAM

### Test ELFs

The following pre-compiled ELFs are provided in the `app/precompiled_elfs` folder. Source code is
available in the `app/examples` directory.

The ELFs exercise different parts of the `ramloader` and it's suggested to try them in the following
order.

1. `led`, simplest program that turns on an LED
2. `static-variables`, turns an LED if static variables are correctly initialized
3. `interrupts`, blinks an LED using interrupts

The `led-bad-address` ELF uses the normal Flash+RAM memory layout so it should be rejected by the
`ramloader`. This ELF can be produced by modifying `app/memory.x` to locate `FLASH` at address
`0x0000_0000`.
