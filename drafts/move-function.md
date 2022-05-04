---
layout: page
title: An Introduction to the Move Function in Swift for System Programmers
categories: draft
---

NOTE: In the following the word binding is referring to a `let` or a `var` like
entity in order to distinguish in between the "binding" itself and the "value"
contained within the "binding". We never invalidate "values" using `move`, only
invalidate "bindings".

NOTE: It is assumed that the reader, as a prerequisite for reading this
document, understands Swift's copy on write model and how that interplays with
uniqueness.

In Swift today, one is unable to end the lifetime of a binding and move its
contents into a different binding. With this in mind, the Swift team is adding a
new special function to Swift called `move`. `move` is inspired by functionality
like `drop` from other languages like Rust. To work with `move`, one applies
`move` to a binding. move will then invalidate the binding and emit an error if
one uses the binding ever again in the case of a let like construct (`let` or
`__owned` arg) or if one uses the binding before reinitializing it for a var
like binding (`var`, `inout`, `mutating self`). As a simple example, consider a
routine that takes in a user data struct that contains a String that we wish to
preprocess and pass off to a system framework afterwards while having the
compiler enforce that uniqueness is preserved:

```
struct SystemFrameworkState {
  var _innerState: String
  var state: String { set { /* put newValue into _innerState and mutate _innerState */ } }
}

extension String {
  init(data: UserData) { /* ... */ }
}

var globalState = SystemFrameworkState()

func f(input: UserData) {
    // Construct a new string from user data.
    let x = String(data: input)

    // Use x in read only ways.

    // Hand off x as an argument to another function while preserving uniqueness.
    // We do not want to break uniqueness since we do not need more copies!
    globalState.state = _move(x)
}
```

Without using `move` here, we have two potential issues. The first is that we
are relying on the optimizer to notice that `f` doesn’t use `x` after passing
`x` to `globalState.state`. Of course most likely the optimizer will do so, but
when writing system code, "likely" is insufficient, so the guarantee allows us
to know that our requirement is maintained. Secondly and perhaps more
importantly, we are relying on our teammates to remember that they should not
add another use of `x` when modifying this code. If they innocently change it to

```
globalState.state = x
x.doSomething()
```

they have accidentally forced Swift to copy `x`, which is what we were trying to
avoid! By using `move` we avoid both of these issues; the optimizer has a hard
guarantee that no copies are required, and code that would innocently introduce
a copy produces an error when compiling, like so:

```
globalState.state = _move(x)
x.doSomething() // Error! 'x' used after being moved.
```

`move` also allows for one to rely on the compiler to safely move state from a
`let` into a `var` without breaking uniqueness of the value stored in the
`let`. For example, consider the setter from `SystemFrameworkState` above this
time with actual code:

```
struct SystemFrameworkState {
  var _innerState: String
  var state: String {
    set {
       // newValue is a __owned argument binding. We assume that it was passed in
       // unique.
       precondition(newValue.isUnique)

       // Now we want to move it into a var so we can perform inout operations
       // upon it. We move newValue so that the compiler will enforce at compile time
       // that extra copies of newValue are not created since no further uses of newValue
       // can occur in the function.
       var x = _move(newValue)

       // Now we can invoke mutating methods on x without causing a copy.
       x.performMutatingMethod()

       // And then store x into _innerState guaranteeing that if someone later
       // puts more mutating code later on x, we get a compile time error.
       _innerState = _move(x)
    }
  }
}
```

Notice, how we are able to move `newValue` (which is a `__owned` argument that behaves
like a let binding) into `x`. We know that `move` will prevent any later local
uses of `newValue`, so we know that the value (now in `x`) must still be unique. Thus
we can call mutating methods on the String without worrying about creating
copies.

Notice how in the previous paragraph, we mentioned that a `__owned` argument
behaves similarly to a `let` binding when it comes to `move`. A natural question
then is to wonder how `move` applies to `inout` and `mutating self` since such
arguments are in the language `var` like arguments. In this case, `move` behaves
in a similar way to `var`, but has an additional constraint: the programmer must
re-initialize the `inout` or `mutating self` argument before the end of the
function. The reason for this restriction is that from an ABI perspective,
`inout` and `mutating self` have a requirement that even though one can move the
input argument out of the memory slot while the function is running, upon
return, a valid object must be in the argument slot. This allows for one to
safely move out `self` or an `inout` argument and later move in a new value
without needing to worry about future programmers forgetting to re-initialize
the value when adding new code. As an example, consider the following example
code that involves replacing a stored `KlassState` with a new updated `KlassState`
from the kernel:

