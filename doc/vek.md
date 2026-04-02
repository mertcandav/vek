# vek Reference

This document is the complete reference for the Reference vek language implementation.

## Table of Contents

1. [Overview](#overview)
2. [Data Types](#data-types)
3. [Operators](#operators)
4. [Keywords](#keywords)
5. [Identifiers](#identifiers)
6. [Functions](#functions)

## Overview

## Data Types

| Type     | Typical Bit Width  | Typical Value                                                       |
|----------|--------------------|---------------------------------------------------------------------|
| integer  | arbitrary-precise  | theoretically unlimited                                             |
| float    | 8 bytes            | -1.797693134862315708e+308 to 1.797693134862315708e+308             |

### Integer Literals

#### Decimal

Decimal expressions are literals that do not contain any special prefixes (such as `0x` or `0`).
```asm
12345
```

#### Binary
Binary literals are literals that begin with `0b`. Only the digits [0, 1] can be used in these literals.
```asm
0b0001010101
```

#### Octal
Octal literals are literals that begin with `0o` or `0`. Only the digits [0, 7] can be used in these literals. Using `0` alone is interpreted as a decimal number.
```asm
0455
0o455
```

#### Hexadecimal
Hexadecimal literals are literals that begin with `0x`. Only the digits [0, 9] and the letters [A, F] can be used in these literals.
```asm
0xDFF90
```

### Float Literals
```asm
3.14
32.60
032.60
3.
.3
0.3
1E2
.12345E+6
1.e+0
0x1p-2
0x2.p10
0x1.Fp+0
0X.8p-0
0x1fffp-16
```

## Operators

### Arithmetic Operators
Arithmetic operators are used to perform common mathematical operations.

| Operator | ISA Instruction | Description | Supported Type(s) |
| -------- | --------------- | ----------- | ----------------- |
| `+` | `ADD`, `ADDIN`, `ADDFL` | Addition | integers, floats |
| `-` | `SUB`, `SUBIN`, `SUBFL` | Subtraction | integers, floats |
| `*` | `MUL`, `MULIN`, `MULFL` | Multiplication | integer, floats |
| `/` | `QUO`, `QUOIN`, `QUFL` | Division | integer, floats |
| `%` | `REM`, `REMIN`, `RELFL` | Modulus | integers, floats |
| `<<` | `SHL` | Shift left | integers, floats |
| `>>` | `SHR` | Shift rifgt | integers, floats |

### Precedences
| Precedence (Descending) | Operator(s) |
| ----------------------- | ----------- |
| 2 |  `*` `%` `/` `<<`, `>>` |
| 1 | `+` `-` |

## Keywords

| Keyword | Description |
| -------- | ----------- |
| `let` | Declares a function |

## Identifiers

Identifiers are special names provided to use the definitions.

An identifier must comply with some rules:
- Must begin with an underscore or letter.
- Can include numbers (`0-9`) except first character.
- It can only consist of numbers, underscore and letters.

vek allows the use of any letter from the list of Unicode characters it supports.\
So it is possible to write identifiers in many languages.\
A keyword cannot be used as identifier.

## Functions

Functions are defined using the `let` keyword, followed by a unique identifier.\
If the function accepts arguments, they are provided as a comma-separated list of identifiers within parentheses.\
An assignment operator (`=`) then separates the signature from the expression that computes the return value.\
Every function is strictly required to return a value.

Example function declarations:
```
let myConstant() = 1234
```
```
let sum(a, b) = a + b
```