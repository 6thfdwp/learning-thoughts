## Data structure
Fundamentally (physically), all data structure could be categorised as two: **Contiguous** vs **Linked**  
Contiguous: array, heaps, hash table  
 • O(1) index based retrieve    
 • Space efficiency, only data   

 **?** How to do dynamic array analysis amortised guarantee

Most algorithms operating on contiguous space (array, string etc) involve some
iteration techniques:
- forward (from first to last), backward
```py
for i in xrange(reversed(len(A))):
# or using while
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
- iterate two array at the same time, merge phase of merge sort
```py
# has general form, need to deal with different size of arrays
while i < len(A) or j < len(B):
  if i >= len(A):
    # only do B[j]
  if j >= len(B):
    # only do A[i]
  do(A[i], B[j])
```


Linked: linked list, tree, graph (chunks of memory bound by pointer)  
 • Simpler insertion and deletion  
 • O(n) to retrieve

**Stack** and **Queue** can be implemented with either array or linked list.

**Priority Queue**   
`heap` is one of efficient implementation for PQ, use an array to maintain partial order.   
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
 • Substring pattern match (Rabin-Karp)   
 `n = |S|` length of original string,  `m = |p|` length of pattern string   
 `n - m + 1` windows hashing of s, 1 hashing of p, total *O(n-m+2)* hash computation. The next window hash H(S, j+1) can be based on previous one H(S, j)


 • Is a new document duplicate with a large corpus    
 • Is a new document plagiarized from a document in large corpus   

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

 **?** How to make tree close to perfect balance `O(lgN)`


 ### § Sorting and Searching

 merge sort by divide and conquer  
 quick sort by randomization   
 heap sort via specific data structure   
 distribution sort by bucketing

**Search Consideration**   


Generalised binary search pattern

 ### § Recursion and Dynamic programming

**Bottome up**  
Solve base case (size 1 or 2), build up solution to get full result, e.g Fibonacci

**Top down**

**Half-Half**  
Binary search and merge sort, each recursion reduces problem size to half


**Backtracking**  
Backtracking is suitable for problems like enumerating all possibility. It naturally yields a traversal tree, each node is partial solution, the edge is constructed by adding new valid candidate (could be multiple candidates) from current node,  essentially work like `DFS` on an implicit graph.

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
   # do things
   c & 1 > 0
   c = c >> 1 # c /= 2
 ```
