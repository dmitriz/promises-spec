# Entension proposal fo the [Promises/A+ spec](https://promisesaplus.com/)

## Goals

1. Extend the existing spec with simpler atomic methods that are frequently requested by library writers but currently are not provided.
1. Simplify the current spec.

A *promise* represents the eventual result of an asynchronous operation. The primary way of interacting with a promise remains through its `then` method, which registers callbacks to receive either a promise's eventual value or the reason why the promise cannot be fulfilled. 

The current proposal suggests no change to the existing functionality. In particular, no existing code using [any of the current `Promise` methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) will be affected.


## Current Terminology

1. "promise" is an object or function with a `then` method whose behavior conforms to this specification.
1. "thenable" is an object or function that defines a `then` method. A "promise" is always a "thenable".
1. "value" is any legal JavaScript value (including `undefined`, null, a thenable, or a promise).
1. "exception" is a value that is thrown using the `throw` statement.
1. "reason" is a value that indicates why a promise was rejected.

### Proposed Simplification. 
In this terminology a promise is not excluded from being a possible "value".
However, the currently available methods do not make it possible to construct a promise with fulfilled value being another promise. This proposal aims to remove this restriction and thereby the number of cases and exceptions, making the spec simpler to read, test and use, and without compromising any existing functionality.


## Proposed new Terminology

1. "atomic" construction is one that cannot be broken into several simpler ones.
1. "wrapping" a value `x` into a promise is a basic atomic construction creating a new promise that is fulfilled with `x`. Here `x` is allowed to be a promise `p` that would be wrapped into a promise fulfilled with `p`, making no distinction between `p` and other objects or primitive values and thus working uniformly across the language. (The currently implemented native `Promise.resolve` method can only wrap a non-promise. If a promise is passed, it will instead recusrively unwrap it, making this operation not atomic. Although it is currently not possible to wrap a promise into another promise, we prove here that adding this construction can peacefully coexist along the current spec and will make usage simpler and adressing needs of more people.)
1. "unwrapping" (or "flattening") a promise `p` is another basic atomic construction, converse to the "wrapping", that creates a new promise `np` attempting to "unwrap" `p`, that behaves identically to `p` except when `p` resolves with value equal to another promise `q`. In the latter case, `np` adopts the complete behavior of `q`.
1. "recursive unwrapping" a promise `p` is a non-atomic construction consisting of repeatedly unwrapping as long as possible, i.e. as long as each new unwrapped promise gets fulfilled with another promise. This is precisely the current behavior of `Promise.resolve(p)`. (Warning. Recursive unwrapping e.g. using current `then` and `resolve` methods may lead to infinite loops and system crashes.)


## Proposed new atomic Methods

