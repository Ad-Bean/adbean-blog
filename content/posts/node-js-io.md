+++
title = '阅读：Ryan Dahl: NodeJS'
date = 2024-01-30T13:23:51-05:00
draft = false
tags = ['阅读']
+++

## NodeJS

> 这学期要用 JS 写分布式，看 Ryan Dahl 在 2009 年 JSConf 上分享 NodeJS 背后的概念，刚好复习一下 JS 的一些知识

NodeJS 简述：

- Server Side Javascript
- Built on Google's V8
- Evented, non-blocking IO, similar to EventMachine or Python's Twisted.
- ~~CommonJS module system~~ (用 ES6 Module 替代 CommonJS)
- 8k lines of C/C++, 2k lines of JavaScript, 14 Contributes

## IO needs to be done differently

> 事件驱动，单线程异步 IO 是否能理解为非阻塞的协程？

许多 Web 应用使用类似 `var result = db.query("select * from T);` 的语句，当查询在运行时，框架在做什么呢？大部分时间下，它在一直等待。

> IO Latency 在硬盘、网络级别是较长的

好的软件是可以处理多任务的，当前线程等待时，其他的线程可以执行其他任务

> Apache 和 NGINX 是如何处理 IO 的？

从 Benchmark 来看，随着处理量增长，concurrency NGINX 是 Apache 的两倍，但内存占用上，Apache 显著大于 NGINX。

- Apache uses one thread per connection
- NGINX doesnot use threads, it uses an **event loop**

对于大量的并发操作，不能用 OS threads 处理每个连接：上下文切换是昂贵的，调用栈也非常占用空间。而是用单线程和事件循环。

> green threads / coroutines can improve 但还是可能是阻塞的
>
> threaded concurrency is a leaky abstraction

使用事件循环，，不需要 machinery，query 结束之后执行相应的函数：

```javascript
db.query('select * from T', () => {});
```

### Cultural Bias

> everybody is talking about threads

学 IO 的时候大部分人都是先接触阻塞的 `puts()` 和 `gets()`

### Missing Infrastructure

> Why isn't everyone using event loops?
>
> Single threaded event loops require I/O to be non-blocking

- POSIX async
- closures and aynonymous functions
- async queries
- ...

### Too much Infrastructure

EventMachine, Twisted, AnyEvent

easy to create efficient servers (Ruby)

## Javascript

> 事件循环：微任务、宏任务
>
> https://javascript.info/event-loop
>
> https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop#queue

Javascript designed specifically to be used with an event loop

- anonymous functions, closures
- only one callback at a time
- I/O through DOM event callbacks

## NodeJS

provide a purely **evented**, **non-blocking** infrastructure to script highly **concurrent** programs

## Design Goals

no functions should direct perform I/O, to receive info from disk/network/another process there must be a callback

### Low level

> 这部分讨论了 Ryan Dahl 是怎么考虑和设计的

stream everything, never force the buffering of data

donot remove functionality present at the POSIX layer (support half-closed TCP connections)

have built-in support for the most important protocols: TCP, DNS, HTTP

support many HTTP features:

- chunked requests and responses
- keep-alive
- hang requests

API should be both familiar to client-side JS programmers and old school UNIX hackers

platform independent

## Usage and Examples

> 一些事件循环的例子，由于都比较久而且 PPT 实在看不清，这里用 MDN 最新的

```Javascript
(() => {
  console.log("this is the start");

  setTimeout(() => {
    console.log("Callback 1: this is a msg from call back");
  }); // has a default time value of 0

  console.log("this is just a message");

  setTimeout(() => {
    console.log("Callback 2: this is a msg from call back");
  }, 0);

  console.log("this is the end");
})();

// "this is the start"
// "this is just a message"
// "this is the end"
// "Callback 1: this is a msg from call back"
// "Callback 2: this is a msg from call back"

```

process object emits an **event** when it receives a signal. Like in the DOM, you need only add a listener to catch them.

```javascript
process.addListener('SIGINT', () => {
  puts('good bye');
  process.exit(0);
});
```

A TCP server emits a "connection" event each time someone connects

HTTP upload emits a "body" event on each packet.

All objects which emit events are instances of `process.EventEmitter`

### File I/O is non-blocking

> something typically hard to do
>
> 这部分主要读取文件，代码实在看不清

A promise is a kind of EventEmitter which emits either "success" or "error"

All file operations return a promise (API sugar)

### HTTP Server

> 展示了 Simple HTTP server 和 Streaming HTTP server，后者可以 hang request

此外还可以使用 `sys.exec('ls -l /').addCallback()`

> 这部分提到的 buffer 不太理解是什么意思，不知道是不是用于处理 TCP 流、文件系统操作、上下文之类的二进制流数据的 Buffer。

## IRC demo

> Ryan 展示了一个 Internet Relay Chat 聊天服务器 https://gist.github.com/ry/a3d0bbbff196af633995

## Internal Design

- V8 (Google)
- libuv / libev / libeio
- http-parser
- evocom
- udns

Thread Pool underneath everything:

- Blocking (possibly blocking) system calls are executed in the thread pool
- Signal handlers and thread pool callbacks are marshaled back into to the main thread via a pipe

`STDIN_FILENO` will refer to a file, cannot `select()` on files: `read()` will **block**

Solution: start a pipe and pumping thread, pump data from blocking fd into pipe. Main thread can pool for data on the pipe.

> 这一段源码 https://github.com/nodejs/node/blob/d52f63d9b29446b26ec831acd8ec6da9147896e5/deps/coupling/coupling.c
>
> 不知道是不是已经被弃用了，最早的源代码很多都是 C 写的

## 结语

> 09 年的时候提出这种单线程非阻塞的异步设计，是非常惊艳的。希望有时间可以看看 NodeJS 的源码。
>
> 03/06/2024 补充，当时忽略了 libuv 的引入，只知道 NodeJS 是事件循环的，但是不知道他是怎么实现的。
