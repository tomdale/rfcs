- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Tracked Properties

## Summary

Tracked properties introduce a simpler, more modern, and more ergonomic
system for tracking state changes. By taking advantage of new JavaScript
features, tracked properties allow Ember to reduce its API surface area while
producing code that is much more intuitive to understand for new learners who
already know JavaScript.

A small example:

```js
export default class Person {
  @tracked firstName = 'Chad';
  @tracked lastName = 'Hietala';

  @tracked get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

let person = new Person();
console.log(person.fullName); // "Chad Hietala"

person.firstName = 'Chadwick';
console.log(person.fullName); // "Chadwick Hietala"
```

## Terminology

#### Computed Property (CP)

A property on an Ember object whose value is produced by evaluating a
function, and updated whenever one of its dependencies changes.

```js
const Person = EmberObject.extend({
  loudFirstName: computed('firstName', function() {
    return this.firstName.toUpperCase();
  })
});

const person = Person.create();
person.set('firstName', 'Godfrey');
console.log(person.loudFirstName); // 'GODFREY'
```

In this document, "computed property" _always_ refers to the feature of the
Ember object model. The native JavaScript feature of using a function to dynamically determine the value of a property is referred to as a _getter_.

#### Getter

The JavaScript feature that allows a function to be evaluated to determine
the value of a property. One way of writing a getter is to use the `get`
syntax:

```js
class Person {
  get loudFirstName() {
    return this.firstName.toUpperCase();
  }
}

const person = new Person();
person.firstName = 'Godfrey';
console.log(person.loudFirstName); // 'GODFREY'
```

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

Compare this to other component APIs which become more verbose than necessary
when JavaScript syntax isn't available:

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

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
