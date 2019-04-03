# Static Custom Attributes

* Proposal: [SE-XXXX](https://github.com/apple/swift-evolution/blob/master/proposals/XXXX-static-custom-attributes.md)
* Authors: [David Hart](https://github.com/hartbit), [Doug Gregor](https://github.com/DougGregor), [Vini Vendramini](http://github.com/vinivendra)
* Review Manager: TBD
* Status: **Pitch only**

## Introduction

Swift currently supports marking declarations with compiler-defined attributes, such as `@objc` and `@deprecated`. The compiler uses these attributes to customize the way it interprets the corresponding code. As a first step towards opening attributes to users and library authors, this proposal introduces static custom attributes which exist only at compile time, allowing tools that parse Swift code to use them as markers.

Previous discussions on the subject:

* [Dec '15](https://forums.swift.org/t/adding-custom-attributes/783)
* [Dec '15](https://forums.swift.org/t/user-defined-attributes-annotations/385)
* [Jan '16](https://forums.swift.org/t/custom-annotation-processors/1050)
* [Jun '16](https://forums.swift.org/t/custom-attribute-in-userland-annotation/2932)
* [Sep '16](https://forums.swift.org/t/java-style-annotations-attributes/4120)
* [Jun '18](https://forums.swift.org/t/custom-attributes/13976)
* [Aug '18](https://forums.swift.org/t/example-of-attributes-with-current-version-of-swift/15662)

## Motivation

Tools like [Sourcery](https://github.com/krzysztofzablocki/Sourcery) and [SwiftLint](https://github.com/realm/SwiftLint), which parse Swift code as part of their operation, often need some way to customize their behaviour by attaching metadata to certain declarations. For example, [Sourcery](https://github.com/krzysztofzablocki/Sourcery) templates can access declaration annotations expressed in comments:

```swift
// sourcery: skipPersistence
// sourcery: anotherAnnotation = 232, yetAnotherAnnotation = "value"
/// Some documentation comment
var precomputedHash: Int
```

Unfortunately, because those annotations live in comments:

* they are less discoverable and more error-prone as they can't take advantage of code completetion,
* they can't be declared or documented in code and,
* they can't be type-checked by the compiler.

Adding custom attributes to Swift would enable those annotations to be expressed as types and resolve all the previous issues.

Furthermore, while static attributes only exist at compile-time, they are a first step in the direction of runtime introspectable custom attributes, which enable more powerful use-cases. The second motivation of this proposal is to lay the foundation for those, before they can be brought forward in a follow-up proposal.

## Design

This proposal introduces a new compiler-defined attribute called `@staticAttribute` for declaring static custom attributes through `struct`s. It takes a single `usage` argument that defines the type of declarations the attribute an be used on. For example:

```swift
@staticAttribute(usage: [.struct, .class])
struct AutoCoding {}

@AutoCoding
class MyClass: NSObject {
    // ...
}

@AutoCoding // error: the `AutoCoding` attribute can't be used on enums
enum MyEnum {
    // ...
}
```

The custom attributes take the name of their defining type and can be constructed through their initializers. This allows for different construction patterns:

```swift
enum Rule {
    case cyclomaticComplexity
    case lineCount
    // ...
}

@staticAttribute(usage: [.struct, .class, .enumm, .property, .function, .subscript])
struct Ignore {
    init(rule: String) {}
    init(rules: [String]) {}
}

@Ignore(rules: [.cyclomaticComplexity, .lineCount])
func a() {
	// ...
}

@Ignore(rule: .lineCount)
func b() {
	// ...
}
```

### Rules

To recap, static custom attribtues follow the following rules:

* The `@staticAttribute` attribute can only annotate structs.
* The `@staticAttribute` takes a single mandatory `usage` argument of `Set<AttributeUsage>` type.
* The `AttributeUsage` type is an enum with the following cases:
    * `case` which represents enum case declarations,
    * `class` which represents class declarations,
    * `enum` which represents enum declarations,
    * `function` which represents any free, static or instance function declarations,
    * `property` which represents stored or computed property declarations in types,
    * `protocol` which represents protocol declarations,
    * `struct` which represents struct declarations and,
    * `subscript` which represents subscript declarations.
* Custom attributes are named after their declaration type and can be disambiguated by prefixing the attribute name by the name of the module they are defined in followed by a period (for example, `@SwiftLint.Ignore`).
* Custom attributes can only be constructed through the initializers accessible at the use-site.
* Using a custom attribute on a declaration not within the set of declarations defined in the attribute's `usage` argument causes a compilation error.

While `@staticAttribute` is a compiler-defined attribute, it can help to visualize it as the following static custom attribute meta-definition:

```swift
@staticAttribute(usage: [.struct])
public struct staticAttribute {
    public init(usage: Set<AttributeUsage>) {}
}

public enum AttributeUsage {
    case `case`
    case `class`
    case `enum`
    case function
    case property
    case `protocol`
    case `struct`
    case `subscript`
}
```

### Coding Style

The proposal suggests that custom attributes should start by an uppercase letter to differentiate them from compiler-defined attributes and avoid any collision with future compiler attributes.

# Source compatibility

This proposal is purely additive.

# Effect on ABI stability

As static custom attributes only exist at compile-time, they don't affect ABI stability.

# Effect on API resilience

As static custom attributes only exist at compile-time, they don't affect ABI resilience.

# Alternatives considered

A previous version of this proposal suggested introducing a new delcaration syntax for declaring custom attributes. But this solution has the disavantage of being harder to learn, and of having a higher implementation cost.