## Data structure
Fundamentally (physically), all data structure could be categorised as two: **Contiguous** vs **Linked**  
Contiguous: array, heaps, hash table  
 • O(1) index based retrieve    
 • Space efficiency, only data   

 **?** How to do dynamic array analysis, amortised guarantee

Linked: linked list, tree, graph (chunks of memory bound by pointer)  
 • Simpler insertion and deletion  
 • O(n) to retrieve

Stack and Queue can be implemented with either array or linked list.

#### § Dictionary
It is an abstract data type to allow access data via given key, also efficiently insert and delete.   

**Hash table**   
Hash function maps keys into large integers, mod by 'm' which is number of slots of hash table. The result int is index where we store the actual data.
When two keys are mapped into same integers, causing collision.  
One way to address is using an array of 'm' linked list, each linked list called bucket. If keys are evenly distributed, we have 'm' close to n, each 'slot' contains just about 1 item.   
Another is open addressing   
Problems nicely solved by hash:   
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




 **Sorting and Searching**   
 merge sort by divide and conquer  
 quick sort by randomization   
 heap sort via specific data structure   
 distribution sort by bucketing
