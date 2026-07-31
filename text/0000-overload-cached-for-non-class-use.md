---
stage: accepted
start-date: 2026-07-30T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/NullVoxPopuli/rfcs/pull/19
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# Overload `cached` to work outside of classes 

## Summary

This RFC introduces an overload to the existing `cached` function, allowing it to be used outside of classes.

This is the derived-state companion to [RFC#1071](https://github.com/emberjs/rfcs/blob/master/text/1071-overload-tracked-for-non-class-use.md), which overloaded `tracked` for use outside of classes, and this RFC re-uses the interfaces defined there.

## Motivation

Our documentation / guides currently don't have much of anything on reactivity, and over the years, it's been useful to talk about reactive primitives as things _outside_ of classes, and compose/wrap them in to refactoring boundaries (classes, components, etc). 

[RFC#1071](https://github.com/emberjs/rfcs/blob/master/text/1071-overload-tracked-for-non-class-use.md) gave us `tracked()` for _root state_ outside of classes, but there is no ergonomic equivalent for _derived state_ -- today, memoized derivation outside of a class requires either a class with a `@cached` getter, or dropping down to the memoization primitives from [RFC#615](https://github.com/emberjs/rfcs/blob/master/text/0615-autotracking-memoization.md).

Enabling `cached` to be used outside of a class makes it a good tool for demos[^demos] for creating derived values in function-based APIs, such as _helpers_, _modifiers_, or _resources_ (or even in module space)[^apps]. They also provide a benefit in testing as well, since tests tend to want to assert on some derived state. 

This is not too dissimilar to the [Autotracking Memoization primitives in RFC#615](https://github.com/emberjs/rfcs/blob/master/text/0615-autotracking-memoization.md) (`createCache` / `getValue`). Making `cached` work outside of classes provides the same benefit without requiring 2 imports from a `primitives` path to use. This RFC intends to provide a tool enabling us to de-emphasize (and potentially later deprecate) the `@glimmer/tracking/primitives/cache` import path for app developers[^primitives-future]. 

`cached`-as-non-decorator was prototyped in [Starbeam](https://starbeamjs.com/guides/fundamentals/functions.html) (as `CachedFormula`) and similar utilities have been available for folks to try out in ember via [ember-resources](https://github.com/NullVoxPopuli/ember-resources) and [reactiveweb](https://github.com/universal-ember/reactiveweb). 

[^apps]: Apps typically should not have reactive state in module space, becaues it doesn't get automatically reset between tests, since we don't reload modules between each tests (partly for perf reasons). Derived state in module space is safer than root state (it resets when its inputs reset), but the same caution applies to what it reads.

[^demos]: demos _must_ over simplify to bring attention to a specific concept. Too much syntax getting in the way easily distracts from what is trying to be demoed. This has benefits for actual app development as well though, as we're, by focusing on concise demo-ability, gradually removing the amount of typing needed to create features. 

[^primitives-future]: The primitives remain useful for library authors, and the implementation of `cached()` would sit on the same machinery -- but most application code should never need to import from a `primitives` path.

## Detailed design

Developers will continue to use: 

```ts
import { cached } from '@glimmer/tracking';
```

however, when called with a function (not used as a decorator), a different return value will be available:
- a `ReadOnlyReactive`

> [!IMPORTANT]
> This particular return value gives us the abilitiy in the future guides talking about reactivity a way to describe what the `@cached` decorator is doing (since decorators are not in every ecosystem), we can describe it as a syntactic sugar on top of a `ReadOnlyReactive`.

### Types 

This RFC re-uses the interfaces defined in [RFC#1071](https://github.com/emberjs/rfcs/blob/master/text/1071-overload-tracked-for-non-class-use.md):

~~~ts
interface Reactive<Value> {
   /**
    * The underlying value
    *
    * Allows easy usage of reactive values in templates.
    */
    value: Value;
}

// Useful internal concept for optimizations
interface ReadOnlyReactive<Value> extends Reactive<Value> {
    /**
    * The underlying value.
    * Cannot be set.
    */
    readonly value: Value;
}
~~~

and adds:

~~~ts
/**
* Utility to create a cached (memoized) derived value. 
*/
function cached<Value>(
    fn: () => Value,
    options?: { 
        equals: (a: Value, b: Value) => boolean, 
        description?: string 
    } = {}
) {
  return new CachedValue(
    fn,
    {
      equals: options?.equals ?? Object.is,
      description: options?.description
    }
  );
}

interface CachedValue<Value> extends ReadOnlyReactive<Value> {
    /**
    * Function short-hand of reading the value
    * of the CachedValue
    */
    get: () => Value;
}
~~~

Unlike RFC#1071's `TrackedValue`, there is no `set`, `update`, or `freeze` -- a `CachedValue` is derived entirely from the tracked state its function reads, so it is _born_ a `ReadOnlyReactive`.

Behaviorally, `cached()` behaves almost the same as this function:
```js
import { createCache, getValue } from '@glimmer/tracking/primitives/cache';

function cached(fn, { equals, description } = {}) {
  return new CachedValuePolyfill(fn, { equals: equals ?? Object.is, description });
}

class CachedValuePolyfill {
    #cache;
    #hasPrevious = false;
    #previous;

    constructor(fn, options) {
        this.#cache = createCache(() => {
            let next = fn();

            if (this.#hasPrevious && options.equals(this.#previous, next)) {
                // retain the previous value (and its identity)
                return this.#previous;
            }

            this.#hasPrevious = true;
            this.#previous = next;
            return next;
        }, options.description);
    }

    get value() {
        // reading entangles with the tracked state `fn` reads
        return getValue(this.#cache);
    }

    get() {
        return this.value;
    }
}
```

(the real implementation would live lower in the reactivity system, because less abstraction layers are speedier)

The function passed to `cached` is only re-invoked when tracked state it previously read has changed -- exactly the memoization semantics of the `@cached` decorator from [RFC#566](https://github.com/emberjs/rfcs/blob/master/text/0566-memo-decorator.md).

The `equals` option decides whether a re-computed value _counts_ as a new value. When the freshly computed value is `equals` to the previous one, the previous value is returned instead, preserving referential identity for downstream consumers. The default is `Object.is`.

For example, with this `CachedValue` and equality function:

```gjs
const letters = tracked(['a', 'b']);

const upper = cached(
    () => letters.value.map((letter) => letter.toUpperCase()),
    { equals: (a, b) => a.length === b.length && a.every((x, i) => x === b[i]) }
);

const reassign = () => letters.value = ['a', 'b'];

<template>
    <output>{{upper.value}}</output>

    <button {{on 'click' reassign}}>Click me</button> 
</template>
```

Clicking the button re-runs the function (the `tracked` value was dirtied), but `upper.value` keeps returning the _same array instance_, so anything consuming `upper.value` does not need to re-process it.

### Usage

Deriving from local state in a template.

```gjs
import { tracked, cached } from '@glimmer/tracking';

const double = (reactive) => cached(() => reactive.value * 2);
const increment = (c) => c.value++;

<template>
    {{#let (tracked @initialCount) as |count|}}
        {{#let (double count) as |doubled|}}
            Count is: {{count.value}}, doubled is: {{doubled.value}}

            <button {{on "click" (fn increment count)}}>add one</button>
        {{/let}}
    {{/let}}
</template>
```

Deriving from module state.
This is already common in demos.

```gjs
import { tracked, cached } from '@glimmer/tracking';

const count = tracked(0);
const doubled = cached(() => count.value * 2);
const increment = () => count.value++;

<template>
    Count is: {{count.value}}, 
    doubled is: {{doubled.value}}

    <button {{on "click" increment}}>add one</button>
</template>
```

Using private mutable properties providing public, memoized, read-only access:

```gjs
export class MyAPI {
    #state = tracked(0);

    #expensive = cached(() => veryExpensiveFunction(this.#state.value));

    get expensive() {
        return this.#expensive.value;
    }

    doTheThing() {
        this.#state.value = secretFunctionFromSomewhere(); 
    }
}
```

### Re-implementing `@cached` 

> [!NOTE]
> This is a conceptual exercise, and for performance reasons it won't be implemented this way

For most current ember projects, using the TC39 Stage 1 implementation of decorators:

```js
import { cached as glimmerCached } from '@glimmer/tracking';

function cached(target, key, { get }) {
  let caches = new WeakMap();

  function getCache(obj) {
    let cache = caches.get(obj);

    if (cache === undefined) {
      cache = glimmerCached(() => get.call(obj), { description: `cached:${key}` });
      caches.set(obj, cache);
    }

    return cache;
  };

  return {
    get() {
      return getCache(this).value;
    },
  };
}
```

<details><summary>Using spec / standards-decorators</summary>

```js
import { cached as glimmerCached } from '@glimmer/tracking';
    
export function cached(target, context) {
  let caches = new WeakMap();

  return function (this: object) {
    let cache = caches.get(this);

    if (cache === undefined) {
      cache = glimmerCached(() => target.call(this), { description: `cached:${String(context.name)}` });
      caches.set(this, cache);
    }

    return cache.value;
  };
}
```

</details>

## How we teach this

The `cached` function is a low-level tool, for folks that want specific behavior and for most real applications, folks should continue to use classes, with `@cached` getters, as the combination of classes with decorators provide unparalleled ergonomics in state management. 

However, developers may think of `@cached` (or decorators in general) as magic -- we can utilize `cached()` as a storytelling tool to demystify how `@cached` works -- since `cached()` will be public API, we can easily explain how `cached()` is used to _create the `@cached` decorator_ (without discussing the real private APIs that we _don't_ want folks using (such as those exported from `@glimmer/validator`).

We can even use the example over-simplified implementation of `@cached` from the _Detailed Design_ section above.

Together with `tracked()` from RFC#1071, this completes the story for teaching reactivity without classes: `tracked()` is root state, `cached()` is derived state.

### When to use `value`

Allows for easy use in templates as well as in getters:

```gjs
import { tracked, cached } from '@glimmer/tracking';

const count = tracked(0);
const doubled = cached(() => count.value * 2);
const increment = () => count.value++;

<template>
    Count is: {{count.value}},
    doubled is: {{doubled.value}}

    <button {{on "click" increment}}>add one</button>
</template>
```

### When to use `get()`

Allows passing the read as a function, e.g. to utilities that accept a thunk:

```gjs
import { tracked, cached } from '@glimmer/tracking';

const count = tracked(0);
const doubled = cached(() => count.value * 2);

const logLater = (read) => setTimeout(() => console.log(read()));

<template>
    {{doubled.value}}

    <button {{on "click" (fn logLater doubled.get)}}>log it</button>
</template>
```

### When to use `equals`

When re-computation may produce a new-but-equivalent value (arrays, objects, strings built from parts), and downstream consumers benefit from referential stability -- for example, avoiding re-renders of `{{#each}}` blocks or re-runs of downstream `cached` values that received an equivalent value.

## Drawbacks

- same API does multiple things based on usage, but developers should be used to this somewhat as overloading is nothing new -- TS will also be agreeable with the overloads -- and RFC#1071 has already established this pattern for `tracked`
- potential confusion between `@cached` (decorates a getter) and `cached(fn)` (wraps a function) -- though the mental model is the same: "memoize this computation against the tracked state it reads"

## Alternatives

- completetly new API, such as `memo` or `formula`
- continue pointing folks at `createCache` / `getValue` from `@glimmer/tracking/primitives/cache` (2 imports, and a `primitives` path that signals "not for app developers")

## Unresolved questions

- none yet

## Appendix

### Naming: value

Consistent with RFC#1071's `TrackedValue`. Value is generic enough, and is a generally understood concept without nuance.

### Naming: `get` and `read` 

The same reasoning as RFC#1071 applies:

- `get` implies that you are always going to do something with what is given to you 
- `read` somewhat implies that you want to see the state of the cached value, but is ambigous about if you want to do anything with that information

`get`, in particular, (while ~ unfortunately ~, matches legacy naming in our history), matches existing JS concepts from Map, WeakMap, other other concepts.

### Why no `set`, `update`, or `freeze`

A `CachedValue` has no storage of its own -- its value is entirely a function of the tracked state its function reads. Writing to it is meaningless, and it is already permanently "frozen" from the consumer's point of view. This is also why `CachedValue` extends `ReadOnlyReactive` rather than `Reactive`.

### Extension

If folks wanted, they could make their own cached value with previous or historical values. This could be useful for extremely expensive operations that depend on previous computations. 
To do this, folks would need to implement their own class:
```js
class CachedValueWithHistory {
    #previous;
    #current;

    constructor(fn) {
        this.#current = cached(() => {
            this.#previous = untrack(() => this.#current?.value);
            return fn(this.#previous);
        });
    }

    get value() {
        return this.#current.value;
    }

    get previous() {
        return this.#previous;
    }

    // ...
}
```
