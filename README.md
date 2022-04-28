# xv6 Patch 2 Report: Scheduling
**Multi-Level Feedback Queue and Proportional Scheduling**

## Methodology
---
>The purpose of this patch is to implement a different scheduling scheme into xv6. The decided design of this scheduling system is to add a multi level feedback queue with a lottery system. This means that features are taken from both types of scheduling methods. 

**Multi-Level Feedback Queue:**

First, a quick description of what the scheduler does. The scheduler determine when a process gets to use the CPU and gives the process a time-slice to run on. When the process traps or the time-slice is over and triggers a trap then the schedular gets to run on the CPU and take the next process of the queue. An overview of multilevel feedback queue:
- there are multiple queues where each queue represents a different priority level
- a job that uses an entire time-slice gets a lower priority
- a job that gives up CPU time gets a higher priority (higher queue)
- after a certain amount of time that job is moved to the highest priority.

For proportional share scheduling, which also is implemented, alongside MLFQ features is optimized for each process receiving a fair share of the CPU. More specifically proportional share scheduling as an early implementation is called lottery scheduling. The key with lottery scheduling is to have user processes use alloted tickets to determine their access to the CPU when the ticket is randomly selected. The randomly selected winner's ticket value is used to loop through a list and check the ticket value.

This experiment involves building known system calls followed by adding queues to the existing scheduler function and then add random selection of a process's ticket to determine it's access to the CPU.

The structure of this MLFQ scheduler implementation is as follows:
![MLFQ](
  https://github.com/ztbochanski/operating-systems-reference/raw/main/images/mlfq.png)

>Note: randomness is introduced in both the high priority and low priority queue to determine the winner. This is proportional to the count of processes in the queue using `total count` of processes. This behavior mimicks a priority queue data structure. 
## Required Changes to xv6
---
To accomplish upgrading xv6's simple scheduler most of the business logic is changed in `proc.c` in the `scheduler`.

1. Right off the bat 4 variables must be added to the process's state to keep track of:
```c
// Per-process state
struct proc {
  ...
  int priority;
  int tickets;
  int high_tics;
  int low_tics;
  ...
```
>high `tics` and low `tics` are for time costs

2. When a new `p struct` is created in the process table with `alloc` initialize all the required feilds:
```c
// initialize struct p process state for scheduler
p->tickets = 1;
p->priority = 1;
p->high_tics = 0;
p->low_tics = 0;
```

3. Now, inside the infinite for loop `for(;;)`, the first task is check for `lowest priority` processes that need to get moved to the first priority.
4. Next, move all jobs in the system to top priority after some time S. This is done with an arbitrary `time S` of `1000`. 
```c
if (total_tics % 1000 == 0 )
  // increase priority
```
5. Next, for each process get a random value to be the winning ticket for both the `low priority` end of the queue and the `high priority` end of the queue.
   >The main idea here is that we are iterating through the process table and when we get a winner that `p` process struct is saved in the scope of the local function and loaded into the CPU.
6. Select highest priority node (process in table) and load into cpu struct.
  ```c
  c->proc = p;
  ```

## Supplemental changes to xv6
---
To peer under the hood of this implementation we need to look into into what's going on with the processes and how they are scheduled.

To accomplish this **two** system calls are implemented that take a look into each process in the process table and some of their important properties including `pid`, `status`, `priority`, and `tickets`.

The two system calls are `getpinfo` which returns process info and `settickets` which sets a ticket value according to a process according to it's `pid`

### Usage

**Get process info**
`getpinfo` usage in the shell uses the command `ps` (process status)
```bash
$ ps
```

**Set tickets for process**
`settickets` usage in the shell uses the command `st` (set tickets)
arg1: `int pid`
arg2: `int ticket_number`
```bash
$ st pid ticket_number
```
### Methods for adding supplemental requirements 

System call implementation:
```c
sys_getpinfo(void)
...
sys_setticket(void)
...
```

Implementation for the rest of the method signatures as follows the normal pattern of adding a system call to xv6.
**syscall.h**
```c
#define SYS_settickets 22
#define SYS_getpinfo 23
```

**syscall.c**
```c
int sys_settickets(void);
int sys_getpinfo(void);
...
[SYS_settickets]  sys_settickets,
[SYS_getpinfo]   sys_getpinfo
```

**user.h**
Add method signatures to user space 
```c
int settickets(int pid, int tickets);
int getpinfo(void);
```

## Random number generation
---
>This method is used for the proportional lottery section of how to stratify or choose what process to select and run.

For generating random numbers a script published [here](https://github.com/joonlim/xv6/blob/master/random.c)
```c
random(void)
{
  // http://stackoverflow.com/questions/1167253/implementation-of-rand
  ...
   b  = ((z1 << 6) ^ z1) >> 13;
  z1 = ((z1 & 4294967294U) << 18) ^ b;
  b  = ((z2 << 2) ^ z2) >> 27; 
  z2 = ((z2 & 4294967288U) << 2) ^ b;
  b  = ((z3 << 13) ^ z3) >> 21;
  z3 = ((z3 & 4294967280U) << 7) ^ b;
  b  = ((z4 << 3) ^ z4) >> 12;
  z4 = ((z4 & 4294967168U) << 13) ^ b;
  ...
}
```
## Testing
---

The implementation is very difficult to test because precise functionality of the scheduler may not have been made in the most optimal way `(space/time complexity)`. 
>The goal with this implementation was to follow the 5 rules of MLFQ in the most efficient way possible. 

The outcome of this scheduler design is very procedural. Further testing will need to be created to test for optimality and better time complexity.

Incremental changes to the xv6 code are critical to the "time is money" mantra. With more time `unit tests` can be done for each of the `6` procedural steps noted above in `required xv6 upgrades`

For the purpose of this scheduler, the primary focus is on `system tests` to verify the usability of the scheduler.

SYSTEM TESTS:

- [x] Verify `getpinfo` functionality
- [x] Verify `settickets` functionality
- [x] Verify CLI `st` to set tickets
- [x] Verify CLI `ps` to print process info
- [x] Test scheduler is working
- [x] Test scheduler can change tickets

Figure 1. Tickets manually changed

 ![settickes]()

Pass✅

Figure 2. Process priority level increased and tickets count reset.

 ![priority]()

Pass✅


## Summary
---
This project involved implementation of a scheduler in xv6. The scheduler planned for was a multilevel feedback queue where processes were given a priority based on some rules. For example if a process used it's entire alloted time slice it would be reduced to the lowest queue; or the opposite could happen where a process can get promoted if it does not use the whole time slice. Processes that have been in the bottom queue for a set amount of time are automatically given the highest priority. To determine order within the queue the a process is selected proportionally by probability through a random number generator. The patch generated demonstrates the code changes required for the implementation of these requirements.