---
title: 'The FP Epiphany (Part 2) - Applicatives'
description: 'A serie about learning functional programming. This post is about applicative functors.'
pubDate: 'Oct 31 2024'
---

In a [previous post](/blog/the-fp-epiphany-functors/), we learned about functors and how they allow us to apply/map
a function taking a **single** argument over a *functorial* structure, eg a list, a Maybe or Either...

One way to look at ***applicative functors*** exposed by Graham Hutton in his book [Programming Haskell](https://www.cambridge.org/sa/universitypress/subjects/computer-science/programming-languages-and-applied-logic/programming-haskell-2nd-edition) 
is as a generalization of functors to pure functions with multiple arguments. 

Indeed functors give us:

```haskell
fmap :: functor f => (a -> b) -> f a -> f b
```

But what if the function you want to map takes more than one argument?

You would think that currying would help and that you could repeatedly use `fmap`:

```haskell
f :: a -> b -> c 
f = ...

curried_f = fmap f (Just a)
fully_applied = fmap curried_f (Just b)
...
```

Unfortunately this does not typecheck. Indeed the type of `curried_f` above is `Maybe (b -> c)` (our partially applied function wrapped into a `Maybe`).
But the 2nd `fmap` application requires `b -> c`.

Instead, we could define a whole bunch of functions to apply functions of 2, 3, ..., N arguments to wrapped arguments as follow:

```haskell
fmap2 :: (a -> b -> c) -> f a -> f b -> f c -- 
fmap3 :: (a -> b -> c -> d) -> f a -> f b -> f c -> f d
...
```

But that wouldn't be very practical. How many of these would you need ? 2 ? 3 ? One hundred ?!

## Applicative to the rescue!

With ***applicative*** we can define all these functions (including fmap) with only 2 simple functions. How? you would ask!

Well, let's pursue our first intuition about currying. The problem is that partially applying a N-ary function (N > 1) using `fmap` 
gives us back a partially applied function wrapped in a functor, which stop us from further using `fmap` with the remaining arguments as we have seen above.
But if we could apply a wrapped function to a wrapped argument, this would allow us to further apply the function to the remaining arguments.

So if we had something like:

```haskell
(<*>) :: f (a -> b) -> f a -> f b
```

We could rewrite our example above as: 

```haskell
f :: a -> b -> c 
f = ...

curried_f = fmap f (Just a)
fully_applied =  curried_f <*> (Just b)
...
```

And this could be generalized to any number of arguments. Then our `fmap2`, `fmap3`... functions would be defined as:

```haskell
fmap :: (a -> b) -> f a -> f b

fmap2 :: (a -> b -> c) -> f a -> f b -> f c -- 
fmap2 f fa fb = fmap f fa <*> fb

fmap3 :: (a -> b -> c -> d) -> f a -> f b -> f c -> f d
fmap3 f fa fb fc = fmap f fa <*> fb <*> fc
...
```

Now, to get rid of `fmap`, which is used to *bootstrap* the currying process in some sort, we need to introduce the 
second function from the `Applicative` typeclass: `pure` 

```haskell
pure :: (Applicative f) => a -> f a

fmap :: (a -> b) -> a -> b
fmap f fa = pure f <*> fa

fmap2 :: (a -> b -> c) -> f a -> f b -> f c
fmap2 f fa fb = pure f <*> fa <*> fb
...
```

And there you have it! 

An ***applicative functor*** is a functor with an `Applicative` instance, where `Applicative` is 
the following class:

```haskell
class Functor f => Applicative f where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
```

It is worth noting that an `Applicative` instance is also a `Functor` instance, not because the constraint:
`Functor f => Applicative f` in the typeclass definition, but because for any `Applicative` instance you 
can define a valid `Functor` instance for that `Applicative` as we have seen above using `pure` and `<*>`.

An `Applicative` must satisfy the following laws:

```haskell
pure id <*> x = x 
pure f <*> pure x = pure (f x)
x <*> pure y = pure (\f -> f y) <*> x
pure (.) <*> u <*> v <*> w = u <*> (v <*> w)
```

Together, these laws ensure that every well-typed expression built with `pure` and `<*>` can be rewritten 
in ***applicative style***:

```haskell
pure f <*> x1 <*> x2 <*> ... <*> xn
```

## Applicatives as monoidal functors

Another view at `Applicative`s expressed in the book [Haskell Programming](https://haskellbook.com) 
is as ***monoidal functors***. What's that?! 

The idea is that `Functor`s apply a function to a value within a structure while 
leaving that structure unchanged. 

```haskell
fmap :: (a -> b) -> f a -> f b
```

That is *easy* because the function only receives 1 value wrapped in a structure, so *just* apply the function
to the value and leave the result in a similar structure.

But with `Applicative`, the function itself is also wrapped in a structure, so we get 2 values wrapped in a structure
and we must return a value wrapped in a structure as well:

```haskell
<*> :: f (a -> b) -> f a -> f b
```

So there are two... *sides* in `Applicative` apply (`<*>`):
- the function application 
- and the *reduction* of the 2 structures containing the function and its argument.

For example: 

```haskell
ghci> (Nothing :: Maybe (Int -> Int)) <*> (Just 1 :: Maybe Int)
Nothing
ghci> (Just (+1) :: Maybe (Int -> Int)) <*> (Just 1 :: Maybe Int)
Just 2
ghci> (Just (+1) :: Maybe (Int -> Int)) <*> (Nothing :: Maybe Int)
Nothing
ghci> (Nothing :: Maybe (Int -> Int)) <*> (Nothing :: Maybe Int)
Nothing
```

But although I get the intuition, I find it a bit confusing here as we are *reducing* values from 2 different types:
`Maybe(Int -> Int)` and `Maybe Int`, none of them being `Monoid` instances.

The importance of the `Monoid` constraint to be able to *reduce* the struture becomes more apparent 
with the `Applicative` instance for the 2-tuple: the functor part only applies to the 2nd value 
of the tuple, so how do we *reduce* the 2 1st values:

```haskell
instance Functor ((,) a) where
  fmap f (a, b) = (a, f b)
...

-- do not compile
(1, (+1)) <*> (2, 2) -- (??, 3)
```

To make this works, we **need** the type of the first element of the tuple to be an instance of `Monoid`:

```haskell
instance Monoid a => Applicative ((,) a)
```

```haskell
ghci> import Data.Monoid
ghci> pure (+1) <*> (Sum 1, 3)
(Sum {getSum = 1},4)
```

## Show me some examples!

### Maybe

Our first example of applicative functor is `Maybe`:

```haskell
ghci> pure (+) <*> Just 1 <*> Just 2
Just 3
ghci> pure (+) <*> Nothing <*> Just 2
Nothing
ghci> pure (+) <*> Just 1 <*> Nothing 
Nothing
```

So using `Applicative` we can lift a pure total function to operate on arguments *which may fail*, and turn it into 
a function which may fail if evaluation of its arguments fails, although this function must now be apply using 
the special applicative function application operator `<*>`.

This can be usefull for validation of data constructor arguments. 

Say you have a  `Rectangle` data type whose constructor takes 2 arguments: a width and a height that must be positive:
```haskell
data Rectangle = Rectangle Integer Integer
  deriving (Show, Eq)
```

You can define a `mkLength` function that will *fail* if the given length is negative:

```haskell
mkLength :: Integer -> Maybe Integer
mkLength l = if l > 0 then Just l else Nothing
```

You can then lift your data constructor to get a new constructor that fails if one of the dimension is negative:

```haskell
mkRect :: Integer -> Integer -> Maybe Rectangle
mkRect w h = Rectangle <$> mkLength w <*> mkLength h
```

```haskell
ghci> mkRect 5 3
Just (Rectangle 5 3)
ghci> mkRect 0 3
Nothing
```

### List

The second example is the `[]` and it is a little bit less intuitive:

```haskell
ghci> pure (+) <*> [1, 2, 3] <*> [4, 5, 6]
[5, 6, 7, 6, 7, 8, 7, 8, 9]
```

So what it does is lift a pure function taking (normal) single value arguments and returning a single value 
into a function taking some sort of multi-valued arguments (in the form of a list of values) and returning a *multi-value* containing 
the result of the original function applied to every pairs of single values from the mutli-valued arguments.

So using the list applicative we can get some sort of ***non-deterministic function***.

But that is only one way to define an applicative instance for `[]` and contrary to `Functor` there are more than 
one way to define a valid `Applicative` instance for ***polymorphic type***. 
We might want to apply the `(+)` function pairwise, i.e. 
to the first values in the argument lists, and then to the 2nd values, and so on... 

Just as for the multiple `Monoid` instances of numerical types, we can define that `Applicative` instance on a newtype.

```haskell
ghci> pure (+) <*> ZipList [1, 2, 3] <*> ZipList [4, 5, 6]
ZipList {getZipList = [5, 7, 9]}
```

## Are Applicative stacked like Functors ? 

In the previous [post](/blog/the-fp-epiphany-functors/) about `Functor`, I mentioned that `Functor`s are stacked.
That is you can `fmap` a function at any *depth* in a stack of `Functor` by composing `fmap`, for instance:

```haskell
ghci> (fmap . fmap) (+1) [Just 1, Just 2]
[Jsut 2, Just 3]
```

What if I want to add elements from lists of `Maybe Integer`? 

```haskell
ghci> pure ??? <*> [Just 1, Just 2] <*> [Nothing, Just 3]
```

Which value shall I pass to the `pure` function above? 

We need a function of type `Maybe Integer -> Maybe Integer -> Maybe Integer`.

Do you remember our `fmap2` above? Its type was:

```haskell
fmap2 :: Applicative f => (a -> b -> c) -> f a -> f b -> f c
```

So if we partially apply it to the `(+)` function, we get what we need in return: 

```haskell
(+) :: Num a => a -> a -> a
fmap2 (+) :: (Applicative f, Num a) => f a -> f a -> f a
```

It turns out that `fmap2` is provided to you by Haskell under the name `liftA2` in `Control.Applicative`:

```haskell
ghci> pure (liftA2 (+)) <*> [Just 1, Just 2] <*> [Nothing, Just 3]
[Nothing, Just 4, Nothing, Just 5]
```

Which can be rewritten a bit nicer using `<$>`:

```haskell
ghci> liftA2 (+) <$> [Just 1, Just 2] <*> [Nothing, Just 3]
[Nothing, Just 4, Nothing, Just 5]
```

