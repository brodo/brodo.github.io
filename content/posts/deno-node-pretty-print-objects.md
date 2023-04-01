+++
title = "String Representations of Javascript Objects"
date = "2023-04-01T00:35:30+02:00"
author = "Julian Dax"
authorTwitter = "" #do not include @
cover = ""
tags = ["Javascript", "Node.js", "Deno"]
keywords = ["Javascript", "Node.js", "Deno"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## `toString()` and `Symbol.toStringTag`

The default `Object.prototype.toString()` function is
[defined to the specification](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.prototype.tostring)
to return `[object X]` even for non-objects:

```Javascript
Object.prototype.toString.call(undefined) // returns "[object Undefined]"
Object.prototype.toString.call(true) // returns "[object Boolean]"
{}.toString() // returns "[object Object]"
```

`X` is a so-called 'string tag' and in newer versions of Javascipt, you can set it yourself:

```Javascript
{[Symbol.toStringTag]:"MyObject"}.toString() // returns "[object MyObject]"
```

But for objects, you can also just implement the complete `toString` function:

```Javascript
{toString: ()=> "My Cool Object!"}.toString() // returns "My Cool Object!"
```

**However, the `toString()` function is not used by `console.log()` or in the Node.js or Deno REPLs.**

## `inspect()` is the real `toString()`

In Node.js, objects are converted to strings using the [`util.inspect()`](https://nodejs.org/dist/latest-v19.x/docs/api/util.html#utilinspectobject-options) function.
In Deno, there is the equivalent [`Deno.inspect()`](https://deno.land/api@v1.32.1?s=Deno.inspect). These methods are used
both in the REPLs and when using `console.log()`.
You can influence what these do by implementing a function with the symbol `nodejs.util.inspect.custom` and
`Deno.customInspect` respectively. If you implement both, pretty printing works in both.
Check out this example for a binary tree:


```Javascript
export class Tree {
    constructor(value, left, right) {
        this.value = value;
        this.left = left;
        this.right = right;
    }

    [Symbol.for("Deno.customInspect")]() {
        return this.toString();
    }
    [Symbol.for("nodejs.util.inspect.custom")](){
        return this.toString();
    }
    toString() {
        const printNode = (node, prefix = '', isLeft) => {
            let str = '';
            if (node !== null) {
                str += `${prefix}${isLeft ? '├──' : '└──'} ${node.value}\n`;
                const newPrefix = `${prefix}${isLeft ? '│   ' : '    '}`;
                str += printNode(node.left, newPrefix, true);
                str += printNode(node.right, newPrefix, false);
            }
            return str;
        };
        return printNode(this).trim();
    }
}

```
And here is a Deno session that uses it:

```
> tree
└── 52
    ├── 41
    │   ├── 51
    │   └── 44
    └── 94
        ├── 54
        └── 36
> console.log(tree);
└── 52
    ├── 41
    │   ├── 51
    │   └── 44
    └── 94
        ├── 54
        └── 36
```

Isn’t that beautiful? Sadly there is no way to accomplish this in browsers.