```
struct SystemFrameworkStruct {
    var innerPtr: KlassState

    mutating func updateState() {
        // Grab the current inner ptr state off of the moved self.
        let val = _move(self).innerPtr

        // Now pass off our current inner ptr to the kernel, asking for
        // a potentially different pointer to updated state:
        let newValue = askKernelForNewPointer(val)

        // If we don't reinitialize self we get an error! Uncomment next line to
        // fix this.
        // self = SystemFrameworkStruct(innerPtr: newValue)
    }
}
```

Thus the user has communicated to the compiler using `move` that after grabbing
`innerPtr`, self has now been invalidated and must be reinitialized before the
end of the current function. This results in the following error being emitted:

```
test.swift:10:19: error: 'self' used after being moved
    mutating func updateState() {
                  ^
test.swift:12:19: note: move here
        let val = _move(self).innerPtr
                  ^
test.swift:21:5: note: use here
    }
    ^
```

This property becomes even more important when working in the context of
throwing code. Consider the following code where we want to updateState, but
want to be able to throw:

```
struct SystemFrameworkStruct {
    var innerPtr: KlassState

    mutating func updateState() throws {
        // Grab the current inner ptr state off of the moved self.
        let val = _move(self).innerPtr

        // Now pass off our current inner ptr to the kernel, asking for
        // a potentially different pointer to updated state:
        let newValue = askKernelForNewPointer(val)

        // Call a throwing function to check if newValue is a safe value.
        try checkNewValAndThrowIfBad(newValue)

        // Reinitialize self here.
        self = SystemFrameworkStruct(innerPtr: newValue)
    }
}
```

Even though the code looks pretty reasonable, there is a hidden bug: the user
has not reinitialized self along the catch case of try causing a potential subtle
memory error if checkNewValAndThrowIfBad throws. By using `move` though, we get a
compile time guarantee that self will be reinitialized before exit, so the compiler
properly flags that we did not clean up self along the catch path:

```
test.swift:10:19: error: 'self' used after being moved
    mutating func updateState() throws {
                  ^
test.swift:12:19: note: move here
        let val = _move(self).innerPtr
                  ^
test.swift:19:44: note: use here
        try checkNewValAndThrowIfBad(newValue)
                                             ^
```

Another case where `move` comes in handy is when one wishes to yield a unique
view to the internal state of a data structure and reinitialize self upon
return. Consider the following code taken from the open source project WebURL:

```
// CopyOnWrite URLStorage
struct URLStorage { /* ... */ }
internal let _tempStorage = URLStorage()

public struct PathComponents {
    var storage: URLStorage

    mutating func addComponent() { /* ... */ }
}

public struct WebURL {
  var storage: URLStorage

  public var pathComponents: PathComponents {
    get { /* ... */ }
    set { /* ... */ }
    _modify {
      var view = PathComponents(storage: storage)
      storage = _tempStorage
      defer { storage = view.storage }
      yield &view
    }
  }
}
```

In this case, because the author does not have access to `move`, they must use a
global instance of the `URLStorage` class to enable it to move out its internal
storage while ensuring that `storage` stays unique. Additionally, the author
needs to be able to guarantee that they do not forget to re-update storage with
the potentially mutated `PathComponent` view. By using `move` as below, one can
guarantee all of these properties:

```
public struct WebURL {
  public var pathComponents: PathComponents {
    get { /* ... */ }
    set { /* ... */ }
    _modify {
      // Move out self ensuring view contains a unique URLStorage.
      var view = PathComponents(storage: _move(self).storage)

      // _move can look through defer and understand that it is going to
      // reinitialize self before the end of the function.
      defer { self = WebURL(storage: view.storage) }

      // Then we yield view. view may be mutated by our caller and the defer
      // and _move together ensure that self is reinitialize with the storage
      // from the inside of the view.
      yield &view
    }
  }
}
```
