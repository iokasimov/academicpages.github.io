---
title: 'A story of an arrow and a comma'
date: 2020-10-31
permalink: /posts/2020/10/arrow-and-comma
tags:
  - haskell
  - functors
  - adjunctions
  - state
  - store
  - lens
---

In this article I’m going to show you what are adjoint functors about through real programming examples. We will build a lot of useful stuff just with two simplest constructions: an arrow and a comma.

![](http://iokasimov.github.io/images/zwqr80uhebtcuamsndizfraxhru.png)

I will deliberately avoid full definitions, axioms and proofs of category theory - I don’t want to copy text from Wikipedia or books. But if you really want to understand these concepts, I strongly recommend  [this blog](https://www.math3ma.com/categories/category-theory) cause it helped me a lot. I want to show you how related programming constructions via math abstractions. And there will be Haskell code.

# Functions and tuples
Let’s start from simple constructions that should exist in all general-purpose languages.

What is a function? It’s a map between two sets, it’s a code which takes some argument **s** and returns some result **a**: **s -> a**.

![](http://iokasimov.github.io/images/qxaxqp3znnibq8pitzvc3xwqjxk.png)

What is a tuple? It’s elementary product of any two types **s** and **a**: **(s, a)**.

![](http://iokasimov.github.io/images/qzgifnlnr_gwt02ldlym5zrd5gq.png)

We get use to look at these two constructions as infix expressions. What if we change our point of view and look at them as prefix expressions? In other words, we pull out operands after operator.

![](http://iokasimov.github.io/images/pqlrbhdaxtdbz0bvxp54ebyl4t4.png)
![](http://iokasimov.github.io/images/fpny8njlpfnss73h_qwxiwuipos.png)

Now we see that they are pretty similar! They take two arguments in the same order. But there is a difference: first argument of arrow (**s**) is in negative position, but first argument of tuple (**s**) is in positive (which makes an arrow **profunctor** and a comma **bifunctor**) . So let’s just ignore those two arguments and just fix them:

![](http://iokasimov.github.io/images/zdckeyvi125yn2ycwmtbztvisgy.png)

![](http://iokasimov.github.io/images/wegmqssuvduiuo_pqj-lk9iqhre.png)

Now they are covariant functors and, also they are in such an interesting relation as **adjunction**!

![](http://iokasimov.github.io/images/rm9ne9die4hnny0_eupyq8yrpoe.png)

But let’s take one thing at a time.

# Categories, functors, adjunctions
Category is a set of objects and arrows:

![](http://iokasimov.github.io/images/bb38146cfcad387787dc419b9f0c27f0.jpg)
_«What is a Category? Definition and Examples» © Math3ma_

Functors are maps between categories:

![](http://iokasimov.github.io/images/25796b599438eb261fa1e9546fc78507.jpg)
_«What is a Functor? Definition and Examples, Part 1» © Math3ma/_

**Adjunction** is a special relation between **functors**. That it, if we can build two such commutative diagrams and all equations hold, then we can say that F and G are **adjoint functors** (**F ⊣ G**):

![](http://iokasimov.github.io/images/724095e1f8d2d8797c295b3ed52e9f33.jpg)
_«What is an Adjunction? Part 2 (Definition)» © Math3ma_

Maybe it looks pretty complicated now but as soon as we get used these concepts on practice we will make sure that this is not so.

# Adjunction of a comma and an arrow
This is how **Functor** definition usually looks like in languages with parametric polymorphism:

![](http://iokasimov.github.io/images/ytoxdiirpssum0wxiakj7bhwyzs.png)

In case of a tuple, first argument is fixed therefore we can change only second one with function:

![](http://iokasimov.github.io/images/os9obc76wesm2iv_edn9nnwnds0.png)

In case of a function, **fmap** definition is just a function composition:

![](http://iokasimov.github.io/images/xc42x5qucuofsgwxndsxonwoawk.png)

Now it’s time to define **adjunction** between tuple and function and … it looks pretty complicated.

![](http://iokasimov.github.io/images/17mehpicsggrd3wpyq_ht83fhry.png)

So instead of these methods, let’s define something simpler - **unit** and **counit**. Do you remember those two commutative diagrams? Morphism **ε** corresponds to **counit** and morphism **η** corresponds to **unit**:

![](http://iokasimov.github.io/images/aigd2fgsbwpakxtz5ia-gkuaugi.jpg)
_«What is an Adjunction? Part 2 (Definition)» © Math3ma_

Let’s start from **counit**:
```haskell
ε :: f (g a) -> a
-- Replace f and g with comma and arrow:
ε :: (((,) s) ((->) s a)) -> a
-- Disclose paretheses for comma - from prefix to infix:
ε :: (s, ((->) s a)) -> a
-- Disclose paretheses for arrow - from prefix to infix:
ε :: (s, s -> a) -> a
```

What we got here? **counit** for a comma and an arrow takes some argument **s** and function from  **s** to **a**. Looks like function application!
```haskell
ε :: (s, s -> a) -> a
ε (x, f) = f x
```

Now it’s turn for `unit`:
```haskell
η :: a -> g (f a)
-- Replace f and g with comma and arrow:
η :: a -> ((->) s ((,) s a))
-- Disclose paretheses for arrow - from prefix to infix:
η :: a -> s -> ((,) s a)
-- Disclose paretheses for comma - from prefix to infix:
η :: a -> s -> (s, a)
```

So `unit` takes **s** and **a** as arguments and build a tuple from them:

```haskell
η :: a -> s -> (s, a)
η x s = (s, x)
```

Now let’s go back to those scary methods:

![](http://iokasimov.github.io/images/ollwlgpudwt1rojw8m6vynirkfc.png)

And implement **left adjunction** and **right adjunction**:

![](http://iokasimov.github.io/images/bdf_bu7baurhrn44suif-1fnrum.png)

Well, it didn’t get any easier. Sometimes, trying to understand something we have no choice except looking at this very carefully. We can do it now.

**Left adjunction**: takes a function which takes a tuple  (**(s, a) -> b** ) and as result we have a function which takes **a** and **s** as arguments:

![](http://iokasimov.github.io/images/n-0nrufn_ozxc658l9o5enbeog8.png)

**Right adjunction**: takes a function which return another function (**a -> (s -> b)**)  and as result we have a function with takes a tuple **(s, a)**:

![](http://iokasimov.github.io/images/lc0ofonefnbxncp9gcy24tj2wve.png)

Does this remind you of anything? That is **currying**!
```haskell
curry :: ((a, s) -> b) -> a -> s -> b
uncurry :: (a -> s -> b) -> (a, s) -> b
```
_The only difference is reverse order of arguments_

So if you are asked one day ”what is currying” you can answer: ”This is just left adjunction of tuple functor and function functor, what’s the problem?”

# Functor combinatorics
There is an idea: let’s try to create simple **functor compositio**n of an arrow and a tuple. Simple **functor composition** is just a wrapping one functor into another one, they are polymorphic over type argument anyway. We can do it in two ways:

![](http://iokasimov.github.io/images/e7qcjoneatbms4fgrdafgs1g7ge.png)

Substitute **f** and **g** with comma and arrow (with fixed arguments) accordingly:

![](http://iokasimov.github.io/images/wfwqzbn_r38kvmw6kijluv_oosk.png)

Looks familiar, right? That it, with first combination we get **Store** comonad and with reverse combination we get **State** monad:

![](http://iokasimov.github.io/images/4ioxqbqhwxhn3nvta6_xzdcl4s8.png)

![](http://iokasimov.github.io/images/jaa7luz2yre4mzshbgkzcpr51ji.png)

And they are **adjoint functors** also!

![](http://iokasimov.github.io/images/n4vxsmkaypec_3kmmtopibzzo9y.png)

# State and Store
What do we think about when we create program which depends not only on arguments but on some state? Maybe we think about a box where we can put and get something from. Or in case of finite-state machines, we are focusing on state transitions:

![](http://iokasimov.github.io/images/itnexbfdvl2idx0ljpmjayohina.jpg)

Well, for transitions we have arrows and for a box we have tuple:

```haskell
type State s a = s -> (s, a)

modify :: (s -> s) -> State s ()
modify f s = (f s, ())

get :: State s s
get s = (s, s)

put :: s -> State s ()
put new _ = (new, ())
```

**Store** is something completely different. We store some source **s** and an instruction how to get **a** from that **s**:

```haskell
type Store s a = (s, s -> a)

pos :: Store s a -> s
pos (s, _) = s

peek :: s -> Store s a -> a
peek s (_, f) = f s

retrofit :: (s -> s) -> Store s a -> Store s a
retrofit g (s, f) = (g s, f)
```

Where can it be used?

# Focusing with lens
Sometimes we have to work with deeply nested data structures, focusing on some details. Imagine you look trough magnifying glass at your data - all other things just disappear from your sight.

![](http://iokasimov.github.io/images/0r1mjuhncre484omz1eixykzj84.png)

And we can construct this magnifying glass with **Store** comonad:

![](http://iokasimov.github.io/images/hyy2nrtwo3o3sfon62-nwbm4jj8.png)

As soon as we can focus on the part of data we need, we can see what is there, replace this part with new or modify it with function:

```haskell
view :: Lens s t -> s -> t
view lens = pos . lens

set :: Lens s t -> t -> s -> s
set lens new = peek new . lens

over :: Lens s t -> (t -> t) -> s -> s
over lens f = extract . retrofit f . lens
```

# Change State with lenses

Let’s use this magic on practice. There was a man:
```haskell
data Person = Person Name Address (Maybe Job)
```

We have lenses which can focus on his/her job and address:
```haskell
job :: Lens Person (Maybe Job)
address :: Lens Person Address
```

He/she is looking for a job. Some positions have relocation opportunity, depends on the offer he/she take, he/she needs to move to some city:
```haskell
hired :: State (Maybe Job) City
relocate :: City -> State Address ()
```

So we have effectful **hired** expression which operates on the state of the current job (**Maybe Job**). And there is expression **relocate** wich takes city as an argument and depends on that city changes **Address** state. These both effectful expressions operate on some state but we can’t use them together because type of state is different.

But there is a brilliant **zoom** function that let us change type of state if we have according lens (it’s pretty similar to **lift** function in **monad transformers**).
```haskell
zoom :: Lens bg ls -> State ls a -> State bg a
```

We take lenses that we already have and apply them to according expressions:

```haskell
zoom job hired :: State Person City
zoom address (relocate _) :: State Person ()
```

… and bind them monadically! Now we have an access to **Person** state in this combined expression:
```haskell
zoom job hired >>= zoom address . relocate
```

You can find those definition sources  [here](https://github.com/iokasimov/joint) .
