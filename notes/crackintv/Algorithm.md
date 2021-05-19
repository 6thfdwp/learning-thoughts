
### Schedule
```sh
✓ Stack     DP, recursive
✓ Hash      BFS, DFS
✓ Tree      BST
- Heap      Bit
- Trie

System design / Interview experience
```

## Data structure
Fundamentally (physically), all data structure could be categorised as two:

### Contiguous vs Linked  
**Contiguous**: array, heaps, hash table  
 - O(1) index based retrieval    
 - Space efficiency, only data   
   ⍰ For dynamic array. The amortised cost for appending is O(1), only a few items require O(N) to move old items to resized new array

**Linked**: linked list, tree, graph (different chunks of memory bound by pointer)  
 - Efficient insertion and deletion  
 - O(n) to retrieve   


**Do iteration**  
Most algorithms operating on contiguous space (array, string etc) involve some common iteration techniques:

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
  i += 1
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
Keep two pointers (indices) is common, e.g,
- Remove duplicates in place (sorted is easier, one pass)  
  One for iteration i, one for current non-dup item's index j, so j+1 would be next placement for the another non-dup
- 2sum in **Ordered** array  
  Find 2 indices that adds up to given number. From two end, each time either increase left or decrease right

For the string problems, need to clarify before solving, here are a few common ones to consider:   
- Is it case sensitive,
- How should we deal with special chars, `space`, `tab` etc
- Is the string ASCII (256 char set) or unicode (2 bytes),

### Stack & Queue
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
  if _top == -1:
    return None
  item = _array[_top]
  _top -= 1
  return item

def peek():
  if _top == -1:
    return None
  return _array[_top]


# linked list for stack
# head pointer that's updated in every op
# all we care is just about the top node that support O(1) insert and remove
top = None
def push(item):
  t = new Node()
  if not top:
    top = t
    return
  # insert from front for O(1)
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
The stack is useful when doing backtrack in the recursion. You put the temporary result in the stack, when current level of recursion finishes or fails (not meet the requirement), pop one item from stack, go back to previous level. So every level of call stack will have right state associated.

Another use of stack is to change recursive procedure to iteration. e.g BST in (pre, post) order traversal. Stack can used to store the nodes that need to be processed later. Like in-order traversal, we push nodes until reach to left-most leaf node, and pop one to continue.

Queue is most commonly used in BFS, For each node, keep adding adjacent items to the queue

### Priority Queue
`heap` is one of efficient implementation for PQ, use an array to maintain partial order. One item dominates its children by having the smaller key (min heap) than itself or bigger key (max heap).   
```sh
          5
        /   \
       7    10   =>  5, 7, 10, 9, 15, 12
    /  \   /
   9   15  12
```
We use a slick array to represent tree structure without storing pointers in each node. The parent and children for item at position k can be determined
```sh
parent(k): return k/2
left(k):   return 2k
right(k):  return 2k+1  
```
Heap can also be implemented with binary tree, but take more space as need to store two pointers.

The common ops in `heap`:
- insert: O(logN)  
  Insert to the right most position to keep tree complete, bubble up to the right position

- extract min/max: O(logN)  
  Return the root node value (or the first el), rebuild the heap to maintain the property (swap the last leaf node to the top, bubble down to right position)

- retrieve min/max: O(1)

As `heap` maintaining the priority (either descending or ascending) efficiently, it is used as core data structure in BFS for shortest path  


### § Dictionary
It is an abstract data type that allow accessing item by given key (like associated array). Good dict implementation would achieve about O(1) retrieval by key, also aim to support efficient insert and delete.   

**Hash table**   
Hash function maps keys into large integers, mod by `m` which is number of slots of hash table. The result integer is index where we store the actual data. When two keys are mapped into same integers, causing collision.   

*Collision by chaining*   
Use an array of **`m`** linked lists, If keys are evenly distributed, we have **`m`** close to **`n`** (total number of items), list in each 'slot' contains just a few items. The search is O(1) to locate bucket, and linear search through linked list      
 **?** How to choose `m`, reasonably large prime number

*Collision by open addressing*   

☞ Problems nicely solved by hash:   
 - Substring pattern match (Rabin-Karp)   
 `n = |S|` length of original string,  `m = |p|` length of pattern string   
 `n - m + 1` windows hashing of S, 1 hashing of p, total *O(n-m+2)* hash computation. The next window hash H(S, j+1) can be based on previous one H(S, j)

 - Is a new document duplicate with a large corpus    
 - Is a new document plagiarized from a document in large corpus   

