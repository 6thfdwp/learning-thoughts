## System design
 For those general system design questions, like design Youtube, Twitter etc. It is broad, generally there are some common aspects to talk for any live product:

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

It can be also focused on specific features on a particular products, e.g recommendation, timeline  

https://www.interviewbit.com/courses/programming/
