---
title: "The FP Epiphany - Functions are Functors"
description: "Mapping over function is really function composition."
pubDate: "Nov 7 2024"
tags: ['functional-programming']
---

While continuing reading Haskell Programming from 1st principle, I stumble upon
this snippet:

```haskell
ghci> (fmap . fmap) sum Just [1, 2, 3]
Just 6
```

And I just could not make sense of it!

Even now after asking for help on the Haskell discourse site, I am like: "huh, what's going on again?"

The first thing to remember here is that function application has the highest precedence and
is left associative, so what's really going on is:

```haskell
(((fmap . fmap) sum) Just) [1, 2, 3]
```

or

```haskell
((fmap . fmap) sum Just) [1, 2, 3]
```

From there, there are 2 ways to see it.

### Mapping over function is composing functions

The first way suggested by [haanss](https://discourse.haskell.org/t/understanding-evaluation-with-fmaps/10689/2?u=applequist) is to apply `sum` to `fmap`:

```haskell
(fmap (fmap sum) Just) [1, 2, 3]
```

So we're mapping `fmap sum` over `Just`? Yep! And `Just` is a function of type `a -> Maybe a` and as such, it is a `Functor`:

```haskell
instance Functor ((->) r) -- function that takes a r
```

> Functions are Functors and mapping a function over a function is just composing functions.

So we're down to:

```haskell
((fmap sum) . Just) [1, 2, 3]
```

Which is:

```haskell
fmap sum (Just [1, 2, 3])
```

And that is simply fully applying `fmap`.

### Functions maybe Stacked Functors

The second way to look at it suggested by [int-index](https://discourse.haskell.org/t/understanding-evaluation-with-fmaps/10689/4?u=applequist) is to recognize that we are apply `sum` over
two levels of Functors.

Two levels ?!

Yep! `Just` is not _just_ a Functor but 2 nested Functors:

- the outer Functor is the function itself (see above)
- the inner Functor is its Maybe return value.

So `(fmap . fmap) sum Just` returns a function that returns a `Maybe` containing a value that is the `sum` of the function argument.

```haskell
ghci> :t (fmap . fmap) sum Just
(fmap . fmap) sum Just :: (Foldable t, Num b) => t b -> Maybe b
```
