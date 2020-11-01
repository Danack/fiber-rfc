# PHP RFC: Fibers 
  * Version: 0.1
  * Date: 2020-09-04
  * Author: Aaron Piotrowski <trowski@php.net>
  * Status: Draft
  * First Published at: http://wiki.php.net/rfc/fibers

## Introduction 

For a long time since PHP was created, most people have written PHP as synchronous code. They have code that runs synchronously and in turn, only calls functions that run synchronously.

More recently, there have been multiple projects that have allowed people to write asynchronous PHP code. That is code that can be called asynchronously, and can call functions that either run synchronously or asynchronously. Examples of these projects are [AMPHP](https://amphp.org/), [ReactPHP](https://reactphp.org/), [Swoole](https://www.swoole.co.uk/).

The problem this RFC seeks to address is a difficult one to explain, but can be referred to as the ["What color is your function?"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) problem. That link contains a detailed explanation of the problem. 

A summary is that: 

* Asynchronous functions need to be called in a special way.
* It's easier to call synchronous functions.
* All PHP core functions are synchronous.
* If there is any asynchronous code in the call stack, then any call to a synchronous function 'breaks' the asynchronous code. 

For people who are familiar with using Promises and await to achieve writing asynchronous code, it can be expressed as; once one function returns a promise somewhere in your call stack, the entire call stack needs to return a promise, **TODO** as otherwise SOMETHING BAD WOULD HAPPEN. need to specify why it is _need_ **TODO** 

This RFC seeks to solve this problem by allowing functions to be interruptible without polluting the entire call stack. This would be achieved by:
 
 * adding support for [Fibers](https://en.wikipedia.org/wiki/Fiber_(computer_science)) to PHP.
 * adding two interfaces in PHP core to define Awaitable and FiberScheduler, so that userland code can interact with Fiber's 
 * adding some Throwable to represent errors, so userland code can catch those errors. **TODO**need details.


### Definition of terms

To allow better understanding of the RFC, this section defines what the names Fibers, Awaitable, and FiberScheduler mean for the RFC being proposed.

### Fibres

Fibers allow you to create full-stack, interruptible functions that can be used to implement cooperative concurrency in PHP. These are also know as coroutines or green-threads.

Unlike stack-less Generators, each Fiber contains a call stack, allowing them to be paused within deeply nested function calls. A function declaring an interruption point (i.e., calling `Fiber::await()`) need not change its return type, unlike a function using `yield` which must return a `Generator` instance.

Fibers pause the entire execution stack, so the direct caller of the function does not need to change how it invokes the function.

A fiber could be created with any callable and variadic argument list through `Fiber::run(callable $callback, mixed ...$args)`.

The callable may use `Fiber::await()` to interrupt execution anywhere in the call stack (that is, the call to `Fiber::await()` may be in a deeply nested function or not even exist at all).

### Awaitable 

An awaitable is a placeholder for a future value (or exception) of an asynchronous operation, most typically I/O. Such objects are also commonly called promises or futures. An awaitable is resolved when the operation succeeds or fails. An awaitable contains a list of callbacks that are invoked when the awaitable is resolved.

Adding an interface to represent Awaitables would allows users to write userland implementations of Awaitables to be used with Fibres.

### FiberScheduler

**TODO** This bit needs work

A `FiberScheduler` is a class that is able to: 

* create new fibers and ---invoke awaitable callbacks---. 
* other stuff. 

It  would be an event loop that responds to events on sockets, timers, and deferred functions.

**endTODO**

This RFC does not include an implementation for FiberScheduler. Instead it proposes only defining an interface, and any implementation would be done in userland. Please see Future scope.

## Proposal

#### Fiber 

A Fiber would be represented as class that would be defined in core PHP. It would have the following signature:

``` php
final class Fiber
{
    /**
     * Can only be called within {@see FiberScheduler::run()}.
     *
     * @param callable $callback Function to invoke when starting the Fiber.
     * @param mixed ...$args Function arguments.
     */
    public static function run(callable $callback, mixed ...$args): void { }

    /**
     * Private constructor to force use of {@see run()}.
     */
    private function __construct() { }

    /**
     * Suspend execution of the fiber until the given awaitable is resolved.
     *
     * @param Awaitable $awaitable
     * @param FiberScheduler $scheduler
     *
     * @return mixed Resolution value of the awaitable.
     *
     * @throws FiberError Thrown if within {@see FiberScheduler::run()}.
     * @throws Throwable Awaitable failure reason.
     */
    public static function await(Awaitable $awaitable, FiberScheduler $scheduler): mixed { }
}
```

`Fiber::await()` accepts an instance of `Awaitable` and an instance of `FiberScheduler`. These interfaces are implemented by user code.

#### Awaitable 

Awaitable would be an interface defined in core PHP. It would have the following signature:

``` php
interface Awaitable
{
    /**
     * Register a callback to be invoked when the awaitable is resolved.
     *
     * @param callable(?Throwable $exception, mixed $value):void $onResolve
     */
    public function onResolve(callable $onResolve): void;
}
```

 **TODO** what does 'registered' mean? I think this bit needs to be at least understand to people who don't grok awaitables, and currently it doesn't.

These callbacks are registered on the awaitable using `Awaitable::onResolve(callable $onResolve)`. The registered callbacks must accept two arguments, an exception or null representing the failure reason and a mixed result value if successful: `function (Throwable|null $exception, mixed $value)`. If the operation failed, the first argument must be a `Throwable` instance and the second argument must be null. If the operation was successful, the first argument must be null and the second may be any value (including null). This pattern is similar to error-first callbacks in node.js.

When an `Awaitable` is provided to `Fiber::await()`, the `Awaitable::onResolve()` method is invoked with a callback to be used to resume the fiber. The provided callback must be invoked within the fiber created from of `FiberScheduler::run()` (discussed below).

#### FiberScheduler 

``` php
interface FiberScheduler
{
    /**
     * Run the scheduler.
     */
    public function run(): void;
}
```

When an instance of `FiberScheduler` is provided to `Fiber::await()` for the first time, internally a fiber is created for that instance and invokes `FiberScheduler::run()`.

The fiber created is paused when it invokes an on-resolve callback or resumed when the same instance is provided again to another call to `Fiber::await()`. It is expected that `FiberScheduler::run()` not return until all pending events have been processed and any awaiting fibers have been resumed. In practice this is not difficult, as the scheduler is paused when resuming a fiber and only re-entered upon awaiting another awaitable that has created more events in the scheduler.

If `FiberScheduler::run()` throws an exception, that would result in the engine giving an 'uncaught exception' error and the script would be exited. **TODO a uncaught exception handler would be called? **

A fiber //must// be resumed from the fiber created from the instance of `FiberScheduler` provided to `Fiber::await()`. Doing otherwise results in a fatal error. In practice this means that a resolving awaitable must defer invocation of registered callbacks to the `FiberScheduler` instance. Often it is desirable to ensure resolution of awaitables is asynchronous, making it easier to reason about program state before and after resolving an awaitable.

#### Unfinished Fibers 
Fibers that are not finished (i.e. that have not completed execution) are destroyed similar to unfinished generators, executing any pending `finally` blocks. `Fiber::await()` may not be invoked in a force-closed fiber, just as `yield` cannot be used in a force-closed generator.

## Backward Incompatible Changes 
The classes/interfaces `Awaitable`, `Fiber`, `FiberScheduler`, `FiberError`, and `FiberExit` would be defined the root namespace, and so no longer usable by userland code. 

No other BC breaks known.

## Proposed PHP Version
PHP 8.1

## Future Scope 

There are two things that have been thought about, but have been excluded from this RFC, in order to make the scope of it be manageable.

#### 'async' and 'await' keywords 

The code `Fiber::await()` is a little bit awkward to read or write.

At some point, `Fiber::await()` could be replaced with the keyword `await` and new fibers could be created using the keyword `async`. That would allow programmers to indicate that the a particular function or method call should return an awaitable and start a new fiber (green-thread).

This is different from the languages JS and Hack. In those languages the usage of `async` is only used to declare asynchronous functions. It doesn't allow programmers to declare that functions should be called asynchronously.

This idea would require PHP including an internally defined `FiberScheduler` and `Awaitable` implementation. But that is not included in the scope of this RFC.

It might look something like this:

``` php
$awaitable = async functionOrMethod();
// async modifies the call to return an awaitable, creating a new fiber, so execution continues immediately.
await $awaitable; // Await the function result at a later point.
```

#### 'defer' keyword

Fibers may be used to implement a `defer` keyword that executes a statement sometime after the current scope is exited within a new fiber. Such a keyword would also require an internal implementation of `FiberScheduler` and likely would be an addition after async/await keywords.

To reiterate, async/await and defer keywords are not part of this RFC. They are future scope that would make it easier to write code that interacts with Fibres, but this RFC provide library interoperability without requiring those keywords.

## Proposed Voting Choices 
Accept this RFC, yes/no.

## Patches and Tests

Implementation at [amphp/ext-fiber](https://github.com/amphp/ext-fiber).

## References 
  * [Boost C++ fibers](https://www.boost.org/doc/libs/1_67_0/libs/fiber/doc/html/index.html)
  * [Ruby Fibers](https://ruby-doc.org/core-2.5.0/Fiber.html)
  * [Lua Fibers](https://wingolog.org/archives/2018/05/16/lightweight-concurrency-in-lua)
  * [Project Loom for Java](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)
