+++
title = "How to detect Errors in JavaScript"
date = "2023-03-31T19:16:51+02:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["JavaScript", "Errors"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## TL/DR

Finding out if an object is an error is unreliable and full of edge-cases. A modern,
[isomorphic](https://en.wikipedia.org/wiki/Isomorphic_JavaScript) way to
do it is the following[^1]:

```JavaScript
function isError (object) {
    return object instanceof Error || (
    object?.constructor?.name === 'Error' ||
    object?.constructor?.name === 'DOMException'
    )
}
```

Many custom error classes in the JavaScript ecosystem are hard to detect
because creating custom error classes used to be hard before the introduction
of the `class` syntax.


## Part One: WTF is an error anyway?

In JavaScript different things can be meant by "error". This part
discusses the different kinds of errors.

### Native Errors

The [JavaScript
specification](https://tc39.es/ecma262/#sec-error-objects) defines a
number of "Native Errors" which are all `instanceof Error`. According to
the specification, these errors are marked in a way that is not
accessible from JavaScript itself[^2]. However, in practice they can be
[detected reliably](#is-native-error) in Node.js and [unreliably in the
browser](#to-string).

### DOMExceptions


`DOMException`s[^3] are errors that are thrown when an error
happens in one of the web APIs. It is defined in the [DOM
specification](https://www.w3.org/TR/DOM-Level-3-Core/core.html#ID-17189187).
DOMExceptions are no native errors and are not `instanceof Error`.
The `DOMException` class does exist both in the browser and in Node.js.


### Instances of `Error`

There are a lot of custom errors defined in popular libraries that are
not native but still instances of `Error`. This happenes if the
prototype of the error object is manually set to `Error.prototype`.


### Objects with a constructor named "Error"


If an object crosses a [realm boundary](https://tc39.es/ecma262/#realm)
[^4], it is not instance
of anything in the current realm. However, the name of the constructor
is still `"Error"`.

### Objects with the `string` properties `name` and `message`

For TypeScript, an `Error` is any object that conforms to [this
interface](https://github.com/microsoft/TypeScript/blob/4b6fb95f040c5c81743e19a56df8fa99bf3d139f/lib/lib.es5.d.ts#L1052):

```TypeScript
interface Error {
    name: string;
    message: string;
    stack?: string;
}
```

The optional `stack` property is not defined in the standard but present
in every modern JavaScript engine.

## Part Two: How to Check for Errors

### Detecting Native Errors

Clever people came up with a way to check for native errors by using the
`Object.prototype.toString()` method:

```JavaScript
Object.prototype.toString.call(new Error()) // returns '[object Error]'
Object.prototype.toString.call({}) // returns '[object Object]'
```


Objects marked as errors are *used to be* the only ones for which
`Object.prototype.toString.call()` would return `[object Error]` instead
of `[object Object]`. However, this is no longer the case since the
introduction of
[`Symbol.toStringTag`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/toStringTag):

```JavaScript
const fakeError = {[Symbol.toStringTag]: 'Error'};
Object.prototype.toString.call(fakeError) // returns '[object Error]'
```

Therefore, there can be *false positives* when checking with
`Object.prototype.toString.call()`.

There are two other reasons for not using this method: firstly, using
`Object.prototype.toString.call()` for type testing is discouraged [in
the
specification](https://tc39.es/ecma262/#sec-object.prototype.tostring).
And secondly, many custom errors and `DOMException` will not return
`[object Error]` when checked like this:

```JavaScript
// I'm using AxiosError as an example for an error that is not a native error.
// I don't want to pick on Axios; this is very common.
import {AxiosError} from "axios";
Object.prototype.toString.call(new AxiosError()); // returns '[object Object]'
Object.prototype.toString.call(new DOMException()); // returns '[object DOMException]'
```
Thus, *False negatives* will also happen quite regularly when using this
method.

Because of all these problems, the Node.js developers decided to expose
the V8-internal `IsNativeError()` function as
[`util.types.isNativeError()`](https://nodejs.org/api/util.html#utiltypesisnativeerrorvalue).
This gives you a reliable way to check if a value is marked as an error.
However, since many custom error values are not marked, this way of
checking will still produce many false negatives.

```JavaScript
util.types.isNativeError({[Symbol.toStringTag]: 'Error'}) // returns false
util.types.isNativeError(new Error()) // returns true
```

**I recommend not checking if an error is native. There are too many
false negatives and cross-realm checking can be done using constructor
names.**

### Detecting `Error` instances

Another way of checking if something is an error is using the
`instanceof` operator. It is quite common for errors to be
`instanceof Error` but not native, so this will catch more errors than
the methods described above. It also finds `DOMException`s as they are also
`instanceof Error`. It is also faster than most other methods as `instanceof`
is quicker than property access. However, as already mentioned, this does
not work across realms:

```JavaScript
(new Error()) instanceof Error; // returns true
(new DOMException()) instanceof Error; // returns true
const context = vm.createContext({});
const myError = vm.runInContext('new Error()', context); // creates an error in a different realm
myError instanceof Error; // returns false
util.types.isNativeError(myError); // returns true
```
To detect errors from other realms we can check the constructor name.
This may lead to false positives though:

```JavaScript
(new Error()) instanceof Error; // returns true
const context = vm.createContext({});
const myError = vm.runInContext('new Error()', context); // creates an error in a different realm
myError.constructor.name === "Error"; // returns true
{constructor: {name: "Error"}}.constructor.name === "Error"; // returns true
```


However, the example for the false positive is contrived and if people
really want to fake an error, they probably have reasons for it.

### Checking for `message` and `name`

If TypeScript's error definition is good enough for you, this type guard
will do the job:

```TypeScript
function isError(e: unknown): e is Error {
    return typeof e === "object" &&
    (typeof e["name"] === "string") &&
    (typeof e["message"] === "string")
}
```


This is a pretty loose definition however, and one can easily imagine
real-world examples for objects with these properties that are not
errors. So why not also check for `stack`? The problem with that is that the string representation of that stack
trace is created lazily when it is accessed. So checking for the `stack` attribute to find out if
something is an `Error` is about 8.5 times slower[^5] than just checking `name` and `message`.

### Summary Table

In the following table, more ✓s are always better:

| Method                            | No False Positives | No False Negatives | Is Isomorphic | Not Deprecated |
|-----------------------------------|--------------------|--------------------|---------------|----------------|
| Object.prototype.toString.call()  | ⤬                  | ⤬                  | ✓             | ⤬              |
| util.types.isNativeError()        | ✓                  | ⤬                  | ⤬             | ✓              |
| instanceof Error                  | ✓                  | ⤬                  | ✓             | ✓              |
| X.constructor.name === "Error"    | ⤬                  | ✓                  | ✓             | ✓              |
| checking for message and name     | ⤬                  | ✓                  | ✓             | ✓              |



## Using Combinations of several checks

*Using combinations of the methods described above is generally the way
to go.* Node.js itself uses [this
function](https://github.com/nodejs/node/blob/3c0131a4190a88211780dcc07dbaf84c8de97f34/lib/internal/util.js#L95)
for checking if something is an `Error` internally:

```JavaScript
function isError(e) {
    // An error could be an instance of Error while not being a native error
    // or could be from a different realm and not be instance of Error but still
    // be a native error.
    return isNativeError(e) || e instanceof Error;
}
```

While this is better than using only one of the methods, it still can
produce false negatives, if the error was created in another realm and
is not a native error:

```JavaScript
const context = vm.createContext({});
const myError = vm.runInContext('new AxiosError()', context); // creates an error in a different realm
isError(myError); // returns false
```

Node.js's deprecated
[`util.isError()`](https://nodejs.org/api/util.html#utiliserrorobject)
function uses a combination of `instanceof` and
`Object.prototype.toString.call()`. This makes it deprecated but
isomorphic.

```JavaScript
function isError(e) {
    return Object.prototype.toString.call(e) === '[object Error]' ||
    e instanceof Error;
}
```

And finally, as already mentioned above, undici uses `instanceof` and
constructor names. This makes it work across realms and does not use
Node.js-only functions. It will also produce less false positives than
just checking for the `name` and `message` properties. It is also fast
because the `||` short-circuits it to an `instanceof` test in many cases:

```JavaScript
function isError (object) {
    return object instanceof Error || (
    object?.constructor?.name === 'Error' ||
    object?.constructor?.name === 'DOMException'
    )
}
```

[^1]: This code is taken from
[undici](https://github.com/nodejs/undici/blob/4885b11dd60b4d1a785c4e5a519ad87920549d1c/lib/fetch/util.js#L75).
[^2]: An `[[ErrorData]]` [internal
slot](https://tc39.es/ecma262/#sec-object-internal-methods-and-internal-slots)
is used for this.
[^3]: `DOMException`s don't fit into the JavaScript naming scheme for errors. In JavaScript an exception is what happens if an
error is thrown.
[^4]: A realm is a JavaScript execution context which has its own global environment. If an error comes from another realm, its prototype points
to the `Error` prototype of the realm it was created in. Thus,`instanceof Error` will return `false`. 
[^5]: I did some micro benchmarking to find this out. The exact number may depend on the use-case but it is **a lot** slower in any case.

