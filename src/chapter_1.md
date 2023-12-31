# 1. The process
*Definition: It is a running program*

## Crux of the problem: How to provide the illusion of many CPUs?
***
OS creates illusion of many CPUs by virtualizing the CPU.
- Running one process, stopping it and running another, and so forth -> Promotes the illusion that many virtual CPUs exists when in fact there is only one. 
- To implement this, the OS needs: 
	- Mechanisms (Low level machinery): Low level methods or protocols that implement a needed piece of functionality.
	- Policies (intelligence): Algorithms for making some kind of decisions within the OS (i.e. the scheduler is an algorithms that makes the decision of which process to run next, don't worry if you don't know yet what an scheduler is, will get to that later).

*Formal definition: Abstraction provided by the OS of a running program* 

At any given type, we can summarize a process by taking an inventory of the different pieces of the system it accesses or affects during the course of its execution
## Components of a process: 
1. Memory (a.k.a address space): Instructions, data that the programs read and writes sits in memory.
2. Registers; Many instruction read or update registers.
3. Program counter (a.k.a instruction pointer): Tell us which instruction of the program will execute next.
4. Persistent storage devices: I/O information might include list of files the process currently has open. 
## Process API
Usually a process interface of an operating systems includes the following:
* Create: Some method to create a new process. 
* Destroy: Interface to destroy processes forcefully.
* Wait: Waiting interface, to wait for a process to stop running.
* Miscellaneous control: Other control that are possible (i.e methods to suspend process, and then resume it).
* Status: Interface to get status information of process. 

## Process creation
1. OS reads executable bytes of executable file from disks and place them in memory somewhere.
*On simpler OS's the loading process is done eagerly (all at once), in most advanced OS's, this is done lazily (i.e by loading pieces of code or data only as they are needed during program execution)*
2. OS allocates memory for the program's run-time stack, and it initializes the stack with arguments. 
3. OS may also allocate some memory for the program's heap. The OS may get involved and allocate more memory to the process to help satisfy heap memory calls (i.e `malloc` on C)
4. OS makes I/O initialization tasks. i.e on Unix like OS, file decriptors 0, 1 and 2 gets assigned to sderr, stdio, stdin. 
5. Last task for the OS: Start the program running at the entry point, namely `main()`, transferring control of the CPU to the newly-created process. 

## Process states
In a simplified view, a process can be in one of three states:
* Running: Process is executing instructions. 
* Ready: The process is ready to run, but the OS has chosen not to start it.
* Blocked: Process has performed some kind of operation that makes it not ready to run until some other event takes place. 

## Data structures
To keep track of the state of each process, the OS keeps some kind of **process list** for all processes that are ready and some additional information to track which process is currently running. 

<center><img src="./images/ProcessStruct.png"></center>

In the image we can see the different states a process can be, and what information does the `xv6` kernel keeps about a process.

