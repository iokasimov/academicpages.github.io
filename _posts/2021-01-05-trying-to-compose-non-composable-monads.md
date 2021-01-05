---
title: 'Trying to compose non-composable: monads'
date: 2021-01-05
permalink: /posts/2021/01/composable-monad-transformers
tags:
  - haskell
  - functors
  - monad
  - applicative
  - traversable
  - distributive
  - adjunctions
  - state
  - store
---

How many times have you heard “monads do not compose”? I’ve spent a lot of time trying to contradict this statement. But just like in math, if you want to understand something sometimes you have to change your point of view.

It’s recommended to read the first and the second parts of this series.

When we want to unite two effects into one (into transformer) we have two options: wrap a left one into a right or wrap a right one into a left. These two options define with `TU` and `UT` schemes.

```haskell
newtype TU t u a = TU (t :. u := a)
newtype UT t u a = UT (u :. t := a)
```

As we already know from previous parts for computations with a read-only environment (`Reader`) it’s enough to have direct composition, but for error handling effects (`Maybe` and `Either`) reverse composition `UT` fits.

```haskell
type instance Schema (Reader e) = TU ((->) e)
type instance Schema (Either e) = UT (Either e)
type instance Schema Maybe = UT Maybe
```

Covariant and Applicative functor instances look pretty obvious because it’s still functor and as we know functors compose.

```haskell
(<$$>) :: (Functor t, Functor u) => (a -> b) -> t :. u := a -> t :. u := b
(<$$>) = (<$>) . (<$>)

(<**>) :: (Applicative t, Applicative u) => t :. u := (a -> b) -> t :. u := a -> t :. u := b
f <**> x = (<*>) <$> f <*> x

instance (Functor t, Functor u) => Functor (TU t u) where
    fmap f (TU x) = TU $ f <$$> x

instance (Applicative t, Applicative u) => Applicative (TU t u) where
    pure = TU . pure . pure
    TU f <*> TU x = TU $ f <**> x

instance (Functor t, Functor u) => Functor (UT t u) where
    fmap f (UT x) = UT $ f <$$> x

instance (Applicative t, Applicative u) => Applicative (UT t u) where
    pure = UT . pure . pure
    UT f <*> UT x = UT $ f <**> x
```

Problem appear when we try to define `Monad` instances. It’s unclear, how we are going to find generalized method if two effects are unknown.

```haskell
instance (Monad t, Monad u) => Monad (TU t u) where
  x >>= f = ???

instance (Monad t, Monad u) => Monad (UT t u) where
  x >>= f = ???
```

Perhaps we are not up to it yet. Let’s try instead define instances for one known effects in transformer:

```haskell
instance Monad u => Monad (TU ((->) e) u) where
    TU x >>= f = TU $ \e -> x e >>= ($ e) . run . f

instance Monad u => Monad (UT (Either e) u) where
    UT x >>= f = UT $ x >>= \case
        Left e -> pure $ Left e
        Right r -> run $ f r

instance Monad u => Monad (UT Maybe u) where
    UT x >>= f = UT $ x >>= \case
        Nothing -> pure Nothing
        Just r -> run $ f r
```

In case of error handling (`Maybe` and `Either`) we can see the same behavior: if invariant contains parameter `a`  then computation goes on. It’s incredibly similar to `Traversable`! This is how it looks like:

```haskell
class (Functor t, Foldable t) => Traversable t where
    traverse :: Applicative f => (a -> f b) -> t a -> f (t b)

instance Traversable Maybe where
    traverse _ Nothing = pure Nothing
    traverse f (Just x) = Just <$> f x

instance Traversable (Either a) where
    traverse _ (Left x) = pure (Left x)
    traverse f (Right y) = Right <$> f y
```

Let’s try to use it:

```haskell
instance (Traversable t, Monad t, Monad u) => Monad (UT t u) where
    UT x >>= f = UT $ x >>= \i -> join <$> traverse (run . f) i
```

It works! It means that we can bind transformers with a known `Traversable` effect.

But what we should do with `TU` which used for `Reader`? I don’t know yet. But we can try something - we take the opposite class for `Traversable` - `Distributive`. Such a happy day - there is an instance for `Reader` (for its inner type `(->) e` to be precise)!

```haskell
class Functor g => Distributive g where
    collect :: Functor f => (a -> g b) -> f a -> g (f b)

instance Distributive ((->) e) where
    collect f q e = flip f e <$> q
```

But why exactly these classes and why they’re opposite each other? In order to understand it we can take a closer look at their modified methods where we replace `a -> t b` functions with `id`.

