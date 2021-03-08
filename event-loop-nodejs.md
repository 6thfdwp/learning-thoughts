## Intro 
We often hear NodeJS is single-threaded, async and event driven I/O, which make it really good at handling large number of concurrent requests compared to traditional server programs written in e.g Java. 

Essentially NodeJS runtime consists of one stack where JS is running and so called macro and micro queues where all the async event (e.g I/O) handlers are scheduled waiting to be picked up to execute (these are mostly JS code from the user) when the stack is empty.

The way it orchestrates the entire mechanism between JS runtime and native OS level to handle I/O and other long running tasks is called **Event loop**.

## JS runtime
All the JS code from users and NodeJS standard libs are running in the Google V8 JS engine, it's single threaded which means there is only one call stack frame. Each execution of the function will be pushed into the stack and popped upon finishes, all the variables inside the function will be gone.  

NOTE: heap is another data structure for object allocation, objects used in fn execution are only pointers pushed/popped in the call stack). Memory may still hold the object content after function returns if there is reference. Call stack is also what error stack trace prints out when some exception occurs.

## Events
When user's JS code needs to do some long running operations, e.g I/O (reading files, http request, querying db  etc.) it triggers calls through another abstraction layer which actually communicates to the native level.
Each call to the native OS level will generate an event upon completion/failure.

### I/O and CPU bound tasks 
Typically, network I/O can be performed in a non-blocking way using built-in OS level functionalities, epoll on Linux, kqueue on BSD (MacOS), IOCP() event ports and IOCP (Input Output Completion Port) in Windows. But File I/O cannot all be addressed by those built-in OS mechanism. 
Also for tasks like crypto, image processing, which are CPU intensive, that need to run in a thread pool.

> A common misconception among the developers about Node is that Node performs all the I/O in the thread pool.

> To orchestrate this entire process while supporting cross-platform I/O, managing the thread pool, scheduling events, there should be an abstraction layer that encapsulates these inter-platform and intra-platform complexities and expose a generalized API for the upper layers of Node.

This is mainly implemented with a cross-platform library called **libuv**.

Below showing the [building blocks of NodeJS platform](https://blog.insiderattack.net/five-misconceptions-on-how-nodejs-works-edfb56f7b3a6)
![image](https://user-images.githubusercontent.com/2919741/110237729-3b798980-7f89-11eb-9469-edb5e7bded05.png)

## How it roughly loops
1. Users' code triggers I/O requests (fs.readFile, http.get, setTimeout), delegates to **libuv** to leverage either built-in OS async I/O or perform operations in the thread pool (from JS -> native)
2. When I/O operations finish, it will add the corresponding callback handlers in the event (task) queue, waiting to be picked up (native sends completion signals back)
3. When the stack is free and the first event in the queue will be pushed to the stack for execution which could in turn push more function calls into the stack (inside V8 JS runtime)
4. During execution it could trigger more I/O requests, hence go back to the first step again
