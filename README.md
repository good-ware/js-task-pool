# @goodware/task-queue: A lightweight task queue for limiting resource usage

## Links

- [npm](https://www.npmjs.com/package/@goodware/task-queue)
- [git](https://github.com/good-ware/js-task-queue)
- [API](https://good-ware.github.io/js-task-queue/)

## Requirements

ECMAScript 2017

## Installation

`npm i --save @goodware/task-queue`

## Overview

This lightweight, battle-tested, single-dependency (Joi) queue class limits the number of tasks (synchronous or asynchronous) that can execute simultaneously, in order to manage the usage of resources such as memory and database connections. Although many other packages address this use case, this one is apparently unique in that it that allows tasks to be queued post-instantiation without using generators.

## Usage

A task-queue object is instantiated by providing a configuration object to the constructor. The configuration object currently has only one required property:

|Name|Description|
|----|-----------|
|`size`|The maximum number of tasks that can execute smultaneously.|

Functions are scheduled for execution via the ascyhronous method `push()`. This method accepts a function and returns a Promise that settles when the provided function is called.

A common misunderstanding is to think the Promise returned by `push()` settles when the function has finished executing. To wait for the function to finish, use the `.promise` property of the object returned by `push()`.

In summary:

- Create a queue of size 10 via `new (require('@goodware/task-queue'))({ size: 10 })`
- `await queue.push(() => {...})` waits for the provided function to be called
- `await queue.push(() => {...}).promise` waits for the provided function to finish

### Example

This example runs at most two tasks at a time. It outputs: 2, 1, 4, 3.

```js
const queue = new (require('@goodware/task-queue'))({ size: 2 });

// Task #1 : push() returns immediately because the queue is empty. 'await'
// doesn't wait for the task to complete.
await queue.push(
  () =>
    new Promise((resolve) =>
      setTimeout(() => {
        console.log(`Task 1 ${Date.now()}`);
        resolve();
      }, 400)
    )
);

// Task #2 : push() returns immediately because the queue has an open slot
await queue.push(
  () =>
    new Promise((resolve) =>
      setTimeout(() => {
        console.log(`Task 2 ${Date.now()}`);
        resolve();
      }, 300)
    )
);

// The queue is full. Task #2 will finish in about 300 ms.

// Task #3 : push() waits until task #2 finishes
await queue.push(
  () =>
    new Promise((resolve) =>
      setTimeout(() => {
        console.log(`Task 3 ${Date.now()}`);
        resolve();
      }, 200)
    )
);

// The queue is full again. 300 ms have already passed. Task #1 will
// terminate in about 100 ms, leaving task #3 in the queue.

// Task #4 : push() waits until task #1 finishes
const ret = await queue.push(
  () =>
    new Promise((resolve) =>
      setTimeout(() => {
        console.log(`Task 4 ${Date.now()}`);
        resolve();
      }, 100)
    )
);

// Wait for task #4 to finish
await ret.promise;
```

## Advanced Usage

Sometimes it's desirable to limit the number of simultaneous calls to `push()` because, like every other invocation of a function that returns a new object, waiting for a slot in the queue consumes memory.

Consider the following application constraints:

1. Up to 10 'worker' functions can execute simultaneously
2. When all 10 workers are running, up to 50 tasks can wait for one of the workers to finish

This use case can be implemented with two queue instances: `workers` and `waiters`. The `workers` queue executes tasks as demonstrated in the above example. The `waiters` queue counts the number of tasks that are waiting when `workers` is full.

```js
const waiters = new (require('@goodware/task-queue'))({ size: 50 });
const workers = new (require('@goodware/task-queue'))({ size: 10 });

// This is the most basic implementation of backpressure 
if (waiters.full()) throw Error('Not ready');

// 'await's are intentionally omitted. They are unnecessary for this example.
waiters.push(() => {
  // Perform some set-up tasks here, if needed
  // The waiters queue is used again to avoid race conditions that could allow more than 50 waiters
  waiters.push(() => workers.push(() => {
    // Do the work
  }));
});
```
