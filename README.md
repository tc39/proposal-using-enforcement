# Strict Enforcement of `using`

A proposal to extend Explicit Resource Management to add an opt-in strict `using` usage requirement for resources via
`Symbol.enter`.

This is a follow-on proposal originally documented in [tc39/proposal-explicit-resource-management#195](https://github.com/tc39/proposal-explicit-resource-management/issues/195). It's purpose is to add an opt-in mechanism to enforce a stricter
resource management model that forces a user to use `using`, `await using`, `DisposableStack`, and
`AsyncDisposableStack` for resources.

## Status

**Stage:** 1  \
**Champion:** Ron Buckton (@rbuckton)  

_For more information see the [TC39 proposal process](https://tc39.es/process-document/)._

## Authors

- Ron Buckton (@rbuckton)

# Motivations

The current model for resource management in the [Explicit Resource Management][] proposal allows for the creation and
use of _Disposables_ &mdash; objects with a `[Symbol.dispose]()` method whose lifetime is bound to the containing block
scope in tandem with a `using` declaration.

One of the caveats to the current proposal is that there is no way to enforce that a _Disposable_ has been assigned to
a `using` declaration, or attached to a `DisposableStack` object. As a result, users are free to construct a _Disposable_
without registration and neglect to perform cleanup manually.

A mechanism of enforcement is not included in the main proposal in an effort to ease adoption. Initially, it is expected
that the largest source of disposables is likely to come from host APIs, such as those in the DOM, NodeJS, or Electron.
These hosts already have [existing](https://github.com/tc39/proposal-explicit-resource-management?tab=readme-ov-file#relation-to-dom-apis) 
[APIs](https://github.com/tc39/proposal-explicit-resource-management?tab=readme-ov-file#relation-to-nodejs-apis) that
can easily be adapted to support `[Symbol.dispose]()` or `[Symbol.asyncDispose]()`. Were such enforcement to be
mandatory, these hosts would need to implement and maintain a completely parallel set of APIs purely to support `using`.
As a result, we are proposing this as a purely opt-in mechanism that could be leveraged by new APIs that need stronger
enforcement guarantees.

# Proposed Solution

We propose the adoption of an additional resource management symbol &mdash; `Symbol.enter` &mdash; along with additional
semantics for `using`, `await using`, `DisposableStack.prototype.use`, and `AsyncDisposableStack.prototype.use`.

## `using` and `await using` Declarations

When a resource is declared via a `using` or `await using` declaration, we would first inspect the resource for a
`[Symbol.enter]()` method. If such a method is present, it would be invoked (synchronously, in both cases) and its
result would become the actual value that is bound and whose `[Symbol.dispose]()` (or `[Symbol.asyncDispose]()`) method
is registered.

```js
function getStrictResource() {
  // Construct the actual resource.
  const actualResource = {
    value: "resource",
    [Symbol.dispose]() { }
  };

  // Construct and return a resource wrapper that imposes strict enforcement semantics. 
  return {
    [Symbol.enter]() {
      return actualResource;
    }
  };
}

{
  using res = getStrictResource(); // retval[Symbol.enter]() is invoked.
  res.value; // "resource"

} // res[Symbol.dispose]() is invoked.
```

This proposal makes no distinction between `using` and `await using` for the purpose of `[Symbol.enter]()`, as a
`[Symbol.enter]()` method may not be asynchronous. It's function is purely an enforcement guard and is not intended as
a means to delay initialization.

## `DisposableStack` and `AsyncDisposableStack`

Both `DisposableStack.prototype.use()` and `AsyncDisposableStack.prototype.use()` function in a similar fashion to their
syntactic counterparts. If the resource provided has a `[Symbol.enter]()` method, it is invoked and its result is the 
actual resource that is registered and returned from the `use()` method.

## How does `Symbol.enter` cause strict enforcement?

`Symbol.enter` achieves strict enforcement by introducing an additional step between resource acquisition and use. The
strict enforcement wrapper object does not implement any resource-specific capabilities itself. Instead, it guides the
user towards `using` as a more convenient alternative to manually unwrapping by directly invoking `[Symbol.enter]()`.

## What makes this an "opt-in" capability?

If the [Explicit Resource Management][] proposal were to ship with strict enforcement as the only option, hosts would
need to either bifurcate their API surface to introduce new, strictly enforced versions of existing APIs, or add an 
additional `[Symbol.enter]() { return this; }` to their existing APIs to conform.

We believe the most likely driver for adoption of `using` will be as a direct result of adoption in host APIs. We expect
hosts are more likely to chose an approach that provides a path of least resistance for existing developers in an effort
to ease adoption, and thus would expect `[Symbol.enter]() { return this; }` to be the approach that offers the least
resistance.

Many host APIs already guard native resources with a finalizer that is executed during garbage collection. For example,
`fs.promises.FileHandle` in NodeJS will automatically release its underlying file handle if the `FileHandle` instance is
garbage collected. In counterpoint, many user produced disposables are likely either to only contain memory-managed
objects that will already be freed by GC, to be composed of native resources provided by a host API, or a combination
thereof. Due to these factors, strict enforcement is not entirely necessary.

As a result, this proposal introduces `Symbol.enter` as a purely opt-in mechanism. When not present on an object, the
runtime acts as if an implicit `[Symbol.enter]() { return this; }` already exists. This aligns with the "path of least
resistance" approach hosts likely would have taken for existing APIs without the additional overhead of such a method.
It also allows a library or package author the autonomy to require strict enforcement if their API requires it.

## Alternatives

Without `Symbol.enter`, there are two alternatives that API authors can consider as a potential workaround for strict
enforcement:

- Leverage a `FinalizationRegistry` to issue warnings to the console when a resource is garbage collected rather than
  disposed.
- Define `[Symbol.dispose]` as a getter rather than a method.

### Workaround: `FinalizationRegistry`

In this approach, we use a `FinalizationRegistry` to warn users about improper cleanup of resources.

```js
const registry = new FinalizationRegistry(() => {
    console.warn("Resource was garbage collected without being disposed. Did you forget a 'using'?");
});

class Resource {
    ...

    constructor() {
        registry.register(this, undefined, this);
        ...
    }

    [Symbol.dispose]() {
        registry.unregister(this);
        ...
    }
}
```

Here, allocation of `Resource` adds an entry to the finalization registry using itself as the unregister token. When the
`[Symbol.dispose]()` method is invoked, the `Resource` is removed from the finalization registry. If the
`[Symbol.dispose]()` method is not invoked prior to garbage collection, the finalization registry's cleanup callback is
triggered issuing a warning to the console.

### Workaround: `Symbol.dispose` Getter

In this approach, we define `[Symbol.dispose]` as a getter rather than a method. This allows us to guard against
a resource being used in an invalid state with the assumption that a direct access to `[Symbol.dispose]` is enough of an
indication that resource registration has occurred.

```js
class Resource {
  #state = "unregistered"; // one of: "unregistered", "registered", or "disposed"

  ...

  resourceOperation() {
    this.#throwIfInvalid();
    ...
  }

  get [Symbol.dispose]() {
    if (this.#state === "unregistered") {
      this.#state = "registered";
    }
    return Resource.#dispose;
  }

  static #dispose = function() {
    if (this.#state === "disposed") {
      return;
    }
    this.#state = "disposed";
    ...
  };

  #throwIfInvalid() {
    switch (this.#disposeState) {
    case "unregistered":
      throw new TypeError("This resource must either be registered via 'using' or attached to a 'DisposableStack'");
    case "disposed":
      throw new ReferenceError("Object is disposed");
    }
  }
}
```

Here, `Resource` is initially constructed in an `"unregistered"` state. Attempts to perform operations against the
resource are guarded by a call to `#throwIfInvalid()` which checks for both whether the resource has been registered or
whether the resource has already been disposed. Since `[Symbol.dispose]` is a getter, accessing the getter triggers a
state change from `"unregistered"` to `"registered"` and returns the actual dispose method that will be invoked by
`using` at the end of the containing block.

# What is Not Being Proposed

We are explicitly not proposing a feature with the breadth of Python's [Context Managers](https://docs.python.org/3/reference/datamodel.html#with-statement-context-managers).
Context managers allow you to intercept, replace, and even drop exceptions thrown within the context which is a far more
complex mechanism than what is proposed. For further discussion on full exception handling, please refer to
[tc39/proposal-explicit-resource-management#49](https://github.com/tc39/proposal-explicit-resource-management/issues/49).

# Prior Art

- Python [Context Managers](https://docs.python.org/3/reference/datamodel.html#with-statement-context-managers) and
  [`__enter__`](https://docs.python.org/3/reference/datamodel.html#object.__enter__).

# Related Proposals

- [Explicit Resource Management][]

# Potential Semantics

[CreateDisposableResource ( V, hint, method )]() would be modified as follows:

```
  1. If _method_ is not present, then
    1. If _V_ is either *null* or *undefined*, then
      1. Set _V_ to *undefined*.
      1. Set _method_ to *undefined*.
    1. Else,
      1. If _V_ is not an Object, throw a *TypeError* exception.
+     1. Let _enter_ be ? GetMethod(_V_, @@enter).
+     1. If _enter_ is not *undefined*, then
+         1. Set _V_ to ? Call(_enter_, _V_).
+         1. If _V_ is not an Object, throw a *TypeError* exception.
      1. Set _method_ to ? GetDisposeMethod(_V_, _hint_).
      1. If _method_ is *undefined*, throw a *TypeError* exception.
  1. Else,
    1. If IsCallable(_method_) is *false*, throw a *TypeError* exception.
  1. Return the DisposableResource Record { [[ResourceValue]]: _V_, [[Hint]]: _hint_, [[DisposeMethod]]: _method_ }.
```

# API

This proposal would introduce the built-in symbol `@@enter`, accessible as a static property named `enter` on the
`Symbol` constructor.

# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.
* [x] Illustrative [examples][Examples] of usage.
* [x] High-level [API][API].

### Stage 2 Entrance Criteria

* [x] [Initial specification text][Specification].
* [ ] [Transpiler support][Transpiler] (_Optional_).

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].



<!-- # References -->

<!-- Links to other specifications, etc. -->


<!-- * [Title](url) -->


<!-- # Prior Discussion -->

<!-- Links to prior discussion topics on https://esdiscuss.org -->


<!-- * [Subject](https://esdiscuss.org) -->


<!-- The following are shared links used throughout the README: -->

[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: #todo
[Transpiler]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
[Explicit Resource Management]: https://github.com/tc39/proposal-explicit-resource-management
