# 17. Concurrency - An Introduction
## Threads characteristics
- Thread process abstraction: Instead of a process having a single point of execution (single PC), a multi-threaded program has more than one point of execution.
- Threads within the same process share the same address space. They can access the same data. 
- It has a program counter (PC), that track where the program is fetching instructions from. 
- Each thread has his own private registers used for computations. When we switch between T1 to T2, a context switch happens to store the registers of T1 to memory and restore the ones of T2 from memory. 
- To store the state of each thread in a process, we use one or more thread control blocks (TCBs)
- The main difference between a context switch on threads to the one in process is that in threads the address space remains the same.
- Each threads has his own stack, but they share the same heap. 
- Having many stacks, limits how much they can grow, this isn't a problem since stacks do not generally have to be very large. 

## Why use threads
There are two major reasons: 
1. Parallelism: On a multi CPU environment, using a thread per CPU to portions of a task can reduce the time it takes to finish the task. 
2. Avoid blocking on I/O: While one thread waits for an I/O operation to finish, another one can use the CPU to make other work. Hence you avoid blocking your process when doing I/O tasks. 
Why not use multiple processes instead of threads? Because threads share the same address space and thus it's easier to share data. 
## Example thread creation
```C
// main.c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void *mythread(void *arg) {
  printf("%s\n", (char *)arg);
  return NULL;
}

int main(int argc, char *argv[]) {
  pthread_t p1, p2;
  int rc;
  printf("main: begin\n");
  pthread_create(&p1, NULL, mythread, "A");
  pthread_create(&p2, NULL, mythread, "B");

  // join waits for the threads to finish
  pthread_join(p1, NULL);
  pthread_join(p2, NULL);
  printf("main: end\n");
  return 0;
}

// To compile run: gcc -pthread main.c -o main
```

In this program, we can't know which thread will get to run first. A might be printed before B, or B before A. It's up to the thread scheduler which thread to run first. 
## Shared data
We have the following C program: 
```C
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

static volatile int counter = 0;

void *mythread(void *arg) {
  printf("%s: begin\n", (char *)arg);
  int i;
  for (i = 0; i < 1e7; i++) {
    counter = counter + 1;
  }
  printf("%s: done\n", (char *)arg);
  return NULL;
}

int main(int argc, char *argv[]) {
  pthread_t p1, p2;
  int rc;
  printf("main: begin\n");
  pthread_create(&p1, NULL, mythread, "A");
  pthread_create(&p2, NULL, mythread, "B");

  // join waits for the threads to finish
  pthread_join(p2, NULL);
  pthread_join(p1, NULL);
  printf("main: done with both (counter = %d)\n", counter);
  return 0;
}
// To compile run: gcc -pthread main.c
```

Here we have two thread wishing to update the same global variable `counter`. Each worker (thread created) is trying to add a number to the shared variable `counter` 10 million times (1e7) in a loop. Thus, since we have two thread and the initial value is 0, we expect the result to be 20.000.000. 

We compile and run our program: 
```bash
$ gcc -pthread main.c -o main
$ ./main
main: begin
A: begin
B: begin
A: done
B: done
main: done with both (counter = 10250346)
```
We clearly didn't got the desired result. Let's try again: 
```bash
$ ./main
main: begin
A: begin
B: begin
B: done
A: done
main: done with both (counter = 11403750)
```
Each time we run the program, not only we got the wrong result, but we got a different result. 

## Uncontrolled Scheduling 
To understand these weird error, we need to understand the code sequence that the compiler generates for the update to `counter`. In assembly the instructions look something like this: 
```S
mov    0x8049a1c,%eax
add    $0x1,%eax
mov    %eax,0x8049a1c
```
*Note: To see the assembly code of your program, you can run `objdump -d main `*

Here the variable `counter` is located at address `0x8049a1c`. 
In this 3 instruction assembly we have that:
1. The `mov` instruction, moves the value at address `0x8049a1c` (our counter value) to register `eax`
2. The `add` instruction, adds 1 to the content of the register `eax`
3. The content of `eax` is stored back into memory at the same address

Let's image the one of our 2 threads (Thread 1) enters this region of code. It load the value of counter (let's say the value is currently at 50) to the register `eax`, then it adds 1 to the register; thus `eax = 51`. Now Thread 1 is interrupted, the OS saves the state of T1 and now is T2 turn to run. Thread 2 loads the value of the counter (50 still since Thread 1 didn't write into memory) into `eax`, adds 1 (thus `eax = 51`) and saves that back to memory. Now Thread 1 runs again, and when restoring the registers, we have that `eax` is 51, now thread 1 saves `eax` into memory and `counter = 51` (again).

What happened? The code to increment `counter` has been run twice, but `counter`, which started at 50, is now only equal to 51. A correct version of this programs should have result int the variable `counter` equal to 52.

This is called a race condition, where the result depends on the timing of execution. When multiple thread executing a piece of code can result in a race condition, we call this piece of code a critical section. 

## The wish for Atomicity
One way of solving this, is having a single instruction that in a single step, does whatever we needed to do and thus removing the possibility of an interrupt on the middle of the task. Something like this:
```S
memory-add 0x8049a1c, $0x1
```

Atomically, in this context means as a unit, or "all or none". What we want to do, is execute the previous 3 instruction atomically:
```S
mov    0x8049a1c,%eax
add    $0x1,%eax
mov    %eax,0x8049a1c
```

Generally we don't have a single instruction to avoid data races, hence we need to implement **synchronization primitives**

## Crux: How to Support Synchronization

## One more problem: Waiting For Another
This problem happens when a thread must wait for another to complete some action before it continues. For example when a process performs an I/O operation and is put to sleep; when the I/O completes, the process need to be roused from its slumber so it can continue. What mechanisms we need to support this type of sleeping/waking interaction that is common in multi-threaded programs? 