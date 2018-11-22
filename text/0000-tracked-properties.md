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
* A **tracked getter** is a JavaScript getter that has been wrapped using the
  tracked decorator
* A **tracked simple property** is a regular, non-getter property that has been
  wrapped using the tracked decorator.
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
JavaScript. Tracked properties look like normal properties because they
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
automatically detected as they are used. This means that as long as all
dependencies are marked as tracked, they will automatically be detected:

```js
import { tracked } from '@ember/object';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

This also allows us to opt out of tracking entirely, like if we know for
instance that a given property is constant and will never change. In general,
the idea is that _mutable_ properties should be marked as tracked, and
_immutable_ properties should not.

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

Tracked properties are designed from the ground up for native JavaScript (ES6)
classes. Adopting them will make it easier to convert away from the aging Ember
object-model, and

## Detailed design

### Autotracking Stack

Autotracking works by taking advantage of the fact that all mutable properties
will be decorated by `tracked`. This decorator adds a getter and a setter that
wraps the property, be it a tracked getter or a tracked simple property. Every
time we activate a tracked getter, it creates a global stack before it begins
executing the getter's code, and as it executes, any other tracked wrappers
that run add themselves to the stack.

This example demonstrates how this method works, but is simplified to only work
with tracked getters, and to only track keys. It uses the stage 2 decorator
syntax.

```js
let PROPERTY_STACK;

function tracked({ key, descriptor }) {
  let originalGet = descriptor.get;

  descriptor.get = function() {
    // store the stack from the previous getter, if it exists
    let previousStack = PROPERTY_STACK;

    // create a new stack
    PROPERTY_STACK = [];

    // call the original get function to get the value
    let value = originalGet.call(this);

    // At this point, PROPERTY_STACK will contain all of the keys of any values
    // that were marked as tracked and accessed when `originalGet` was called.
    // We can use this knowledge to setup our change tracking, then push our
    // own key and the current stack into the previous stack if it exists.
    if (previousStack) {
      previousStack.push(key, ...PROPERTY_STACK);
    }

    PROPERTY_STACK = previousStack;

    return value;
  }
}
```

Of course, tracking keys would only work if we knew the full paths, or were only
concerned with only changes to the current object. The real implementation uses
[Glimmer VM's internal validators](https://github.com/glimmerjs/glimmer-vm/blob/master/guides/05-validators.md)
to track changes to properties.

As properties are accessed, their validation  _tags_ are pushed on the stack,
and when the property access is complete these tags are combined into a single
tag for that property. Setters are added to tracked getters and tracked simple
properties as appropriate which manually dirty these tags whenever they are
activated, and subsequently invalidate all of the tags for any getters which
ever accessed them.

### Untracked Properties

Revisiting the `middleName` bug from earlier, you may notice that it's still
possible for a new developer to introduce a bug with tracked properties. What if
they don't know that all mutable properties must be tracked? This would fail,
just like our previous example:

```js
import { tracked } from '@ember/object';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';
  middleName = 'Tomster';

  @tracked
  get fullName() {
    return `${this.firstName} ${this.middleName} ${this.lastName}`;
  }
}
```

The last piece of the puzzle here will be a development time only assertion.
When you access _untracked_ properties from an a tracked getter, we will install
something similar to the mandatory setter function. This will throw an error if
the user ever tries to set the property, effectively making it immutable.

To accomplish this, we can wrap any object which has tracked properties with a
native [`Proxy`][proxy], which will subsequently wrap any objects or array properties
which are accessed with another proxy, and so on. These proxies will be able
to check if any untracked properties are being accessed from an tracked context,
and setup immutable assertions if so.

### Tracked Interop with Computed Properties and `Ember.set`

Tracked properties work well in isolation. In systems built from the ground up
using `@tracked`, all mutable values will be marked as tracked, so autotracking
will be able to "watch" them all as they are accessed and changed.

However, tracked properties will have to interoperate with the current system
for tracking changes and mutable state in Ember for some time as the community
transitions forward. The origin of all mutations in the old system is through
`Ember.set` (or more specifically, `notifyPropertyChange`), but there are many
ways to listen to these changes. Tracked properties are concerned with two
specific cases:

1. **Computed Properties.** When `notifyPropertyChange` is called on one of a
computeds dependencies, it will invalidate, and invalidate any computeds which
depend on it. In this way, one call to `set` can invalidate many properties in
the system. Tracked properties must also properly invalidate if they consume
any of these properties.
2. **Ember.set() directly**. If a tracked property accesses a simple, untracked
property which is _later_ set with `Ember.set()`, it must have a way to know
that this property has changed. This can occur in services, helper classes which
have not yet converted to `@tracked`, and in many other edge cases.

In the case of computed properties, we have all the information we need up front
to allow tracked getters to watch for changes in the computed. They will notify
whenever they are updated, so the same technique which is used for the autotrack
stack can be used to watch them as well. Crucially, computed properties will
always have a tag due to the fact that they are defined explicitly in class
definitions, which means we have something to watch the very first time a
computed property is accessed.

In the edge case of non-computed, non-tracked property access, this is not the
case. The property may be completely undecorated, meaning it may never have had
a tag in the first place. With standard Javascript property access, we won't
have an opportunity to _add_ a tag either. Autotrack will not work in these
cases:

```js
const Config = Service.extend({
  polling: {
    shouldPoll: false,
    pollInterval: -1,
  },

  init() {
    this._super(...arguments);

    fetch('config/api/url')
      .then(r => r.json())
      .then(polling => set(this, 'polling', polling));
  }
})

