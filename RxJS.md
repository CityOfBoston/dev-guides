# RxJS

## Resources

 * [Learning Observable By Building Observable](https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87)

## Notes

### Creating chains

A chain off of an `Observable` factory method does not make an observable
itself. It defines the **pattern** of operations that will be kicked off when
`subscribe` is called. When you call `subscribe` on the end of this chain, it
causes the previous operation to `subscribe`, and soforth recursively back to
the “root” `Observable`/`Subject`. This is why when you `subscribe` to the same
chain twice all of the operations will be duplicated when values come through.
The `subscribe` recursion creates two completely distinct `Observable` chains.

Use `share()` in order to have two subscriptions share a same source chain.
`share` makes an observable that will broadcast values from its source out to
all of its subscribers.

### Work on Subscription

Observable roots typically do their work only when they are subscribed to:

 * `fromEvent` registers an event listener
 * `defer` runs its factory method
 * `of` starts pushing its values out

 This is how `retry` works: on error, it causes a resubscribe back to the root,
 which then repeats the original work (and therefore the chained `Observer`s
 that follow).

### Subjects

“Probably a more important distinction between Subject and Observable is that a
Subject has state, it keeps a list of observers. On the other hand, an
Observable is really just a function that sets up observation.” — [On The
Subject Of
Subjects](https://medium.com/@benlesh/on-the-subject-of-subjects-in-rxjs-2b08b7198b93)

`Observable`s do not keep a list of `Observer`s, since each `Observable` chain
is separate for every `subscribe` call.

### Errors

An error is the last thing that will come from an `Observable`. **Full stop.**
Once an `Observable` is in an error state, it unsubscribes from its parent,
which chains back to its source.

Even if you use `catch` to “keep going” from an error, you’re only affecting
downstream `Observer`s from the `catch`. You keep the error from propagating
further, but the result from `catch` will presumably be the last `next` value
that they see coming in.

This means that if you want an error to only terminate _part_ of your observer
chain (for example, an API error that doesn’t bubble up to cause a `fromEvent`
`Observable` from deregistering its listener) you _must_ use a temporary,
intermediate `Observable`.

For example:

```
Rx.Observable
  .fromEvent(source, 'click')
  .mergeMap(
    (ev) =>
      // intermediate observable that can error out without failing
      // the “fromEvent” observable
      Rx.Observable
        .of(ev)
        .map((ev) => somethingThatCouldThrow(ev))
        .catch((err) => Rx.Observable.empty()))
  .do((result) => …)
  .subscribe({
    error: () => console.log("THIS SHOULD NEVER HAPPEN")
  });
```

In this case, errors from `somethingThatCouldThrow` only make it back as far as
the `Observable.of`. We have a `catch` that transforms the error into an
`empty`, which completes this inner chain (and causes `mergeMap` to stop merging
it in).

It is _very_ important to keep errors from bubbling back up to the source of
long-running (_e.g._ event-listening) roots.

### Promises

RxJS automatically wraps `Promise`s when `Observable`s are expected, turning
their `resolve`s into `next`/`complete` events, and `reject`s into `error`s.

`Observable.defer` is a great tool for working with promises. Create the
`Promise` and return it from the factory function you pass to `defer`. Every
time a subscription reaches the `defer` `Observable`, it will call the factory
function and trigger the creation of the `Promise`. 

This pattern is ideal for doing `retry`s, since `retry` works by re-subscribing
(which calls the factory funciton, which makes the promise, _&c._).

### Custom Operators and `let`

To make a custom operator (for example, to do advanced queuing or retrying with
exponential fallback), write it as a function from `Observable -> Observable`.
It should take an input `Observable`, call chained methods on it (_e.g._
`mergeMap`, `catch`, _&c._) and return the result.

You can then insert your custom operator function into a chain by using `let`.
