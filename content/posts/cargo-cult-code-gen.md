+++
title = "Cargo Cult Code Generation"
date = "2023-05-05T21:19:33+02:00"
author = "Julian Dax"
authorTwitter = "" #do not include @
cover = ""
tags = ["JavaScript", "ChatGPT", "Copilot", "Errors"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## Javascript Errors in the Wild

Recently, I came across this code in the popular [Axios library](https://github.com/axios/axios/blob/21a5ad34c4a5956d81d338059ac0dd34a19ed094/lib/core/AxiosError.js):

```javascript
function AxiosError(message, code, config, request, response) {
    Error.call(this);
    // some more code i've left out for brevity
}
```

What does `Error.call(this)` do, you might ask? Nothing! I've tried it out in
Node versions back until 6, Spidermonkey and JavaScriptCore. None of them do anything to the `this` object of the
`Error` constructor. Moreover, [the specification](https://262.ecma-international.org/13.0/#sec-error-constructor)
says that you have to be able to call the `Error` constructor without `new`, so it makes sense to implement it
that way. Depending on how good you know Javascript Errors, you might assume that this call might:

1. Turn `this` into an `instanceof Error` - It does not.
2. Turn `this` into a [native error](https://nodejs.org/dist/latest-v20.x/docs/api/util.html#utiltypesisnativeerrorvalue) - It does not.
3. Add a `stack` property to `this` - It does not.

`Error.call(this)` does absolutely nothing to `this`. Zero, zilch, nada. The only thing it does is **waste a lot of CPU time**.
It creates a new `Error` object and a new stack trace - a costly thing to do - and then lets the GC throw it away.

This is a a case of **[cargo cult programming](https://en.wikipedia.org/wiki/Cargo_cult_programming)**:
> Cargo cult programming is a style of computer programming characterized by the ritual inclusion of code or program structures that serve no real purpose.


I don't want to hate on Axios through, a ton of popular libraries do this! I've `grep`ed through my `node_modules` folder and
found it in [rxjs](https://github.com/ReactiveX/rxjs/blob/6e3e5e49cb735289a8e8796dc58e682468a36028/src/internal/util/createErrorClass.ts)
,[jsonwebtoken](https://github.com/auth0/node-jsonwebtoken/blob/a99fd4b473e257c2f50ff69c716db1c520bf9a78/lib/JsonWebTokenError.js), [nock](https://github.com/nock/nock/blob/76a8df2562536a583c60872665172cc4d965994b/lib/intercept.js#L24) and [js-yaml](https://github.com/nodeca/js-yaml/blob/0d3ca7a27b03a6c974790a30a89e456007d62976/lib/exception.js#L27) (which comes bundled with Node.js).

[Sourcegraph finds more than 133K results in over 10k repos](https://sourcegraph.com/search?q=context:global+%28language:JavaScript+OR+language:TypeScript%29+content:%22Error.call%28this%29%22+count:1000000&patternType=standard&sm=0&groupBy=repo)
for both Typescript and Javascript if you search for `Error.call(this)`.[^1]

## Cargo Cult Copilot

If Github Copilot is trained on this, will it generate code with `Error.call(this)` in it? I've tried it by typing
`function CachingError` and let it generate from there:

```Javascript
function CachingError(message) {
    this.name = 'CachingError'
    this.message = message
    this.stack = (new Error()).stack
}
```
This can be done better, but no `Error.call(this)`. On my second try I got the following result:

```Javascript
function CachingError() {
    Error.apply(this, arguments);
}
```
There we are! Using `.apply` instead of `.call` and passing on the `arguments` array would even be a good idea
if the `Error` constructor would do something to it's `this`.


## CargoGPT4

What about ChatGPT? I've tried this prompt with GPT4:

{{<chat person="Julian">}}

Write me an Error subclass called `CachingError` in Javascript using prototypal inheritance.

{{</chat>}}

{{< chat person="ChatGPT" >}} Here's an example of a CachingError subclass in JavaScript using prototypal inheritance:

```JavaScript
// Define the CachingError constructor
function CachingError(message) {
    // Call the Error constructor with the given message
    Error.call(this, message);

    // Set the error name to 'CachingError'
    this.name = 'CachingError';

    // Capture the current stack trace and store it in the error object
    Error.captureStackTrace(this, CachingError);
}

// Set up the prototype chain between CachingError and Error
CachingError.prototype = Object.create(Error.prototype);
CachingError.prototype.constructor = CachingError;

// Example usage
try {
    throw new CachingError('An error occurred while caching data');
} catch (error) {
    console.error(error.name); // CachingError
    console.error(error.message); // An error occurred while caching data
    console.error(error.stack); // Stack trace
}
```
{{< /chat >}}



On the first try!

## LoRas to the Rescue!

The central problem with AI code generation is [Sturgeon's law](https://en.wikipedia.org/wiki/Sturgeon%27s_law):
> Sturgeon's law (or Sturgeon's revelation) is an adage stating "ninety percent of everything is crap".

90% of code is crap. That means instead of having gigantic models trained on everything that people
can get their hands on, use smaller models and train with better code. In the Stable Diffusion community, people train
[textual inversions](https://arxiv.org/abs/2208.01618) with 5 images and [LoRas](https://arxiv.org/abs/2106.09685) with ~20.
There needs to be a "Javascript Error LoRa" in ChatGPT and Copilot that fixes this.


## How to Do It Properly

If you want to create a subclass of `Error` use `class` syntax:

```JavaScript
class MyError extends Error {
    constructor(){
        super(...arguments);
        this.name = "MyError";
    }
}

```

This way, your error will be a [native error](https://nodejs.org/dist/latest-v20.x/docs/api/util.html#utiltypesisnativeerrorvalue), `instanceof Error` and have a `.name` property that differentiates it from other errors.

If you want to continue this deep dive, check out my [post]({{< ref "posts/detect-js-errors" >}}) on how to detect if an object is an error, parts of which made it into the official Node.js documentation.

[^1]: I'm sure there are a lot for false positives and false negatives (`Error.apply`...) in this search, but you get the idea.



