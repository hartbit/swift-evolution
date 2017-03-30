# Arbitrary precision integer and float literal procotols

* Proposal: [SE-XXXX](XXXX-arbitrary-precision-integer-float-literal-protocols.md)
* Authors: [David Hart](https://github.com/hartbit)
* Review Manager: *TBD*
* Status: *TBD*

## Introduction

We propose a pair of protocols extending `ExpressibleByIntegerLiteral` and `ExpressibleByFloatLiteral` to allow initializing conforming types with arbitrary precision integer and floating-point literals.

## Motivation

The current protocols `ExpressibleByIntegerLiteral` and `ExpressibleByFloatLiteral` are simple and work well but don't support arbitrary precision literal values. Replacing those protocols is a non-goal as they provide a simple interface for work well for most cases. Therefore, to support initializing big integer and big float types with literal values, we need new protocols.

## Proposed solution

We propose:

* renaming `FloatingPointSign` into `Sign`,
* introducing a `ExpressibleByArbitraryPrecisionIntegerLiteral` protocol that extends `ExpressibleByIntegerLiteral` and provides a new arbitrary precision initializer,
* introducing a `ExpressibleByArbitraryPrecisionFloatLiteral` protocol that extends `ExpressibleByFloatLiteral` and provides a new arbitrary precision initializer.

Here is the corresponding code:

```swift
/// The sign of a number.
public enum Sign : Int {
    
    /// The sign for a positive value.
    case plus
    
    /// The sign for a negative value.
    case minus
}

/// A type that can be initialized with an arbitrary precision integer literal.
public protocol ExpressibleByArbitraryPrecisionIntegerLiteral:
    ExpressibleByIntegerLiteral {
    
    /// Creates an instance initialized with the sign and memory value of the
    /// arbitrary precision integer literal.
    ///
    /// Do not call this initializer directly. Instead, initialize a variable or
    /// constant using an arbitrary precision integer literal. For example:
    ///
    ///     let x = 123_456_789_123_456_789_123_456_789
    ///
    /// In this example, the assignment to the `x` constant calls this integer
    /// literal initializer behind the scenes with `sign` as `.plus` and
	/// `buffer` as the memory [0x0000000000661EFD, 0xF2E3B19F7C045F15].
    ///
    /// - Parameters:
    ///     sign: The sign of the integer value.
    ///     buffer: The memory value of the integer.
    init(sign: Sign, buffer: UnsafeBufferPointer<UInt64>)
}

/// A type that can be initialized with an arbitrary precision float literal.
public protocol ExpressibleByArbitraryPrecisionFloatLiteral:
    ExpressibleByFloatLiteral {
    
    /// Creates an instance initialized with the sign, exponent and
    /// significand of the arbitrary precision float literal.
    ///
    /// Do not call this initializer directly. Instead, initialize a variable or
    /// constant using an arbitrary precision integer literal. For example:
    ///
    ///     let x = 123_456_789_123_456.789_123_456_789
    ///
    /// In this example, the assignment to the `x` constant calls this float
    /// literal initializer behind the scenes with `sign` as `.plus`, `exponent`
    /// as the memory [0x0000000000661EFD, 0xF2E3B19F7C045F15] and `significand`
    /// with value [0x000000000000000C].
    ///
    /// - Parameters:
    ///     sign: The sign of the integer value.
    ///     exponent: The memory value of the exponent.
    ///     significand: The memory value of the significand.
    init(
        sign: Sign,
        exponent: UnsafeBufferPointer<UInt64>,
        significand: UnsafeBufferPointer<UInt64>)
}
```