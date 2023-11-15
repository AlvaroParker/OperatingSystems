# Locks 
We want to execute a series of instruction atomically, due to the presence of interrupts, we couldn't. To solve this we introduce lock, which we put around critical section and ensure this sections is executed as a single atomic instruction. 

## The basic idea
We have a critical section that looks like this:
```C
balance = balance + 1;
```
To use a lock, we add some code around the critical section like this:
```C
lock_t mutex; // Some globally-allocated lock `mutex`
...
lock(&mutex);
balance = balance + 1;
unlock(&mutex);
```
- Lock is just a variable 
- It stores the state of the lock at any given time
- It's either available or acquired
- We can store more data in the lock such as the current thread that's holding it, or a list of some kind for ordering lock acquisition

The general flow of locks are describe here: 
- Calling `lock()` tries to acquire the lock
- If no other thread holds the lock, the thread will acquire the lock and enter the critical section
- If another thread tries to acquire the lock, the thread will not return while the lock is held by another thread. This way, other threads are prevented from entering the critical section. .
- Once the owner of the lock calls `unlock()`, the lock is now available again to other threads. 
- If there are no other threads waiting for the lock, the state of the lock is set to free. 
- If there are waiting threads, one of them will notice and will acquire the lock and enter critical section.

With locks we make sure than only a single thread can access a critical section of code. 

## Pthread locks 
The name the POSIX library uses for a lock is `mutex` as it is used to provide mutual exclusion between threads. When you see the following threads code, assume that's doing the same thing as above: 
```C
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

Pthread_mutex_lock(&lock); // wrapper; exits on failure
balance = balance + 1;
Pthread_mutex_unlock(&lock);
```
Passing a variable to lock and unlock helps us avoid locking all threads with one lock (coarse-grained locking strategy) and doing a more specific thread lock (a more fine-grained approach) 

## Evaluating Locks 
To build locks we must defined some evaluation criteria:
- Does the lock provides mutual exclusion. Does the lock work, preventing multiple threads from accessing a critical section.
- Fairness. Does each thread contending the lock gets a fair shot at acquiring it once its free. 
- Performance. The time overheads added by using the lock. 
	- When a single thread grabs and releases the lock, what's the overhead of doing so
	- Is there a performance overhead when multiple threads are contending for a lock. 
	- How does the lock performs when there are multiple CPU's involved and threads on each contending lock. 

## Controlling interrupts
One solution to implement mutex was to disable interrupts for critical sections. The could would look like this: 
```C
void lock() {
	DisableInterrupts();
} 
void unlock() {
	EnableInterrupts();
}
```
Assuming that we are running on a single processor system, by turning off interrupts before entering a critical section, we ensure that the code inside this section won't be interrupted 

#### Benefits
- Simplicity: Easy to implement and to grasp. 
#### Negatives
- This allows any calling thread to perform a privileged operation, we must trust that this is not abused. 
- Greedy program calls lock and never `unlock` hence taking over the entire CPU
- Buggy or malicious program could call `lock` and enter a loop, OS never regains control and we must restart the entire system
- Does not work on multiple CPUs systems, if a thread disables interrupts, a thread running on a different CPU can still access critical section. 
- Running off interrupts for extended period of time can lead to interrupts becoming lost. 
- Inefficient: Code that mask and unmask interrupts are CPU inefficient. 

This negatives **might** be acceptable when running OS level programs, since the OS trust it self. 

## Just Using Loads/Stores
Block using single flag variable (`flag`) to indicate whether some thread has possession of a lock. 
- The first thread that enters the critical section calls `lock()` which check if the `flag` is set to 1 (in this case, is not), and then sets the flag to 1 to indicate that the thread now holds the lock. 
- When the thread finishes with the critical section, it calls `unlock()` which clears the flag
- If another thread calls `lock()` while the lock is held, it will find that the flag is set to 1 so it will simply *spin-wait* in a while loop for that thread to call `unlock` and clear the flag
- Once that first thread does clear the flag, the waiting thread fall out of the while loop, sets the flag to 1 for itself and proceeds into the critical section .

