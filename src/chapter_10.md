# 10. Free-Space Management
- It becomes more difficult when the free space you are managing consist of variable sized units
- Usually arises in user-level memory-allocation libraries (i.e. `malloc()` and `free()`)
- Arises in the OS when using segmentation to implement virtual memory
- Problem known as external fragmentation: The free space gets chopped into little pieces of different sizes and is thus fragmented; request may fail because there is no contiguos space of memory. 

Example: 
In this image we see that the total available space is 20 bytes, however if a process request 15 bytes it will fails since there's not contiguos space of memory that adds up to 15 bytes. 

<center><img src="./images/externFrag.png"></center>

## Assumptions
We have a basic allocation library that has the following functions: 
	- `void *malloc(size_t size)` where `size` is the number of bytes request by the application, it hands back a point (void pointer) to a region with the requested size. 
	- `void free(void *ptr)` takes a single pointer and frees the corresponding chunk that the pointer is pointing to. 
2. The space this library manages is known as the heap, the generic data structured used to manage free space in the heap is some kind of free list
3. Free list containers references to all of the free chunks of space in the managed region of memory. 
4. We are primarily concerned with external fragmentation
5. When a virtual memory is handed out to a client, it cannot be relocated to another location in memory. The region where the pointer is pointing to wont be relocated. 
6. The allocator manages a contiguous region of bytes. For this case in particular we assume that the region is a single fixed size throughout its life. 
## Low level Mechanisms 
### Splitting and Coalescing
We have the following 30 bytes heap: 

![[./images/bytesheap.png]]

Assuming we have the following free list: 

```
head -> {addr: 0, len: 10} -> {addr: 20, len: 10} -> NULL
```

**Splitting:** If we have a request for a single bytes, the allocator performs this action (Splitting). It finds a free chunk of memory that can satisfy the request and split it in two, the first part will be returned to the caller, the second chunk will remain on the list.
For example if we choose the second element on the free list, we will end up with this free list after the allocation: 

```
head -> {addr: 0, len: 10} -> {addr: 21, len: 9} -> NULL
```

Here we return the address `20` to the caller, and the start of the second element would be then `21` instead of `20`
Splitting is usually used when the requested sized is smaller than the size of any particular free chunk. 

**Coalesce** Free space when a chunk of memory is freed. 
Imagine we free the memory that's on the middle of our heap, we would then have the following free list: 

```
head -> {addr: 10, len: 10} -> {addr: 0, len: 10} -> {addr: 20, len: 9} -> NULL
```
And when a user requests, for example, 20 bytes, we won't be able to find a 20 bytes chunk even tho the first and second chunk are neighbors and could be used as a 20 bytes chunk. 
To fix this problem we use **coalesce**: When freeing a chunk of memory, check if the newly freed space sits right next to one existing free chunk, if it does, merge them into a single large chunk. 
### Tracking size of allocated regions
- To track the size of allocated regions, we add a header to the top of the request memory, for example if we request 20 bytes `malloc(20)`

![[./images/headermalloc.png]]

- The header struct may look like this: 
```C
typedef struct {
	int size;
	int magic;
} header_t;
```
- Here magic can be used to detect memory corruptions
- `size` is used to save the `size` of the allocated region
- When we free this space, the total size would be `size` + the size of the header
## Strategies for managing free space
### Best fit
- Search for memory chunks on the free list that can hold the requested size.
- Select the smallest from the resulting chunks that can hold the requested size. 
- Literally the best fit. 
- Full search list is required, hence bad performance
### Worst fit
- The oposite to best fit
- Scan the free list and search for all possible chunks that can hold the requested size
- Choose the biggest chunk
- Full search list is required, hence bad performance
This strategies tries to keep big chunks free instead of lots of small chunks (which is what the best fit strategies does)
### First fit 
- Search for the first memory chunk that can hold the request size
- Better speed as it stops as soon as it finds a chunk that can hold the requested size
- Pollutes the beginning of the free list with small chunks, this can be mitigated by using address-based ordering list
### Next fit
- Instead of starting at the beginning of the list like **first fit** does, we keep an extra pointer that holds the location we where last looking. 
- From that location, we start the first fit strategies 
- Performance similar to first fit
- We avoid the pollution at the beginning of the list
