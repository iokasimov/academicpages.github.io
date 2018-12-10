---
title: 'Observable as an open interface for handling executing processes and property testing'
date: 2018-12-10
permalink: /posts/2018/12/observable
tags:
  - haskell
  - continuations
  - observable
  - network
  - testing
---

It all began with this tweet:
![Alt text](http://iokasimov.github.io/images/denis_stoyanov_kan_tweet.png)

I found this idea interesting, I've messed around with it for a while and wrote a small library: [observable](http://hackage.haskell.org/package/observable). I will not go into details of the code internals, such as continuations or applicative functors. I warn you that I will not rely on OOP definition, cause it doesn't fit our needs (in my opinion, thinking in categories of objects with an internal state is harmful).

How does observing look like? Well, we definitely can say that we need two things here: observing action and observer handler. Action produces events, handler handles them - easy. But we will consider two options - one-time actions/handlers and repeated actions/handlers. Based on these assumptions we can describe four observable patterns:

* `(.:~.), notify` - Listen only first event, handle just once
* `(.:~*), follow` - Listen only first event, handle forever
* `(*:~.), subscribe` - Listen all events, handle each once
* `(*:~*), watch` - Listen all events from action, handle each forever

Sorry if you found these definitions not semantically correct, my english is not so good.

Before you continue reading, I recommend you to read a simple example in [README](https://github.com/iokasimov/observable).

But recently I realized where it can be used effectively - in network components. We will build a toy distributed system with two types of nodes - administrator and worker (or master and slave, as you wish). For our simple example we will use [network-simple](http://hackage.haskell.org/package/network-simple) cause it uses CPS-style for incoming connections.

```haskell
administrator :: Observable IO Socket ()
administrator = dispatch . (<$>) fst . ContT $ serve (Host "127.0.0.1") "9000"
```

```haskell
worker :: Observable IO Socket ()
worker = dispatch . (<$>) fst . ContT $ connect "127.0.0.1" "9000"
```

```haskell
data Command = Do | Wait -- Administrator's messages
data Query = More | Done -- Worker's messages
```

As we defined nodes and messages, we're ready to implement theirs workflows.

Administrator should receive message from worker and reply what to do next:

```haskell
administer :: Socket -> IO ()
administer socket = receive @Query socket >>= interpret

interpret :: Query -> IO ()
interpret More = print "Master: you got the task." *> send socket Do
interpret Done = print "Master: well done!" *> send socket Wait
```

Worker just wants to work, an ideal employee. It asks about a task, receives a command and decides what to do next:

```haskell
work :: Socket -> IO ()
work socket = send socket More *> receive @Command socket >>= interpret

interpret :: Command -> IO ()
interpret Do = print "Worker: copy that..." *> threadDelay 1000000 *> send socket Done
interpret Wait = print "Worker: okay, I will wait..." *> threadDelay 1000000
```

Despite our observable actions and theirs handlers looking the same, we need to treat them differently:
* Administrator for each incoming connection should run a listening loop: what worker wants and to do with it. This is `(*:~*), watch` pattern.
* Once worker connects to administrator it should repeatedly send messages and get answers. This is `(.:~*), follow` pattern.

```haskell
async $ administrator *:~* administer -- watch administrator administer
async $ worker .:~* work -- follow worker work
```

And actually, it's easier to test, when the interface of producing/consuming events is opened. We can change handlers for the sake of property testing:

```haskell
data Beaten a = Alive a | Dead

-- | Listen first event from action untill time limit is up
alive :: Int -> Observable IO a r -> (a -> IO r) -> IO (Beaten r)
alive limit observable handle = race (threadDelay limit) (notify observable handle)
	>>= either (const . pure $ Dead) (pure . Alive)

-- | Listen every event one by one from action until limit time between events is up
heartbeat :: Int -> Observable IO a r -> (a -> IO r) -> IO (Beaten r)
heartbeat limit observable handle = race (threadDelay limit) (notify observable handle)
	>>= either (const . pure $ Dead) ((*>) (heartbeat limit observable handle) . pure . Alive)

data Check = Check

administrator_starts_and_reply :: Int -> Property
administrator_starts_and_reply seconds = withTests (TestLimit 1) . property $ do
	lift . async $ administrator .:~. (const $ pure ())
	lift (alive seconds worker $ const $ pure ()) >>=
		\case { Alive _ -> success; Dead -> failure }

worker_receives_administrator_message :: Int -> Property
worker_receives_administrator_message seconds = withTests (TestLimit 1) . property $ do
	lift . async $ administrator .:~* (flip send Check)
	lift $ threadDelay seconds -- Give some time to administrator to up
	lift (alive seconds (obs . notify worker $ void . receive @Check) pure) >>=
		\case { Alive _ -> success; Dead -> failure }
```

You can find full code snippet [here](https://gist.github.com/iokasimov/f67c1a117f806d63522c85f66742aa1f).

P.S. All of this was built because of a lot of pain I felt from using Cloud Haskell. Maybe I will find some time to build a small network components library based on observable patterns. If you had pleasure reading this - star [observable](https://github.com/iokasimov/observable) repo.
