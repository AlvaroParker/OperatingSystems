# 4. Scheduling
## Metrics

### Turnaround time: 
* Defined as `time of completion` - `time of arrival` 
* T = TCompletion - TArrival
### Response time
* Time of response = Time first run minus time of arrival. 
* T = TFirstRun - TArrival

## FIFO 
- First in first out: The first process to arrive gets executed first. 
- Problems: 
	- Given 3 processes: A, B and C (which arrives at that order). A runs for 100 seconds and B and C for 10 seconds each. 
	- Time of completion for: A = 100 secs, B = 110 and C = 120. Average 110 secs. 
	- **FIFO behaves poorly on processes of different lengths** 
## Shortest Job First (SJF)
* The shorts job runs first, then the next one and so on. 
* Average turn around time for process A, B and C decreases from 110 secs to 50 secs. 
* **Problems** 
	* If A arrives first and 10 seconds later B and C arrive, we will get similar turnaround time than in FIFO. 

## Shortest time to completion first (STCF)
* The shortest job to completion runs.
* If process A which takes 100 seconds it's running and a process B which takes 5 seconds arrives 10 seconds later A starts running, we switch to run process B because it will end before.
* Bad response time. 

## Round Robin 
- Runs a jobs for a time slice
- The time slice must be a multiple of the timer-interrupt period
- Better response time
- Shorter time slice makes RR perform better on the response time metric 
- Too short makes context switching slow down the systems
***

- Fair scheduling policies perform poorly on metrics like turn around time but good on metrics like response time. (SJF, STCF)
- Unfair scheduling policies perform poorly on response time and better on turn around time.  (RR)

## I/O
* If process A  has I/O operations: |AAAAA|I/O|AAAAA|
	* It spits the process in two, the I/O it's the delimiter
* When the I/O is in progress another process can run (i.e B)
