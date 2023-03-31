+++
title = "Pretty Printing Objects in Javscript REPLs"
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

In the Node.js REPL, objects are converted to strings using the [`util.inspect()`](https://nodejs.org/dist/latest-v19.x/docs/api/util.html#utilinspectobject-options) function.
In Deno, there is the equivalent [`Deno.inspect()`](https://deno.land/api@v1.32.1?s=Deno.inspect).
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

// Helper function to generate a balanced binary tree of a given depth
export function createBalancedTree(depth) {
    if (depth === 0) {
        return null;
    }
    const value = Math.floor(Math.random() * 100);
    const left = createBalancedTree(depth - 1);
    const right = createBalancedTree(depth - 1);
    return new Tree(value, left, right);
}

```

And here is a Deno session that uses it:

```
> import {Tree, createBalancedTree} from "./Tree.js"
let tree = createBalancedTree(5);
undefined
> tree
└── 49
    ├── 25
    │   ├── 44
    │   │   ├── 2
    │   │   │   ├── 58
    │   │   │   └── 84
    │   │   └── 85
    │   │       ├── 71
    │   │       └── 25
    │   └── 6
    │       ├── 89
    │       │   ├── 74
    │       │   └── 23
    │       └── 18
    │           ├── 78
    │           └── 29
    └── 55
        ├── 54
        │   ├── 79
        │   │   ├── 36
        │   │   └── 66
        │   └── 14
        │       ├── 69
        │       └── 51
        └── 81
            ├── 58
            │   ├── 1
            │   └── 67
            └── 29
                ├── 93
                └── 98
```


Isn’t that beautiful?
