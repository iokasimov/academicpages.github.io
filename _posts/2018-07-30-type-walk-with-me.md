---
title: 'Type walk with me'
date: 2018-07-30
permalink: /posts/2018/07/type-walk-with-me
tags:
  - haskell
  - product
  - with
---

I lost count of how many times I've seen types/functions/families and another first-class abstractions that shouldn't exist if we want to use more universal constructions. A little bit of theory can reduce and beautify our production code, and now I will try to demonstrate that.

Problem description
--------------------------------------------------------------------------------

Let's suppose you write a JSON API server that operates on some entities, users, for example. What are you going to do first? Right, define a type:

```haskell
data User = User { name :: String, age :: Int }
```

We have a type, then, we need to write an instance of `Aeson` to convert a value into JSON:

```haskell
instance ToJSON User where
   user = object ["name" .= name user, "age" .= age user]
```

So far so good. But someday, a frontend guy says: "Oh, man, on this endpoint, I need an additional information to user object: a list of followers".
Okay, you think. So, I should change my `User` type and add a field with a list of followers, not a big deal:

```haskell
type Followers = [User]

data User = User { name :: String, age :: Int, followers :: Followers }

instance ToJSON User where
   toJSON user = object ["name" .= name user, "age" .= age user, "followers" .= followers user]
```

And yet another day, the frontend guy says: "On that endpoint, I need to get some stats with the user object: a view counter". You think: "Hmmm, that's not good, I should create the same datatype that differs from the previous one in just one field."

But there is a better way.

Open product types
--------------------------------------------------------------------------------

A field of a record syntax is just a multiplier. Can we manipulate these multipliers the way we want and pay nothing for that? Yes. Let's create an open product type:

```haskell
{-# language TypeOperators #-}

data (:&:) a b = a :&: b
```
Looks very easy, right? Let's decompose the user object and the additional information in this style:

```haskell
{-# language FlexibleInstances #-}

instance ToJSON (User :&: Followers) where
   toJSON (user :&: followers) = object
      ["user" .= toJSON user, "followers" .= toJSON followers]
```

But then the frontend guy appears again saying "Man, I need a new endpoint with user, followers and view counter, as fast as possible, please".
And you reply: "No, problem, here you go!"

```haskell
{-# language FlexibleInstances #-}

type Counter = Int

instance ToJSON (User :&: Followers :&: Counter) where
   toJSON (user :&: followers :&: counter) = object
      ["user" .= toJSON user, "followers" .= toJSON followers, "counter" .= toJSON counter]
```

There is a little zero dependency package with this type definition called [with](https://github.com/iokasimov/with) that I created recently.
