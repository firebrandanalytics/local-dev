# A Conceptual Guide to FireFoundry's Async Streams Library

## Introduction

Modern AI workflows are rarely simple, single-threaded operations. They are complex pipelines of concurrent, asynchronous tasks: fetching data, calling models, processing results, and streaming progress. To manage this complexity, FireFoundry provides a powerful and opinionated async streams library.

This guide is not an exhaustive API reference. Instead, it explains the core concepts and design philosophy behind the library, helping you understand *why* it's built this way and *how* to choose the right tools for your orchestration needs.

At its heart, the library is designed to solve one problem: **how to treat complex asynchronous data flows as simple, reusable, and composable building blocks.**

## 1. The "Obj" Pattern: Structured, Composable Iterators

The foundation of the library is the "Obj" pattern (e.g., `PullObj`, `PushObj`). Instead of using raw `AsyncGenerator` functions, we wrap them in classes.

**Why use classes?**

1.  **Reusability and Composition:** It turns a transient generator into a solid, reusable component. You can pass instances of these objects around, chain them together, and build complex pipelines from simple parts.
2.  **Consistent Interface:** Every object implements the standard `AsyncGenerator` interface, with `next()`, `return()`, and `throw()` methods. This guarantees predictable behavior, especially for resource management and graceful shutdown.
3.  **State and Configuration:** Classes can hold state and configuration. A `SourceBufferObj` holds its buffer; a `HierarchicalTaskPoolRunner` holds its capacity source. This is difficult to do cleanly with standalone generator functions.

```typescript
// Every "Obj" is a structured component with a predictable lifecycle.
export class PullObj<T> implements AsyncIterable<T> {
    // ... state management ...
    protected generator?: AsyncGenerator<T,T|undefined,void>;

    constructor() {
        this.generator = this.pull_impl();
    }
    
    // The core logic is implemented here by subclasses.
    protected async* pull_impl() : AsyncGenerator<T,T|undefined,void> {
        return undefined;
    }

    // Standard async iterator methods ensure consistent behavior.
    public async next(): Promise<IteratorResult<T>> { /* ... */ }
    public async return(value?: T): Promise<IteratorResult<T>> { /* ... */ }
    public async throw(error?: any): Promise<IteratorResult<T>> { /* ... */ }
}
```

## 2. The Pull Model: Lazy, Convergent (Many-to-One) Processing

The library's design is built on a fundamental symmetry. The pull model represents the **convergent**, or **many-to-one**, half of this design. It excels at taking multiple input streams and combining them into a single output stream.

This model is also **lazy**, meaning work is only performed when a value is explicitly requested or "pulled" by a consumer. Think of it as an assembly line: the line only moves forward when the worker at the end pulls the next item.

**Building Blocks:**

*   **`SourceObj`**: The start of a chain. It generates values from an internal source.
*   **`PullObj...Link`**: The middle of a chain. It pulls a value from an upstream source, transforms it, and passes it downstream.
*   **Consumer**: Typically a `for await...of` loop that pulls values from the end of the chain.

**When to Use It:**
Use the pull model for standard data processing pipelines and when you need to **combine** multiple data sources into one.

```typescript
// A simple pull pipeline:
const numberSource = new SourceBufferObj([1, 2, 3, 4, 5]);
const squareLink = new PullMapObj(numberSource, (n) => n * n); // 1:1 transform

// The consumer loop pulls from the end of the chain, triggering all upstream work.
for await (const squaredNumber of squareLink) {
  console.log(squaredNumber); // 1, 4, 9, 16, 25
}
```

## 3. A Tour of Common Pull Stream Operations

While this guide isn't a full API reference, it's helpful to see the breadth of built-in tools for common pull-based manipulations. The library provides composable objects for nearly any convergent pattern you might need.

### Single-Stream Transformations

These objects operate on a single stream of data, much like array methods.

*   **`PullMap`**: Transforms each item in a stream. (e.g., `(n) => n * 2`)
*   **`PullFilter`**: Keeps only the items that match a condition. (e.g., `(n) => n > 10`)
*   **`PullDedupe`**: Removes duplicate items from a stream.
*   **`PullReduce`**: Combines all items into a result, yielding each intermediate step. (e.g., calculating a running total)
*   **`PullFlatMap`**: Transforms one item into many, flattening the result into the main stream.
*   **`PullWindow` / `PullBuffer`**: Groups items into batches, either by a fixed size (`Window`) or a dynamic condition (`Buffer`).

### Combining Multiple Streams (Many-to-One)

These tools orchestrate multiple pull streams together, demonstrating the convergent nature of the pull model.

*   **`PullConcat`**: Chains streams together sequentially, exhausting one before starting the next.
*   **`PullZip`**: Combines streams into a single stream of tuples, pairing corresponding items from each source.
*   **`PullRoundRobin`**: Merges streams by taking one item from each in a fair, rotating sequence.
*   **`PullRace`**: Merges streams by yielding the next item from whichever stream produces one first, enabling resolution-order processing.

This is just a sample; these building blocks allow you to construct sophisticated, declarative data pipelines.

## 4. The Push Model: Eager, Divergent (One-to-Many) Processing

