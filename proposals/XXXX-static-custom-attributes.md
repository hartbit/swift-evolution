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

This proposal introduces a new compiler-defined attribute called `@staticAttribute` for declaring static custom attributes through `struct`s. It takes a mandatory `declarations` argument that defines the type of declarations the attribute can annotate and an optional `scopes` arguments that further refines the scope of the allowed declarations. For example:

```swift
@staticAttribute(declarations: [.struct, .class], scopes: [.global])
struct AutoCoding {}

@AutoCoding
class MyClass: NSObject {
    // ...
}

@AutoCoding // error: the `AutoCoding` attribute can't be used on enums
enum MyEnum {
    // ...
}

func foobar() {
    @AutoCoding // error: the `AutoCoding` attribute can't be used on local declarations
    struct Inner {
        // ...
    }
}
```

The custom attributes take the name of their defining type and can be constructed through their initializers. This allows for different construction patterns:

```swift
enum Rule {
    case cyclomaticComplexity
    case lineCount
    // ...
}

@staticAttribute(declarations: [.struct, .class, .enum, .property, .function, .subscript])
struct Ignore {
    init(rule: Rule) {}
    init(rules: [Rule]) {}
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

### Definition

The `@staticAttribute` attribute is itself a custom static attribute defined in the Standard Library with the following definition:

```swift
/// The meta static attribute to define static attributes.
@staticAttribute(declarations: [.struct])
public struct staticAttribute {

    /// Represents the type of declaration the attribute can be annotated to.
    public enum Declaration {

        /// An enumeration type declaration.
        case `enum`

        /// A structure type declaration.
        case `struct`

        /// A class type declaration.
        case `class`

        /// A protocol declaration.
        case `protocol`

        /// A typealias declaration.
        case `typealias`

        /// An associated-type declaration.
        case `associatedtype`

        /// A mutable or immutable variable declaration.
        case variable

        /// A function parameter declaration.
        case parameter

        /// A subscript declaration.
        case `subscript`

        /// An initializer declaration.
        case initializer

        /// A deinitializer declaration.
        case deinitializer

        /// A function declaration.
        case function

        /// A property accessor declaration.
        case accessor

        /// An extension declaration.
        case `extension`

        /// A module import declaration.
        case `import`

        /// An enumeration case declaration.
        case enumCase
    }

    /// Represents a declaration scope the attribute can be annotated to.
    public enum Scope: CaseIterable {

        /// Global scope declaration.
        case global

        /// Local scope declaration (within a function, method, etc...)
        case local

        /// Static member of a type.
        case `static`

        /// Instance member of a type.
        case instance
    }

    /// Creates an instance of the `staticAttribute` attribute.
    /// - Parameters:
    ///   - declarations: The declarations the attribute can annotate.
    ///   - scopes: The scopes of the declarations the attribute can annotate.
    public init(declarations: Set<Declaration>, scopes: Set<Scope> = Set(.allCases)) {}
}
```

The configuration of declarations and scopes creates a matrix of different constraints that can be placed on the placement of custom attributes. The following code snippets demonstrates the different combinations:

#### Enum

```swift
@GlobalEnumAttribute
enum GlobalEnum {}

func foo() {
    @LocalEnumAttribute
    enum LocalEnum {}
}

struct Foo() {
    @StaticEnumAttribute
    enum StaticEnum {}
}
```

#### Struct

```swift
@GlobalStructAttribute
struct GlobalStruct {}

func foo() {
    @LocalStrutAttribute
    struct LocalStruct {}
}

struct Foo() {
    @StaticStructAttribute
    struct StaticStruct {}
}
```

#### Class

```swift
@GlobalClassAttribute
class GlobalClass {}

func foo() {
    @LocalClassAttribute
    class LocalClass {}
}

struct Foo() {
    @StaticClassAttribute
    class StaticClass {}
}
```

#### Protocol

```swift
@ProtocolAttribute
protocol Protocol {}
```

#### Typealias

```swift
@GlobalTypealiasAttribute
typealias GlobalTypealias = Foo

func foo() {
    @LocalTypealiasAttribute
    typealias LocalTypealias = Foo
}

struct Foo {
    @StaticTypealiasAttribute
    typealias StaticTypealias = Foo
}
```

#### Associatedtype

```swift
protocol Foo {
    @StaticAssociatedtypeAttribute
    associatedtype StaticAssociatedtype
}
```

#### Variable

```swift
@GlobalVariableAttribute
let globalVariable = 0

func foo() {
    @LocalVariableAttribute
    let localVariable = 0
}

struct Foo {
    @StaticVariableAttribute
    static let staticVariable = 0

    @InstanceVariableAttribute
    let staticVariable = 0
}
```

#### Parameter

```swift
func foo(@ParameterAttribute parameter: Int) {
}
```

#### Subscript

```swift
struct Foo {
    // If and once SE-0254 is accepted.
    @StaticSubscriptAttribute
    static subscript(index: Int) -> Int {
        get { ... }
        set { ... }
    }