```haskell
sequence :: (Traversable t, Applicative u) => t (u a) -> u (t a)
sequence = traverse id

distribute :: (Distributive t, Functor u) => u (t a) -> t (u a)
distribute = collect id
```

That’s it! We can see that these methods let us change the order of effects. If `Traversable` worked for reverse composition, could `Distributive` to be fitted for direct one?

```haskell
instance (Monad t, Distributive t, Monad u) => Monad (TU t u) where
    TU x >>= f = TU $ x >>= \i -> join <$> collect (run . f) i
```

Yes, it works too! So we have an algorithm to define `Monad` instances for such transformers:

* Reverse composition `TU` - rely on `Traversable`
* DIrect composition `UT` - rely on `Distributive`

But we also have a more complicated scheme - `TUT` which is used for monad `State `and comonad  `Store`.

```haskell
newtype TUT t t' u a = TUT (t :. u :. t' := a)

newtype State s a = State ((->) s :. (,) s := a)
newtype Store s a = Store ((,) s :. (->) s := a)

type instance Schema (State s) = TUT ((->) s) ((,) s)
type instance Schema (Store s) = TUT ((,) s) ((->) s)
```

Looks pretty complicated, right? The Composition of covariant `Functor`’s still works, but not for `Applicative`’s - we can’t just map effectful function to the composition of an arrow and a comma because they’re knitted. In other words, tuple content is dependent of an argument of function and we have to make a function application to get an updated value.

```haskell
instance (Functor t, Functor t', Functor u) => Functor (TUT t t' u) where
    fmap f (TUT x) = TUT $ f <$$$> x
```

So, the arrow (`(->) s`) is `Distributive` and the comma (`(,) s`) is `Traversable`… But there is a more tight connection between these two types called `Adjunction` (you can read more about it [here](https://iokasimov.github.io/posts/2020/10/arrow-and-comma)).

```haskell
class Functor t => Adjunction t u where
    leftAdjunct  :: (t a -> b) -> a -> u b
    rightAdjunct :: (a -> u b) -> t a -> b
    unit :: a -> u :. t := a
    unit = leftAdjunct id
    counit :: t :. u := a -> a
    counit = rightAdjunct id

instance Adjunction ((,) s) ((->) s) where
    leftAdjunct :: ((s, a) -> b) -> a -> (s -> b)
    leftAdjunct f a s = f (s, a)
    rightAdjunct :: (a -> s -> b) -> (s, a) -> b
    rightAdjunct f (s, a) = f a s
    unit :: a -> s -> (s, a)
    unit x = \s -> (s, x)
    counit :: (s, (s -> a)) -> a
    counit (s, f) = f s
```

Let’s try to deal with `State` effect first. Wrapping into an effect is completely identical to `unit` method and we use right adjoint to bind two effects in `Monad` chain.

```haskell
instance Monad (State s) where
    State x >>= f = State $ rightAdjunct (run . f) <$> x
    /— Or: State x >>= f = State $ counit <$> ((run . f) <$$> x)/
    return = State . unit
```

But what about transformers? We have an unknown effect between the two ones. To wrap a value into a transformer we need left adjunction and for binding, we need right adjunction:

```haskell
instance (Adjunction t' t, Monad u) => Monad (TUT t t' u) where
    x >>= f = TUT $ (>>= rightAdjunct (run . f)) <$> run x
    return = TUT . (leftAdjunct pure)
```

Amazing, to define `Comonad` instance for the same scheme we need to do everything vice versa:

```haskell
instance (Adjunction t' t, Comonad u) => Comonad (TUT t' t := u) where
    extend f x = TUT $ (=>> leftAdjunct (f . TUT)) <$> run x
    extract = rightAdjunct extract . run
```

And what we need to do to `lift` an unknown effect up to the transformer layer? We can use `adjunctions` here too! Moreover, instances for `Monad` and `Comonad` transformers are shocking symmetric…

```haskell
instance (Adjunction t' t, Distributive t) => MonadTrans (TUT t t') where
    lift = TUT . collect (leftAdjunct id)

instance (Adjunction t' t, Applicative t, forall u . Traversable u) => ComonadTrans (TUT t' t) where
    lower = rightAdjunct (traverse id) . run
```

We can draw some conclusions from what has been said:

* If effect has `Traversable` instance - reverse schema `UT` fits.
* If effect has `Distributive` instances - direct schema `TU` fits.
* If components of effect form `Adjunction` - `TUT` schema fits.

It turns out that some monads can unite in transformers based on the properties of known effects.

You can find the source of the definitions [here](https://github.com/iokasimov/joint). Also, you can take a look at examples of application described effect system [here](https://github.com/iokasimov/experiments/tree/master/Problems/joint).
