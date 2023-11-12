# 21. Condition variables
- Threads might want to check a condition is true before continuing its execution. 
- A parent thread might wish to check whether a child thread has completed before continuing 
- We would want to put the parent to sleep until the condition is met. 

## Definition and Routines
- **Condition variable**: An explicit queue that threads can put themselves on when some state of execution (i.e. some condition) is not as desired (by waiting on the condition)
- When some other threads changes said state, it can then wake one (or more) of those waiting threads and thus allow them to continue. 
- To use condition variables we have `wait()` and `signal()`:
	- `wait()` is used by threads to put themselves to sleep. 
	- `signal()` is executed when a thread has changed something in the program and thus wants to wake a sleeping thread waiting this condition


```C
int done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c = PTHREAD_COND_INITIALIZER;

void thr_exit() {
	Pthread_mutex_lock(&m);
	done = 1;
	Pthread_cond_signal(&c);
	Pthread_mutex_unlock(&m);
}

void *child(void *arg) {
	printf("child\n");
	thr_exit();
	return NULL;
}

void thr_join() {
	Pthread_mutex_lock(&m);
	while (done == 0) 
		Pthread_cond_wait(&c, &m);
	Pthread_mutex_unlock(&m);
}

int main(int argc, char *argv[]) {
	printf("parent: begin\n");
	pthread_t p;
	Pthread_create(&p, NULL, child, NULL);
	thr_join();
	printf("parent: end\n");
	return 0;
}
```

**In this code we have two cases to consider, the first:**
- Parent create child thread
- Parent continues execution and jumps to `thr_join`
- It acquires the lock and check if `done` is 1
- Since `done` is not 1 yet, it calls `Pthread_cond_wait` which takes the condition and a lock
- `Pthread_cond_wait` releases the lock and puts the thread to sleep
- The child thread runs and calls `thr_exit`
- `thr_exit` acquires the lock and sets done to 1 and sends a signal to threads that are watching for `c`
- The parent threads wakes up with the lock held and checks if `done` is 1
- `done` is 1 so we jump to `Pthread_mutex_unlock` to release the lock 

**On the seconds options we have:**
- Parent creates thread
- Child runs executes the code and set `done` to 1 
- The parent calls `thr_join` and since `done` is 1, it skips the `Pthread_cond_wait` part and directly calls `Pthread_mutex_unlock` and continues execution. 

## The Producer/Consumer (Bounded Buffer) Problem
- We have one or more producer threads and one or more consumer threads. 
- Producers generate data and place them in a buffer
- Consumers grab the items from the buffer and consume them in some way
- The bounded buffer is a shared resource, we rquire sync access to it. 
Example: 
```C
void *producer(void *arg) {
	int i; 
	int loops = (int) arg;
	for (i = 0; i)
}
```