+++
title = "TypeScript Piped Inference"
date = 2023-01-29
[taxonomies]
tag=["TypeScript","Types", "Inference"]
+++

# Summary

In this article, we demonstrate a powerful TypeScript trick that allows types to
be inferred using the `extends` keyword on `unknown` types.

# The challenge

We want to be able to write something like this:

```typescript
const bar: 'bar' =
  getMember(
    { foo: 'bar', value: 123 } as const,
    'foo')
```

If you'd like, go ahead and give this a try in the [TypeScript
Playground](https://www.typescriptlang.org/play) before reading on...

# Generic functions

Let's say that we want to write a function which returns its only argument. This
is trivially easy in JavaScript:

```typescript
function identity(value) {
  return value
}
```

However, in TypeScript, this gives us a type error (when `noImplicitAny` is
enabled... you're using `noImplicitAny`, right?):

```typescript
// Parameter 'value' implicitly has
// an 'any' type.(7006)
function identity(value) {
  return value
}
```

We could fix the type that the user is going to supply:

```typescript
function identity(value: number): number {
  return value
}
```

But now this function only works for numbers... Is there a better way?

Yes! We can make this a _generic function_:

```typescript
function identity<A>(value: A): A {
  return value
}
```

When usages of `identity` are type-checked, TypeScript substitutes `A` for the
given type. For example, `identity` works for `number`:

```typescript
const value: number = identity<number>(1234)
```

TypeScript can also _infer_ the type of its argument, so we don't need to
specify the type variable:

```typescript
const value: number = identity(1234)
```

## Back to our challenge...


Now that we know a little more about how generics work, let's see if we can
leverage them to implement `getMember`:

```typescript
function getMember
  <T, R extends Record<string, T>>
  (record: R, field: keyof R): T {
  return record[field]
}
```

Unfortunately this does not work like we would want it to:

```typescript
const bar = getMember(
  { foo: 'bar', value: 123 } as const,
  'foo')
// bar: 'bar' | 123
```

The problem here is that `R` is typed as `Record<string, 'bar' | 123>`, because
`T` is typed as `'bar' | 123` in order to cover the type of all values in our
record.

This is a bit of a problem - how are we going to specify a type for each field
in our record?

# Piped inference

The trick is to leverage type inference here, using a combination of `extends`
and `unknown`:

```typescript
function getMember
  <R extends Record<string | number | symbol, unknown>,
   F extends keyof R>
  (record: R, field: F): R[F] {
  return record[field]
}
```

We need to type this with enough information for `R` to be dereferenced - in
this case, it needs to be a record - and we need to use `F extends keyof R` in
order to narrow down the key we're using.

To see why this works, we can expand the type variables (and in this case, we
can get rid of the `as const` assertion, but note that `as const` will help when
the type variables are inferred):

```typescript
const bar = getMember<{foo: 'bar', value: 123}, 'foo'>(
  { foo: 'bar', value: 123 }, 'foo')
// bar: 'bar'
```

I call this technique _piped inference_ - I can't find another name for this in
the wild.

# More recipes

This technique is pretty powerful, and is useful in all sorts of other
situations:

### Variadic combiner

This pattern allows us to write a function which takes a variable number of
arguments, and combine them in a strongly typed way.

The trick here is to take an argument which extends a non-empty tuple of unknown
values, ie. `extends [unknown, ...unknown[]]`.

Let's say we want a function which takes a variadic number of arguments which
may or may not be `undefined`, returns `undefined` if any argument is
`undefined`, or runs a combiner function on the arguments otherwise:

```typescript
combineNonUndefineds(
  'foo' as const,
  'bar' as const,
  'baz' as const,
  (a: 'foo', b: 'bar', c: 'baz') => a + b + c)
```

We can type `combineNonUndefineds` as:

```typescript
/** Similar to `Exclude`, but can exclude `O` from an entire array */
type ExcludeAll<T extends [unknown, ...unknown[]], O> =
  T extends [infer A, ...infer B] ?
    B extends [unknown, ...unknown[]] ? [A, ...ExcludeAll<B, O>] : [A, B]
  : never

function combineNonUndefineds
  <Ts extends [unknown, ...unknown[]], A>
  (...args: [
    ...values: Ts,
    combiner: (...values: ExcludeAll<Ts, undefined>) => A
  ]): A | undefined { /* ... */ }
```

# Takeaways

In general, TypeScript's inference is really powerful. It can be leveraged to
make generics more powerful.

The main lesson here is that generics should be used in order to specify just
enough information to get the job done, and inference can help by making types
based off of generics more specific.
