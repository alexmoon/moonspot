+++
title="Async await demystified"
date=2022-05-21
draft=false

[taxonomies]
tags=["test", "code"]

[extra]
disable_comments = true
+++

## In the beginning...

When Javascript was first created it was intended for simple tasks within the
context of a single webpage. Most scripts were one or two lines embedded in the
attributes of an HTML element. A script was triggered in response to an event
such as clicking on a link and that was it.

This worked great for scripts that had everything they needed to complete
immediately. But what if you wanted to do something on a delay, such as updating
a clock on the website? Javascript was single-threaded and ran in the UI thread
of the browser so if you tried to `sleep` or spin until the right time you would
end up locking up the whole browser.

Javascript's solution to this was callbacks:

```javascript
setTimeout(5000, function () {
  alert("It's 5 seconds later!");
});
```

Because Javascript allowed you to easily define functions in-line this solution
was simple and worked well as long as the programs people were writing in Javascript were realtively simple.

## Callback hell

As Javascript gained ever more capability the limitations of callbacks began
to appear. The main issue with callbacks is that the logical sequence of your
program becomes very convoluted. For example:

```javascript
console.log("This comes first");
setTimeout(1000, function () {
  console.log("This comes fourth");
});
setTimeout(100, function () {
  console.log("This comes third");
  setTimeout(1000, function () {
    console.log("This comes fifth");
  });
});
console.log("This comes second");
```

You can also see that as you start nesting callbacks, the code becomes more and
more indented, experiencing a sort of rightward drift. There had to be a better
way.

## Promises, promises

The `Promise` API was developed by library authors to try to tame the callback
hell. It introduced an object called a `Promise` which represents a computation
that will complete at some point in the future.

```typescript
// This function converts the setTimeout() API into a Promise API
function timeoutPromise(delay: number, value: string): Promise<string> {
  return new Promise((resolve) => setTimeout(delay, () => resolve(value)));
}
```

As you can see, we now have a function that returns a `Promise` rather than
taking a callback as an argument. Inside that function we pass a callback to
the existing `setTimeout` API that "resolves" the promise.

What can we do with that `Promise` object? Well, `Promise` has a method
`then()` which allows us to give it a callback to be called when the `Promise`
resolves.

```typescript
let promise = timeoutPromise(100, "a");
promise.then((value) => {
  console.log("The timeout finished! Here's your value:", value);
});
```

But now we're back where we started, just passing callbacks to promises instead
of the original functions! What's the point?

Well `Promise` has another trick up its sleeve. The `then()` method itself
returns a `Promise` so you can chain its calls:

```typescript
let promise = timeoutPromise(100, "a");
promise
    .then((value) => {
        console.log("The timeout finished! Here's your value:", value);
        return "b";
    })
    .then((value) => {
        console.log("This time the value is 'b':", value);
    });
```

What if we want to do something asynchronous inside `then()`? Won't we still
need to nest promises?

Well, here's the final trick. If you return a `Promise` from the `then()`
callback, then the chained promise will wait for the promise you returned. That
lets you turn that chain of nested callbacks into a linear chain of promises
instead.

```typescript
// This function converts the setTimeout() API into a Promise API
function timeoutPromise(delay: number, value: string): Promise<string> {
  return new Promise((resolve) => setTimeout(delay, () => resolve(value)));
}

console.log("This comes first");
timeoutPromise(100, "a").then((value) => {
  console.log("This comes second, value is 'a'", value);
  return timeoutPromise(100, "b");
}).then((value) => {
  console.log("This comes third, value is 'b'", value);
  return timeoutPromise(100);
}).then((value) => {
  console.log("This comes forth, value is 'c'", value);
});
```

As you can see, this API solves the rightward drift problem. It makes the flow
of control more clear in that things now generally happen from top to bottom in
the code, instead of having to look for ever more nested callbacks. However,
there's still an awful lot of callbacks being created and asynchronous code
using this API is still much more convoluted than equivalent synchronous code
would be. Maybe there's a better way...

## A little sugar please

Finally we come to `async await`. This feature was added to the Javascript
language after the `Promise` API was standarized and is really just syntax sugar
for that API. Everything you do with `async await` can be done with the
underlying `Promise` API, `async await` just makes the code easier to read,
write and understand.

First let's look at `async`:

```typescript
async function foo(): Promise<string> {
  return "bar";
}
```

`async` does exactly two things. First, it wraps the return type of your
function in a `Promise`. Second it allows you to use the `await` keyword. That's
it. So the above function is exactly the same as:

```typescript
function foo(): Promise<string> {
  return new Promise((resolve) => resolve("bar"));
}
```

The real magic is in the `await` keyword. `await` is used on a `Promise` object
(for example, the one returned by an `async` function, but any `Promise` will
do). When you use the `await` keyword it takes everything in the function that
comes after it and essentially turns it into a callback that is passed to the
`then()` method of the `Promise`. In other words it lets you write code
sprinkled with `await`s that looks just like normal synchronous code, but
automatically transforms it into code that uses the `Promise` API to wait for
completion at each step. So our promise example above would become:

```typescript
// This function converts the setTimeout() API into a Promise API
function timeoutPromise(delay: number, value: string): Promise<string> {
  return new Promise((resolve) => setTimeout(delay, () => resolve(value)));
}

async function main(): Promise<void> {
  console.log("This comes first");
  let value = await timeoutPromise(100, "a");
  console.log("This comes second, value is 'a'", value);
  let value = await timeoutPromise(100, "b");
  console.log("This comes third, value is 'b'", value);
  let value = await timeoutPromise(100);
  console.log("This comes forth, value is 'c'", value);
}

main();
```

You'll note we had to wrap things in an `async` function in order to use the
`await` keyword. But the code is now much shorter and clearer than it was
before.

And that's all there is to it!
