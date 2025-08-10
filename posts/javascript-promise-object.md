---
id = "javascript-promise-object"
title = "Understanding bind Through Promise Implementation"
abstract = "When thinking about the this keyword in JavaScript, the first thing that comes to mind is that its value changes based on how the function is called. Today, we'll look into how to implement a Promise object from scratch and explore the usage of bind(), a method we don't use very often in everyday development."
tags = ["javascript"]
date = 2024-02-21
status = "show"
---


When thinking about the this keyword in JavaScript, the first thing that comes to mind is that its value changes based on how the function is called. Today, we'll look into how to implement a Promise object from scratch and explore the usage of bind(), a method we don't use very often in everyday development.

## What is bind?

First, let's understand the bind method. The bind method allows you to set the this value for a function. In JavaScript, the value of this is dynamically determined based on how a function is called. This can lead to unexpected behavior if this isn't what you expected. The bind method helps solve this issue.

```jsx
Function.prototype.vbind = function (newThis) {
  // If the target is not a function, throw an error
  if (typeof this !== 'function') {
    throw new Error("Cannot bind - target is not callable");
  }
  let boundTargetFunction = this;
  let boundArguments = Array.prototype.slice.call(arguments, 1);
  // Return a new function that calls the original function with the new this value
  return function boundFunction() {
    let targetArguments = Array.prototype.slice.call(arguments);
    return boundTargetFunction.apply(newThis, boundArguments.concat(targetArguments));
  };
};

```

This code is a simple implementation of the bind method. By adding the vbind method to Function.prototype, we can use it with any function. This method takes a new this value as its first argument, followed by any arguments that should be passed to the bound function. Internally, it uses the apply method to call the original function with the new this value and arguments.


>💡 boundFunction is a great example of a closure. A closure is a function that "remembers" the environment in which it was created, allowing it to access variables from its enclosing scope. 

Here, boundFunction remembers boundTargetFunction, newThis, and boundArguments from the context in which it was created, allowing it to execute boundTargetFunction with the specified newThis context later.


## Combining Promise and bind

With an understanding of the bind method, let's create a simple Promise-like object called VPromise. A Promise represents the eventual completion (or failure) of an asynchronous operation.

```jsx
class VPromise {
  constructor(executionFunction) {
    this.promiseChain = [];
    this.handleError = () => {};

    this.onResolve = this.onResolve.vbind(this);
    this.onReject = this.onReject.vbind(this);

    executionFunction(this.onResolve, this.onReject);
  }

  then(handler) {
    this.promiseChain.push(handler);
    return this; // For chaining
  }

  error(handler) {
    this.handleError = handler;
    return this; // For error handling chain
  }

  onResolve(value) {
    let storedValue = value;
    try {
      this.promiseChain.forEach((nextFunction) => {
        storedValue = nextFunction(storedValue);
      });
    } catch (error) {
      this.onReject(error);
    }
  }

  onReject(error) {
    this.handleError(error);
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
    console.log(value); // Asynchronous result
  })
  .error((error) => {
    console.log(error); // Error occurred
  });

```

Similar to JavaScript's native Promise object, the VPromise class performs asynchronous operations and calls the appropriate handler (onResolve or onReject) based on the result. The key here is using vbind to permanently bind the this context of the onResolve and onReject methods to the current instance.

Without using bind, if these methods were called asynchronously, this could be set to an unexpected value, preventing access to instance properties like promiseChain or handleError. This reinforces the understanding that:

“this in JavaScript is dynamically determined based on how the function is called.”

## The Role of bind

In VPromise, the onResolve method isn't called as a method of the instance, but rather as a standalone function. This happens when the executionFunction (which performs the asynchronous operation) calls resolve. If bind is not used, this within onResolve would refer to the global object (or be undefined in strict mode), causing issues with accessing this.promiseChain.

Using vbind ensures that this is correctly bound to the instance, making the asynchronous behavior predictable and ensuring that the onResolve and onReject methods function as intended, even when called asynchronously.

Understanding this and the bind method is crucial in JavaScript, especially when dealing with asynchronous code.

---

**References:**

- [**JavaScript**: *Learn JavaScript Promises by Building a Promise from Scratch*](https://levelup.gitconnected.com/understand-javascript-promises-by-building-a-promise-from-scratch-84c0fd855720)
- [**JavaScript**: *Implementing promises from scratch (TDD way)*](https://www.mauriciopoppe.com/notes/computer-science/computation/promises/)
- [**JavaScript**: *Implement your own — call(), apply() and bind() method in JavaScript*](https://blog.usejournal.com/implement-your-own-call-apply-and-bind-method-in-javascript-42cc85dba1b)