```C
typedef struct __lock_t { int flag; } lock_t;

void init(lock_t *mutex) {
	// 0 -> lock is available, 1 -> held
	mutex->flag = 0;
}
void lock(lock_t *mutex) {
	while (mutex->flag == 1) // TEST the flag
		;
	mutex->flag = 1; // now SET it!
}
void unlock(lock_t *mutex) {
	mutex->flag = 0;
}
```

This implementation has two errors.
1. The first one is that we can easily produce a case where both threads set the flag to 1, and both threads are thus able to enter critical section. 
2. The second being a performance error, the `lock` routine does a **spin-waiting**, which wastes time waiting for another thread to release a lock but at the same time running on the CPU. On a single CPU system, how can a thread be waiting for a lock while using the CPU if the other threads needs to use it to free the lock? Doesn't makes sense. 
## Building working spin locks with test and set 
System designers started to invent hardware support for locking. The test-and-set instruction is one implementation of this hardware support. 
In this example we see test-and-set in practice via C code: 
```C
int TestAndSet(int *old_ptr, int new) {
	int old = *old_ptr; // Fetch old value at old ptr;
	*old_ptr = new; // Store 'new' into old_ptr
	return old; // return the old value
}

typedef struct __lock_t {
	int flag;
} lock_t;

void init(lock_t *lock) {
	// 0: lock is available, 1: lock is held
	lock->flag = 0;
}

void lock(lock_t *lock) {
	while (TestAndSet(&lock->flag, 1) == 1)
		; // spin-wait (do nothing)
}

void unlock(lock_t *lock) {
	lock->flag = 0;
}
```

First case, a thread calls `lock`:
- No other threads currently holds the lock (thus `flag` is 0).
- Thread calls `TestAndSet(flag, 1)`, which returns the old value (0)
- The thread breaks the loop since the returned value is 0
- The thread also atomically set the value of `flag` to 1 thus indicating that the thread is now held 
- We the threads finishes execution of the critical section, `unlock` is called. 
The second case is:
- Other thread already has the lock held (`flag` is 1)
- Another thread calls `TestAndSet(flag, 1)`
- `TestAndSet` returns the old values which is 1, while simultaneously setting it to 1 again 
- As long as other threads holds the lock, 1 will be returned and thus this thread will spin and spin until the lock is finally released. 

This `TestAndSet` is actually an atomtic instruction implemented on the hardware level, hence it can't be interrupted. Hence we ensure that only one thread acquire the lock. 

## Evaluating Spin Locks
Given the previous spin lock we can evaluate it. 
- Correctness: Does it provide mutual exclusion? Yes, it only allows a single thread to enter critical section at a time. 
- Fairness. Spin locks doesn't provide any fairness guarantees. A thread might spin forever, under contention and wont execute the critical section. 
- Performance: We analyze this on a single process and a multi processor system:
	- Single processor: Performance is bad. If a thread holding the lock is preempted within the critical section, the scheduler might run every other thread, each of them runs for a slice of time. A waste of CPU cycles. 
	- On multiple CPU's. Performance is reasonably well. Thread A hold a lock in CPU 1, thread B spin in CPU2, spinning  to wait the lock on another processor doesn't waste many cycles in this case. 

## Compare-And-Swap
Hardware primitive that some systems provide. The C pseudocode looks like this: 
```C
int CompareAndSwap(int *ptr, int expected, int new) {
	int original = *ptr;
	if (original == expected)
		*ptr = new;
	return original;
}
```
And the lock instruction: 
```C
void lock(lock_t *lock) {
	while (CompareAndSwap(&lock->flag, 0, 1) == 1)
		; // spin
}
```
This tests if the value that `ptr` is pointing to, is the `expected`. If so, update the value that `ptr` is pointing to, to `new`. Finally return the `original` value. 
## Load-Linked and Store-Conditional
Hardware instruction, the C pseudocode looks like this: 
```C
int LoadLinked(int *ptr) {
	return *ptr;
}
int StoreConditional(int *ptr, int value) {
	if (no update to *ptr since LoadLinked to this address) {
		*ptr = value;
		return 1; // success!
	} else {
		return 0;
	}
}

void lock(lock_t *lock) {
	while (1) {
		while (LoadLinked(&lock->flag) == 1)
			; // Spin until it's zero
		if (StoreConditional(&lock->flag, 1) == 1)
			return; // If set-it-to-1 was a success: all done
					// Otherwise: try all over again
	}
}

void unlock(lock_t *lock) {
	lock->flag = 0;
}
```

