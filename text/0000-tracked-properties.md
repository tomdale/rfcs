- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Tracked Properties

## Summary

Tracked properties introduce a simpler and more ergonomic system for tracking
state change in Ember applications. By taking advantage of new JavaScript
features, tracked properties allow Ember to reduce its API surface area while
producing code that is both more intuitive and less error-prone.

This simple example shows a `Person` class with three tracked properties:

```js
export default class Person {
  @tracked firstName = 'Chad';
  @tracked lastName = 'Hietala';

  @tracked get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

## Terminology

Because of the occasional overlap in terminology when discussing similar
features, this document uses the following language consistently:

* A **getter** is a JavaScript feature that executes a function to determine the value of a property.
  The function is executed every time the property is accessed.
* A **computed property** is a property on an Ember object whose value is
  produced by executing a function. That value is cached until one of computed
  property's dependencies changes.
* A **tracked getter** is a JavaScript getter that has been 
* A **tracked simple property** is a regular, non-getter property that has been
  instrumented to detect when it is mutated.
* A **tracked property** refers to any property that has been instrumented. 

## Motivation

Tracked properties are designed to be simpler to learn, simpler to write, and
simpler to maintain than today's computed properties. In addition to clearer
code, tracked properties eliminate the most common sources of bugs and mental
model confusion in computed properties today.

### Leverage Existing JavaScript Knowledge

Ember's computed properties provide functionality that overlaps with native
JavaScript getters and setters. Because native getters don't provide Ember
with the information it needs to track changes, it's not possible to use them
reliably in templates or in other computed properties.

New learners have to "unlearn" native getters, replacing them with Ember's
computed property system. Unfortunately, this knowledge is not portable to
other applications that don't use Ember that developers may work on in the
future.

Tracked properties are as thin a layer as possible on top of native
JavaScript. Tracked properties look like a normal properties because they
*are* normal properties.

Because there is no special syntax for retrieving a tracked property,
any JavaScript syntax that feels like it should work does work:

```js
// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];
```

Similarly, syntax for changing properties works just as well:

```js
// Simple assignment
this.firstName = "Yehuda";
// Addition assignment (+=)
this.lastName += "Katz";
// Increment operator
this.age++;
```

This compares favorably with APIs from other libraries, which becomes more
verbose than necessary when JavaScript syntax isn't available:

```js
this.setState({
  age: this.state.age + 1
})
```

```js
this.setState({
  lastName: this.state.lastName + "Katz";
})
```

### Avoiding Dependency Hell

Currently, Ember requires developers to manually enumerate a computed
property's dependent keys: the list of _other_ properties that _this_
computed property depends on. Whenever one of the listed properties changes,
the computed property's cache is cleared and any listeners are notified that
the computed property has changed.

In this example, `'firstName'` and `'lastName'` are the dependent keys of the
`fullName` computed property:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  })
})
```

While this system typically works well, it comes with its share of drawbacks.

First, it's annoying to have to type every property twice: once as a string
as a dependent key, and again as a property lookup inside the function. While
explicit APIs can often lead to clearer code, this verbosity often obfuscates
the intent of the property. People understand intuitively that they are
typing out dependent keys to help _Ember_, not other programmers.

Second, people tell us that this syntax is not very intuitive. You have to
read the Ember documentation at least once to understand what is happening in
this example.

It's also not clear what syntax goes inside the dependent key string. In this
simple example it's a property name, but nested dependencies become a
property path, like `'person.firstName'`. (Good luck writing a computed
property that depends on a property with a period in the name.)

You might form the mental model that a JavaScript expression goes inside the
string—until you encounter the `{firstName,lastName}` expansion syntax or the
magic `@each` syntax for array dependencies.

The truth is that dependent key strings are made up of an unintuitive,
unfamiliar microsyntax that you just have to memorize if you want to use
Ember well.

Lastly, it's easy for dependent keys to fall out of sync with the
implementation, leading to difficult-to-detect, difficult-to-troubleshoot bugs.

For example, imagine a new member on our team is assigned a bug where a
user's middle name is not appearing in their profile. Our intrepid developer
finds the problem, and updates `fullName` to include the middle name:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.middleName} ${this.lastName}`;
  })
})
```

They test their change and it seems to work. Unfortunately, they've just
introduced a subtle bug. If the user's `middleName` were to change,
`fullName` wouldn't update! Maybe this will get caught in a code review,
given how simple the computed property is, but noticing missing dependencies
is a challenge even for experienced Ember developers when the computed
property gets more complicated.

Tracked properties have a feature called _autotrack_, where dependencies are
automatically detected as they are used.

### Reducing Memory Consumption

By default, computed properties cache their values. This is great when a
computed property has to perform expensive work to produce its value, and
that value gets used over and over again.

But checking, populating, and invalidating this cache comes with its own
overhead. Modern JavaScript VMs can produce highly optimized code, and in
many cases the overhead of caching is greater than the cost of simply
recomputing the value.

Worse, cached computed property values cannot be freed by the garbage
collector until the entire object is freed. Many computed properties are
accessed only once, but because they cache by default, they take up valuable
space on the heap for no benefit.

For example, imagine this component that checks whether the `files` property is supported in
input elements:

