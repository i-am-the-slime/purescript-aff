# purescript-aff

[![Latest release](http://img.shields.io/github/release/slamdata/purescript-aff.svg)](https://github.com/slamdata/purescript-aff/releases)
[![Build status](https://travis-ci.org/slamdata/purescript-aff.svg?branch=master)](https://travis-ci.org/slamdata/purescript-aff)

An asynchronous effect monad and threading model for PureScript.

# Example

```purescript
main = launchAff do
  response <- Ajax.get "http://foo.bar"
  log response.body
```

# Getting Started

## Installation

```
bower install purescript-aff
```

## Introduction

An example of `Aff` is shown below:

```purescript
deleteBlankLines path = do
  contents <- loadFile path
  let contents' = S.join "\n" $ A.filter (\a -> S.length a > 0) (S.split "\n" contents)
  saveFile path contents'
```

This looks like ordinary, synchronous, imperative code, but actually operates
asynchronously without any callbacks. Error handling is baked in so you only
deal with it when you want to.

The library contains instances for `Semigroup`, `Monoid`, `Apply`,
`Applicative`, `Bind`, `Monad`, `Alt`, `Plus`, `MonadEff`, and `MonadError`.
These instances allow you to compose asynchronous code as easily as `Eff`, as
well as interop with existing `Eff` code.

## Escaping Callback Hell

Hopefully, you're using libraries that already use the `Aff` type, so you
don't even have to think about callbacks!

If you're building your own library, then *purescript-aff* provides a
`makeAff` function:

```purescript
makeAff :: forall eff a. ((Either Error a -> Eff eff Unit) -> Eff eff (Canceler eff)) -> Aff eff a
```

This function expects you to provide a handler, which should call the
supplied callback with the result of the asynchronous computation.

You should also return `Canceler`, which is just a cleanup effect. Since
`Aff` threads may be killed, all asynchronous operations should provide a
mechanism for unscheduling it.

*purescript-aff* also provides functions for easily binding FFI definitions in
`Control.Monad.Aff.Compat`.

```javascript
exports._ajaxGet = function (request) { // accepts a request
  return function (onError, onSuccess) { // and callbacks
    var req = doNativeRequest(request, function (err, response) { // make the request
      if (err != null) {
        onError(err); // invoke the error callback in case of an error
      } else {
        onSuccess(response); // invoke the success callback with the reponse
      }
    });

    // Return a canceler, which is just another Aff effect.
    return function (cancelError) {
      return function (cancelerError, cancelerSuccess) {
        req.cancel(); // cancel the request
        cancelerSuccess(); // invoke the success callback for the canceler
      };
    };
  };
};
```

```purescript
foreign import _ajaxGet :: forall eff. Request -> EffFnAff (ajax :: AJAX | eff) Response
```

We can wrap this into an asynchronous computation like so:

```purescript
ajaxGet :: forall eff. Request -> Aff (ajax :: AJAX | eff) Response
ajaxGet = fromEffFnAff <<< _ajaxGet
```

This eliminates callback hell and allows us to write code simply using `do`
notation:

```purescript
example = do
  response <- ajaxGet req
  log response.body
```

## Eff

All purely synchronous computations (`Eff`) can be lifted to asynchronous
computations with `liftEff` defined in `Control.Monad.Eff.Class` (see
[here](https://github.com/purescript/purescript-eff)).

```purescript
import Control.Monad.Eff.Class

liftEff $ log "Hello world!"
```

This lets you write your whole program in `Aff`, and still call out to
synchronous code.

If your `Eff` code throws exceptions (`err :: Exception`), you can remove the
exception label using `liftEff'`. Exceptions are part of `Aff`s built-in
semantics, so they will always be caught and propagated anyway.

## Dealing with Failure

`Aff` has error handling baked in, so ordinarily you don't have to worry
about it.

When you need to deal with failure, you have a few options.

 1. **Alt**
 2. **MonadError**
 3. **Bracketing**

#### 1. Alt

Because `Aff` has an `Alt` instance, you may also use the operator `<|>` to
provide an alternative computation in the event of failure:

```purescript
example = do
  result <- Ajax.get "http://foo.com" <|> Ajax.get "http://bar.com"
  pure result
```

#### 2. MonadError

`Aff` has a `MonadError` instance, which comes with two functions:
`catchError`, and `throwError`.

These are defined in
[purescript-transformers](http://github.com/purescript/purescript-transformers).
Here's an example of how you can use them:

```purescript
example = do
  resp <- Ajax.get "http://foo.com" `catchError` \_ -> pure defaultResponse
  when (resp.statusCode /= 200) do
    throwError myErr
  pure resp.body
```

#### 3. Bracketing

`Aff` threads can be cancelled, but sometimes we need to guarantee an action
gets run even in the presence of exceptions or cancellation. Use `bracket` to
acquire resources and clean them up.

```purescript
example =
  bracket
    (openFile myFile)
    (\file -> closeFile file)
    (\file -> appendFile "hello" file)
```

In this case, `closeFile` will always be called regardless of exceptions once
`openFile` completes.

## Forking

Using `forkAff`, you can "fork" an asynchronous computation, which means
that its activities will not block the current thread of execution:

```purescript
forkAff myAff
```

Because Javascript is single-threaded, forking does not actually cause the
computation to be run in a separate thread. Forking just allows the subsequent
actions to execute without waiting for the forked computation to complete.

Forking returns a `Fiber eff a`, representing the deferred computation. You can
kill a `Fiber` with `killFiber`, which will run any cancelers and cleanup, and
you can observe a `Fiber`'s final value with `joinFiber`. If a `Fiber` threw
an exception, it will be rethrown upon joining.

```purescript
example = do
  fiber <- forkAff myAff
  killFiber (error "Just had to cancel") fiber
  result <- try (joinFiber fiber)
  if isLeft result
    then (log "Canceled")
    else (log "Not Canceled")
```


## AVars

The `Control.Monad.Aff.AVar` module contains asynchronous variables, which
are very similar to Haskell's `MVar`. These can be used as low-level building
blocks for asynchronous programs.

```purescript
example = d
  v <- makeEmptyVar
  _ <- forkAff do
    delay (Milliseconds 50.0)
    putVar v 1.0
  a <- takeVar v
  log ("Succeeded with " <> show a)
```

## Parallel Execution

The `Parallel` instance for `Aff` makes writing parallel computations a breeze.

Using `parallel` from `Control.Parallel` will turn a regular `Aff` into
`ParAff`. `ParAff` has an `Applicative` instance which will run effects in
parallel, and an `Alternative` instance which will race effects, returning the
one which completes first (canceling the others). To get an `Aff` back, just
run it with `sequential`.

```purescript
-- Make two requests in parallel
example =
  sequential $
    Tuple <$> parallel (Ajax.get "https://foo.com")
          <*> parallel (Ajax.get "https://bar.com")
```

```purescript
-- Make a request with a 3 second timeout
example =
  sequential $ oneOf
    [ parallel (Just <$> Ajax.get "https://foo.com")
    , parallel (Nothing <$ delay (Milliseconds 3000.0))
    ]
```

```purescript
tvShows =
  [ "Stargate_SG-1"
  , "Battlestar_Galactics"
  , "Farscape"
  ]

getPage page =
  Ajax.get $ "https://wikipedia.org/wiki/" <> page

-- Get all pages in parallel
allPages = parTraverse getPage tvShows

-- Get the page that loads the fastest
fastestPage = parOneOfMap getPage tvShows
```

# API Docs

API documentation is [published on Pursuit](http://pursuit.purescript.org/packages/purescript-aff).
