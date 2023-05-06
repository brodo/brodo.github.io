+++
title = "How (Not) to Subclass Errors in JavaScript"
date = "2023-05-05T21:19:33+02:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["JavaScript", "Errors"]
keywords = ["", ""]
description = "A Tale of Cargo Cult Programming"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## TL/DR

If you want to create a subclass of `Error` use `class` syntax:

```JavaScript
class MyError extends Error {
    constructor(){
        super(...arguments);
        this.name = "MyError";
    }
}

```
- The `Error` constructor does **absolutely nothing** to its `this` argument. It returns a completely new object.
- Creating `Error` instances costs CPU because a stack trace is created when the `Error` constructor is called.
- Many popular JavaScript libraries do this wrong.

## How not to subclass Errors

In order to see how not to create a subclass of Error, let’s take a look at Axios’s
[`AxiosError`](https://github.com/axios/axios/blob/21a5ad34c4a5956d81d338059ac0dd34a19ed094/lib/core/AxiosError.js):

```javascript
function AxiosError(message, code, config, request, response) {
    Error.call(this);

    if (Error.captureStackTrace) {
        Error.captureStackTrace(this, this.constructor);
    } else {
        this.stack = (new Error()).stack;
    }

    this.message = message;
    this.name = 'AxiosError';
    code && (this.code = code);
    config && (this.config = config);
    request && (this.request = request);
    response && (this.response = response);
}
```

Let's go through this line by line. This is an old-school Javascript [constructor function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain#constructors)
meant to be called with `new` in front of it. In the second line - `Error.call(this)` - the following happens:

1. The `Error` constuctor is called with the newly created `this` object of `AxiosError`.
2. The `Error` constuctor creates a stack trace, which is quite a costly thing to do.
3. The `Error` constuctor ignores it's `this` and returns a brand new `Error` object.
4. The newly created `Error` object is not saved anywhere and will be deleted the next time the garbage collector runs.

Who ever wrote this obviously assumed that the `Error` constructor would do something to it's `this` object.
I have found code like this in many popular Javascript libraries. Just `grep` your `node_modules` folder for
`Error.call(this)` and you will find plenty. [Sourcegraph finds more than 133K results in over 10k repos](https://sourcegraph.com/search?q=context:global+%28language:JavaScript+OR+language:TypeScript%29+content:%22Error.call%28this%29%22+count:1000000&patternType=standard&sm=0&groupBy=repo)
for both Typescript and Javascript if you search for it. I don't know if this ever did work but I doubt it. I've checked
Node versions back until 6, Spidermonkey and JavaScriptCore. None of them do anything to the object the
`Error` constructor is called with. Moreover, [the specification](https://262.ecma-international.org/13.0/#sec-error-constructor)
says that you have to be able to call the `Error` constuctor without `new`, so it makes sense to implement it
that way. This is a a case of [cargo cult programming](https://en.wikipedia.org/wiki/Cargo_cult_programming).
Someone wrote this somewhere and people copied it without checkig if it works.

Now let's go over the `if(Error.captureStackTrace)`. This checks if the V8-only `Error.captureStackTrace` method exists.
It allows you to manually create a stack trace and adds it as a `stack` property to it's first parameter. The second parameter
is a function which should not show up in the stack trace. This way, if we throw an `AxiosError` the top of the stack trace
is the function we have thrown the error in and not `AxiosError` itself. (I've recently improved the documentation of this function in
the [Node.js documentation](https://nodejs.org/dist/latest-v20.x/docs/api/errors.html#errorcapturestacktracetargetobject-constructoropt), if you want to know more details.)
So if we are in a V8 engine, this will add a `stack` property to the `this` object.
Bear in mind, adding this stack trace is prosumably what people had in mind when they came up with `Error.call(this)`.
So this is the second **expensive** stack trace we do. If we are not under V8 we also create a second stack trace (this time with
the `AxiosError` on top) and add it to `this`. The rest of the code is sensible, `message` and `name` should be defined.

The next problem with this code is that `AxiosError` (and errors like it) are hard to detect. [I wrote a whole other post about this](../detect-js-errors),
but the gist of it is this: `AxiosError` is not returned by one of the built-in error constuctors and therefore harder to detect then
necessary. `AxiosError`s prototype is set to the correct object later in the code, so `instanceof Error` will work in most cases.