In declaring the identities below satisfied by the new methods, we write `==` to denote the [terminology of equivalence](https://github.com/fantasyland/fantasy-land#terminology) of values:

>The definition should ensure that the two values can be safely swapped out in a program that respects abstractions. For example:
> - Two lists are equivalent if they are equivalent at all indices.
> - Two JavaScript objects are equivalent when they hold equivalent values for all keys.
> - Two promises are equivalent when they have the same state at the same time, and get fulfilled or rejected with equivalent values.
> - Two functions are equivalent if they yield equivalent outputs for equivalent inputs.

The following new methods are proposed:

1. `of`: class/static method wrapping value into a promise that (together with `map` below) conforms to [the Pointed Functor spec](https://stackoverflow.com/a/41816326/1614973) spec, i.e. satisfying `of(f(x)) == of(x).map(f)`
for all values `x` and functions `f`. No automatic unwrapping occurs as with `resolve`.
1. `map`: instance method transforming promise into promise, that conforms to the [Functor spec](https://github.com/fantasyland/fantasy-land#functor), i.e. satsifying `p.map(t=>t) == p` and `p.map(f).map(g) == p.map(t=>g(f(t)))` for all promises `p` and functions `f`, `g`.
1. `flatMap` (aka `chain`): instance method that conforms to the [Monad spec](https://github.com/fantasyland/fantasy-land#monad), i.e. satisfying `of(x).flatMap(f) == f(x)` and `p.flatMap(of) == p` for all values `x`, promises `p` and functions `f`,
in addition to the Pointed Functor spec.



## Requirements

### Promise States

A promise must be in one of three states: pending, fulfilled, or rejected.

1. When pending, a promise:
    1. may transition to either the fulfilled or rejected state.
1. When fulfilled, a promise:
    1. must not transition to any other state.
    1. must have a value, which must not change (for an object, its reference must not change).
1. When rejected, a promise:
    1. must not transition to any other state.
    1. must have a reason, which must not change.

Here, "must not change" means the strong identity `===`, but does not imply immutability for objects.


### The `then` Method

A promise must provide a `then` method to access its current or eventual value or reason.

A promise's `then` method accepts two arguments:

```js 
promise.then(onFulfilled, onRejected)
```

1. Both `onFulfilled` and `onRejected` are optional arguments:
    1. If `onFulfilled` is not a function, it must be ignored.
    1. If `onRejected` is not a function, it must be ignored.
1. If `onFulfilled` is a function:
    1. it must be called after `promise` is fulfilled, with `promise`'s value as its first argument.
    1. it must not be called before `promise` is fulfilled.
    1. it must not be called more than once.
1. If `onRejected` is a function,
    1. it must be called after `promise` is rejected, with `promise`'s reason as its first argument.
    1. it must not be called before `promise` is rejected.
    1. it must not be called more than once.
1. `onFulfilled` or `onRejected` must not be called until the [execution context](https://es5.github.io/#x10.3) stack contains only platform code. [[3.1](#notes)].
1. `onFulfilled` and `onRejected` must be called as functions (i.e. with no `this` value). [[3.2](#notes)]
1. `then` may be called multiple times on the same promise.
    1. If/when `promise` is fulfilled, all respective `onFulfilled` callbacks must execute in the order of their originating calls to `then`.
    1. If/when `promise` is rejected, all respective `onRejected` callbacks must execute in the order of their originating calls to `then`.
1. `then` must return a promise [[3.3](#notes)].

    ```js
    promise2 = promise1.then(onFulfilled, onRejected);
    ```

    1. If either `onFulfilled` or `onRejected` returns a value `x`, run the Promise Resolution Procedure `[[Resolve]](promise2, x)`.
    1. If either `onFulfilled` or `onRejected` throws an exception `e`, `promise2` must be rejected with `e` as the reason.
    1. If `onFulfilled` is not a function and `promise1` is fulfilled, `promise2` must be fulfilled with the same value as `promise1`.
    1. If `onRejected` is not a function and `promise1` is rejected, `promise2` must be rejected with the same reason as `promise1`.

### The Promise Resolution Procedure

The **promise resolution procedure** is an abstract operation taking as input a promise and a value, which we denote as `[[Resolve]](promise, x)`. If `x` is a thenable, it attempts to make `promise` adopt the state of `x`, under the assumption that `x` behaves at least somewhat like a promise. Otherwise, it fulfills `promise` with the value `x`.

This treatment of thenables allows promise implementations to interoperate, as long as they expose a Promises/A+-compliant `then` method. It also allows Promises/A+ implementations to "assimilate" nonconformant implementations with reasonable `then` methods.

To run `[[Resolve]](promise, x)`, perform the following steps:

1. If `promise` and `x` refer to the same object, reject `promise` with a `TypeError` as the reason.
1. If `x` is a promise, adopt its state [[3.4](#notes)]:
   1. If `x` is pending, `promise` must remain pending until `x` is fulfilled or rejected.
   1. If/when `x` is fulfilled, fulfill `promise` with the same value.
   1. If/when `x` is rejected, reject `promise` with the same reason.
1. Otherwise, if `x` is an object or function,
   1. Let `then` be `x.then`. [[3.5](#notes)]
   1. If retrieving the property `x.then` results in a thrown exception `e`, reject `promise` with `e` as the reason.
   1. If `then` is a function, call it with `x` as `this`, first argument `resolvePromise`, and second argument `rejectPromise`, where:
      1. If/when `resolvePromise` is called with a value `y`, run `[[Resolve]](promise, y)`.
      1. If/when `rejectPromise` is called with a reason `r`, reject `promise` with `r`.
      1. If both `resolvePromise` and `rejectPromise` are called, or multiple calls to the same argument are made, the first call takes precedence, and any further calls are ignored.
      1. If calling `then` throws an exception `e`,
         1. If `resolvePromise` or `rejectPromise` have been called, ignore it.
         1. Otherwise, reject `promise` with `e` as the reason.
   1. If `then` is not a function, fulfill `promise` with `x`.
1. If `x` is not an object or function, fulfill `promise` with `x`.

If a promise is resolved with a thenable that participates in a circular thenable chain, such that the recursive nature of `[[Resolve]](promise, thenable)` eventually causes `[[Resolve]](promise, thenable)` to be called again, following the above algorithm will lead to infinite recursion. Implementations are encouraged, but not required, to detect such recursion and reject `promise` with an informative `TypeError` as the reason. [[3.6](#notes)]

## Notes

1. Here "platform code" means engine, environment, and promise implementation code. In practice, this requirement ensures that `onFulfilled` and `onRejected` execute asynchronously, after the event loop turn in which `then` is called, and with a fresh stack. This can be implemented with either a "macro-task" mechanism such as [`setTimeout`](https://html.spec.whatwg.org/multipage/webappapis.html#timers) or [`setImmediate`](https://dvcs.w3.org/hg/webperf/raw-file/tip/specs/setImmediate/Overview.html#processingmodel), or with a "micro-task" mechanism such as [`MutationObserver`](https://dom.spec.whatwg.org/#interface-mutationobserver) or [`process.nextTick`](http://nodejs.org/api/process.html#process_process_nexttick_callback). Since the promise implementation is considered platform code, it may itself contain a task-scheduling queue or "trampoline" in which the handlers are called.

1. That is, in strict mode `this` will be `undefined` inside of them; in sloppy mode, it will be the global object.

1. Implementations may allow `promise2 === promise1`, provided the implementation meets all requirements. Each implementation should document whether it can produce `promise2 === promise1` and under what conditions.

1. Generally, it will only be known that `x` is a true promise if it comes from the current implementation. This clause allows the use of implementation-specific means to adopt the state of known-conformant promises.

1. This procedure of first storing a reference to `x.then`, then testing that reference, and then calling that reference, avoids multiple accesses to the `x.then` property. Such precautions are important for ensuring consistency in the face of an accessor property, whose value could change between retrievals.

1. Implementations should *not* set arbitrary limits on the depth of thenable chains, and assume that beyond that arbitrary limit the recursion will be infinite. Only true cycles should lead to a `TypeError`; if an infinite chain of distinct thenables is encountered, recursing forever is the correct behavior.
