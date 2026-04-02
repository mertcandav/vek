# vek ISA Reference

This document is the complete technical reference for its Instruction Set Architecture (ISA).

Every language that will be compiled into the vek bytecode that conforms to this ISA.\
Some specific content, such as the VM's `ECALL` definitions, may be specific to the reference vek VM implementation.\
The ISA defines only the standard.

## Table of Contents

1. [Overview](#overview)
2. [Binary Representation](#binary-representation)
3. [Entry Point](#entry-point)
4. [Stack and Indexes](#stack-and-indexes)
5. [Registers](#registers)
6. [Data Types](#data-types)
   - [Integers](#integers)
   - [Floating-Point](#floating-point)
7. [Stack Frames and Calls](#stack-frames-and-calls)
8. [Instruction Reference](#instruction-reference)
   - [Control Flow](#control-flow)
   - [Stack Management](#stack-management)
   - [Data Movement](#data-movement)
   - [Arithmetic - Register-to-Register](#arithmetic---register-to-register)
   - [Arithmetic - Immediate Integer](#arithmetic---immediate-integer)
   - [Arithmetic - Immediate Float](#arithmetic---immediate-float)
   - [Arithmetic - Special](#arithmetic---special)
9. [Notation Guide](#notation-guide)

## Overview

The vek ISA is purpose-built for numerical computing. While it adopts a **register-based** architecture, it diverges substantially from conventional hardware ISAs in how registers are managed.

The key design principle is that the **stack acts as a register map**. Every memory slot on the stack is addressable as a general-purpose register. Predefined (reserved) registers are also mapped onto the stack, occupying fixed indices. This approach allows direct data manipulation on the stack while minimizing overhead - memory offsets serve as register indices, eliminating the need for a separate register file.

Developers do not need familiarity with physical hardware ISAs or assembly to work with vek. The instruction set is intentionally simplified and focused on mathematical computation.

## Binary Representation

The binary format is **byte-aligned**. Every opcode, operand, and data encoding is serialized sequentially, byte by byte, with each byte being a standard 8-bit octet.

All structures (opcodes, immediates, encoded integers, floating-point values) must be placed contiguously in the binary stream with no padding between them unless explicitly required by an instruction's encoding format.

## Entry Point

The first **9 bytes** of a valid vek binary encode the entry point as a `JMP` instruction:

```
[1-byte JMP opcode] [8-byte PC address]
```

When the virtual machine initializes, it decodes this sequence and sets the Program Counter (PC) to the target address. Execution begins from that address.

**Structural convention:** The entry point routine must be the **last executable routine** in the binary. All subroutines (if any) are encoded before it. Once the entry point routine completes, the program terminates.

```
[subroutine A] [subroutine B] ... [entry point routine]
^                                  ^
|__ encoded first                  |__ encoded last, pointed to by the 9-byte header
```

## Stack and Indexes

The stack is a **FIFO-managed** (First-In, First-Out) memory region. It is the primary working memory for all computations.

- Indexing is **zero-based**: index `0` refers to the first (oldest) element pushed.
- Indices are always **non-negative integers**.
- As the index value increases, it refers to memory segments allocated later.
- Every stack slot is a **generic container** capable of storing heterogeneous data types - primitives, structured values, or any data transported during execution.

> **Note:** Within a stack frame, the index space is local to that frame. See [Stack Frames and Calls](#stack-frames-and-calls) for details.

## Registers

The ISA defines a set of **reserved registers**, each mapped to a fixed index on the stack. These registers are initialized and placed on the stack before any instruction executes.

Unless otherwise noted, all registers are **ready for use immediately** but hold an **uninitialized (indeterminate) value** until explicitly written. Always assign a value before reading.

| Register | Description |
|----------|-------------|
| `RV` | **Return Value.** A multi-purpose register used to store the final result of a computation or intermediate values. Serves as the primary destination for return values and temporary data during instruction execution. |

## Data Types

The ISA has native support for two numerical types: **integers** and **floating-point** values.

### Integers

Integers are treated as **arbitrary-precision** values (big integers). Most operations produce integer results by default. Some instructions (such as division between mixed types) may promote the result to floating-point.

#### Binary Encoding

Integers use a variable-length encoding scheme:

```
[BYTE-LEN : index-encoded] [SIGN : 1 byte] [BYTES... : absolute value]
```

| Field | Description |
|-------|-------------|
| `BYTE-LEN` | The byte-length of the absolute value, encoded as an index (see [Notation Guide](#notation-guide)). |
| `SIGN` | `0` = positive, `1` = negative. |
| `BYTES...` | The raw byte representation of the absolute value. |

### Floating-Point

Floating-point values are **64-bit double-precision**, encoded according to the **IEEE 754** standard. All arithmetic is performed in strict IEEE 754 compliance.

- Subject to overflow conditions inherent to 64-bit doubles.
- Always occupy a **fixed 8-byte** footprint in the binary.

#### Binary Encoding

```
[BYTE0] [BYTE1] [BYTE2] [BYTE3] [BYTE4] [BYTE5] [BYTE6] [BYTE7]
```

The 8 bytes are the raw bit-level representation of the 64-bit double, serialized byte by byte in order.

## Stack Frames and Calls

The ISA has **native support for function calls** through stack frames. Each function call operates in its own isolated stack frame.

- The program's **entry point** also runs inside a primary stack frame.
- A new frame is created with the `ASF` (Allocate Stack Frame) instruction.
- The `ASF` instruction accepts an **offset** parameter that defines how many recently pushed stack slots from the *preceding* frame are inherited (i.e., made accessible) in the new frame. This is the mechanism for passing arguments to callees.
- Each frame **preserves the return address** (the caller's PC). When a `RET` instruction is encountered, execution resumes from the saved return address.
- Within a frame, the stack index is **reset** to the local scope. Stack operations within a function address local memory, plus any inherited slots defined by `ASF`.

**Call sequence:**

```
ASF [offset]    ; allocate new frame, inherit recently pushed 'offset' slots from caller
CALL [pc]       ; jump to callee, save return address
...             ; callee body
RET             ; restore caller's stack frame and resume
```

External functions follow the same protocol and must also execute `RET`.

## Instruction Reference

### Notation Guide

The following notation is used throughout this reference:

| Notation | Meaning |
|----------|---------|
| `[INDEX]` | A stack slot index (zero-based, non-negative integer), encoded as a variable-length index value. |
| `[PC]` | An 8-byte program counter address. |
| `[AS-INDEX]` | Alias for `[INDEX]` emphasizes the value is encoded as an index. |
| `[SIGN]` | A 1-byte sign indicator: `0` = positive, `1` = negative. |
| `[BYTE-LEN;INDEX]` | The byte-length of an integer's absolute value, encoded as an index. |
| `[BYTES...]` | A variable number of raw bytes constituting an integer's absolute value. |
| `[BYTE]` | A single raw byte. Eight consecutive `[BYTE]` fields denote an IEEE 754 64-bit float. |
| `[SOURCE;INDEX]` | A stack index used as the source operand. |
| `[DESTINATION;INDEX]` | A stack index used as the destination operand (modified in-place). |
| `[VALUE;INDEX]` | A stack index used as a value operand. |

### Control Flow

#### `NOP`

Performs no operation.

Used as a special-purpose marker, most notably as the **mandatory terminator** for integer encoding sequences.

```
NOP
```

#### `JMP`

Unconditionally branches to the specified PC address. The current PC is replaced with the target. Does **not** affect the stack frame.

```
JMP [PC;AS-INDEX]
```

| Operand | Description |
|---------|-------------|
| `[PC;AS-INDEX]` | The target program counter address, encoded as an index. |

#### `CALL`

Must be executed **immediately after** an `ASF` instruction. Jumps to the target PC and saves the return address in the current frame's metadata. The return address is later used by `RET` to resume execution.

```
CALL [PC;AS-INDEX]
```

| Operand | Description |
|---------|-------------|
| `[PC;AS-INDEX]` | The target program counter address of the callee. |

#### `ECALL`

Invokes an **external function** registered with the VM, a function defined outside the binary. Behaves identically to `CALL` in terms of stack frame and return address semantics. The external function must execute `RET` to restore the caller's context.

```
ECALL [INDEX]
```

| Operand | Description |
|---------|-------------|
| `[INDEX]` | The resolution index of the external function in the VM's extern table. |

#### `RET`

Terminates the current stack frame and restores the preceding execution context. Uses the return address stored in the active frame's metadata to jump back to the caller's next instruction.

Every function **must** execute `RET` to ensure stack consistency.

```
RET
```

### Stack Management

#### `ASF`

Allocates a new stack frame and shifts execution scope into it. The `offset` parameter specifies how many stack slots from the preceding frame are accessible in the new frame (argument passing).

Must always be paired with a subsequent `CALL` or `ICALL` instruction.

```
ASF [OFFSET;AS-INDEX]
```

| Operand | Description |
|---------|-------------|
| `[OFFSET;AS-INDEX]` | Number of stack slots inherited from the preceding frame. |

#### `SETSP`

Explicitly sets the stack pointer to the specified index. The stack pointer must always point to the **next available** memory location for insertion. Use this instruction to directly control stack growth.

```
SETSP [INDEX]
```

| Operand | Description |
|---------|-------------|
| `[INDEX]` | The index the stack pointer will be set to. |

### Data Movement

#### `AIN`

Pushes an arbitrary-precision integer onto the stack.

```
AIN [BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

| Operand | Description |
|---------|-------------|
| `[BYTE-LEN;INDEX]` | Byte-length of the absolute value. |
| `[SIGN]` | `0` = positive, `1` = negative. |
| `[BYTES...]` | Raw bytes of the absolute value. |

#### `AFL`

Pushes a 64-bit IEEE 754 floating-point value onto the stack.

```
AFL [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

The 8 bytes are the bitwise representation of the double-precision float.

#### `AMOV`

Pushes a **copy** of the value at the specified index onto the stack.

```
AMOV [INDEX]
```

| Operand | Description |
|---------|-------------|
| `[INDEX]` | Source index whose value is copied and pushed. |

#### `MOV`

Copies the value from a source index into a destination index. The source is not modified.

```
MOV [SOURCE;INDEX],[DESTINATION;INDEX]
```

| Operand | Description |
|---------|-------------|
| `[SOURCE;INDEX]` | Index of the value to copy. |
| `[DESTINATION;INDEX]` | Index where the value is written. |

#### `MOVIN`

Copies an arbitrary-precision integer directly into the slot at the specified index, overwriting existing data.

```
MOVIN [INDEX],[BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

| Operand | Description |
|---------|-------------|
| `[INDEX]` | Destination index. |
| `[BYTE-LEN;INDEX]` | Byte-length of the integer's absolute value. |
| `[SIGN]` | `0` = positive, `1` = negative. |
| `[BYTES...]` | Raw bytes of the absolute value. |

#### `MOVFL`

Copies a 64-bit floating-point value directly into the slot at the specified index, overwriting existing data.

```
MOVFL [INDEX],[BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

| Operand | Description |
|---------|-------------|
| `[INDEX]` | Destination index. |
| `[BYTE] × 8` | IEEE 754 bitwise representation of the double. |

#### `NEG`

Inverts the sign of the numerical value at the specified index. Positive becomes negative and vice versa.

```
NEG [INDEX]
```

| Operand | Description |
|---------|-------------|
| `[INDEX]` | Index of the value to negate. |

### Arithmetic - Register-to-Register

All register-to-register arithmetic instructions operate on values identified by stack indices. They support **mixed-mode arithmetic** with automatic type promotion according to the their rules.

#### `ADD`

Adds the value at `[VALUE;INDEX]` to the value at `[DESTINATION;INDEX]`. Result is stored in the destination.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
ADD [DESTINATION;INDEX],[VALUE;INDEX]
```

#### `SUB`

Subtracts the value at `[SOURCE;INDEX]` from the value at `[DESTINATION;INDEX]`. Result is stored in the destination.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
SUB [DESTINATION;INDEX],[SOURCE;INDEX]
```

#### `MUL`

Multiplies the value at `[DESTINATION;INDEX]` by the value at `[VALUE;INDEX]`. Result is stored in the destination.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
MUL [DESTINATION;INDEX],[VALUE;INDEX]
```

#### `QUO`

Divides the value at `[DESTINATION;INDEX]` by the value at `[SOURCE;INDEX]` using **truncating division (T-division)**. Result is stored in the destination.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation (fractional part truncated toward zero) |
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
QUO [DESTINATION;INDEX],[SOURCE;INDEX]
```

#### `REM`

Calculates the remainder of `[DESTINATION;INDEX]` divided by `[VALUE;INDEX]` using **truncating division (T-division)**. Result is stored in the destination.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
REM [DESTINATION;INDEX],[VALUE;INDEX]
```

### Arithmetic - Immediate Integer

These instructions apply an arithmetic operation using a literal integer constant encoded directly in the instruction stream. They bypass register lookups for improved performance when working with known constants.

The immediate integer encoding follows the same format as `AIN`.

#### `ADDIN`

Adds an immediate integer to the destination index.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
ADDIN [DESTINATION;INDEX],[BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

#### `SUBIN`

Subtracts an immediate integer from the destination index.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
SUBIN [DESTINATION;INDEX],[BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

#### `MULIN`

Multiplies the destination index by an immediate integer.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
MULIN [DESTINATION;INDEX],[BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

#### `QUOIN`

Divides the destination index by an immediate integer using **T-division**.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation (fractional part truncated toward zero) |
| Float | Integer | Float | Integer is promoted to float before operation |

```
QUOIN [DESTINATION;INDEX],[BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

#### `REMIN`

Calculates the remainder of the destination index divided by an immediate integer using **T-division**.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Integer | Integer | Integer | Direct operation |
| Float | Integer | Float | Integer is promoted to float before operation |

```
REMIN [DESTINATION;INDEX],[BYTE-LEN;INDEX] [SIGN] [BYTES...]
```

### Arithmetic - Immediate Float

These instructions apply an arithmetic operation using a literal 64-bit IEEE 754 floating-point constant encoded directly in the instruction stream.

#### `ADDFL`

Adds an immediate float to the destination index.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |

```
ADDFL [DESTINATION;INDEX],[BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

#### `SUBFL`

Subtracts an immediate float from the destination index.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |

```
SUBFL [DESTINATION;INDEX],[BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

#### `MULFL`

Multiplies the destination index by an immediate float.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |

```
MULFL [DESTINATION;INDEX],[BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

#### `QUOFL`

Divides the destination index by an immediate float using **T-division**.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |

```
QUOFL [DESTINATION;INDEX],[BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

#### `REMFL`

Calculates the remainder of the destination index divided by an immediate float using **T-division**.

| Left Operand | Right Operand | Result Type | Behavior |
|---|---|---|---|
| Float | Float | Float | Direct operation |
| Integer | Float | Float | Integer is promoted to float before operation |

```
REMFL [DESTINATION;INDEX],[BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE] [BYTE]
```

### Arithmetic - Special

These instructions implement a specific process whose behavior can vary depending on various conditions.

#### `SHL`

Performs a bitwise left shift on the value stored at the target index.
The operation requires a constant index as an argument, which defines the specific shift count.

If the target value is an integer, the operation shifts the bits of the integer representation accordingly.

If the target value is a float, the operation shifts the underlying bit pattern of the float.

In the event of a numeric or bitwise overflow, the operation does not trap or throw an exception.
Instead, the result is strictly defined to resolve to 0.

```
SHL [DESTINATION;INDEX],[SHIFT;INDEX]
```

#### `SHR`

Performs a bitwise right shift on the value stored at the target index.
The operation requires a constant index as an argument, which defines the specific shift count.

If the target value is an integer, the operation shifts the bits of the integer representation accordingly.

If the target value is a float, the operation shifts the underlying bit pattern of the float.

In the event of a numeric or bitwise overflow, the operation does not trap or throw an exception.
Instead, the result is strictly defined to resolve to 0.

```
SHR [DESTINATION;INDEX],[SHIFT;INDEX]
```

## Quick Reference - Instruction Summary

| Instruction | Opcode | Operands | Description |
|---|---|---|---|
| `NOP` | 0 | N/A | No operation / integer encoding terminator |
| `JMP` | 1 | `[PC]` | Unconditional branch |
| `ASF` | 2 | `[OFFSET]` | Allocate stack frame with inherited slots |
| `CALL` | 3 | `[PC]` | Call subroutine (after ASF) |
| `ECALL` | 4 | `[INDEX]` | Call external function |
| `RET` | 5 | N/A | Return from subroutine |
| `SETSP` | 6 | `[INDEX]` | Set stack pointer |
| `AIN` | 7 | `[LEN] [SIGN] [BYTES...]` | Push integer |
| `AFL` | 8 | `[BYTE×8]` | Push float |
| `AMOV` | 9 | `[INDEX]` | Push copy of value at index |
| `MOV` | 10 | `[SRC],[DST]` | Copy index to index |
| `MOVIN` | 11 | `[DST],[LEN] [SIGN] [BYTES...]` | Write integer into index |
| `MOVFL` | 12 | `[DST],[BYTE×8]` | Write float into index |
| `NEG` | 13 | `[INDEX]` | Negate value |
| `ADD` | 14 | `[DST],[VAL]` | Add register to register |
| `ADDIN` | 15 | `[DST],[INT]` | Add immediate integer |
| `ADDFL` | 16 | `[DST],[FLOAT]` | Add immediate float |
| `SUB` | 17 | `[DST],[SRC]` | Subtract register from register |
| `SUBIN` | 18 | `[DST],[INT]` | Subtract immediate integer |
| `SUBFL` | 19 | `[DST],[FLOAT]` | Subtract immediate float |
| `MUL` | 20 | `[DST],[VAL]` | Multiply register by register |
| `MULIN` | 21 | `[DST],[INT]` | Multiply by immediate integer |
| `MULFL` | 22 | `[DST],[FLOAT]` | Multiply by immediate float |
| `QUO` | 23 | `[DST],[SRC]` | Divide register by register (T-div) |
| `QUOIN` | 24 | `[DST],[INT]` | Divide by immediate integer (T-div) |
| `QUOFL` | 25 | `[DST],[FLOAT]` | Divide by immediate float (T-div) |
| `REM` | 26 | `[DST],[VAL]` | Remainder of division (T-div) |
| `REMIN` | 27 | `[DST],[INT]` | Remainder by immediate integer (T-div) |
| `REMFL` | 28 | `[DST],[FLOAT]` | Remainder by immediate float (T-div) |
| `SHL` | 29 | `[DST],[INT]` | Bitwise left shift |
| `SHR` | 30 | `[DST],[INT]` | Bitwise right shift |