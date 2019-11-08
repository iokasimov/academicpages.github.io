---
title: 'Trying to compose non-composable: joint schemes'
date: 2019-11-02
permalink: /posts/2019/11/joint
tags:
  - haskell
  - effect
  - composition
  - transformer
---

Introduction
--------------------------------------------------------------------------------

In this blog post I would like to show you an experimental method for dealing with multiple effects in Haskell that I have called `joint schemes`. What is the `joint schema`? It's a `schema` that describes how some specific effect should be composed with other effects. I never seen something similar before, so I decided to dig into the capabilities of this approach.

In Haskell we get used to work with effects as functors, whose objects (arguments) are some expressions, which we are interested at some particular moment.

Definitions
--------------------------------------------------------------------------------

Before we continue, let's define some type annotations that will be used in this article, that will be very helpful when you're starting thinking in terms of compositions over objects.

Let `t` be determined, some specific effect and let `u` be undetermined, just any other effect:

Functor's object:
```haskell
type (:=) t a = t a
```
If we have a simple one `Maybe` effect over some value it looks like: :
```haskell
Maybe := a
```

Composition of functors
```haskell
type (:.) t u a = t (u a)
```

Let's say, we want to add new `List` effect over existing one:
```haskell
List :. Maybe := a
```

Natural transformation
```haskell
type (~>) t u = forall a . t a -> u a
```
Function signature for safely accessing a list item would look like this:
```haskell
List ~> Maybe
```

When we see type annotation like `Maybe a`, we abstract from existing of this `a`, focusing all our attention on this `a` like it always exists. The same story with `List a` - multiple `a` values, `State s a` - `a`, which depends from some current state, `Either e a` - some `a` that can throw an error `e`. For example: `List :. Maybe := a` - such expression is easy to imagine, this is a list of values, whose existing is unknown.

Compositions and transformers
--------------------------------------------------------------------------------

In most real world programs we use many effects at the same time and the effect must then be composed in some way. There are mainly two different ways to do this: compositions and transformers. So, what are the pros and cons of these both methods?

Compositions:
* There is no need in additional datatypes
* There is no generalised method for merging effects
* Everything is composable, as long as you don't need monadic behaviour

Transformers:
* They let you merge several effects into one
* With `lift` you can do computations on any level of monad transformer stack
* But it needs to have additional datatype (usually, newtype) (like `ReaderT`, `StateT`, `ExceptT`, `MaybeT`)
* You cannot consider effects alone, but there are special functions for that (like `mapReaderT`, `mapStateT`, `mapExceptT`, `mapMaybeT`)

Transformers differ from compositions that they propagate effects on each other. For example, if you have an error on `Either`-level in `StateT _ (Either _) a` - all computation should stop. If you have just a composition `State _ :. Either _ := a` - in this case (just a functors composition) an error will not affect the whole computation.

Decomposing familiar effects
--------------------------------------------------------------------------------

If we take a closer look on types for monad transformers from Kmett's `transformers` library, we can see some patterns:

```haskell
newtype ReaderT r m a = ReaderT { runReaderT :: r -> m a }
newtype MaybeT m a = MaybeT { runMaybeT :: m (Maybe a) }
newtype ExceptT e m a = ExceptT { runExceptT :: m (Either e a)) }
newtype StateT s m a = StateT { runStateT :: s -> m (a,s) }
```

Transformers describe special case of how two effects - determined and undetermined should be composed. We will try to rewrite type definitions above in terms of determined and undetermined effects:

```haskell
Reader: r -> u a ===> (->) r :. u := a ===> t :. u := a -- determined - t ~ (->) r, undetermined - u
Maybe: u (Maybe a) ===> u :. Maybe := a ===> u :. t := a -- determined - t ~ Maybe, undetermined - u
Either: u (Either e a) ===> u :. Either e := a ===> u :. t := a -- determined - t ~ Either e, undetermined - u
```

Some effects are pretty complicated and can be defined via composition of more simpler effects:

```haskell
State: s -> m (a, s) ===> (->) s :. (,) s := a ==> t :. u :. t’ := a
newtype State s a = State ((->) s :. (,) s := a)
```

If we take a closer look again on first three examples, we can consider some patterns: in `Reader`, determined effect wraps undetermined one; but in the case of `Either` and `Maybe` we do the opposite thing - undetermined effects wrap determined. In case of `State` we put undetermined effect between two determined effects.

Fit familiar effects in joint schemes
--------------------------------------------------------------------------------

Let's try to express those patterns in types:

```haskell
newtype TU t u a = TU (t :. u := a)
newtype UT t u a = UT (u :. t := a)
newtype TUT t u t' a = TUT (t :. u :. t' := a)
```
We just defined `joint schemes` - they are functors compositions in wrappers, that can point on the positions of determined and undetermined effects. In the essence, methods of transformers, which names start with `run` unwrap these wrappers and return functors composition. We can generalise this operation:

```haskell
class Composition t where
	type Primary t a :: *
	run :: t a -> Primary t a
```

