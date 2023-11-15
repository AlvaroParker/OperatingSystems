# Lock-based Concurrent Data Structures
Adding locks to data structures to make them usable by threads makes the structure thread safe. 
## Crux: How to add locks to data structures
How should we add locks to data structures, in order to make it work correctly, high performance, many threads at once? i.e. concurrently. 

## Concurrent Counters
We have the following counter: 

```C
typedef struct __counter_t {
	int value;
	pthread_mutex_t lock;
} counter_t;

void init(counter_t *c) {
	c->value = 0;
	Pthread_mutex_init(&c->lock, NULL);
}

void increment(counter_t *c) {
	Pthread_mutex_lock(&c->lock);
	c->value++;
	Pthread_mutex_unlock(&c->lock);
}

void decrement(counter_t *c) {
	Pthread_mutex_lock(&c->lock);
	c->value--;
	Pthread_mutex_unlock(&c->lock);
}

int get(counter_t *c) {
	Pthread_mutex_lock(&c->lock);
	int rc = c->value;
	Pthread_mutex_unlock(&c->lock);
	return rc;
}
```

- This structures has a single lock, which is acquired when we write and the released when returning from  the write call. 
- This code has performance costs:
	- Benchmarking this code, shows us that from a 0.03 seconds that it takes to run on a single threads, it jumps to nearly 5 seconds on 2 threads.
### Scalable counting
Definition: When see the threads complete just as quickly on multiple processors as the single threads does on one.

The most famous technique to attack this problem is called **approximate counter**:
- Multiple threads on multiple CPU's 
- Each threads has a local counter, and there a single global counter. 
- When a thread running on a given core wishes to incremente the counter, it increments its local counter
- Access to this local pointer is sync via a lock
- To keep the global counter up to date, local values are periodically transferred to the global counter. 
- How often this local-to-global transfer occurs is given by a threshold *S*:
	- The small *S* is, the more the counter behaves like the non-scalable counter above
	- The bigger *S*, the more scalable but less precise 
	 - *S* is local to the CPU, meaning that for each CPU there's a timer that when it reaches *S*, it transfers his counter to the Global counter. 

## Concurrent Linked List
Check the code on the book, page 362. 

- The code acquires a lock on insertion and releases it once finished. 
-  The code acquires a lock for lookup, since we don't want our list to change when we are searching for an element inside of it. 
### Scaling Linked Lists
- Instead of having a single lock for the entire list, we have a lock per node. 
- This is called hand-over-hand locking
- When traversing the list, the code first grabs the next node's lock and then releases the current node's lock
- This has performance issues since grabbing a lock for each node can be a time expensive task.

## Concurrent queues

Check the code on the book, page 365. 

- We have two locks, one for the tail and one for the head
- Thanks to this locks, we can perform concurrent enqueue and dequeue operations. 
- We can also add a dummy code to separate the head part of the queue, with the tail part. 

## Concurrent hash tables
Code at page 366

- Instead of having a lock for the entire structures we have a lock per hash bucket. 
- This enables many concurrent operations to take place. 