    @InstanceSubscriptAttribute
    subscript(index: Int) -> Int {
        get { ... }
        set { ... }
    }
}
```

#### Initializer

```swift
struct Foo {
    @InitializerAttribute
    init() {
    }
}
```

#### Deinitializer

```swift
struct Foo {
    @DeinitializerAttribute
    deinit() {
    }
}
```

#### Function

```swift
@GlobalFunctionAttribute
func globalFunction() {
}

func foo() {
    @LocalFunctionAttribute
    func localFunction() {
    }
}

struct Foo {
    @StaticFunctionAttribute
    static func staticFunction() {
    }

    @InstanceFunctionAttribute
    func instanceFunction() {
    }
}
```

#### Accessor

```swift
var foo: Int {
    @GlobalAccessorAttribute
    get { ... }
}

func foo() {
    var foo: Int {
        @LocalAccessorAttribute
        get { ... }
    }
}

struct Foo {
    static var foo: Int {
        @StaticAccessorAttribute
        get { ... }
    }

    var foo: Int {
        @InstanceAccessorAttribute
        get { ... }
    }
}
```

#### Extension

```swift
@ExtensionAttribute
extension Foo {}
```

#### Import

```swift
@ImportAttribute
import Foo
```

#### EnumCase

```swift
enum Foo {
    @EnumCaseAttribute
    case foo
}
```

### Structs

As per the definition, `@staticAttribute` can only be attached to structs. They are not supported on enums as those don't have natural initializers that can be used for the creation pattern. They are neither supported on classes as it is too early to predict if those will work with future compile-time evaluation features in Swift.

### Namespacing

Custom attributes are named after their declaration type and can be disambiguated by prefixing the attribute name by the name of the module they are defined in followed by a period (for example, `@SwiftLint.Ignore`).

### Visibility

Custom attributes can only be constructed if they are visible at the use-site and only through the initializers accessible at the use-site. For example, if a `public` custom attribute defines an `internal` and a `public` initializer, it can be constructed using both initializers inside the same module, but only the `public` initializer can be used from other modules.

### Naming

The proposal suggests that custom attributes with names starting with an underscore or a lowercase ASCII letter are reserved for the Standard Library, to help differentiate and avoid collision between language provided attributes like `@inlinable` and user-defined custom attributes like `@Ignore`.

### From Compiler to Standard Library

Static custom attribtues are powerful enough to allow several compiler-defined attributes to be re-defined inside the Standard Library. Here a few examples which could take advantage of them:

```swift
@staticAttribute(declarations: [.function])
public struct discardableResult {}

@staticAttribute(declarations: [.enum, .struct, .class, .protocol])
public struct dynamicCallable {}

@staticAttribute(declarations: [.enum, .struct, .class, .protocol])
public struct dynamicMemberLookup {}

@staticAttribute(declarations: [.import])
public struct testable {}

@staticAttribute(declarations: [.function, .variable, .subscript, .initializer, .deinitializer])
public struct inlinable {}

@staticAttribute(declarations: [.function, .variable, .subscript, .initializer, .deinitializer])
public struct usableFromInline {}

@staticAttribute(declarations: [.function])
public struct warn_unqualified_access {}

@staticAttribute(declarations: [.function, .variable, .subscript, .initializer])
public struct nonobjc {}

@staticAttribute(declarations: [.class])
public struct objcMembers {}

@staticAttribute(declarations: [.class], scopes: [.global])
public struct NSApplicationMain {}

@staticAttribute(declarations: [.variable, .function], scopes: [.instance])
public struct NSManaged {}

@staticAttribute(declarations: [.class])
public struct requires_stored_property_inits {}

@staticAttribute(declarations: [.function], scopes: [.instance])
public struct IBAction {}

@staticAttribute(declarations: [.class, .extension], scopes: [.global])
public struct IBDesignable {}

@staticAttribute(declarations: [.variable], scopes: [.instance])
public struct IBInspectable {}

@staticAttribute(declarations: [.variable], scopes: [.instance])
public struct IBOutlet {}

@staticAttribute(declarations: [.class], scopes: [.global])
public struct UIApplicationMain {}

@staticAttribute(declarations: [.variable], scopes: [.instance])
public struct GKInspectable {}
```

## Source compatibility

This proposal is purely additive.

## Effect on ABI stability

As static custom attributes only exist at compile-time, they don't affect ABI stability.

## Effect on API resilience

As static custom attributes only exist at compile-time, they don't affect ABI resilience.

## Alternatives considered

A previous version of this proposal suggested introducing a new declaration syntax for declaring custom attributes. It was abandoned due to a higher implementation and learning cost.
