+++
title = "Converters in Haskell"
date = 2022-07-03
[taxonomies]
tag=["Haskell","Design patterns"]
+++

# Summary

In this article, we explore options for converting from one type to another in
Haskell. We suggest using functions to do this work, unless converting to/from
multiple types - in that case, we suggest using typeclasses, or the
`Convertible` typeclass if you expect to need many converters in your project
and can take on extra dependencies, and you're disciplined enough to not abuse
its power.

# The Problem

I've been working through some [Project Euler](https://projecteuler.net/)
problems lately. Many of the early problems deal with digits in a number; for
example, [finding palindromic numbers](https://projecteuler.net/problem=4).

So, in order to help, I created a simple type for dealing with digits:

```haskell
newtype Digit = Digit Natural
```

`Natural` is nice here - digits in a number are always positve - but `Natural`
is way too big for each digit `1..9`. So I created a smart constructor to parse
digits:

```haskell
toDigit :: Natural -> Either String Digit
toDigit n
  | n >= 0 && n <= 9 = Right $ Digit n
  | otherwise = Left $ "Can't parse " <> show n <> " to a digit!"
```

This is fine and dandy when creating `Digit`s from `Int`s, but it's also common
to create `Digit`s from `Char`s - for example, when turning a `Natural` into a
`[Digit]`:

```haskell
digits :: Natural -> Either String [Digit]
digits = mapM toDigit . show
```

The problem here being that `toDigit :: Char -> Either String Digit` is not
defined. I could map again using some `Char -> Int` here, but it seems more
general (and safer, using `Either` to catch failures) to use `toDigit` here. So,
I need a way to convert `Natural` to `Digit`, and `Char` to `Digit`.


# Prior work

It's quite common to convert from one type to another. Usually the language
you're using will support many conversions to and from builtin types, but it's
common to convert to/from types you define.

## Regular old functions

How about we just write a function to convert one type to another? This is how I
originally approached this, with `toDigit`.

Pros:
  - Works fine for single type conversions. This is probably what you should use
    unless you know that you'll need to convert more than one type.

Cons:
  - It'd be nice to be able to convert multiple types in this way - say,
    having a `toFoo` function that could take a `Bar` or a `Baz`.

## Using typeclasses

How about using a typeclass for each of these? eg.

```haskell
newtype MyNumber = MyNumber Int

class ToDigit a where
  toDigit :: a -> Digit

class PartialToDigit a where
  partialToDigit :: a -> Either String Digit

class FromDigit a where
  fromDigit :: Digit -> a

class PartialFromDigit a where
  partialFromDigit :: Digit -> Either String a
```

Pros:
  - Simple
  - Provides a consistent API: can be adapted for any type using this naming
    convention

Cons:
  - More boilerplate, but can probably be alleviated with some Template Haskell.

## Using the [`convertible`](https://hackage.haskell.org/package/convertible) package

This package declares [the `Convertible`
typeclass](https://hackage.haskell.org/package/convertible-1.1.1.1/docs/Data-Convertible-Base.html#g:1),
for converting from `a` to `b`:

```haskell
type ConvertResult a = Either ConvertError a

class Convertible a b where
  safeConvert :: a -> ConvertResult b
```

There's also a handy `convert` function, but it's partial:

```haskell
convert :: Convertible a b => a -> b
convert x =
    case safeConvert x of
      Left e -> error (prettyConvertError e)
      Right r -> r
```

### Similarities to Scala's implicit conversions

This `Convertible` typeclass reminds me of Scala's implicit conversions in
some ways: you can convert between two types by simply using the `convert`
keyword, as long as there's a way to convert between the two. In this way,
you can make a function take just about any argument as long as it converts
to the type that you need.

For example, you could implement some sort of boolean coercion using
`Convertible`:

```haskell
instance Convertible Int Bool where
  safeConvert n = Right $ n /= 0

-- | Prints "bar".
main :: IO ()
main =
  if convert 0
  then putStrLn "foo"
  else putStrLn "bar"
```

Pros:
  - One typeclass to rule them all
  - Some nice utilities for failure modes

Cons:
  - This technique is pretty powerful, and can be used to make types more
    "dynamic" than they probably should be. Care should be taken to avoid
    over-abstraction.