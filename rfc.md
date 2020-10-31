# PHP RFC: Fibers 
  * Version: 0.1
  * Date: 2020-09-04
  * Author: Aaron Piotrowski <trowski@php.net>
  * Status: Draft
  * First Published at: http://wiki.php.net/rfc/fibers

## Introduction 
Fibers create full-stack, interruptible functions that may be used to implement cooperative concurrency in PHP. These are also know as coroutines or green-threads. Unlike stack-less Generators, each Fiber contains a call stack, allowing them to be paused within deeply nested function calls. A function declaring an interruption point (i.e., calling `Fiber::await()`) need not change its return type, unlike a function using `yield` which must return a `Generator` instance. Fibers pause the entire execution stack, so the direct caller of the function does not need to change how it invokes the function.

Implementations of cooperative concurrency using promises suffer from the ["What color is your function?"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) problem.   Once one function returns a promise somewhere in your call stack, the entire call stack needs to return a promise.

Fibers aim to solve this problem, allowing functions to be interruptible without polluting the entire call stack.

## Proposal 
**This RFC proposes adding components to PHP implementing interruptible fibers and top-level (main-thread) await.**

A fiber is created with any callable and variadic argument list using `Fiber::run(callable $callback, mixed ...$args)`. The callable may use `Fiber::await()` to interrupt execution anywhere in the call stack (that is, the call to `Fiber::await()` may be in a deeply nested function or not even exist at all).

#### Fiber 

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

An awaitable is a placeholder for a future value (or exception) of an asynchronous operation, most typically I/O. Such objects are also commonly called promises or futures. An awaitable is resolved when the operation succeeds or fails. An awaitable contains a list of callbacks that are invoked when the awaitable is resolved. These callbacks are registered on the awaitable using `Awaitable::onResolve(callable $onResolve)`. The registered callbacks must accept two arguments, an exception or null representing the failure reason and a mixed result value if successful: `function (Throwable|null $exception, mixed $value)`. If the operation failed, the first argument must be a `Throwable` instance and the second argument must be null. If the operation was successful, the first argument must be null and the second may be any value (including null). This pattern is similar to error-first callbacks in node.js.

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

A `FiberScheduler` defines a special class is able to create new fibers using `Fiber::run()` and invokes awaitable callbacks. In general, a fiber scheduler would be an event loop that responds to events on sockets, timers, and deferred functions.

When an instance of `FiberScheduler` is provided to `Fiber::await()` for the first time, internally a fiber is created for that instance and invokes `FiberScheduler::run()`. The fiber created is paused when it invokes an on-resolve callback or resumed when the same instance is provided again to another call to `Fiber::await()`. It is expected that `FiberScheduler::run()` not return until all pending events have been processed and any awaiting fibers have been resumed. In practice this is not difficult, as the scheduler is paused when resuming a fiber and only re-entered upon awaiting another awaitable that has created more events in the scheduler.

`FiberScheduler::run()` throwing an exception results in an uncaught exception and exits the script.

A fiber //must// be resumed from the fiber created from the instance of `FiberScheduler` provided to `Fiber::await()`. Doing otherwise results in a fatal error. In practice this means that a resolving awaitable must defer invocation of registered callbacks to the `FiberScheduler` instance. Often it is desirable to ensure resolution of awaitables is asynchronous, making it easier to reason about program state before and after resolving an awaitable.

#### Unfinished Fibers 
Fibers that are not finished(do not complete execution) are destroyed similar to unfinished generators, executing any pending `finally` blocks. `Fiber::await()` may not be invoked in a force-closed fiber, just as `yield` cannot be used in a force-closed generator.

## Backward Incompatible Changes 
Declares `Awaitable`, `Fiber`, `FiberScheduler`, `FiberError`, and `FiberExit` in the root namespace. No other BC breaks.

## Proposed PHP Version(s) 
PHP 8.1

## Future Scope 
#### async/await keywords 
Using an internally defined `FiberScheduler` and `Awaitable` implementation, `Fiber::await()` could be replaced with the keyword `await` and new fibers could be created using the keyword `async`. The usage of `async` differs slightly from languages such as JS or Hack. `async` is not used to declare asynchronous functions, rather it is used at call time to modify the call to any function or method to return an awaitable and start a new fiber (green-thread).

``` php
$awaitable = async functionOrMethod();
// async modifies the call to return an awaitable, creating a new fiber, so execution continues immediately.
await $awaitable; // Await the function result at a later point.
```

#### defer keyword 
Fibers may be used to implement a `defer` keyword that executes a statement sometime after the current scope is exited within a new fiber. Such a keyword would also require an internal implementation of `FiberScheduler` and likely would be an addition after async/await keywords.
## Proposed Voting Choices 
Merge implementation into core, 2/3 required.

## Patches and Tests 
Implementation at [amphp/ext-fiber](https://github.com/amphp/ext-fiber).

## References 
  * [Boost C++ fibers](https://www.boost.org/doc/libs/1_67_0/libs/fiber/doc/html/index.html)
  * [Ruby Fibers](https://ruby-doc.org/core-2.5.0/Fiber.html)
  * [Lua Fibers](https://wingolog.org/archives/2018/05/16/lightweight-concurrency-in-lua)
  * [Project Loom for Java](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)