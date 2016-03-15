# Improved importing of Objective-C null_resettable properties

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-importing-null_resettable.md)
* Author(s): [Jason Patterson](https://github.com/patters)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

Objective-C includes the keyword `null_resettable` to decorate `readwrite` properties whose getter will always return a value, and whose setter can be passed in `nil` to "reset" the property, presumably to a "default" value.

Currently, Swift imports these properties as implicitly unwrapped optionals. This proposal nominates that such properties be imported as non-optional properties with an additional "reset" method added to the interface.

Swift-evolution thread: [Original email](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160314/012676.html)

## Motivation

Objective-C includes the keyword `null_resettable` to decorate `readwrite` properties whose getter will always return a value, and whose setter can be passed in `nil` to "reset" the property, presumably to a "default" value. Pragmatically, this means the getter implementation returns the default value when the property has been left unset.

For example, this Objective-C class:

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


## Proposed solution

The proposed syntax for importing a `nonnull, null_resettable` property is to import the property as though it were a plain `nonnull` property and to add a "reset" method on the imported class. The method's name is derived by uppercasing the first letter of the property name and prepending the string "reset" to that name. For example, this Objective-C class:

```objc
@interface Foo: NSObject
@property (nonatomic, nonnull, null_resettable) NSString *firstName;
@end
```

would be imported as:

```swift
class Foo: NSObject {
  var firstName: String
  func resetFirstName()
}
```

For completeness, Swift classes visible to Objective-C would adopt the pattern in reverse: any non-optional properties that also have a matching "reset" method would be exposed to Objective-C as `null_resettable` properties, omitting the reset method altogether.

## Detailed design

Resetting an Objective-C `null_resettable` property is as simple as calling the setter with a `nil` parameter:

```objc
[foo setBar:nil];
```

Therefore, the "reset" method that is added to Swift to reset the property would be bridged to the setter's selector. For example, this Objective-C class:

```objc
@interface Foo: NSObject
@property (nonatomic, nonnull, null_resettable) NSString *firstName;
@end
```

would be imported as:

```swift
class Foo: NSObject {
  var firstName: String
  func resetFirstName() 
}
```

Getting and setting `foo.firstName` sends the `firstName` and `setFirstName:` selectors, respectively. Calling `foo.resetFirstName()` would also send the `setFirstName:` selector to the receiver with a `nil` first argument.

Conversely, Swift classes visible to Objective-C would be analyzed for properties that could be marked as `null_resettable` if the type is non-optional and there exists a matching "reset" method. The reset method would not be exposed as a separate method on the Objective-C class. Swift would need to bridge calls from a `setSomeProperty:nil` message to the `resetSomeProperty()` method on the Swift class.

For example, this class authored in Swift:

```swift
class Foo: NSObject {
  var firstName: String
  init(firstName: String) {
    self.firstName = firstName
  }
  func resetFirstName() {
    firstName = "Foo"
  }
}
```

would be visible to Objective-C as:

```objc
@interface Foo: NSObject
@property (nonatomic, nonnull, null_resettable) NSString *firstName;
@end
```

and used in the following way:

```objc
Foo *foo = [[Foo alloc] initWithFirstName:@"Foo"];
[foo firstName]; // calls the firstName getter and returns "Foo"
[foo setFirstName:"Bar"]; // calls the firstName setter with newValue = "Bar"
[foo setFirstName:nil]; // calls resetFirstName()
```

### Naming conflicts

When importing, if the Objective-C class defines a resettable property as well as a reset method (that matches the naming scheme), such as:

```objc
@interface Foo: NSObject
@property (nonatomic, nonnull, null_resettable) NSString *firstName;
- (void)resetFirstName;
@end
```

then a warning would be generated when the Objective-C header is bridged into Swift, indicating that the Objective-C method implementation available via the `resetFirstName` selector is unavailable to Swift.

## Impact on existing code

Swift code that was setting imported `null_resettable` properties (typed `T!`) to nil would need to be changed to call the new reset method, since the property is no longer optional. The compiler *should* generate a helpful message to indicate that the syntax has changed.

Swift code visible to Objective-C may now have properties decorated with `null_resettable` that weren't before, which is not a breaking change. However, matching "reset" methods will no longer be visible to Objective-C, which may break some Objective-C code, so the compiler *should* generate a helpful message to indicate that the caller should set the property to `nil` instead of sending the `resetSomeProperty` message.


## Alternatives considered

No alternatives for importing `null_resettable` Objective-C code have been considered. 

Resettable properties are a useful concept in general, not specific to Objective-C. If Swift were to feature first-class support for them, then that opens up new alternatives and possibilities that would supercede this proposal. 

However, the cost versus the benefit of first-class support, combined with the limited usage of resettable properties in practice, may mean that the feature should be limited to Objective-C and does not warrant a prominent place in Swift at this time.

