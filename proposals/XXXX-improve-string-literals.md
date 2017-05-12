# Improve String Literals

* Proposal: [SE-XXXX](XXXX-improve-string-literals.md)
* Authors: [David Hart](https://github.com/hartbit), [Adrian Zubarev](https://github.com/devandartist)
* Review Manager: TBD
* Status: TBD

## Introduction

This proposal builds on top the new features of [SE-0168 Multi-Line String Literals](0168-multi-line-string-literals.md) by widening the use-cases for unescaped double-quotes and resolving certain issues around long lines in single and multi-line string literals.

## Motivation

In Swift 3, String literals have three pain points:

* Strings containing double-quotes
* Multi-line strings
* Long single-line strings

Proposal [SE-0168](0168-multi-line-string-literals.md) fixed the two first problems with the same syntax. Unfortunately, while an improvement on Swift 3, several problems remain:

* Long single-line strings still require the less than ideal concatenation syntax:

Some project styles (like the Standard Library) mandate a maximum line length, requiring long single-line strings to be hard-wrapped. This still requires odd solutions:

```swift
assert(condition, "This is a long assertion message that requires " +
    "string concatenation when the project style enforces maximum line " +
    "lengths")
```

* Long lines in a multi-line strings can't be manually wrapped:

```swift
let markdown = """
    # Title

    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer elementum commodo sem, a congue orci porta sit amet. Duis facilisis, est et vehicula congue, turpis dui ultricies nunc, ut elementum quam elit nec felis. Integer aliquam id risus nec laoreet. Vivamus vitae odio sit amet quam iaculis fermentum nec sed neque.

    ## Subtitle

    Cras et nibh velit. Praesent eleifend sagittis quam, pellentesque lobortis lectus commodo vel. Vivamus suscipit, nulla quis blandit ullamcorper, velit neque euismod nibh, nec blandit mi diam molestie ex. Cras porttitor, est sed pharetra interdum, ipsum mauris viverra quam, sit amet eleifend purus elit sit amet odio.
    """
```

* Short strings containing double-quotes have to use the multi-line syntax to benefit from unescaped double-quotes:

```swift
print("""
    { "success": false, "error": "Wrong parameter" }
    """)
```

## Proposed solution

By implementing multi-line string literals and support for unescaped double-quotes with the same syntax, [SE-0168](0168-multi-line-string-literals.md) has made those features unusable on their own. By dissociating them and supporting two extra syntax features, we can solve all the above problems:

### Divorce the `"""` delimiter from the multi-line syntax and have them only support unescaped double-quotes

The change allows us to express short strings containing double-quotes without resorting to the multi-line syntax:

```swift
print("""{ "success": false, "error": "Wrong parameter" }""")
```

As a consequence, multi-line strings are now only defined by a newline following the leading delimiter and the whitespace preceeding the trailing delimiter. They gain support for `"` delimiters, which has the nice advantage of saving a few characters in multi-line strings which are known to never contain double-quotes:

```swift
print("""
    Triple " are still valid delimiters
    """)

query("
    SELECT 'name'
    FROM 'people'
    WHERE age > 20
    ")
```

### Support escaping newlines in multi-line strings with a trailing `\`

This change allows hard-wrapping long lines in multi-line strings. They also have the added benefit of making trailing white-space at the end of source-code lines explicit.

```swift
let markdown = """
    # Title

    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer \
    elementum commodo sem, a congue orci porta sit amet. Duis facilisis, est \
    et vehicula congue, turpis dui ultricies nunc, ut elementum quam elit nec \
    felis. Integer aliquam id risus nec laoreet. Vivamus vitae odio sit amet \
    quam iaculis fermentum nec sed neque.

    ## Subtitle

    Cras et nibh velit. Praesent eleifend sagittis quam, pellentesque \
    lobortis lectus commodo vel. Vivamus suscipit, nulla quis blandit \
    ullamcorper, velit neque euismod nibh, nec blandit mi diam molestie \
    ex. Cras porttitor, est sed pharetra interdum, ipsum mauris viverra \
    quam, sit amet eleifend purus elit sit amet odio.
    """
```

### Adopt the C/Objective-C syntax that concatenates single-line strings

This change will be familiar to C developers and provides a cleaner and more performant solution for long single-line strings:

```swift
assert(condition, "This is a long assertion message that flows "
    "from one line to the next without requiring the concatenation "
    "operator")

assert(condition, """This is another "single-line" message that """
    """supports up to two double-quotes (" and "") without any """
    """escaping""")
```

## Source compatibility

This feature is purely additive; it has no effect on source compatibility.

## Effect on ABI stability

This feature is purely additive; it has no effect on ABI stability.

## Effect on API resilience

This feature is purely additive; it has no effect on API resilience.

## Alternatives considered

A different syntax for supporting long single-line strings was discussed where ending delimiters were replaced with the `\` escaping character, mirroring their use in multi-line strings:

```swift
assert(condition, "This is a long assertion message that flows \
    "from one line to the next without requiring the concatenation \
    "operator")

assert(condition, """This is another "single-line" message that \
    """supports up to two double-quotes (" and "") without any \
    """escaping""")
```

That syntax saved two characters per line in strings with `"""` delimiters but had several disadvantages:

* It loses the familiarity with C syntax
* It introduces an asymmetry between the last line and those above
* It does not do any actual escaping, introducing developer ambiguity with their use in multi-line literals