---
title: 'Cofree will tear us apart'
date: 2018-05-31
permalink: /posts/2018/05/cofree-will-tear-us-apart
tags:
  - haskell
  - cofree
  - apart
---

Recently I've been working on distributed systems and I often need to deal with the data that may be located anywhere. We'll see how just one algebraic construction helped to describe a problem with recycling data in general and then to solve it.

I've created a library [apart](https://github.com/iokasimov/apart), whose definitions are used in this post.

Introduction and motivation
--------------------------------------------------------------------------------

Let's suppose you need to write a program that works with some data. If it's too big, you should think about its managing and storing because keeping it in RAM is too dangerous. We have to split these data into the segment that we need right now and the rest that we don't.

How we're going to do that? First of all, we need an abstraction that allows us to describe various data structures algebraically. And such an abstraction exists!

Basic construction `Cofree`
--------------------------------------------------------------------------------

Many Haskellers know `Free` type, but `Cofree` is unfairly bypassed. The difference is really small - while `Free` is a sum of `a` and `t (Free t a)`, `Cofree` is a product:

```haskell
Free t a = a + t (Free t a)
Cofree t a = a * t (Cofree t a)
```
If we pick `Cofree` as basic construction, we'll get those functions for free:
* `extract` - Get a focused value, so our structure can't be empty
* `unwrap` - Get a segment of structure without focused value
* `extend` - Update values with function that takes care of context
* `coiter` - Generate structure from seed

So, how we're going to build various data structures with `Cofree` help? All we need is a `Functor`, to be more precise, we need to instantiate `t` type in `Cofree t a`.

Suppose we need a stack or non-empty list - fairly simple structure. We can take `Maybe`, cause it has everything we need: `Just` lets you continue describing a structure and `Nothing` lets you terminate an expression (cause it's a null constructor).

```haskell
type Stack = Cofree Maybe

stack :: Stack Int
stack = 1 :< Just (2 :< Just (3 :< Nothing))
```

Helper construction `Shape`
--------------------------------------------------------------------------------

Okay, now we know how to describe structures algebraically via `Cofree`. But we talked about separation of data that is located in different places. So we'll just add one more construction:

```haskell
data Shape t raw value = Ready (t value) | Converted raw

data Apart t raw value = Apart (Cofree (Shape t raw) value)
```
`Ready` constructor describes values in RAM, `Converted` describes data located somewhere else. `Cofree` and `Shape` types can give us wonderful type `Apart`.

Simple example of usage `Apart`
--------------------------------------------------------------------------------

Imagine that we want to work with binary tree. How can we describe it via `Cofree`? We need a `Functor` of branching here. A node of a binary tree can either have left child, right child, both of them or no one. Let's write it down.

```haskell
data Crotch a = End | Less a | Greater a | Crotch a a

type Binary = Cofree Crotch
```

Example of binary tree we can get from Wikipedia page:

![Alt text](https://upload.wikimedia.org/wikipedia/commons/d/da/Binary_search_tree.svg)

```haskell
example :: Binary Int
example = 8 :< Crotch
	(3:< Crotch
		(1 :< End)
		(6 :< Crotch
			(4 :< End)
			(7 :< End)))
	(10 :< Greater
		(14 :< Less
			(13 :< End)))
```

Suppose we need to keep in memory this tree with height not greater than 3. We can [`limit`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Combinators.hs#L23) that.

```haskell
limit 3 do_something_with_the_rest example
```

I'm intentionally not focusing on the way to persist the segments - it's not a goal of this post. We can persist out of range segment in file and function `do_something_with_the_rest` can return filename and line number. Or we can put them in `Redis`/`Memcashed`/`Tarantool` and return connection parameters and keys for the saved segments. It doesn't matter.

```haskell
8 :< Crotch
	(3 :< Crotch
		(1 :< {RESTORE_INFO})
		(6 :< {RESTORE_INFO}))
	(10 :< Greater
		(14 :< {RESTORE_INFO}))
```
So we cut off our tree and put information for restoring where these unnecessary segments were. With [`recover`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Combinators.hs#L16) we can put back to memory the whole structure.

```haskell
recover back_to_RAM scattered
```

Or, we can traverse with effect on scattered data structure and restore all segments with applying this function to them as well - just use [`fluent`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Combinators.hs#L31).

```haskell
fluent do_something_wherever_they_are scattered
```

Conclusion
--------------------------------------------------------------------------------

At this moment, I've successfully defined [`acyclic oriented graphs`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Structures/Graph.hs), [`binary`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Structures/Tree/Binary.hs), [`prefix`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Structures/Tree/Prefix.hs),
[`rose`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Structures/Tree/Rose.hs),
[`AVL`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Structures/Tree/Binary/AVL.hs) and
[`Splay`](https://github.com/iokasimov/apart/blob/master/Data/Apart/Structures/Tree/Binary/Splay.hs) trees. And some functions to deal with them.

The idea of using `Cofree` as base construction for another structures was captured from description module [`Control.Comonad.Cofree`](https://hackage.haskell.org/package/free-5.0.2/docs/Control-Comonad-Cofree.html) in package [`free`](https://hackage.haskell.org/package/free) of Edward Kmett.

The idea of algebraic graphs was used from [paper of Andrew Mokhov](https://doi.org/10.1145/3122955.3122956).

But yet there is a lot to do, actually:
* Try to define Finger tree, Scapegoat, ART and another complicated structures.
* Check compatibility with streaming libraries ([`pipes`](http://hackage.haskell.org/package/pipes), [`conduit`](http://hackage.haskell.org/package/conduit), [`io-streams`](http://hackage.haskell.org/package/io-streams), [`machines`](http://hackage.haskell.org/package/machines)).
* Natural transformations between structures (cause of `Functor`).
* Benchmarks, with popular [`containers`](http://hackage.haskell.org/package/containers), for the first.
