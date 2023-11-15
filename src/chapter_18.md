# Thread API
Here we present the API the OS provides for thread creation and control

## Thread creation
```C
#include <pthread.h>

int pthread_create(pthread_t *thread, 
	const pthread_attr_t *attr, 
	void *(*start_routine)(void *), 
	void *arg
);
```
Arguments of this function: 
1. `thread`: A pointer to a structure of type `pthread_t`, this struct is used to interact with threads
2. `aatr`: Used to specify any attributes this thread might have (Stack size, scheduling priority, etc).
3. `start_routine`: Function the thread should start running. This is a pointer to a function with return type `void *`. 
4. `arg` is the argument to be passed to the function where the thread begins execution. 

## Thread completion
What if we want to wait for a thread to complete? You must call the routine `pthread_join()`
```C
pthread_join(pthread_t th, void **thread_return);
```
This routine takes two arguments: 
1. `th`: Of type `pthread_t` indicates which thread to wait for. 
2. `thread_return`: It's a pointer to a `void *` pointer, used to store the return value of the function that the thread is executing. If we don't care about the return value, we can use `NULL`

You should never return a pointer to a value allocated in the stack of a thread. When the thread ends its execution, it is destroyed alongside his stack, hence the value that the pointer in pointing to is lost and we get UB.

## Locks
Mutual exclusion to a critical section via locks. The most basic pair of routines to use for this purpose is provided by the following: 
```C
pthread_mutex_lock(pthread_mutex_t *mutex);
pthread_mutex_unlock(pthread_mutex_t *mutex);
```
These can be used to create a "container" around our critical block of code: 
```C
pthread_mutex_t lock;
...
pthread_mutex_lock(&lock);
x = x + 1; // Or whatever your critical section is
pthread_mutex_unlock(&lock);
```
This works as follows:
- If no other thread holds the lock when `pthread_mutex_lock(&lock);` is called, then the thread will acquire the lock. 
- If other thread holds the lock, `pthread_mutex_lock(&lock);` will "wait" for the lock to become available. 
- The thread makes the critical section operation. 
- The thread releases the lock using `pthread_mutex_unlock(&lock);`

This code snippet is badly written (for simplicity) since: 
- The lock is poorly initialized, and it should be initialized either using `PTHREAD_MUTEX_INITIALIZER` at compile time or `pthread_mutex_init()` at runtime. 
- The code doesn't check error codes when calling lock and unlock.

## Condition Variables
Use when a thread has to wait for other thread on some condition state. The two primary routines look like this: 
```C
pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
pthread_cond_signal(pthread_cond_t *cond);
```
To use a condition variable, we need a lock that is associated with this condition. When calling either of the above routines, the lock should be held.