### § Tree & Graph
Tree can be seen as connected graph (every node can be reached) without cycle, as one node can only have one parent.

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
Change process order, would yield pre / post order traversal. We can see different traversals are just using DFS in left and right subtree of a node interleaved with processing current node.

When doing recursion, the leaf node can be seen as subtree without left and right parts. When it tries to go deeper, the procedure directly returns doing nothing. So this is where we reach the recursion base case, start going back.

 ⍰ How to make tree close to balance `O(logN)`

**Graph traversal**
- DFS  
  It uses recursion or (stack) to traverse. Typical use case is topological sort to check if it is valid DAG, which is commonly used to represent the tasks depending on each other or some prerequisites. The search tree generated from DFS shows the correct order needed to finish tasks. If there is `back edge` means there is cycle, hence invalid DAG.

- BFS  
  It uses queue to control the visiting order for graph nodes, also a visited `set` to efficiently check if a node has been explored (all its connections have been checked)

  BFS on unweighted graph from s -> d yields the shortest path (the minimal hops) between s and d

  Time: `O(V + E)`, V = number of nodes, since every node and edge will be explored in worst. O(1) < |E| < O(V^2) depends on how dense the graph is. V + E means whichever dominates. If it is very dense, E dominates, hence Time: O(V^2)

- Dijkstra non-negative weight   
  It is an enhanced BFS. Each node put in the Q will be associated with some value, (could be price, cost, time, traffic volume etc). The processing order for Q is based on the current min / max value. These value also gets updated (relaxed) if node is still in the Q (explored but not check its neighbours yet) when there is another optimal path found which yields better than current cost.

  So need an efficient structure to constantly extract min/max node. When the node's value gets updated, it will also re-organise for next extraction.

  Depends on structure we used to maintain a list of nodes that will be extracted. It has different complexity.

### § Sorting and Searching

For algorithm  analysis with Big(O), we have the following formula:

```O(n!) > 2^n > n^3 > n^2 > nlog(n) > n > log(n)```   
- n! and 2^n do not have practical use case in real.
- n^2 and n^3 remains usable when `n < 1000`.
- nlog(n) and n can be used for up to billions of items

**Sorting Consideration**
- How many items?  
If it's more than hundreds, consider `nlog(n)`, e.g heapsort, quicksort, mergesort
- Duplicate keys?  
Do we need stable sort, i.e same keys remain original relative position before sorted

- What do we know about Data   
partially sorted: `insertion sort` would be better   
the distribution of keys   
the range of keys, consider bucket sort when it is small range, e.g sort people by ages

- Do we need external sort?

Short comparisons for a few sorting algorithms:

- merge sort by divide and conquer

  Divide is recursive procedure to continue halving the array until only one item,  
  Conquer is to walk back the recursion, merge two parts in each level

- quick sort by randomization   
  Randomly choose one item, find its right position, so its left ones are all smaller and right ones are all bigger

- heap sort via specific data structure

  This is actually the selection sort on top of priority queue (implemented as heap). The data structure helps to extract min (find and delete) from the rest of items in log(N)

distribution sort by bucketing

**Search Consideration**   

Binary search is important, and has some variances, e.g. in rotating sorted array, or sorted array with duplicates.

Binary search can be generalised beyond find the element in sorted array. As long as the search space is monotonic (increasing or decreasing in one direction). Or think in this way: we check element x in the search space against some predicate of the domain *f*:  
if f(x2) == true, elements that are in the **`right`** side (could means bigger) of x2 are also true, f(x+1) == true ..  
if f(x1) == false, elements that are in the **`left`** side (could means smaller) of x1 are also false, f(x-1) == false ..

Search space checked against some predicate looks like this:

`false false false false(<-x1) true(x2->) true true true`   
With generalised approach, some problems can also be solved by BS, like find closest square root number for given x: `floor(sqrt(x))`

### § Recursion and Dynamic programming

**Bottome up**  
Solve base case (size 1 or 2), build up solution to get full result, e.g Fibonacci. Iterative Fibonacci is one example.

