---
id = "first-days-functional-programming"
title = "First few days into FP"
abstract = "Functional programming isn't just different syntax. It's rewiring how you think about solving problems, and that mental shift is harder than any mathematical theory."
tags = ["functional-programming"]
date = 2024-08-19
status = "show"
---

I barely know stuff at the moment but a few things I've learned are starting to click.

## Functional Programming is Hard

Not just because of all the mathematical theories behind functional programming, but because you have to rewire your brain circuits on how to do things in functional programming languages compared to programming languages I'm used to like TypeScript, C#, Python, and others.

## It Goes Back Over Years

Most programming languages were designed around how computers actually work - memory, processors and etc. Functional languages were designed around mathematical abstractions that existed.

## Purity

In purely functional programming languages based on Lambda Calculus, we have rigorous requirements:

- Take one parameter
- Perform a computation
- Return one result

That's it. No side effects, no modifying global state, no printing to console inside your function. If your function does anything other than transform input to output, it's not pure. This sounds too extreme... because what meaningful program can you build without all the side effect. No write database or any files and etc..

Apparently you can, but unlike `console.log("hello, world)"`, side effects are more advanced concept in functional programming. Haven't got to that part yet.

## In Functional Programming, Everything is Immutable

An impure functional language generally doesn't allow rewriting memory, but still allows I/O, random numbers, and messaging other threads. But the core data structures are immutable. - like [Gleam](https://gleam.run/frequently-asked-questions/).

Why immutability? It actually comes with huge optimizations. It makes it easier for things like memoization and compiler-level optimizations.

## No Loops

Immutability means no loops. for-loops and while-loops don't work, so all looping is done with recursion.
