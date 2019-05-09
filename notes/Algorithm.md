https://www.interviewbit.com/courses/programming/

## Data structure
Fundamentally (physically), all data structure could be categorised as two:

**Contiguous** vs **Linked**  
Contiguous: array, heaps, hash table  
 - O(1) index based retrieve    
 - Space efficiency, only data   

Linked: linked list, tree, graph (chunks of memory bound by pointer)  
 - Simpler insertion and deletion  
 - O(n) to retrieve

⍰ How to do dynamic array amortised analysis

**Do iteration**  
Most algorithms operating on contiguous space (array, string etc) involve some
iteration techniques:
- forward (from first to last), or backward
```py
for i in reversed( xrange(len(A) ):
# or using while
while j > 0:
  # do stuff
  j -= 1
```

- from two end to the middle
```py
# check palindrome
i, j = 0, len(A)-1
while i < j:
  if not A[i] == A[j]: return False
  i, j = i+1, j-1
# or do reverse manually
while i < n/2:
  j = n - 1 - i
  A[i], A[j] = A[j], A[i]
```

- from middle or some part of the array, expanding towards two ends
```py
# longest sub-palindrome
def expand(i, j, A):
  while i>=0 and j<len(A) and A[i]==A[j]:
    i, j = i-1, j+1
  return A[i+1:j]
```

- iterate two array at the same time   
  merge phase of merge sort
```py
# general form, need to deal with different size of arrays
while i < len(A) or j < len(B):
  if i >= len(A):
    # only do B[j]
  if j >= len(B):
    # only do A[i]

  # do both
  do(A[i], B[j])

# find all appearing in two sorted arrays
while i < ilen and j < jlen:
  if A[i] < B[j]:
    i += 1
  elif A[i] > B[j]:
    j += 1
  else:
    res.append(A[i])
    i, j = i+1, j+1
```
Keep two pointers (indices) is common, a few problems using this:  
- Remove duplicates in place (sorted is easier, one pass)  
  One for iteration, one for current non-dup item's index
- 2sum in **Ordered** array  
  Find 2 indices that adds up to given number. From two end, each time either increase left or decrease right



**Stack & Queue**  
can be implemented with either array or linked list.
```py
# array for stack
_array, top = [], -1
def push(item):
  # if it's not dynamic size of array, need consider overflow
  # if _top >= _len:
  #   resize array or not push
  _array.append(item)
  _top += 1

def pop():
  # current top will be overwritten next push
  item = _array[_top]
  _top -= 1
  return item

# linked list for stack
top = None # also head pointer updated in every op
def push(item):
  t = new Node()
  if not top:
    top = t
    return
  # push from front for O(1)
  t.next = top
  top = t

def pop():
  if not top:
    return None
  v = top.val
  top = top.next
  return v

# Queue will use two pointers to track, as it needs to do op from two ends
```

**Priority Queue**   
`heap` is one of efficient implementation for PQ, use an array to maintain partial order. One item dominates its children by having the smaller(min heap) than they do or bigger keys (max heap).   
```sh
          5
        /   \
       7    10   =>  5, 7, 10, 9, 15, 12
    /  \   /
  9   15  12
```
We use a slick array to represent tree structure without storing pointers in each node. The parent and children for item at position k have been determined
```sh
parent(k): return k/2
left(k):   return 2k
right(k):  return 2k+1  
```

`heap` is particular suitable for:
- The items in the queue frequently get updated (priority value), also inserted, deleted, still be able to retrieve min/max item efficiently


#### § Dictionary
It is an abstract data type that allow accessing item by given key (like associated array). Good dict implementation would achieve about O(1) retrieval by key, also aim to support efficient insert and delete.   

**Hash table**   
Hash function maps keys into large integers, mod by `m` which is number of slots of hash table. The result `int` is index where we store the actual data. When two keys are mapped into same integers, causing collision.   

*Collision by chaining*   
Use an array of **`m`** linked list, If keys are evenly distributed, we have **`m`** close to **`n`** (total number of items), list in each 'slot' contains just about 1 item. The search is O(1) to locate bucket, and linear search through linked list      
 **?** How to choose `m`, reasonably large prime number

*Collision by open addressing*   

☞ Problems nicely solved by hash:   
 - Substring pattern match (Rabin-Karp)   
 `n = |S|` length of original string,  `m = |p|` length of pattern string   
 `n - m + 1` windows hashing of s, 1 hashing of p, total *O(n-m+2)* hash computation. The next window hash H(S, j+1) can be based on previous one H(S, j)


 - Is a new document duplicate with a large corpus    
 - Is a new document plagiarized from a document in large corpus   