**Top down**   
[DP explained](www.quora.com/Are-there-any-good-resources-or-tutorials-for-dynamic-programming-DP-besides-the-TopCoder-tutorial/answer/Michal-Danilák)  
When using DP to come up with the sub-problems patterns, we start thinking with top-down fashion. Normally it is denoted as recurrent formula.  
A few steps to follow:
- Identify the sub-problem   
  It is easy to evaluate those small sized problem to get sub-optimal (local or current) result
- Clearly express it as a recurrent formula, F(i) = F(i-1) + F(i-2)  
  We also decide the necessary parameters (minimal state space), this also used to determine the complexity as it indicates number of calls that can be made with different parameter combinations.
- Identify the overlapping  
  add memoization (cache result to directly use when given same input parameters). Mostly array (1 or 2 dimensions depends on number of parameters) to store partial result  
  Some problems may not have clear recurrent pattern and there is no overlapping. They can be reasoned from small sized problem to quickly get local result. These result can be used when growing the problem size until the global optimal value

This usually converts an exponential alg to square or cubic complexity. e.g give the problem size N and recurrent formula:   
`F(s, e) = max(F(s+1, e)+year * A[s], F(s, e-1)+year * A[e])`  
Each state could have 2 possible branches (two recursive calls)   
Then without cache it is O(2^N). But we only have 2 variables, each can be N, total combination is bound to N^2. So the complexity should be bound to O(N^2) without repeatedly recursion.

Both bottom up and top down needs memoization to cache previous result from sub-problem. Bottom up is similar to recursive thinking, but implement in reverse. Need the proper cached result anyway.

**Half-Half**  
Binary search and merge sort, each recursion or iteration reduces problem size to half.  

**Backtracking**  
Backtracking is suitable for problems like enumerating all possibility. It naturally yields a traversal tree, each node is partial solution, the edge is constructed by adding new valid candidate (could be multiple candidates) to current node,  essentially work like `DFS` on an implicit graph.   
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

**Memory limits**   

Bit vector can be used when the input size cannot fit in memory entirely. If using bool array or int array, it will take more space (4 bytes or so) to indicate each number's presence. (e.g int A[1000] = 1 A[1005] = 0). Need to be able to calculate the rough vector size based on available memory.

Normally 4 bytes int would have `2^31` positive integers (up to billions)  

`2^10`:  1024,  ( ~10^3 thousand),  1 Kb, x bytes if every item is represented as 1b  
`2^16`:  65536,  64K  
`2^20`:  1048576 (~10^6 million)    1 Mb,   
`2^30`:          (~10^9 billion)  1 Gb  
`2^32`  ~4 billions (the typical long int range)

If we have 1Gb memory, 8 billion bits, enough to represent all integers   
If we only have 10Mb, cannot use one vector to indicate all ints. The idea is to first identify which range the missing int falls into   

We divide the ints into blocks (stored as int [], each block means one entry in the array, assume each takes 4 bytes), array size (number of blocks) can be `2^23 (10M)/4 = 2^21` entries at most (should be less, otherwise no space for bit vector),     
Assume each entry represents number range
- entries = 2^31 / number_range <= 2^21
- 2^10 <= number_range <= 2^26   
  upper bound is max bits we can fit in 10M = 2^23 * 2^3 = 2^26 bits (need each bit indicates one int) once we know which range the missing number falls into

 ```py
 # if choose 2^20 as number range (the number of ints represented by each entry)
 # we have 2^11 entries (2 * 4 Kb for int [])
 # 0 to (1 million-1) increment number at 0 block
 # 1 million to (2 million-1) increment number at 1 block
 # 2 million to (3 million-1) increment number at 1 block
 def find():
   size = 2^31 / range_size
   blocks = [0] * size # number of entries in int []
   for each number in input:
     blocks[number/range_size] += 1

   for each block:
     if blocks[i] < range_size:
       # If one block misses int, the counter < range_size,
       # If no missing int, each block[i] == 1 million
       # other blocks with duplicates, the counter > range_size   
       start_range = i * range_size
       end_range = start_range + range_size

     # another pass of loop
     # only allocate range_size bit vector for that block
     # 2^20 / 2^3: size of bv, 2^17 bytes ~= 128k
     # with the previous int [], total memory S(128 Kb + 8 Kb)
     bvsize = [0] * (range_size >> 3)
     bv = bytearray(bvsize)
     for each value in input:
       if value >= start_range and value <= end_range:
         v = value - start_range
         idx, offset = v / 8, v % 8
         bv[idx] |= 1 << offset

    # finally check bv, return number corresponding to any bit 0
 ```
