+++
title = "Zipper Data Stucture in Unison"
date = "2023-04-02T00:07:39+02:00"
author = "Julian Dax"
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
draft = true
+++

In the last community meetup for the [Unison](https://www.unison-lang.org) programming language
we used and talked about Zipper data structures. In this post I'm going to explain what these
are and give an implementation for them in Unison. The tutorial is based on the one in
[Learn You a Haskell for Great Good](http://learnyouahaskell.com/zippers).

## The Problem

Zippers are data structures that are used to efficiantly navigate and change other data structures.
They make it way easier to work with the [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure)
common in functional languages like Unison, Haskell or Clojure. Zippers where intrduced to handle
trees, so the example used here is also a binary tree. Here is how to define a binary tree in Unison:

```
structural type BTree a = Leaf a | Node a (BTree a) (BTree a)

```

A binary tree consists of a lead