# Improving capture semantics of local functions

* Proposal: [SE-00XX](https://bitbucket.org/snippets/lynchrb/r487zn)
* Authors: [Alex Lynch](https://bitbucket.org/iam_apps/), [David Hart](http://github.com/hartbit)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

This proposal improves local functions by giving them the same memory safety and expressive power as closures in regards to capture semantics.

Swift-evolution thread: [Making capturing semantics of local](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

Local functions can be an elegant alternative to closures. But the compiler diagnostics and their lack of capture semantics management makes them error-prone in regards to retain cycles.

For example, the compiler will enforce a `self.` prefix when accessing class members from closures to make the capture semantics to `self` explicit, but it doesn't have a similar diagnostic for local functions.

```swift
class Foo {
    func foo() {
        func localFunction() {
            bar()
        }
        
        DispatchQueue.main.async {
            bar() // error: call to method 'bar' in closure requires explicit 'self.' to make capture semantics explicit
        }
    }

    func bar() {
        print("bar")
    }
}
```

This makes it very easy to unkowingly keep a strong reference to `self` and potentially create retain cycles, as in the example below:

```swift
class Foo {
    let observation: NSKeyObservation?

    func foo() {
        func localFunction() {
            bar()
        }
        
        observation = object.observe(\.property) { [weak self] (_, _) in
            self?.bar()
            localFunction()
        }
    }

    func bar() {
        print("bar")
    }
}
```

This creates a retain cycle: `self` -> `observation` -> _closure_ -> `localFunction` -> `self`. Moreover, even if the programmer is aware of the strong reference to `self` in the local function, there is no way to modify the capture semantics and break the cycle.

## Proposed solution

This proposal suggests resolving this disparity by extending the `self.` prefix diagnostic and capture lists syntax to local functions.

## Detailed design

This proposal suggests two changes to the language:

### Explicit self diagnostic

First of all, this proposal suggests extending the requirement of the `self.` prefix to local functions, but only if the local function is used as or used inside an escaping closure.

For example, the following example shows several cases:

```swift
class Foo {
    func foo() {
        // The compiler should generate an error inside `local1` because it is used as an escaping closure.
        
        func local1() {
            bar() // error: call to method 'bar' in local function requires explicit 'self.' to make capture semantics explicit
        }

        DispatchQueue.main.async(execute: local1)

        // The compiler should generate an error inside `local2` because it is used inside an escaping closure.

        func local2() {
            bar() // error: call to method 'bar' in local function requires explicit 'self.' to make capture semantics explicit
        }

        DispatchQueue.main.async { local2() }

        // The compiler not should generate an error inside `local3` because it is used inside a non-escaping closure.

        func local3() {
            bar()
        }

        { local3() }()

        // The compiler not should generate an error inside `local4` because it is called from the same scope.

        func local4() {
            bar()
        }

        local4()
    }

    func bar() {
        print("bar")
    }
}
```

### Capture list

Secondly, this proposal suggests allowing the same capture list syntax from closures in local functions. Capture lists would still be invalid in top-level and member functions.

```swift
class Foo {
    func foo() {
        func local() { [weak self] in
            self?.bar()
        }
    }

    func bar() {
        print("bar")
    }
}
```

## Source compatibility

The extension of the `self.` prefix diagnostic to local functions is an important aspect of the proposal as strong references to `self` are one of the most common sources of retain cycles. Unfortunately, this change brings a source breakage as it introduces as new error which did not exist previously.

But by only applying the rule to local functions used as/in escaping contexts, we should greatly reduce the amount of cases where the new error will trigger. Furthermore, the cases where it will trigger will undoubtly point users to potential retain cycles.

## Effect on ABI stability

The proposed change will have no effect on the ABI.

## Effect on API resilience

The proposed feature has no effect on API. It is a modification to local functions which are visible only in the containing function scope. 

## Alternatives considered

An alternative syntax for the capture list in local functions without the `in` keyword was considered but ultimately rejected for consistency reasons:

```swift
class Foo {
    func foo() {
        func local() { [weak self]
            self?.bar()
        }
    }

    func bar() {
        print("bar")
    }
}
```