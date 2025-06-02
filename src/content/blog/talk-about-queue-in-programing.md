---
title: queue-programming
author: ICE
pubDatetime: 2024-06-02T16:55:12.000+00:00
slug: queue-programming
featured: false
draft: false
tags:
  - Rxjs
  - js
description: "javascript"
---

# Understanding Queues in Software Programming
A queue is a fundamental data structure in computer science, designed to store and manage data in a specific order, following the First In, First Out (FIFO) principle. Queues are widely used in everyday programming scenarios, such as caching, task scheduling, and data flow control. In this article, we'll explore the concept of queues through practical examples, focusing on their ability to handle asynchronous tasks and complex data processing scenarios.
What is a Queue?

A queue operates like a line at a ticket counter: the first element added is the first to be processed. This makes queues ideal for scenarios where order matters, such as task scheduling or event logging. Let’s dive into real-world examples to understand how queues solve common programming problems.
Example 1: Asynchronous Printing Task Queue
In scenarios like a printer queue, tasks (e.g., printing documents) must be processed in the order they are received. Each task may involve asynchronous operations, such as fetching user data from a backend or transforming data before printing. A queue ensures tasks are executed sequentially, with error handling (e.g., retries) if needed.
Here’s a TypeScript implementation of an asynchronous queue that processes tasks one by one:
```javascript
class AsyncFunctionQueue {
    private queue: Array<() => Promise<void>> = []; // Store async function tasks
    private isProcessing: boolean = false; // Track if tasks are being processed

    // Add an async function task to the queue
    async addTask(task: () => Promise<void>): Promise<void> {
        this.queue.push(task);
        console.log("Added task to queue");
        // Start processing if not already processing
        if (!this.isProcessing) {
            await this.processTasks();
        }
    }

    // Process tasks in the queue
    private async processTasks(): Promise<void> {
        this.isProcessing = true;
        while (this.queue.length > 0) {
            const task = this.queue.shift()!; // Dequeue task
            console.log("Processing task...");
            await task(); // Execute the async task
            console.log("Completed task");
        }
        console.log("No tasks to process, waiting...");
        this.isProcessing = false;
    }

    // Check if the queue is empty
    isEmpty(): boolean {
        return this.queue.length === 0;
    }
}

// Example usage
async function main() {
    const queue = new AsyncFunctionQueue();
    for (let i = 0; i < 3; i++) {
        const task = async () => {
            console.log(`Printing Document_${i}.pdf`);
            await new Promise(resolve => setTimeout(resolve, 1000));
        };
        await queue.addTask(task);
    }
}
main();
```
### Key Points:

Tasks are asynchronous functions, ensuring non-blocking execution.

The queue processes tasks sequentially, preventing parallel execution to maintain order.

Example 2: Collecting and Sending Events to Backend
Another common use case is collecting events (e.g., user interactions) on the frontend and sending them to the backend for logging. Instead of sending each event immediately, a queue can batch events to reduce network requests and improve performance.
Here’s a JavaScript implementation of a queue that flushes tasks when the browser is idle or after a microtask:
```javascript
export class Queue {
    private stack: Array<() => void> = []; // Store tasks
    private isFlushing: boolean = false; // Track if queue is being flushed

    constructor() {}

    // Add a task to the queue
    addFn(fn: () => void): void {
        if (typeof fn !== "function") return;
        if (!("requestIdleCallback" in window || "Promise" in window)) {
            fn();
            return;
        }
        this.stack.push(fn);
        if (!this.isFlushing) {
            this.isFlushing = true;
            if ("requestIdleCallback" in window) {
                requestIdleCallback(() => this.flushStack());
            } else {
                Promise.resolve().then(() => this.flushStack());
            }
        }
    }

    // Clear the queue
    clear(): void {
        this.stack = [];
    }

    // Get current tasks
    getStack(): Array<() => void> {
        return this.stack;
    }

    // Execute all tasks in the queue
    flushStack(): void {
        const temp = this.stack.slice(0);
        this.stack = [];
        this.isFlushing = false;
        for (let i = 0; i < temp.length; i++) {
            temp[i]();
        }
    }
}

// Example usage
const queue = new Queue();
queue.addFn(() => console.log("Sending event: Click"));
queue.addFn(() => console.log("Sending event: Scroll"));
```
Key Points:

Tasks are batched and executed when the browser is idle (requestIdleCallback) or in a microtask (Promise.resolve).
This approach optimizes resource usage by avoiding immediate execution.

Example 3: Complex Queue with RxJS
For more complex scenarios, such as consuming data only when a certain number of items are collected or after a specific time interval, we can use RxJS, a powerful library for reactive programming. RxJS treats data as streams, making it easier to handle complex logic.
Here’s an RxJS-based queue that consumes data when either 3 items are collected or 5 seconds have passed:

```javascript
import { Subject, merge, buffer, filter, bufferCount, bufferTime, map } from "rxjs";

class QueueConsumer<T> {
    private dataSubject = new Subject<T>();
    private triggerSubject = new Subject<void>();

    constructor(
        private threshold: number,
        private timeout: number,
        private consumer: (data: T[]) => void
    ) {
        this.setupQueue();
    }

    // Set up the RxJS stream
    private setupQueue() {
        this.dataSubject
            .pipe(
                buffer(
                    merge(
                        this.dataSubject.pipe(bufferCount(this.threshold), map(() => void 0)),
                        this.dataSubject.pipe(bufferTime(this.timeout), map(() => void 0)),
                        this.triggerSubject
                    )
                ),
                filter((data) => data.length > 0)
            )
            .subscribe((data) => {
                this.consumer(data);
            });
    }

    // Add data to the queue
    push(data: T) {
        this.dataSubject.next(data);
    }

    // Force flush the queue
    flush() {
        this.triggerSubject.next();
    }

    // Clean up resources
    destroy() {
        this.dataSubject.complete();
        this.triggerSubject.complete();
    }
}

// Example usage
function example() {
    const consumer = (data: number[]) => {
        console.log("Consumed:", data);
    };

    const queue = new QueueConsumer<number>(3, 5000, consumer);

    queue.push(1);
    queue.push(2);
    setTimeout(() => queue.push(3), 1000);
    setTimeout(() => queue.push(4), 2000);
    setTimeout(() => queue.push(5), 3000);

    setTimeout(() => queue.destroy(), 10000);
}
example();
```
### Key Points:

The queue buffers data until either the threshold (3 items) or timeout (5 seconds) is reached.
RxJS’s stream-based approach simplifies complex logic, treating data as a continuous flow.
The consumer function processes batched data, making it ideal for scenarios like logging or analytics.

### Why Use RxJS for Complex Logic?
RxJS introduces a streaming mindset, where data is treated as a sequence of events over time. This makes it easier to:

Handle asynchronous data flows with operators like buffer, filter, and merge.
Implement complex conditions (e.g., thresholds or timeouts) without manual state management.
Write declarative, maintainable code for advanced scenarios.

## Conclusion
Queues are a versatile tool in programming, enabling orderly data processing, asynchronous task management, and efficient resource usage. From simple sequential task queues to batched event logging and complex RxJS-based strategies, queues adapt to a wide range of problems. By understanding and applying queues, developers can build robust, scalable, and maintainable systems.
