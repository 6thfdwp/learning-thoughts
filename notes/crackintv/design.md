## System design
 For those general system design questions, like design Youtube, Twitter etc. It is broad, generally there are some common aspects to talk for any live product:

It is hard to get started, some general approaches to follow:  
- Define the scope   
  Like MVP, what the core features we need for the system. So we know what to design for.
- Draw high level key components  
  Good to start storage, almost every system needs some kind of storage. So it is good point to start digging in a bit more. Based on scope, what are the tables and their relationship.  
  What kind of API specs needed   
  What is the user flow look like
- Pick one component to detailed design
- Scalability and Reliability
- Issues and trade-off


//
- Storage  
  How business data stored in e.g proper designed relational DB, how large blob files stored and served   
  Partition (Sharding) is one way to scale db

- Server (Web or API)  
  Handle client request, produce response   

- Scalability  
  Both storage (large volume of data) and server (serving more concurrent requests)

- Cache  
  Different level of caching strategy, business data, db indexing, query results.   
  Generally not good for long-tailed content (only popular would have higher cache hits, hence useful)   
  When to invalidate the cache

- Security  

Some numbers to know :
```sh
`2^10`:  1024,  ( ~10^3 thousand),  1 Kb, x bytes if every item is represented as 1b  
`2^16`:  65536,  64K  
`2^20`:  1048576 (~10^6 million)    1 Mb,   
`2^30`:          (~10^9 billion)  1 Gb  
`2^32`  ~4 billions (the typical long int range)

Read 1 MB sequentially from memory:  0.25 ms
Round trip with data center: 0.5 ms
Dish seek: 10 ms
Read 1 MB sequentially from dish: 20 ms
```
With these numbers, we can roughly analyse the storage, memory usage, data transmitted, bandwidth
