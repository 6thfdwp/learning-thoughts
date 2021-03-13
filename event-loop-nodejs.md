## Intro 
We often hear NodeJS is single-threaded, async and event driven I/O, which make it really good at handling large number of concurrent requests compared to traditional server programs which maintain a thread pool to handle each request per thread.

Essentially NodeJS runtime consists of one stack where JS is running and a few queues ([x1] need to elaborate) where all the async event (e.g I/O) handlers are scheduled to execute (e.g JS code from the user to process the result) when the stack is empty.

The way it orchestrates the entire mechanism between JS runtime and native OS level to handle I/O and other long running tasks is called **Event loop**.

## JS runtime
All the JS code from users and NodeJS standard libs are running in the Google V8 JS engine, it's single threaded (main thread) which means there is only one call stack frame. Each execution of the function will be pushed into the stack and popped one after another upon finishes, all the associated variables inside the function will be gone.  

NOTE: heap is another data structure for object allocation, objects used in fn execution are only pointers pushed/popped in the call stack). Memory may still hold the object content after function returns if there is reference. Call stack is also what error stack trace prints out when some exception occurs.

## Events
When user's JS code needs to do some long running operations, e.g I/O (reading files, http request, querying db  etc.) it triggers calls through another abstraction layer which actually communicates to the native level.
Each call to the native OS level will generate an event upon completion/failure.

## Non-blocking 
How does NodeJS achieve it with single-threaded call stack ?

Typically, network I/O can be performed in a non-blocking way using kernel primitives, `epoll` on Linux, `kqueue` on BSD (MacOS), and `IOCP` (Input Output Completion Port) in Windows.    
But for tasks like crypto, image processing, which are CPU intensive, also other types of I/O that cannot be addressed by those built-in OS mechanism. that need to run in a thread pool.

In [Not So Single Threaded](https://www.youtube.com/watch?v=zphcsoSJMvM), it gives some simple visualisation on how CPU bound tasks differ from I/O ones. In short, for CPU intensive tasks requires dedicated threads to process, it's limited to computing power (the CPU processors) and the thread pool (can only assign number of tasks up to size of the thread pool). But for I/O tasks, no dedicated thread allocation in thread pool, CPU is mostly idle.  

Consider we have NodeJS server running on a 2 core computer, one of the endpoints handles the signup which normally need lots of computing for password hashing. When NodeJS receives each request, it actually assign it to a background thread to process. 

 ▪︎ 2 concurrent requests

![image](https://user-images.githubusercontent.com/2919741/110912726-004bd180-8360-11eb-9bf6-3ac5eeb2572d.png)

This is easy, 2 threads running on 2 core CPU in true parallel, each takes about 100ms

 ▪︎ 4 concurrent requests

![image](https://user-images.githubusercontent.com/2919741/110913794-4bb2af80-8361-11eb-802b-6aaf64b2dd69.png)

Again NodeJS assigns each request to a thread, but because we only have 2 core, each core has to switch between 2 threads back and forth, make it look like in parallel. (that's how CPU does multi-tasking). In the end it doubles the amount of time for each request, and they start and finish at similar time.

 ▪︎ 6 (or more) concurrent requests

![image](https://user-images.githubusercontent.com/2919741/110915136-fd061500-8362-11eb-8cbb-10930376b57a.png)

Turns out NodeJS actually spins up a thread pool ([x2 how to config] default 4) when it receives the first request that requires a dedicated thread. When more requests coming in exceed the size of thread pool, it will queue up the fifth request until one of the first 4 finishes.   
So we see this long tails where fifth/sixth actually starts processing after 200ms.

But for I/O tasks, say sending requests to API, it roughly takes similar time for each request to finish (receiving the data) no matter how many concurrent requests. As mostly it's waiting for the response from network, not consuming server's power. 


> A common misconception among the developers about Node is that Node performs all the I/O in the thread pool.

> To orchestrate this entire process while supporting cross-platform I/O, managing the thread pool, scheduling event handler execution, there should be an abstraction layer that encapsulates these inter-platform and intra-platform complexities and expose a generalized API for the upper layers of Node.

This abstraction layer is mainly implemented with a cross-platform library called **libuv**.

Below showing the [building blocks of NodeJS platform](https://blog.insiderattack.net/five-misconceptions-on-how-nodejs-works-edfb56f7b3a6)
![image](https://user-images.githubusercontent.com/2919741/110237729-3b798980-7f89-11eb-9469-edb5e7bded05.png)

## How it roughly loops
Let's try to put things together:

1. User's JS code triggers async tasks (fs.readFile, http.get, setTimeout), delegates to **libuv** to leverage either built-in kernal primitives or allocate it in the thread pool (from JS -> native [x3 through C++ binding?])
2. When I/O operations finish, it will add the corresponding callback handlers in the event (task) queue, waiting to be picked up (native sends completion signals back)
3. When the stack is free and the first event in the queue will be pushed to the stack for execution which could in turn push more function calls into the stack (inside V8 JS runtime)
4. During execution it could trigger more I/O requests, hence go back to the first step again
