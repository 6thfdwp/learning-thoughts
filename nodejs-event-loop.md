## Intro 
We often hear NodeJS is single-threaded, async and event driven I/O, which make it really good at handling large number of concurrent requests as it's non-blocking.

Essentially NodeJS runtime consists of one stack where JS is running and a few queues (discussed later) where all the async event (e.g I/O) handlers are scheduled to execute (e.g JS code from the user to process the result) when the stack is empty.

The way it orchestrates the entire mechanism between JS runtime and native OS level to handle I/O and other long running tasks is called **Event loop**.

## JS runtime
All the JS code from users and NodeJS standard libs are running in the Google V8 JS engine, it's single threaded (main thread) which means there is only one call stack frame. Each execution of the function will be pushed into the stack and popped one after another upon finishing.

NOTE: heap is another data structure for object allocation, objects used in fn execution are only pointers pushed/popped in the call stack). Memory may still hold the object content after function returns if there is reference. Also closure forms the scope chain which could still hold some variables to be used when those callbacks triggered.

Call stack is also what error stack trace prints out when some exception occurs.

## Events and its Queues
When user's JS code needs to do some long running operations, e.g I/O (reading files, http request, querying db  etc.) it triggers calls through another abstraction layer which actually communicates to the native level.
Each call to the native OS level will generate an event upon completion/failure. 

There are 4 main types of queues for different phases of the loop:
- Expired timers and intervals 
- IO events queue 
- Immediate queue - callbacks added with `setImmediate`
- Close handlers 

Besides 4 main queues, there are 2 other intermediate queues:
- Next ticks
- Micro tasks - resolved promise callbacks 

When each main phase is completed, event loop will check the two intermediate queues, see if there is any pending items. It will immediately process them all before moving to next main phase. 
 
## Non-blocking 
How does NodeJS achieve it with single-threaded call stack ?

Typically, network I/O can be performed in a non-blocking way using kernel primitives, `epoll` on Linux, `kqueue` on BSD (MacOS), and `IOCP` (Input Output Completion Port) in Windows.    
But for tasks like crypto, image processing, which are CPU intensive, also other types of I/O that cannot be addressed by those built-in OS mechanism. that need to run in a thread pool.

In [Not So Single Threaded](https://www.youtube.com/watch?v=zphcsoSJMvM), it gives some simple visualisation on how CPU bound tasks differ from I/O ones. In short, for CPU intensive tasks requires dedicated threads to process, it's limited to computing power (the CPU processors) and the thread pool (can only assign number of tasks up to size of the thread pool). But for I/O tasks, NodeJS tries not allocating threads in thread pool, keep CPU mostly idle.  

Consider we have NodeJS server running on a 2 core computer, one of the endpoints handles the signup which normally need lots of computing for password hashing. When NodeJS receives each request, it actually assign it to a background thread to process. 

 ▪︎ 2 vs 4 concurrent requests

![image](https://user-images.githubusercontent.com/2919741/111007701-35e1d080-83db-11eb-8f20-24aa27cfd87d.png)

For 2 requests, each gets assigned to one thread, so 2 threads running on 2 core CPU in true parallel, each takes about 100ms. As for 4 we need 4 threads, but each core has to switch between 2 threads back and forth, make it look like in parallel. (that's how CPU does multi-tasking). In the end it doubles the amount of time for each request, also they start and finish at similar time (CPU has a fair switch ~).

 ▪︎ 6 (or more) concurrent requests

![image](https://user-images.githubusercontent.com/2919741/110915136-fd061500-8362-11eb-8cbb-10930376b57a.png)

Turns out NodeJS actually spins up a thread pool ([x1]) when it receives the first request that requires a dedicated thread. When more requests coming in exceed the size of thread pool, it will queue up the following request (the fifth this case) until one of the first 4 finishes.   
So we see this long tails where fifth/sixth only start processing after about 200ms.

But for I/O tasks, say sending requests to API, it roughly takes similar time for each request to finish (receiving the data) no matter how many concurrent requests. Because once the main thread handed the request over to underlying kernel, it will immediately finish and continue to take more requests. Mostly it depends on network speed and remote resources, not the server where NodeJS is running.

> A common misconception among the developers about Node is that Node performs all the I/O in the thread pool.

> To orchestrate this entire process while supporting cross-platform I/O, managing the thread pool, scheduling event handler execution, there should be an abstraction layer that encapsulates these inter-platform and intra-platform complexities and expose a generalized API for the upper layers of Node.

This abstraction layer is mainly implemented with a cross-platform library called **libuv**.

Below showing the [building blocks of NodeJS platform](https://blog.insiderattack.net/five-misconceptions-on-how-nodejs-works-edfb56f7b3a6)
![image](https://user-images.githubusercontent.com/2919741/110237729-3b798980-7f89-11eb-9469-edb5e7bded05.png)

## How it roughly loops
Let's try to put things together:

1. User's JS code triggers async tasks (fs.readFile, http.get, setTimeout), delegates to **libuv** to leverage either built-in kernal primitives or allocate it in the thread pool (from JS -> native [x2])
2. When I/O operations finish, it will add the corresponding callback handlers in the event (task) queue, waiting to be picked up (native sends completion signals back)
3. When the stack is free and the first event in the queue will be pushed to the stack for execution which could in turn push more function calls into the stack (inside V8 JS runtime)
4. During execution it could trigger more I/O requests, hence go back to the first step again

## Some own afterthoughts 
I feel something in common for hugely successful open source softwares: right (or better) abstraction over the existing problems, provide better programming model for devs, avoid dealing with low level error-prone APIs, even better cross platform!

For NodeJS it mainly abstracts away the complexities to manage CPU and IO tasks efficiently. And It gives users a performant runtime and simple unified programming model by combining other open source softwares, it does not even create a new server programming language!  From dev's view, most of time we just use corresponding `async` API to send commands and register callbacks to process, not care if it's CPU and IO, no complex threading API to learn. (Promise, async/await are mainly syntax level to help write better async code without nested callbacks).

Same as React, to build the [UI runtime](https://overreacted.io/react-as-a-ui-runtime/) with declarative composition, abstract away the imperative costly UI mutations (e.g dealing with DOM API in browser) . React manages the whole lifecycle to figure which part and when and how updates need to be applied, avoid blocking the main UI thread as much as possible. While NodeJS manages the async tasks lifecycle to avoid blocking JS thread. 

---
- [x1] Is it NodeJS or `libuv` maintaining the thread pool, default 4 ?
- [x2] From JS -> native side, is it through C++ binding?