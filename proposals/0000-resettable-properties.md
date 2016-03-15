# Resettable Properties

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-resettable-properties.md)
* Author(s): [Jason Patterson](https://github.com/patters)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

Resettable properties are mutable properties that can also be "reset" to some default value. Instead of requiring client code to manually set the property to its default by value, "resetting" allows the property implementation to define special heuristics for "resetting" and allows consumers to forego knowing what the default value actually is.

This proposal nominates new syntax to support resettable properties in Swift which can then be leveraged by new Swift code and the Objective-C importer.

Swift-evolution thread: [Original email](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160314/012676.html)

## Motivation

Objective-C includes the keyword `null_resettable` to decorate `readwrite` properties whose getter will always return a value, and whose setter can be passed in `nil` to "reset" the property, presumably to a "default" value. Pragmatically, this means the getter implementation returns the default value when the property has been reset (or left unset). This is a useful pattern in general, not just limited to legacy Objective-C code. 

Currently, Swift imports these properties as `ImplicityUnwrappedOptional<T>`. This Objective-C class:

```objc
@interface Foo: NSObject
@property (nonatomic, nonnull, null_resettable) NSString *name;
@end
```

uses the `null_resettable` keyword to allow callers to specify a new `name` for an instance, or to reset `name` to some default by specifying `nil`. Querying `name` after resetting it will not return `nil`. In other words, the implementation decouples the nullability of the getter and setter and looks as follows:

```objc
- (nonnull NSString *)name { ... }
- (void)setName:(nullable NSString *)newValue { ... }
```

This class is currently imported into Swift as:

```swift
class Foo: NSObject {
  var name: String!
}
```

Currently, there is no way to represent `null_resettable` in Swift, so the importer declares the property as a `T!`. This impacts readability of the code and uses implicitly unwrapped optionals, which many developers try to avoid. 

By importing code in this way, the `null_resettable` specification has lost its semantic intent, since `null_unspecified` properties are also imported as `T!`. Developers must resort to detailed documentation to explain the discrepancy. For example, API documentation must detail why their property is typed as an implicity unwrapped optional, and documentation must assure users that the getter will never return `nil` and cause runtime crashes.

Moreover, *new* code written in Swift cannot cleanly capture this useful pattern without sacrifices (see Alternatives considered).

By adopting a solution to this problem, Swift gains flexibility in representing property semantics, developers gain readability, and the Objective-C importer can reflect the true intent of the developer.


## Proposed solution

The proposed syntax for representing a resettable property is to use a new `set?` operator. This indicates that the type of the value (i.e. `newValue`) available in the setter implementation is an `Optional<T>` and that consumers may set the property to `nil` (or `.None`). For example:

```swift
protocol Resettable {
  var name: String { get set? }
}

class Character {
  private let initialName: String
  init(name: String) {
    myName = initialName = name
  }
  private var myName: String? = nil
  var name: String {
    get { return myName ?? initialName }
    set? { myName = newValue }
  }
}

let c = Character("Narrator")
c.name // "Narrator"
c.name = "Cornelius"
c.name // "Cornelius"
c.name = nil
c.name // "Narrator"
c.name = "Rupert"
c.name // "Rupert"
c.name = .None
c.name // "Narrator"
```

The scope of the proposed solution is to specify the syntax and interface of a resettable property. How the property is actually *reset*, of course, left to the class implementation.

Above, a resettable property is be chosen to be backed with an optional instance variable. In another example, resetting a property has a side effect on another instance variable:

```swift
protocol Serializer { var passphrase: String { get } }

class Document {
  private var serializer: Serializer
  var passphrase: String {
    get { return serializer.passphrase }
    set? {
      if let passphrase = newValue {
        serializer = EncryptedSerializer(passphrase: passphrase)
      } else {
        serializer = PlainSerializer()
      }
    }
  }
}
```


## Detailed design

This proposal nominates these changes to the Swift grammar:

*set-operator* → `set` |­ `set­?`­  
*setter-clause* → *attributes­*<sub>opt</sub>­ *set-operator* *­setter-name*­<sub>opt</sub>­ *­code-block­*  
*setter-keyword-clause* → *attributes*­<sub>opt</sub>­ *set-operator*  
*willSet-operator* → `willSet` |­ `willSet?`­  
*willSet-clause* → *attributes­*<sub>opt</sub>­ *willSet-operator* *­setter-name*­<sub>opt</sub>­ *­code-block­*  

Overloading the `set` operator with `set?` mirrors the implied change in usage associated with the `try` and `try?` operators. Using the `set?` operator for a property of type `T` would affect property implementations in the following ways:

* The getter continues to return `T`.
* The setter takes a `T?`.

Subclasses that override with `willSet` clauses would be required to use the `willSet?` operator, added to match readability with the `set?` operator. Subclasses adding `didSet` clauses do not change. The implementations change in the following ways:

* The `willSet?` clause receives a `T?`.
* The `didSet` clause continues to receive a `T`.

For example:

```swift
class Foo: Resettable {
  var name: String {
    get { /* returns a String */ }
    set? { /* newValue is an Optional<String> */ }
  }
}

class Bar: Foo {
  override var name: String {
    willSet? { /* newValue is an Optional<String> */ }
    didSet { /* oldValue is a String */ }
  }
}
```

This proposal recommends that the Swift compiler *should* generate a warning if the `set?` operator is used on a property with an optional or implicitly unwrapped optional type, since the setter would then be using a `T??` or `T!?`. The Swift compiler *should* generate an error if an overridden property is adding a `willSet` clause to a resettable property and the `willSet?` operator was not used.

Changes to the Objective-C importer:

* Objective-C code that declares a `null_resettable` property would be imported into Swift as a resettable property.
* Swift code that declares a resettable property would be visible to Objective-C as a `null_resettable` property.

It should also be noted that [Property Behaviors](https://github.com/apple/swift-evolution/blob/master/proposals/0030-property-behavior-decls.md) could be used to provide a default implementation of resettable properties in the standard library, but Swift still needs the syntax changes to reflect the intent.


## Impact on existing code

There is no impact to existing Swift-only code since the feature is additive. Re-imported Objective-C code with `null_resettable` properties will use the new grammar. All existing consumers of the previously-imported code (using `T!`) should continue to compile and function as before. 

With the newly-imported version, values will no longer be implicitly unwrapped when retrieved. For example, this code is valid before and after the change (except there is no longer a "danger" of a runtime assertion):

```swift
class Foo {
  let tintColor: UIColor
  init(view: UIView) {
    tintColor = view.tintColor
  }
}
```

## Alternatives considered

Instead of adding new grammar, one can just add reset methods to the class:

```swift
class Foo {
  var visibility = false
  func resetVisibility() { ... }
  var depth = 0
  func resetDepth() { ... }
  var name = "Foo"
  func resetName() { ... }
}
```

Depending on the frequency of use, this could be considered method pollution. This does not solve the `null_resettable` import problem either.

Instead of adding a new `set?` operator, one could a new keyword (`reset`) that would be used alongside of `set` in implementations:

```swift
protocol Foo {
  var name: String { get set reset }
}

class Foo {
  var name: String {
    get { }
    set { }
    reset { }
  }
}
```

This doesn't offer any innate clues to the reader as to how this is actually *used* by client code. Resetting "reads" as a separate operation here, instead of implying `foo.name = nil` syntax for usage. 

Another alternative would be to allow the type of `newValue` to be specified in setters:

```swift
protocol Foo {
  var name: String { get set(newValue: String?) }
}
class Foo {
  var name: String {
    get { }
    set(newValue: String?) { }
  }
}
```

This still introduces grammar changes via new syntax for protocol property definitions. This also would permit a developer to completely decouple the type of a property's getter and setter, which has little use in practice. It also pollutes the setter declaration with an unnecessary `newValue` before the type declaration, which further impacts readability.