Now we have a generalised way to run these schemes:

```haskell
instance Composition (TU t u) where
	type Primary (TU t u) a = t :. u := a
	run (TU x) = x

instance Composition (UT t u) where
	type Primary (UT t u) a = u :. t := a
	run (UT x) = x

instance Composition (TUT t u t') where
	type Primary (TUT t u t') a = t :. u :. t' := a
	run (TUT x) = x
```

But what about transformers? We need to define new typeclass again. It consists of:
* Some corresponding joint scheme that uniquely defined for this type
* `embed` method to lift undetermined effect up to transformer layer
* `build` method to make determined effect a transformer

```haskell
class Composition t => Transformer t where
	type Schema (t :: * -> *) (u :: * -> *) = (r :: * -> *) | r -> t u
	embed :: Functor u => u ~> Schema t u
	build :: Applicative u => t ~> Schema t u

type (:>) t u a = Transformer t => Schema t u a
```

All we need is to define instances, let's start with `Maybe` and `Either`:

```haskell
instance Transformer Maybe where
	type Schema Maybe u = UT Maybe u
	embed x = UT $ Just <$> x
	build x = UT . pure $ x

instance Transformer (Either e) where
	type Schema (Either e) u = UT (Either e) u
	embed x = UT $ Right <$> x
	build x = UT . pure $ x
```

We will create our own type for `Reader` because we don't have it in base. And we need to define `Composition` instance for `Reader` just because it can be expressed via more simpler effect (a function arrow).

```haskell
newtype Reader e a = Reader (e -> a)

instance Composition (Reader e) where
	type Primary (Reader e) a = (->) e a
	run (Reader x) = x

instance Transformer (Reader e) where
	type Schema (Reader e) u = TU ((->) e) u
	embed x = TU . const $ x
	build x = TU $ pure <$> run x
```

Let's do the same with `State`:

```haskell
newtype State s a = State ((->) s :. (,) s := a)

instance Composition (State s) where
	type Primary (State s) a = (->) s :. (,) s := a
	run (State x) = x

instance Transformer (State s) where
	type Schema (State s) u = TUT ((->) s) u ((,) s)
	embed x = TUT $ \s -> (s,) <$> x
	build x = TUT $ pure <$> run x
```

Simple example
--------------------------------------------------------------------------------

Let's try our approach on the real world tasks - we'll write a program that can test the correctness of location of various styles of brackets in a source code. First, we need to define types for brackets: they can be opened and closed; they can have different styles.

```haskell
data Shape = Opened | Closed

data Style = Round | Square | Angle | Curly
```

Our program is indifferent to other symbols:

```haskell
data Symbol = Nevermind | Bracket Style Shape
```

Also, we're going to define what types of errors our program can stumble on:

```haskell
data Stumble
	= Deadend (Int, Style) -- Closed bracket without opened one
	| Logjam (Int, Style)  -- Opened bracket without closed one
	| Mismatch (Int, Style) (Int, Style)  -- The closed bracket's style doesn't match to the opened one
```

What are the effects our program needs? We have to store a stack of styles of open brackets which should be checked later and we should stop all computation on first error. Let's build a transformer:

```haskell
State [(Int, Style)] :> Either Stumble := ()
```

This algorithm is easy: we traverse the structure of indexed brackets, if we haven't stumbled on error and we have some brackets left in the stack - it means that the opened bracket doesn't have the closed one.

```haskell
checking :: Traversable t => t (Int, Symbol) -> Either Stumble ()
checking struct = run (traverse proceed struct) [] >>= \case
	(s : _, _) -> Left . Logjam $ s where
	([], _) -> Right ()
```

We put to the stack each opened bracket, and any closed one we match with the last opened one:

```haskell
proceed :: (Int, Symbol) -> State [(Int, Style)] :> Either Stumble := ()
proceed (_, Nevermind) = pure ()
proceed (n, Bracket style Opened) = build . modify . (:) $ (n, style)
proceed (n, Bracket closed Closed) = build get >>= \case
	[]-> embed $ Left . Deadend $ (n, closed)
	((m, opened) : ss) -> if closed /= opened
		then embed . Left $ Mismatch (m, opened) (n, closed)
		else build $ put ss
```

Conclusion
--------------------------------------------------------------------------------

So, what do we have? According to this approach even very complicated effects can be decomposed on composition of simpler ones (that's what for `Composition` typeclass) and we can inject any other effects into such schemes so they are working as transformers pretty well.

Suddenly, that trick doesn't work with the mother of monads - continuations. Just because we cannot redefine it via functors composition.

I've created a tiny [library](https://github.com/iokasimov/joint) with examples, if you have a bundle of effects in your pet-projects - you can try it. And if you have some troubles with it - email me. Also, there is full [example's code with brackets](https://gist.github.com/iokasimov/e149804f8bf4cb807a1ff6c2ae6a383a).

I must say many thanks to [Joel Wikstrom](https://twitter.com/WikstromJoel) who had helped me a lot - his notes and advice made this blog post happen.