class SomeComponent extends Component {
  @service config;

  @tracked
  get pollInterval() {
    // When we access shouldPoll and pollInterval, they don't have tags or
    // wrapped getters. We have no idea that they could update in the future.
    let { shouldPoll, pollInterval } = this.config.polling;

    return shouldPoll ? pollInterval : -1;
  }
}
```

To support this case, `Ember.get` will manually opt in to adding the tags for
any properties which are accessed during a tracked getter. This allows tracked
getters to safely access services and objects which are still using the legacy
change tracking system:

```js
const Config = Service.extend({
  polling: {
    shouldPoll: false,
    pollInterval: -1,
  },

  init() {
    this._super(...arguments);

    fetch('config/api/url')
      .then(r => r.json())
      .then(polling => set(this, 'polling', polling));
  }
})

class SomeComponent extends Component {
  @service config;

  @tracked
  get pollInterval() {
    // When we access shouldPoll and pollInterval, they don't have tags or
    // wrapped getters. We have no idea that they could update in the future.
    let shouldPoll = this.get('config.polling.shouldPoll');
    let pollInterval = this.get('config.polling.pollInterval');

    return shouldPoll ? pollInterval : -1;
  }
}
```

This solution will _only_ be necessary in cases where users do not know ahead of
time whether a property is A. immutable, B. tracked, or C. a computed property.
As time goes on and more and more services, addons, and apps are converted to
`@tracked`, the need for `Ember.get` will become smaller and smaller, until it
can be removed from the framework entirely (along with `Ember.set`).

### Computed Property Interop with Tracked

The previous section covers the cases where classic Ember properties are
accessed from tracked properties, but we also have to define the behavior for
when the opposite occurs: tracked properties are accessed by from a classic
context.

There are two major ways this can occur:

1. Computed Properties
2. Observers

Observers are strictly within the old paradigm, so tracked properties will _not_
attempt to interoperate with them. Using `Ember.set` on a tracked property will
still notify that the property has updated, and any observers for that property
will fire accordingly.

Computed properties, on the other hand, are similar enough to tracked properties
that interoperability would be useful. As such, computed properties will _also_
automatically add tracked properties to their dependencies as they are accessed.
This will use the same mechanism as tracked properties do with the autotrack
stack, and will allow users to progressively convert from computed properties to
tracked properties over time by whittling away at the explicit dependencies
within a CP.

### Interop with EmberObject

While `@tracked` is designed from the ground up for native Javascript classes,
it will have to interoperate with legacy code for some time. In the previous
sections, we noted how `get` would have to be used with any legacy code which
has not been updated to tracked properties. As such, to make the transition as
easy as possible, `tracked` will also be made to work with the classic
object-model:

```js
const Config = Service.extend({
  polling: tracked({
    value: {
      shouldPoll: false,
      pollInterval: -1,
    }
  }),

  init() {
    this._super(...arguments);

    fetch('config/api/url')
      .then(r => r.json())
      .then(polling => set(this, 'polling', polling));
  }
})
```

Tracked getters and setters will also be allowed, with the `get` and `set` keys
on the passed in configuration. This will allow existing libraries to transition
incrementally, and add tracked support minimally where necessary.

This form will _not_ be allowed on native classes, and will hard error if it is
attempted. Additionally, default values will be defined on the _prototype_ to
maintain consistency with the classic object-model.

## How we teach this

There are two different aspects of tracked properties which need to be
considered for the learning story:

1. **General usage.** Which properties should I mark as tracked? How do I
consume them? How do I trigger changes?
2. **Interop with legacy systems.** How do I safely consume tracked properties
from legacy classes and computeds? How do I safely consume legacy APIs from
tracked properties?

### General Usage

The mental model with tracked properties is that anything _mutable_ should be
tracked. If a value will ever change, it should have the `@tracked` decorator
attached to it.

After that, usage should be "Just Javascript". You can safely access values
using any syntax you like, including desctructuring, and you can update values
using standard assignments.

```js
// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];

