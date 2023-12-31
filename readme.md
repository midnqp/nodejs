<p align=center><img width=30% src="https://github.com/midnqp/nodejs/assets/50658760/d77458b5-fc76-47ac-9915-de5224557c23"></p>

_Notes on Node.js_ by Muhammad Bin Zafar.
- [Preface](#preface)
  - [Features](#features)
  - [Benefits](#benefits)
  - [Drawbacks](#drawbacks)
  - [Workarounds for drawbacks](#workarounds-for-drawbacks)
- [Event Loop](#event-loop)
  - [Phases](#phases)
  - [Phases in Detail](#phases-in-detail)
    - [Timers](#timers)
    - [Pending callbacks](#pending-callbacks)
    - [Poll](#poll)
- [ESM vs CJS](#esm-vs-cjs)
- [Important builtin modules](#important-builtin-modules)
- [Important third-party packages](#important-third-party-packages)
- [Important Node.js backend frameworks](#important-nodejs-backend-frameworks)
- [JavaScript](#javascript)
- [TypeScript](#typescript)
- [Node.js vs Deno](#nodejs-vs-deno)
- [Node.js vs Bun](#nodejs-vs-bun)
- [Changelog of important features across releases](#changelog-of-important-features-across-releases)
- [Implementation of design patterns](#implementation-of-design-patterns)
  * [Creational patterns](#creational-patterns): [Singleton](#singleton), [Abstract Factory](#abstract-factory), ...
  * [Structural patterns](#structural-patterns): [Adapter](#adapter), [Bridge](#bridge), ...
  * [Behavioral patterns](#behavioral-patterns): [Chain of responsibility](#chain-of-responsibility), [Command](#command), ...

## Preface

Node.js is an asynchronous event-driven runtime environment, built on top of the V8 JavaScript engine from Google. Node.js is designed to build scalable network applications. 

Node.js is created by Ryan Dahl, who made the [first ever commit](https://github.com/nodejs/node-v0.x-archive/commit/9d7895c567e8f38abfff35da1b6d6d6a0a06f9aa) on 16 Feb 2009. His intention of building Node.js was to do event-driven I/O in JavaScript to have server-side JavaScript off the ground. 

Node.js is a popular choice for building real-time applications, such as chat applications, streaming applications, and web servers. Node.js is also used to build microservices, which are small, independent services that can be easily scaled and deployed.

##### Features
- Single-threaded event loop. Node.js uses a single-threaded event loop to handle all of its I/O. This makes Node.js very efficient for handling concurrent requests. Almost no function in Node.js directly performs I/O, only through its event loop.
- Non-blocking I/O. Node.js uses non-blocking I/O to handle its I/O operations. This means that Node.js can handle multiple requests at the same time without blocking. This is in contrast to today's more common concurrency model, in which OS threads are employed. Thread-based networking is relatively inefficient and very difficult to use.
- Built-in modules. Node.js has a large number of built-in modules that can be used to build applications.
- Package manager. Node.js has a package manager called npm that can be used to install and manage Node.js modules.

##### Benefits
- Scalability. Node.js is very scalable and can be used to handle a large number of concurrent requests. Primarily due to having single-threaded non-blocking I/O.
- Performance. Node.js is very performant and can handle a large number of requests per second. Not only accept new connections, but also quickly processes and returns responses.
- Versatility. Node.js can be used to build a wide variety of applications, including web servers, real-time applications, and microservices.
- Community. Node.js has a large and active community that provides support and resources for developers.

##### Drawbacks
- Memory usage. Node.js can use a lot of memory, especially for applications that handle a large number of concurrent requests. In truth, this is obvious and a non-issue. More concurrent requests accepted only means more memory.
- Security. Node.js is not as secure as some other programming languages.
- Learning curve. Node.js has a steep learning curve, especially for developers who are not familiar with JavaScript and its convention of async code execution, e.g. callbacks.

##### Workarounds for drawbacks
- Node.js runs on a single CPU core. Node.js being designed without threads doesn't mean you can't take advantage of multiple cores in your environment. 
  - Child processes can be spawned by using the [child_process.fork()](https://nodejs.org/api/child_process.html#child_process_child_process_fork_modulepath_args_options) API, and are designed to be easy to communicate with. 
  - Built upon that same interface is the [cluster](https://nodejs.org/api/cluster.html) module, which allows you to share sockets between processes to enable load balancing over your cores. 
  - Conventionally, during deployments on a production environment through Docker, multiple replicas of the Node.js server application is created, often equal to the number of CPU cores of a machine. Then the replicas are mapped to a Nginx load balancer. Note that, in this approach, the *cluster* module is not used.
- The *worker_threads* builtin module creates new processes to imitate a traditional operating system thread, given a piece of code and some data to work with.
```js
// filename: main.js
import * as threads from 'node:worker_threads'

function sumOfNumbers(workerData) {
  return new Promise((resolve, reject) => {
    const worker = new threads.Worker('piece-of-code.js', { workerData })
    worker.on('message', resolve)
    worker.on('error', reject)
  })
}

console.log(await sumOfNumbers([2, 3])) // 5

// filename: piece-of-code.js
import {workerData, parentPort} from 'node:worker_threads'
parentPort.postMessage(workerData[0] + workerData[1]) // 5
```

## Event Loop
The event loop is what allows Node.js to perform non-blocking I/O operations. Node.js is similar in design to, and influenced by, systems like Ruby's [Event Machine](https://github.com/eventmachine/eventmachine) and Python's [Twisted](https://twistedmatrix.com/trac/). 

Node.js takes the event model a bit further. It presents an event loop as a runtime construct instead of as a library. The event loop of Node.js is single-threaded, however file I/O operations, cryptographic operations, and some network operations run in multiple threads through the thread pool.

##### Phases
The 6 phases of the event loop are as follows.
```
   ┌───────────────────────────┐  this phase executes callbacks scheduled by setTimeout()
┌─>│           timers          │  and setInterval().
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐  executes I/O callbacks, which were deferred to its next 
│  │     pending callbacks     │  loop's iteration.
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  only used internally
│  └─────────────┬─────────────┘    
│  ┌─────────────┴─────────────┐  retrieve new I/O events from the kernel; execute all I/O
│  │           poll            │  related callbacks
│  └─────────────┬─────────────┘  
│  ┌─────────────┴─────────────┐      
│  │           check           │  callbacks of setImmediate()
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  callbacks for the event 'close'
   └───────────────────────────┘
```
Between each run of the event loop, Node.js checks if it is waiting for any asynchronous I/O or timers and shuts down cleanly if there are not any.

Each phase has a FIFO queue of callbacks to execute. While each phase is special in its own way, generally, when the event loop enters a given phase, it will perform any operations specific to that phase, then execute callbacks in that phase's queue until the queue has been exhausted or the maximum number of callbacks has executed. When the queue has been exhausted or the callback limit is reached, the event loop will move to the next phase, and so on.

Since any of these operations may schedule more operations and new events processed in the poll phase are queued by the kernel, poll events can be queued while polling events are being processed. As a result, long running callbacks can allow the poll phase to run much longer than a timer's threshold. 

##### Phases in Detail
###### Timers
Operating system scheduling or the long running callbacks in the "poll" phase are what causes timers such as *setTimeout()* to delay beyond the specified time in milliseconds, resulting us to describe `setTimeout(() => console.log('hello world'), 1000)` as "the callback will execute after at least 1000 ms, but not guarranteed to be precisely 1000 ms, usually delays more than 1000 ms".

```js
now = Date.now()
setTimeout(()=>console.log('timer is done', Date.now() - now), 100)
fs.readFile('small file', () => {})
// timer is done 102     (not exactly 100ms)
```
```js
now = Date.now()
setTimeout(()=>console.log('timer is done', Date.now() - now), 100)
fs.readFile('huuuuuge file', () => {})
// timer is done 100     (always perfectly 100ms)
```
```js
now = Date.now()
setTimeout(()=>console.log('timer is done', Date.now() - now), 100)
fs.readFile('small file', () => {
  const startCallback = Date.now();
  while (Date.now() - startCallback < 1000) { /* busy loop to block for 1000ms */ }
})
// timer is done 1012     (🤯 always delays more than 1000ms, 
//                         whereas it was supposed to delay for 100ms 🤯)
```
The second setTimeout() executes the callback in the correct delay of 100ms because in order to prevent starving the event loop, libuv has a hard maximum time before it stops polling for more events. Since the file read is taking beyond 100ms, event loop moves on to the next phases/iterations and runs the callback for the setTimeout(). 

This is evidence that, the event loop **pauses/waits/sleeps** in the "poll" phase to see whether anything happens; if nothing happens in 100ms (varies by platform) it needs to move on to the next phase.

The third piece of code is evidence that I/O callbacks are executed in the "poll" phase synchronously. This is also evidence that the "poll" phase is responsible for delaying timers, because it has to execute callbacks which can take any arbitrary amount of time.

###### Pending callbacks
This phase executes callbacks for some system operations such as types of TCP errors. For example if a TCP socket receives ECONNREFUSED when attempting to connect, some *nix systems want to wait to report the error. This will be queued to execute in the pending callbacks phase.

###### Poll
The poll phase has two main functions:
- Calculating how long it should block and poll for I/O, then
- Processing events in the poll queue.

When the event loop enters the poll phase and there are no timers scheduled, one of two things will happen:

- If the poll queue is not empty, the event loop will iterate through its queue of callbacks executing them synchronously until either the queue has been exhausted, or the system-dependent hard limit is reached.
- If the poll queue is empty, one of two more things will happen:
  - If scripts have been scheduled by setImmediate(), the event loop will end the poll phase and continue to the check phase to execute those scheduled scripts.
  - If scripts have not been scheduled by setImmediate(), the event loop will wait for callbacks to be added to the queue, then execute them immediately.

Once the poll queue is empty the event loop will check for timers whose time thresholds have been reached. If one or more timers are ready, the event loop will wrap back to the timers phase to execute those timers' callbacks.

## ESM vs CJS

## Important builtin modules
- Filesystem `node:fs` - unique approach of reading a file asynchronously and letting the event loop know through emitting events.
- Streams and pipes `node:stream`
- Buffers `node:buffer`
- Event emitter `node:events`
- Net `node:net`
- HTTP `node:http`
- Crypto `node:crypto`

## Important third-party packages
- Express. Overwhelmingly popular and the default approach to build a web server. 
- Lodash. Highest downloads in all of the Node.js ecosystem
- Sequelize. ORM abstraction over the database, to not write raw SQL and get hacked. 3nd in popularity as an ORM.
- TypeORM and Prisma. The most popular ORM in all of the Node.js ecosystem. No other ORM has more downloads and popularity than them.
- Pino and Winston. Cleaner, more structured logging for production environments. Quintessential!
- Fastify. Alternative to Express. Developed by one of the creators of Node.js itself!

## Important Node.js backend frameworks
- Nest. A really great Express-based backend framework with the perfect architecture.
- Next. Fullstack framework with Node.js and React.
- Koa and Fastify. Provides many relevant packages alongside.

## JavaScript
JavaScript is an interpreted, dynamically-typed, weakly-typed, multi-paradigm, garbage-collected, single-threaded programming language, often used for developing client-side web applications. 

Adding features such as asynchronous filesystem access, native HTTP/QUIC support, cryptography, streams, buffers, etc are what results in a separate runtime named Node.js. Their syntax has no difference. 

Each browser such as Chrome, Edge, Safari, Firefox implements their own JavaScript engine such as V8, Chakra, Webkit, Spidermonkey following the [ECMA-262](https://www.ecma-international.org/publications-and-standards/standards/ecma-262/) specification. However this results in differences among implementations for instance Chakra has a multi-threaded event loop 🤯. 

Note that despite JavaScript being asserted as an "interpreted" language, in truth, it is compiled through various approaches such as JIT on the fly right before executing it - making an impression of an interpreted language.

- *Code* is a set of special instructions to tell the computer what tasks to perform.
- *Syntax* are the rules of valid format and combinations of instructions.
- A *statement* is a group of expressions and operators that performs a specific task.
- Statements are made up of one or more *expressions*. In ECMA and MDN docs, expressions are often coupled with operators in the same section. Clearly because most expressions are made of operators.
- A *variable* is a container to hold a type and a value.
- Based on various aspects of a variable, languages are divided into the following.
  - *Strongly typed*, meaning the compiler will not allow operations between variables.
  - *Weakly typed*, meaning the compiler will allow and may automatically do type coercion.
  - *Statically typed*, meaning variable types are explicitly mentioned during declaration at compile-time and will not change at runtime.
  - *Dynamically typed*, meaning a variable can be assigned new values of different types at runtime.

## TypeScript
TypeScript is a strongly-typed language with optional static-typing.

## Node.js vs Deno
Announced in 2018. The creator of Node.js and Deno is the same person, Ryan Dahl. He built Deno out of the regret of the decisions he made which building Node.js. Deno tries to fix Node.js mistakes and provides a new approach to security and application development.

## Node.js vs Bun
Bun is a more recent competitor of Node.js which gained traction around <month-of-release>. Sumner built this.

## Changelog of important features across releases
- Node.js 18 LTS
  - Top-level await
  - `fetch()`
- Node.js 20 LTS
  - Set default packaging format to ESM or CommonJS.

## Implementation of design patterns
Design patterns are general, reusable solutions to commonly-occuring problems.

Effective software design requires considering issues that may not become visible until later in the implementation.

### Creational patterns
These design patterns are about class and/or object instantiation.

##### Singleton
A class of which only a single instance can exist.

The traditional implementation is as follows.
```ts
class DbConn {
  private static instance: DbConn | undefined;
  private constructor() {}
  public connect(): void {}
  public disconnect(): void {}
  public static getInstance(): DbConn {
    if (!DbConn.instance) DbConn.instance = new DbConn();
    return DbConn.instance;
  }
}

const dbconn = DbConn.getInstance()
```
Another interesting implementation would be as follows. However, using the `new` keyword and not getting a new instance in return can be misleading.
```
class DbConn {}

const dbconn = new DbConn()
```
##### Abstract Factory
##### Factory Method
##### Builder
##### Prototype

### Structural patterns
##### Adapter
##### Bridge
##### Composite
##### Decorator
##### Façade
##### Flyweight
##### Proxy

### Behavioral patterns
##### Chain of responsibility
##### Command
##### Interpreter
##### Iterator
##### Mediator
##### Memento
##### Observer
##### State
##### Strategy
##### Template Method
##### Visitor
