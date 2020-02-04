---
title: 'Trying to compose non-composable: lift everything!'
date: 2020-02-04
permalink: /posts/2020/02/joint
tags:
  - haskell
  - effect
  - composition
  - transformer
---

It's recommended to read the first part: [Trying to compose non-composable: joint schemes](https://iokasimov.github.io/posts/2019/11/joint), but if you're not interested in details of implementation - it's okay, in this article I’ll quickly go through the capabilities of my `joint` library.

Introduction
--------------------------------------------------------------------------------

According to [Stephen Diehl](http://www.stephendiehl.com/posts/decade.html), algebraic effects are one of the most important problems that will be solved in the future for Haskell and I would like to make my modest contribution.

Lift one effects to the others
--------------------------------------------------------------------------------

What does lifting mean, actually? Well, it's very similar to `pure/return`, but it makes not value effectful, but other effect:
```haskell
pure :: a -> t a
lift :: u a -> t u a
```

So we can use any effects in some transfomer, but later we have to interpret them all.

Thus, we can easily compose the lifted effects:
```haskell
let f = lift get :: Configured _ t => t _
let g = lift Nothing :: Optional t => t _
let h = lift (failure _) :: Failable _ t => t _

let x = f *> g *> h :: (Applicative t, Configured _ t, Optional t, Failable _ t) => t _
```

And represent them in any order we like:

```haskell
let x = f *> g *> h :: (Applicative t, Configured _ t, Optional t, Failable _ t) => t _

let y = pure _ :: Reader _ :> State _ :> Either _ :> Maybe := _
let z = pure _ :: State _ :> Either _ :> Maybe _ :> Reader := _

let xy = x *> y :: Reader _ :> State _ :> Either _ :> Maybe := _
let xz = x *> z :: State _ :> Either _ :> Maybe _ :> Reader := _
```

Effects adaptation
--------------------------------------------------------------------------------

Adaptation means that some effects can be replaced by more powerful effects. For example, `Reader` and `Writer` effects can be used in `State` because `State` can read and write, so it can modify stored value:

```haskell
lift put :: Accumulated _ t => t _
lift get :: Configured _ t => t _
(lift . adapt $ put) :: Stateful _ t => t _
(lift . adapt $ get) :: Stateful _ t => t _
```

How is that possible? In the [previous part](https://iokasimov.github.io/posts/2019/11/joint), we divided State into two effects:

```haskell
State s ~ (->) s :. (,) s
```

In case of `Reader` we just lift `->` (arrow) functor, in case of `Writer` - `,` (tuple):

```haskell
Reader s ~ (->) s
Writer s ~ (,) s
```

Also you can adapt `Failable` to `Optional` but we lost error information:

```haskell
(lift $ Just _) :: Optional t => t _
(lift $ failure _) :: Failable _ t => t _
(lift . adapt $ failure _) :: Optional t => t _
```

Simple real world example
--------------------------------------------------------------------------------

Let's imagine that we need to make an HTTP request, it's `IO`, that can throw `HttpException` :

![IO - salesman](http://iokasimov.github.io/images/io_salesman.png)

```haskell
import qualified "wreq" Network.Wreq as HTTP

request :: (Monad t, Liftable IO t, Failable HttpException t) => t (Response ByteString)
request = lift (try @HttpException $ HTTP.get link) >>= lift
```

Wow, what is there? First, we `lift` some `IO`-action, then after `>>=` we `lift` `Either` and we get an expression, that can be used in many effectful expressions than contain such two effects! We can delay using concrete transformers until we really need to evaluate them.

Conclusion
--------------------------------------------------------------------------------

No `free`/`freer` monad, no `GADTs` and other fancy stuff - all you need is a functor composition. If you want to be able to use your own effect you need to pick a joint schema and write several instances (`Functor`/`Applicative`/`Monad`).

The source code for `joint` is [there](https://github.com/iokasimov/joint). Thanks for reading, if you wanna try it or you have some questions - don't hesitate to PM me.
