# General Debugging

The following sections are tips for debugging issues that may arise in a strongly-typed language via the compiler.

## There is currently no "Actual Type / Expected Type" distinction

In the following error...
```
  Could not match type

    A

  with type

    B

  ... rest of error ...
```

... one might expect `A` to be the "actual" type and `B` to be the "expected" type. However, sometimes the two are swapped, so that `A` is the "expected" type and `B` is the "actual" type. This is not desirable, but is currently how the compiler works.

Why? Because [the compiler uses a mixture of unification and type inference to check types](https://github.com/purescript/purescript/issues/3111#issuecomment-335596641). See [purescript/purescript#3399](https://github.com/purescript/purescript/issues/3399) for more information.

## Distinguishing the Difference between `{...}` and `(...)` errors

(thomashoneyman recommended I document this. These examples might be incorrect since I am not fully aware of the comment that garyb made, but the general idea still applies.)

Recall that `{ label :: Type }` is syntax sugar for `Record (label :: Type)`

So, the below error means a `Record` could not unify with some other type:
```
  Could not match type
    { label :: Type }
  with type
    String
```

Whereas the below error means a `Record` was the correct type, but some of its label-type associations were missing.
```
  Could not match type
    Record (label :: Type)
  with type
    Record (label :: Type, forgottenLabel :: OtherType)
```

## Type Directed Search

Otherwise known as "typed holes."

If you recall in `Syntax/Basic Syntax/src/Data-and-Functions/Type-Directed-Search.md`, we can use type-directed search to
1. help us determine what an entity's type is
2. guide us in how to implement something
3. see better ways to code something via type classes

As an example, let's say we have a really complicated function or type
```purescript
main :: Effect Unit
main = do
  a <- computeA
  b <- computeB
  c <- (\a -> (\c -> doX c) <$> box a) <$> (Box 5) <*> (Box 8)
```
If we want to know what the type will be for `doX`, we can rewrite that entity using a type direction search, `?doX`, and see what the compiler outputs:
```purescript
main :: Effect Unit
main = do
  a <- computeA
  b <- computeB
  c <- (\a -> (\c -> ?doX c) <$> box a) <$> (Box 5) <*> (Box 8)
```
If we're not sure what type a function outputs, we can precede the function with our search using `?name $ function`:
```purescript
main :: Effect Unit
main = do
  a <- computeA
  b <- computeB
  c <- (\a -> (\c -> ?help $ doX c) <$> box a) <$> (Box 5) <*> (Box 8)
```

If you encounter a problem or need help, this should be one of the first things you use.

## Getting the Type of an Expression from the Compiler

This is known as "typed wildcards".

In a function body, wrapping some term with `(term :: _)` will cause the compiler to infer the type for you.

```purescript
main :: Effect Unit
main = do
  a <- computeA
  b <- computeB
  c <- (\w x -> ((doX x) :: _)) <$> box a) <$> (Box 5) <*> (Box 8)
```

## Getting the Type of a Function from the Compiler

There are two possible situations where this idea might help:
- You know how to write the body for a function but aren't sure what it's type signature is
- You're exploring a new unfamiliar library and are still figuring out how to piece things together. So, you attempt to write the body of a function but aren't sure what it's type signature will be.

In such cases, we can completely omit the type signature and the compiler will usually infer what it is for us:
```purescript
-- no type signature here for `f`,
-- so the compiler will output a warning
-- stating what its inferred type is
f = (\a -> (\c -> doX c) <$> box a) <$> (Box 5) <*> (Box 8)
```

However, the above is not always useful when one only wants to know what the type of either an argument or the return type. In such situations, one can use typed wildcards from above in the type signature:
```purescript
doesX :: String -> _ -> Int
doesX str anotherString = length (concat str anotherString)
```

## Determining why a type was inferred incorrectly

Sometimes, I wish we could have a 'unification trace' or a 'type inference trace'. I know the code I wrote works, but there's some mistake somewhere in my code that's making the compiler infer the wrong type at point X, which then produces the type inference problem much later at point Y. To solve Y, I need to fix the problem X, but I'm not sure where X is.

Here's an example:
```purescript
type Rec = { a :: String }

f :: String -> String
f theString = wrap (unwrap theString)

  where
    wrap :: String -> Rec
    wrap theString = { a: theString }

    {-
      the mistake! Compiler says
      Cannot match type
        { a :: String }
      with type
        { a :: String, b :: String }
    unwrap :: Rec -> String
    unwrap rec = rec.b
```

From the Slack channel, garyb mentioned passing the `--verbose-errors` flag to the compiler. **This will output a LOT of information**, but it's that or nothing. To do that, run this code:

```bash
pulp --psc-package build -- --verbose-errors
pulp --psc-package build -- -v
```