// Simple assignment
this.firstName = "Yehuda";
// Addition assignment (+=)
this.lastName += "Katz";
// Increment operator
this.age++;
```

#### Triggering Updates on Complex Objects

There may be cases where users want to update values in complex, untracked
objects such as arrays or POJOs. `@tracked` will only be usable with class
syntax at first, and while it may make sense to formalize these objects into
tracked classes in some cases, this will not always be the case.

To do this, users can re-set a tracked value directly after its inner values
have been updated.

```js
class SomeComponent extends Component {
  @tracked items = [];

  @action
  pushItem(item) {
    let { items } = this;

    items.push(item);

    this.items = items;
  }
}
```

### Interop with Legacy Systems

There are two cases, as discussed in the Detailed Design section, that we need
to consider when teaching interoperability:

1. Tracked getters accessing non-tracked properies and computeds
2. Computed getters accessing tracked properties

In the first case, the general rule of thumb is to use `Ember.get` if you want
to be 100% safe. In cases where you are certain that the values you are
accessing are tracked, computeds, or immutable, you can safely standard access
syntax.

In the second case, no additional changes need to be made when using tracked
properties. They can be accessed as normal, and will be automatically added to
the computed's dependencies. There is no need to use `Ember.get`, and you can
use standard assignments when updating them.

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
  let { firstName, lastName } = this;

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

## Alternatives

* We could keep the current computed property based system, and refactor it
  internally to use references only and not rely on chains or the old property
  notification system. This would be difficult, since CPs are very intertwined
  with property events as are their dependencies. It would also mean we wouldn't
  get the DX benefits of cleaner syntax, and the performance benefits of opt-in
  change tracking.

* We could continue to rely on `Ember.set`. There is precedent for this in other
  frameworks, such as React's `setState`. This would open up a lot of design
  possibilities, but likely wouldn't be able to restrict the requirement for
  `@tracked` being applied to all mutable properties for the same reason
  `Ember.get` must be used in interop.

* We could allow `@tracked` to receive explicit dependencies instead of forcing
  `Ember.get` usage for interop. This would be very complex, if even possible,
  and is ultimately not functionality `@tracked` should have in the long run, so
  it would not make sense to add it now.

* We could attempt to wait until Ember's support matrix allows us to use native
  [Proxies][proxy] in production. This would open up huge possibilities for
  automatic change tracking, but would also likely have a major performance
  impact. It's hard to say whether or not proxies will be able to match the
  performance of manually annotated tracked properties, and this could always be
  revisited in the future.

[proxy]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