**Binary Search Tree (BST)**   
This potentially (depend on how balanced it is) allows fast search and efficient update. Unlike unsorted linked list only for fast update or sorted array only for quick binary search.  

 • In-order traversal
 ```py
 def inorder(node):
   if not node: return
   inorder(node.left)
   process(node)
   inorder(node.right)
 ```
 Change process order, would yield pre / post order traversal

 ⍰ How to make tree close to perfect balance `O(lgN)`


 ### § Sorting and Searching

For algorithm  analysis with Big(O), we could have following:

```O(n!) > 2^n > n^3 > n^2 > nlog(n) > n > log(n)```   
- n! and 2^n do not have practical use case in real.
- n^2 and n^3 remains usable when `n < 1000`.
- nlog(n) and n can be used for up to billions of items

**Sorting Consideration**
- How many items?  
If it's more than hundreds, consider `nlog(n)`, e.g heapsort, quicksort, mergesort
- Duplicate keys?  
Do we need stable sort, i.e same keys remain original position before sorted

- What do we know about Data   
partially sorted: `insertion sort` would be better   
the distribution of keys   
the range of keys, consider bucket sort when it is small range, e.g sort people by ages

- Do we need external sort?

 merge sort by divide and conquer  
 quick sort by randomization   

 heap sort via specific data structure   
 This is actually the selection sort on top of priority queue (implemented as heap). The data structure helps to extract min (find and delete) from the rest of items in log(n)

 distribution sort by bucketing

**Search Consideration**   

Binary search is important, and has some variances, e.g. in rotating sorted array, or sorted array with duplicates.

Binary search can be generalised beyond find the element in sorted array. As long as the search space is monotonic (increasing or decreasing in one direction). Or think in this way: we check element x in the search space against some predicate of the domain *f*:  
if f(x) == true, elements that are in the **`right`** side (could means bigger) of x are also true, f(x+1) == true ..  
if f(x) == false, elements that are in the **`left`** side (could means smaller) of x are also false, f(x-1) == false ..

Search space checked against some predicate looks like this:

false false false false(<-x1) true(x2->) true true true   



### § Recursion and Dynamic programming

**Bottome up**  
Solve base case (size 1 or 2), build up solution to get full result, e.g Fibonacci

**Top down**

**Half-Half**  
Binary search and merge sort, each recursion or iteration reduces problem size to half.  

**Backtracking**  
Backtracking is suitable for problems like enumerating all possibility. It naturally yields a traversal tree, each node is partial solution, the edge is constructed by adding new valid candidate (could be multiple candidates) from current node,  essentially work like `DFS` on an implicit graph.   
It ensures never visiting a state () twice, also guarantee all will be covered.


### § Bits
 The basic unit is to group 8 bits into 1 byte, all computation is based on one or more units.   
 `int` is normally represented by 4 bytes, the range is -2^31 to 2^31 - 1. **2's complement** used to represent positive and negative, which is calculated as following (assume 3 bits number):
 ```sh
 # 1's: flip all bits; 2's: add one
 001(1) -> 110 -> 111 (-1)
 010(2) -> 101 -> 110 (-2)
 011(3) -> 100 -> 101 (-3)
 ```
 **Common manipulation**  
 ```py
 # get the i-th (from right, least significant) bit of number n
 # see if this bit is set or not e.g, n=0110 i=3  0[1]10 & [1]00
 # NOTE: i=3 we need to do 2 left shifts
 n & (1 << i-1)

 # set the i-th bit (make it 1 if it's 0), other bit not changed
 n | (1 << i-1)

 # clear the i-th bit (make it 0), other bits not changed
 mask = ~(1 << i-1)
 n & mask

 # check if number is power of 2
 # if a number is 2^x, only one 1 set in left most,
 # n-1 make all the following 0 -> 1, flip the left most to 0
 (n & (n-1)) == 0

 # XOR to swap two variables a,b without temp
 # Number a xor with itself equals 0
 a ^= b
 b ^= a # b = a^b ^ b = a^ (b^b) = a, now b becomes a
 a ^= b # a = a^b ^ a = a^a ^ b = b

 # check every bit for a given Number c
 while c > 0:
   if c & 1 > 0:
     # do things
   c = c >> 1 # c /= 2
 ```


### System design
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