- Store-conditional: It only succeeds if no intervening store to the address has taken place. 
- `lock()`: A thread spins waiting for the flag to be set to 0
- Once so, thread tries to acquire the lock via the store-conditional
## Fetch-and-Add
Hardware primitive, atomically increments a value while returning the old value at a particular address. C pseudocode: 
```c
int FetchAndAdd(int *ptr) {
	int old = *ptr;
	*ptr = old + 1;
	return old;
}
typedef struct __lock_t {
	int ticket;
	int turn;
} lock_t;

void lock_init(lock_t *lock) {
	lock->ticket = 0;
	lock->turn = 0;
}

void lock(lock_t *lock) {
	int myturn = FetchAndAdd(&lock->ticket);
	while (lock->turn != myturn)
		; // spin 
}

void unlock(lock_t *lock) {
	lock->turn = lock->turn + 1;
}
```
- Ticket and turn value. 
- When a thread wishes to acquire a lock it does an atomic fetch-and-add on the ticket value, the value is considered the thread's turn (`myturn`). 
- The globally shared `lock->turn` is used to determine which thread's turn it is.
- When `myturn == turn` then the thread can run
- Unlock is done by adding 1 to the `lock->turn` value.

## Spin performance
We have 2 threads: 
- Thread 0 is in critical section and thus has a lock held, then it get interrumpted. 
- Thread 1 tries to acquire the lock, but finds it held, it start looping and wasting cpu time. 
- Hardware support can't fix this problem, OS support is needed. 

## Spin solution: Yield
When you are going to spin, instead just give up the CPU to another thread. This can be represented in C code: 
```C
void init() {
	flag = 0;
}
void lock() {
	while (TestAndSet(&flag, 1) == 1)
		yield(); // give up the cpu 
}
void unlock() {
	flag = 0;
}
```
Yield is a system call that moves the caller from the running state to the ready state. Thus promoting promotes another thread to running.
- Good enough we have few threads
- Bad when we start having a lot of threads, and we have to yield a lot of times, hence still wasting CPU time (still better that the no yield approach)
- The cost of context switch is present in this solution. 
- Starvation it's still present in this solution

## Using Queues
A thread has either to spin waiting for the lock or yield the CPU, either way there's waste and no prevention of starvation. 
This can be improved with OS support in terms of two calls: 
- `park()` to put calling thread to sleep
- `unpark(threadID)` to wake a particular thread as designated by `threadID`
- This can be used to put the caller to sleep with it tries to acquire a held lock and wakes it when the lock is free 
C code representation of this: 
```C
typedef struct __lock_t {
  int flag; // If flag is set to 0 we can run the thread, else we add it to tue queue and park it
  int guard; // This is to control access of flag and queue
  queue_t *queue;
} lock_t;

void lock_init(lock_t *m) {
  m->flag = 0;
  m->guard = 0;
  queue_init(m->queue);
}

void lock(lock_t *m) {
  while (TestAndSet(&m->guard, 1) == 1) // We try to acquire read and write access to flag and queue
    ;  // acquire guard lock
  if (m->flag == 0) { // If flag = 0, we can run (held lock)
    m->flag = 1;
    m->guard = 0;
  } else { // else we add it to tue queue, park and continue
			// hold held by other thread
    queue_agdd(m->queue, gettid());
	setpark(); 
    m->guard = 0;
    park();
  }
}

void unlock(lock_t *m) {
  while (TestAndSet(&m->guard, 1) == 1)
    ;  // acquire guard lock
  if (queue_empty(m->queue)) {
    m->flag = 0;
  } else {
    unpark(queue_remove(m->queue));
  }
  m->guard = 0;
}
```
