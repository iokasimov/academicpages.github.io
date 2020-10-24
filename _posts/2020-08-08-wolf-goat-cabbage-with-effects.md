---
title: 'Cross wolf, goat and cabbage across the river with effects'
date: 2020-08-08
permalink: /posts/2020/08/wgc-effects
tags:
  - haskell
  - effect
  - composition
  - transformer
---


_Once upon a time a farmer went to a market and purchased a  wolf, a  goat , and a cabbage. On his way home, the farmer came to the bank of a river and rented a boat. But crossing the river by boat, the farmer could carry only himself and a single one of his purchases: the wolf, the goat, or the cabbage. If left unattended together, the wolf would eat the goat, or the goat would eat the cabbage. The farmer’s challenge was to carry himself and his purchases to the far bank of the river, leaving each purchase intact. How did he do it?_

![](http://iokasimov.github.io/images/wdyt9getbzfatxvfihnk3udgia0.png)

In this blogpost we will try to find generalized solution for such puzzles with algebraic effects.

Let’s start from the start: we need a route. Since we do not know in advance, after what guaranteed number of steps we will get a solution, we should generate an infinite route - we’re gonna evaluate it lazily anyway.

```haskell
data Direction = Back | Forward

route :: [Direction]
route = iterate alter Forward

alter :: Direction -> Direction
alter Back = Forward
alter Forward = Back
```

With intention to build generalized solution let’s abstract from exact characters too. Let’s create our own non-transitive symmetric relation between elements of character set:

```haskell
class Survivable a where
	survive :: a -> a -> Ordering

data Character = 🐺 | 🐐 | 🥬

instance Survivable Character where
	survive 🐺 🐐 = GT
	survive 🐐 🐺 = LT
	survive 🐐 🥬 = GT
	survive 🥬 🐐 = LT
	survive _ _ = EQ
```

What do we need effects for? Well, they can help us to fight complexity. Therefore, to realize which effects do we need we should think about where is the complexity hidden:

* To find a solution in which all characters will be transported to opposite bank, we have to go through many permutation options - we can make it with [Monad instance of list](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/List) to simulate plurality effect.
* Also, we need to remember some character location to check condition of compatibility with other characters. We can keep  `type River a = ([a],[a])`  with  `State (River a)`.
* The boat can take someone or sail empty - we can take partiality effect into consideration with `Maybe` datatype.

I’m going to use my experimental `joint` library but it should be pretty easily to translate this code to `transformers` or `mtl`.

So we have three different effects: plurality, state and partiality. But it’s not enough just to have them - we have to decide how to compose them:

* In `Applicative`/`Monad` chain of `Maybe`  evaluation, we we got `Nothing` somewhere, result will be `Nothing`. We are going to keep it apart from other effect because we don’t want to stop evaluation if we send an empty boat.
* Every next choice (plurality effect) should be based on the current bank of characters (state effect) - because we can’t take a character that is not existed on current  bank. So, let’s joint them:  `State (River a) :> []`.

Let’s describe one step in this puzzle solution as sequence of actions:
* Get a list of characters on the current riverbank
* Select and verify next character to transport
* Actually transport character to opposite riverbank

```haskell
step direction = bank >>= next >>= transport
```

Let’s take a look at each step more closely.

Depending on boat moving direction, we apply a lens for source of boat moving to current state and get the list of characters:

```haskell
bank :: (Functor t, Stateful (River a) t) => t [a]
bank = view (source direction) <$> current
```

How to choose the next character to move? We take the list of character with `bank` expression and add option with empty boat:  `\xs -> Nothing : (Just <$> xs)`. For every candidate (empty boat is candidate too) we check that we don’t leave characters on source bank that could be eaten:

```haskell
valid :: Maybe a -> Bool
valid Nothing = and $ coexist <$> xs <*> xs
valid (Just x) = and $ coexist <$> delete x xs <*> delete x xs

coexist :: Survivable a => a -> a -> Bool
coexist x y = survive x y == EQ
```

After filtering list of candidate, we `lift ` plurality effect and return every candidate as a value:

```haskell
next :: (Survivable a, Iterable t) => [a] -> t (Maybe a)
next xs = lift $ filter valid $ Nothing : (Just <$> xs)
```

Okay, there is only one action left - actual transportation. We delete picked character from source bank and add it to target bank:  

```haskell
leave, land :: River a -> River a
leave = source direction %~ delete x
land = target direction %~ (x :)
```

If there was a character in the boat we change the river state:

```haskell
transport :: (Eq a, Applicative t, Stateful (River a) t) => Maybe a -> t (Maybe a)
transport (Just x) = modify @(River a) (leave . land) $> Just x where
transport Nothing = pure Nothing
```

It would be nice to see our program working. To find a solution we need at least 7 steps:

```haskell
start :: River Character
start = ([🐐, 🐺, 🥬], [])

run (traverse step $ take 7 route) start
```

Finally, we got two solutions:

```haskell
[Just 🐐,Nothing,Just 🐺,Just 🐐,Just 🥬,Nothing,Just 🐐]
[Just 🐐,Nothing,Just 🥬,Just 🐐,Just 🐺,Nothing,Just 🐐]
```

The final code for solution is available [here](https://github.com/iokasimov/experiments/blob/master/joint/wolf-goat-cabbage.hs).
