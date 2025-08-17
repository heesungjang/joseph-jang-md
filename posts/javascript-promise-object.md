---
id = "javascript-promise-object"
title = "Understanding bind Through Promise Implementation"
abstract = "Ever had JavaScript's `this` point to something unexpected? Let's build a Promise from scratch to see how `bind()` keeps `this` predictable in asynchronous code."
tags = ["javascript"]
date = 2024-02-21
status = "show"
---

Picture this: you're deep in JavaScript code, everything's working perfectly, and then `this` decides to point to something completely unexpected. Your carefully crafted object method suddenly thinks `this` is the global window object, and your code stops working.

If you've been there (and let's be honest, who hasn't?), you've probably wondered why JavaScript works this way. Today, we're going to dive into this chaos by building a Promise from scratch and discovering how `bind()` can be our lifeline when `this` goes rogue.

## The bind Lifeline

JavaScript's `bind()` method fixes the `this` problem by locking in what `this` should be, regardless of how the function gets called later. Think of it like putting a name tag on a function. No matter where that function ends up, it remembers who it belongs to.

The party analogy works here: instead of spending all night correcting people who call you by the wrong name, you just wear a name tag. `bind()` is that name tag for functions.

Let's build our own version to see what's really happening under the hood:

```jsx
Function.prototype.vbind = function (newThis) {
  // First, a sanity check - we can't bind something that isn't a function
  if (typeof this !== 'function') {
    throw new Error("Cannot bind - target is not callable");
  }
  
  let boundTargetFunction = this; // Remember the original function
  let boundArguments = Array.prototype.slice.call(arguments, 1); // Grab any extra arguments
  
  // Here's the magic - return a new function that "remembers" everything
  return function boundFunction() {
    let targetArguments = Array.prototype.slice.call(arguments);
    return boundTargetFunction.apply(newThis, boundArguments.concat(targetArguments));
  };
};
```

The interesting part is that we're returning a new function that "remembers" the original function and the `this` value we want. When this bound function gets called, it uses `apply()` to run the original function with our chosen context.


>💡 `boundFunction` is a closure. It remembers the variables from when it was created, even after `vbind` finishes. This is why it can still access `boundTargetFunction`, `newThis`, and `boundArguments` later.


## Building Our Own Promise

Building a Promise from scratch reveals why `bind` matters so much in asynchronous code. Promises need to call methods on their instance from within callback functions, which is exactly when `this` starts pointing to unexpected places.

```jsx
class VPromise {
  constructor(executionFunction) {
    this.promiseChain = []; // All the .then() handlers waiting in line
    this.handleError = () => {}; // Our error handler (starts as a no-op)

    // Here's the crucial part - bind these methods to THIS instance
    this.onResolve = this.onResolve.vbind(this);
    this.onReject = this.onReject.vbind(this);

    // Execute immediately, passing our bound methods as resolve/reject
    executionFunction(this.onResolve, this.onReject);
  }

  then(handler) {
    this.promiseChain.push(handler); // Add to the queue
    return this; // Return ourselves for chaining
  }

  error(handler) {
    this.handleError = handler; // Set our error handler
    return this; // Chainable goodness
  }

  onResolve(value) {
    let storedValue = value;
    try {
      // Run through all the .then() handlers in sequence
      this.promiseChain.forEach((nextFunction) => {
        storedValue = nextFunction(storedValue);
      });
    } catch (error) {
      this.onReject(error); // If anything breaks, handle the error
    }
  }

  onReject(error) {
    this.handleError(error); // Pass the error to our handler
  }
}
```

## Using VPromise

```jsx
function delayThenValue() {
  return new VPromise((resolve, reject) => {
    setTimeout(() => {
      if (true) {
        resolve('Asynchronous result');
      } else {
        reject('Error occurred');
      }
    }, 2000);
  });
}

delayThenValue()
  .then((value) => {
    console.log(value); // "Asynchronous result" after 2 seconds
  })
  .error((error) => {
    console.log(error); // Won't happen in this example
  });
```

The `vbind` calls in the constructor are what make this work. Without them, the `onResolve` and `onReject` methods would lose their connection to the Promise instance when called asynchronously.

## Why bind Saves the Day

Here's what would happen without those `vbind` calls: when `setTimeout` eventually calls our resolve function, it wouldn't be called as a method of our `VPromise` instance. Instead, it would be called as a standalone function, meaning `this` would probably point to the global object (or be `undefined` in strict mode).

Our carefully crafted `onResolve` method would suddenly have no idea what `this.promiseChain` or `this.handleError` are supposed to be. The whole thing would break with a cascade of undefined errors.

With `bind`, the methods stay connected to their original instance no matter how they're called.

## The Bigger Picture

JavaScript gives you flexibility with `this`, but that flexibility means you need to be explicit about what you want. `bind` is how you tell JavaScript to stop being clever and just stick to what you specified.

It's one of those methods that seems pointless until you actually need it. Then you realize how much cleaner your asynchronous code becomes when you don't have to worry about `this` wandering off.

---

**References:**

- [**JavaScript**: *Learn JavaScript Promises by Building a Promise from Scratch*](https://levelup.gitconnected.com/understand-javascript-promises-by-building-a-promise-from-scratch-84c0fd855720)
- [**JavaScript**: *Implementing promises from scratch (TDD way)*](https://www.mauriciopoppe.com/notes/computer-science/computation/promises/)
- [**JavaScript**: *Implement your own — call(), apply() and bind() method in JavaScript*](https://blog.usejournal.com/implement-your-own-call-apply-and-bind-method-in-javascript-42cc85dba1b)