```js
import Component from '@ember/component';
import { computed } from '@ember/object';

export default Component.extend({
  inputElement: computed(function() {
    return document.createElement('input');
  }),

  supportsFiles: computed('inputElement', function() {
    return 'files' in this.inputElement;
  }),

  didInsertElement() {
    if (this.supportsFiles) {
      // do something
    } else {
      // do something else
    }
  }
})
```

This component would create and retain an `HTMLInputElement` DOM node for the
lifetime of the component, even though all we really want to cache is the
Boolean value of whether the browser supports the `files` attribute.

Particularly on mobile devices, where RAM is limited and often slow, we
should be more conservative about our memory consumption. Tracked properties
switch from an opt-out caching model to opt-in, allowing developers to err on
the side of reduced memory usage, but easily enabling caching (a.k.a.
memoization) if a property shows up as a bottleneck during profiling.

### Native Classes

Tracked properties are designed from the ground up for native JavaScript (ES6) classes.

### "Just JavaScript"


## Detailed design

### Autotracking Stack
> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

Like any technical design, tracked properties must make tradeoffs to balance
performance, simplicity, and usability. Tracked properties make a different
set of tradeoffs than today's computed properties.

This means tracked properties come with edge cases or "gotchas" that don't
exist in computed properties. When evaluating the following drawbacks, please
consider the two features in their totality, including computed property
gotchas you have learned to work around.

In particular, please try to compensate for [familiarity ][familiarity] and
[loss aversion][loss-aversion] biases. Before you form a strong opinion,
[give it five minutes][5-minutes].

[familiarity]: https://en.wikipedia.org/wiki/Familiarity_heuristic
[loss-aversion]: https://en.wikipedia.org/wiki/Loss_aversion
[5-minutes]: https://signalvnoise.com/posts/3124-give-it-five-minutes

### Tracked Properties & Promises

Dependency autotracking requires that tracked getters access their dependencies synchronously.
Any access that happens asynchronously will not be detected as a dependency.

This is most commonly encountered when trying to return a `Promise` from a
tracked getter. Here's an example that would "work" but would never update if
`firstName` or `lastName` change:

```js
class Person {
  @tracked firstName;
  @tracked lastName;

  @tracked
  get fullNameAsync() {
    return this.reloadUser().then(() => {
      return `${this.firstName} ${this.lastName}`;
    });
  }

  async reloadUser() {
    const response = await fetch('https://example.com/user.json');
    const { firstName, lastName } = await response.json();
    this.firstName = firstName;
    this.lastName = lastName;
  }

  setFirstName(firstName) {
    // This should cause `fullNameAsync` to update, but doesn't, because
    // firstName was not detected as a dependency.
    this.firstName = firstName;
  }
}
```

One way you could address this is to ensure that any dependencies are consumed synchronously:

```js
@tracked
get fullNameAsync() {
  // Consume firstName and lastName so they are detected as dependencies.
  let { _firstName, _lastName } = this;

  return this.reloadUser().then(() => {
    // Fetch firstName and lastName again now that they may have been updated
    let { firstName, lastName } = this;
    return `${this.firstName} ${this.lastName}`;
  });
}
```

However, **modeling async behavior as tracked properties is an incoherent approach
and should be discouraged**. Tracked properties are intended to hold simple
state, or to derive state from data that is available synchronously.

But asynchrony is a fact of life in web applications, so how should we deal
with async data fetching?

**In keeping with Data Down, Actions Up, async behavior should be modeled as
methods that set tracked properties once the behavior is complete.**

Async behavior should be explicit, not a side-effect of property access.
Today's computed properties that rely on caching to only perform async
behavior when a dependency changes are effectively reintroducing observers
into the programming model via a side channel.

A better approach is to call a method to perform the async data fetching, then
set one or more tracked properties once the data has loaded. We can refactor the
above example back to a synchronous `fullName` tracked property:

```js
class Person {
  @tracked firstName;
  @tracked lastName;

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  async reloadUser() {
    const response = await fetch('https://example.com/user.json');
    const { firstName, lastName } = await response.json();
    this.firstName = firstName;
    this.lastName = lastName;
  }

  setFirstName(firstName) {
    // This should cause `fullNameAsync` to update, but doesn't, because
    // firstName was not detected as a dependency.
    this.firstName = firstName;
  }
}
```

Now, `reloadUser()` must be called explicitly, rather than being run
implicitly as a side-effect of consuming `fullName`.

### Accidental Untracked Properties

One of the design principles of tracked properties is that they are only
required for state that _changes over time_. Because tracked properties imply
some overhead over an untracked property (however small), we only want to pay
that cost for properties that actually change.

However, an obvious failure mode is that some property _does_ change over
time, but the user simply forgets to annotate that property as `@tracked`.
This will cause frustrating-to-diagnose bugs where the DOM doesn't update in
response to property changes.

Fortunately, we have several strategies for mitigating this frustration.

The first strategy involves the way most tracked properties will be consumed:
via a component template. In development mode, we can detect when an
untracked property is used in a template and install a setter that causes an
exception to be thrown if it is ever mutated. (This is similar to today's
"mandatory setter" that causes an exception to be thrown if a watched
property is set without going through `set()`.)

The second strategy relies on the fact that `EmberObject.create()` already
returns a JavaScript [`Proxy`][proxy] in development mode for objects that
rely on `unknownProperty`.

[proxy]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
