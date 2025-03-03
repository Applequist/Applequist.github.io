---
title: 'The FP Epiphany (Part 1) - Functors'
description: 'A serie about learning functional programming. This post is about functors.'
pubDate: 'Sep 27 2024'
updatedDate: 'Sep 29 2024'
---

Rust has been my favorite programming language for a while now:
- fast
- safe
- you know the pitch...

And, although I have been sensible to the promises of functional programming since it has been popular again, I really never got it. 

I mean, I was attracted by the rigorous mathematical foundations of Haskell,
the ideas that *referential transparency* and *equational reasoning* would make my code easier to understand, 
that composable abstractions like ***Functors*** and al would make me more productive, it all looked so... cryptic.

So, yeah, I was just happy folding my `Vec`s and mapping my `Result`s in Rust and call it a day...

Until I picked [Haskell Programming from first principles](https://haskellbook.com) and gave it a serious look.

In this first post, we are going to look at functors and how, although the idea behind it has *made it* in Rust, you get more in Haskell.

## So, what are functors ?

The idea behind a functor is to apply or map a function to a value wrapped into a structure while leaving that structure unchanged. 

In Haskell, it is implemented using a typeclass:

```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

A way to look at the `fmap` function from the `Functor` typeclass is that it takes a "normal" function from `a` to `b` 
and returns a new function that can be applied to a value of type `f a` and returns a value of type `f b`. 
This "normal" function is said to have been ***lifted***.  

The classical example of functor is `[]` whose `Functor` instance could be written as follow:

```haskell
instance Functor [] where
  fmap f x:xs =  f x : (fmap f xs)
```

And if we want to double every elements in a list we could do as follow:

```
ghci> double = (*2)
ghci> liftedDouble = fmap double
ghci> liftedDouble []
[]
ghci> liftedDouble [1, 2, 3]
[2, 4, 6]
```

But there are many other functor instances including `Maybe` and `Either` and our `liftedDouble` function would work just as well
with them:

```
ghci> liftedDouble Nothing
Nothing
ghci> liftedDouble (Just 1)
Just 2
```

## What about Rust ?

In stable Rust 1.81, there is no equivalent to the `Functor` typeclass and its `fmap` function
but many types/traits like `Iterator`, `Option<T>` and `Result<T, E>` to name a few, have a `map` method that provides the ability to apply a function to the *contained* value(s):

```rust
fn main() {
  dbg!(vec![1, 2, 3].iter().map(|i| i * 2).collect::<Vec<_>>() == vec![2, 4, 6]);

  dbg!(None::<i32>.map(|i| i * 2) == None);
  dbg!(Some(1).map(|i| i * 2) == Some(2));

  dbg!(Err::<i32, &'static str>("Boom").map(|i| i * 2) == Err("Boom"));
  dbg!(Ok::<i32, &'static str>(1).map(|i| i * 2) == Ok(2));
}
```

If you compile and run it, you'll get the following:

```
❯ rustc map.rs && ./map
[map.rs:2:5] vec![1, 2, 3].iter().map(|i| i * 2).collect::<Vec<_>>() == vec![2, 4, 6] = true
[map.rs:4:5] None::<i32>.map(|i| i * 2) == None = true
[map.rs:5:5] Some(1).map(|i| i * 2) == Some(2) = true
[map.rs:7:5] Err::<i32, &'static str>("Boom").map(|i| i * 2) == Err("Boom") = true
[map.rs:8:5] Ok::<i32, &'static str>(1).map(|i| i * 2) == Ok(2) = true
```

## Ok, so why would I care about Haskell functors?!

Let's say, you have collected a bunch of computation `Result`s in a `Vec` and you want to post-process them:

```rust
fn double(d: i32) -> i32 {
  d * 2
}

fn main() {
  let d = vec![Ok(2), Ok(5), Err("Boom"), Ok(6)];
  dbg!(d.into_iter().map(|r| r.map(double)).collect::<Vec<_>>());
}
```

Compile and run:

```
❯ rustc mapmap.rs && ./mapmap
[mapmap.rs:7:5] d.into_iter().map(|r| r.map(double)).collect::<Vec<_>>() = [
    Ok(
        4,
    ),
    Ok(
        10,
    ),
    Err(
        "Boom",
    ),
    Ok(
        12,
    ),
]
```

Done. But you'd admit it is quite verbose with the nested `map` calls and it get worst as the number of layers of type increases. 
Here we only have 2 layers: a `Vec` containing `Result`s.

In Haskell, functors compose really nicely and all you have to do to "dig" into layers of functors is to compose fmap:

```
ghci> double = (*2)
ghci> d = [Right 2, Right 5, Left "Boom", Right 6]
ghci> pp = (fmap . fmap) double d 
ghci> pp
[Right 4, Right 10, Left "Boom", Right 12]
```

The `fmap` composition is not obvious at first but if we "unpack" it, it becomes clearer:

```
(fmap . fmap) double d

fmap (fmap double) d

liftedDouble :: (Functor f, Num a) => f a -> f a
liftedDouble = fmap double

fmap liftedDouble d
```

So the outer `fmap` will apply `liftedDouble` (see above) to each `Either` values in the list,
and `liftedDouble` will double each integer in a `Right` value and leave the `Left` value alone.

Neat, right?! FP for the win!

## One more thing...

I wrote above that there is no equivalent `Functor` typeclass in stable Rust. 
As [u/Zde-G](https://www.reddit.com/user/Zde-G/) user on reddit explained in [this post](https://www.reddit.com/r/rust/comments/ynvm8a/comment/ivc5n8d/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button), we can write something close using GAT (Generic Associated Types).

I reproduce the solution below:

```rust
trait Functor {
    type Item;
    type Result<U>;
    
    fn fmap<U, F: FnMut(Self::Item) -> U>(self, f: F) -> Self::Result<U>;
}

impl<T> Functor for Vec<T> {
    type Item = T;
    type Result<U> = Vec<U>;
    
    fn fmap<U, F: FnMut(Self::Item) -> U>(self, f: F) -> Self::Result<U> {
        self.into_iter().map(f).collect()
    }
}

impl<T, E> Functor for Result<T, E> {
    type Item = T; 
    type Result<U> = Result<U, E>;
    
    fn fmap<U, F: FnMut(Self::Item) -> U>(self, f: F) -> Self::Result<U> {
      self.map(f)
    }
}

fn double(data: i32) -> i32 {
    data * 2
}

fn main() {
    let data = vec![Ok(2), Ok(5), Err("Boom"), Ok(7)];
    dbg!(data.fmap(|i| i.fmap(double)));
}
```

Compile and run:

```
❯ rustc functor.rs && ./functor
[functor.rs:32:5] data.fmap(|i| i.fmap(double)) = [
    Ok(
        4,
    ),
    Ok(
        10,
    ),
    Err(
        "Boom",
    ),
    Ok(
        14,
    ),
]
```