The push model is the symmetric opposite of pull. It represents the **divergent**, or **one-to-many**, half of the library's design. It excels at taking a single input stream and splitting it out to multiple destinations.

This model is **eager**, meaning a producer can "push" data into the pipeline as soon as it's available, without waiting for a consumer to ask for it. Think of it as an event bus dispatching events to multiple listeners.

**Building Blocks:**

*   **Producer**: Any code that calls `sink.next(value)`.
*   **`PushObj...Link`**: The middle of a chain. It receives a value, transforms it, and pushes it to a downstream sink.
*   **`SinkObj`**: The end of a chain. It consumes values and terminates the pipeline.

**Concurrency and Ordering:**
A key feature of the push model is that `next()` calls can be made concurrently. By default, these operations are not serialized. If you need to guarantee order and prevent race conditions, you must wrap a link or sink with `PushSerialObj`.

### Familiar Transformation Patterns

Just like the pull model, the push model provides a rich set of 1-to-1 transformation links. Developers who understand `PullMap` will find `PushMap` intuitive; they serve the same purpose for a different stream type. This includes:

*   `PushMapObj`
*   `PushFilterObj`
*   `PushReduceObj`
*   `PushFlattenObj`

### Divergent (One-to-Many) Operations

Where the push model truly shines is in its ability to split a stream:

*   **`PushForkObj`**: Broadcasts each incoming item to *all* of its downstream sinks.
*   **`PushDistributeObj`**: Sends each incoming item to a *specific* sink based on a selector function.
*   **`PushRoundRobinObj`**: Sends incoming items to its sinks in a fair, rotating sequence.

```typescript
// A simple push pipeline demonstrating divergence:
const highPrioritySink = new SinkCollectObj<string>();
const lowPrioritySink = new SinkCollectObj<string>();

const distributor = new PushDistributeObj(
  [highPrioritySink, lowPrioritySink], // Array of sinks
  (s: string) => s.includes("URGENT") ? 0 : 1 // Selector function
);

// A producer pushes data into the pipeline.
await distributor.next("INFO: System normal.");
await distributor.next("ALERT: URGENT CPU at 99%.");
await distributor.return();

console.log(highPrioritySink.buffer); // ["ALERT: URGENT CPU at 99%."]
console.log(lowPrioritySink.buffer);  // ["INFO: System normal."]
```

## 5. Combining Streams: The `AsyncIteratorCombiner`

The `AsyncIteratorCombiner` is the powerful engine for managing multiple **pull** streams in parallel. It provides the low-level implementation for many of the multi-stream patterns like `Race` and `RoundRobin`.

It provides several strategies for consuming from multiple iterators:
*   `race_generator()`: Yields the next available value from *whichever* stream produces one first. Perfect for creating a single, interleaved progress stream.
*   `round_robin_generator()`: Yields one value from each stream in a fair, rotating sequence.
*   `all()`: Waits for every stream to produce its next value before yielding an array of all values (similar to `zip`).

**When to Use It:**
Use the combiner when you have a known set of pull streams (e.g., from `entity.start()`) and need to consume them concurrently.

## 6. Synchronizing Operations: The `WaitObject`

A `WaitObject` is a low-level synchronization primitive. You can think of it as a **resettable, one-time signal** or a **sequence of promises**.

*   Code can `await waitObj.wait()` to pause until a signal is received.
*   Another part of the system can call `waitObj.resolve()` to send the signal and unblock the waiting code.

Its key feature is that the signal generator can keep generating signals without worrying whether anyone is waiting. Most developers will not need to instantiate a `WaitObject` directly. It is primarily used as an internal mechanism to build more complex components.

## 7. Bridging the Models: The `PushPullBufferObj`

The `PushPullBufferObj` is the essential bridge that connects the push and pull worlds. It acts as a decoupled, asynchronous queue.

*   It exposes a **sink** interface for eager producers to push data into it.
*   It exposes a **source** interface for lazy consumers to pull data from it.

Internally, it uses a `WaitObject` to signal the pull side when new data has been pushed into the buffer.

**When to Use It:**
Use it whenever you need to decouple a fast producer from a slow consumer (or vice-versa), or when you need to create a dynamic work queue where one part of your system adds tasks and another part consumes them.

## 8. The Grand Unifier: `HierarchicalTaskPoolRunner`

The `HierarchicalTaskPoolRunner` is the highest-level abstraction in the library. It brings all the other concepts together to provide a robust engine for orchestrating complex, capacity-limited workloads.

It is designed to consume a **pull source of task runner functions**.

Key features that build on the lower-level concepts:
*   **Capacity Management (`CapacitySource`):** It only pulls and starts a new task when it has capacity, preventing resource exhaustion. It supports hierarchical capacity for fair resource sharing across multiple pools.
*   **Lazy Task Invocation:** By pulling *functions* that start tasks, it ensures work only begins once capacity is secured.
*   **Eager Execution:** The moment a task finishes, the runner immediately starts the next one to maximize throughput, without waiting for the consumer.
*   **Dynamic Sources:** It can consume from any `PullObj` source, including a `PushPullBufferObj`, allowing you to dynamically inject new tasks into a running pool.

**When to Use It:**
This is your go-to tool for production-grade parallelism in FireFoundry, especially for processing large collections of items or handling dynamically discovered work.