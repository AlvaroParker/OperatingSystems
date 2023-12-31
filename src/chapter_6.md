# Scheduling - Proportional Share
Based around simple concept: **Instead of optimizing for [[4. Scheduling#Turnaround time|turnaround]] or [[4. Scheduling#Response time|response]] time, the scheduler tries to guarantee that each job obtain a certain percentage of CPU time.**
## Lottery scheduling
Every so often, hold a lottery to determine which process should get to run next; processes that should run more often should be given more changes to win the lottery. 
### Tickets represent your share
Tickets are used to represent the share of a resource that a process should receive. i.e. If we have two processes A and B, where a has 75 tickets and B has 25 tickets, that would mean that A should receive 75% of the CPU and B 25% of the CPU. 
The longer these two jobs compete, the more likely they are to achieve the desired percentages.

### Ticket mechanisms 
#### Ticket currency
It allow user to allocate tickets among their own jobs in whatever currency hey would like; the system then automatically converts said currency into the correct global value

**Example:** User A has two jobs, A1 and A2; User B has 1 job, B1. The operating systems gives user A and user B 100 tickets each and user A gives A1 500 tickets and A2 500 tickets while user B gives 1000 tickets to B1.
```
User A -> 500  (A's currency) to A1 -> 50  (global currency)
       -> 500  (A's currency) to A2 -> 50  (global currency)
User B -> 1000 (B's currency) to B1 -> 100 (global currency)
```

#### Ticket transfer
- A process can temporarily hand off its tickets to another process
#### Ticket inflation
- A process can temporarily raise or lower the number of tickest it owns. 
- Only valid on scenarios where processes trust one another. 
- If any one process knows it needs more CPU time, it can boost its ticket value as a way to reflect that need to the system, without needing to communicate it to the other processes.
### How to assign tickets
We could assume that the users know best; in such case, each user is handed some number of tickets, and a user can allocate tickets to nay job they run as desired. 
-> That's a non solution since we are basically passing the problem to the user.

## Stride scheduling
Deterministic fair-share scheduler. 
- Stride: Number that's inverse in proportion to the number of tickets a process has.
- Each job in the system has a stride. 
- The more tickets, the lower stride. 
- Each process has an initial pass value. 
- Pick the process to run that has the lowest pass value so far
- When you ran a process, increment its pass counter by its stride.

A simple pseudocode: 
```C
curr = remove_min(queue);   // pick client with min pass
schedule(curr);             // run for quantum
curr->pass += curr->stride; // update pass using stride
insert(queue, curr);        // return curr to queue
```

With lower stride value, you will run the process more times. 
- A stride scheduling cycle is when all pass value are equal. 
- At the end of each cycle, each process will have run in the same proportion to their ticket values. 

### Main problem with stride scheduling
- If a new job enters in the middle of our stride scheduling, what should its pass value be? Should it be set to 0? This will monopolize the CPU...
- With [[6. Scheduling - Proportional Share#Lottery scheduling|Lottery Scheduling]] this doesn't happen, if a new process arrives, we just increase the total number of tickets and we go on to the next cycle. 

## Linux Completely Fair Scheduler (CFS)
Goal: To fairly divide a CPU evenly among all competing processes.
To achieve this goal, it uses a simple counting-based technique know as virtual runtime (`vruntime`)
- As each process runs, it accumulates `vruntime`
- Most basic case: Each process's `vruntime` increases at the same rate, in proportion with real time. 
- Scheduling decision: CSF will pick the process with the lowest `vruntime`
- How to know when to switch? 
	- CSF switches too often: Fairness increases, costs performance. 
	- CSF switches less often: Near-term fairness, performance increases 
	- `sched_latency` is the value CFS uses to determine how long one process should run before considering a switch. CFS divides this value by the number of (*n*) processes running on the CPU to determine the time slice for a process, ensuring that over this period of time, CFS will be completely fair. 
- How it works: 
	- We have *n* = 4 processes running, CFS divides the value of `sched_latency` (i.e. 48 ms) by *n*  to arrive arive at a per-process time slice of 12 ms. 
	- CSF schedules the first job and runs it until it has used 12 ms of (virtual) runtime.
	- Checks to see if there is a job with lower `vruntime` to run instead
	- In this case there is, CFS switches to any of the other 3 jobs 
	- Repeat. 
