## Concurrent programming in Pharo
@chaconcprog

Pharo is a sequential language since at one point in time there is only one computation carried on. However, it has the ability to run programs concurrently by interleaving their executions. The idea behind Pharo is to propose a complete OS and as such a Pharo run-time offers the possibility to execute different processes in Pharo lingua (or green threads in other languages) that are scheduled by a process scheduler defined within the language.

Pharo's concurrency is priority-based _preemptive_ and _collaborative_. It is _preemptive_ because a process with higher priority interrupts \(preempts\) processes of lower priority. It is _collaborative_ because the current process should explicitly release the control to give a chance to the other processes of the same priority to get executed by the scheduler.

In this chapter, we present how processes are created and their lifetime. 
We will show how the process scheduler manages the system. 

In a subsequent chapter, we will present the semaphores in detail and revisit scheduler principles
then later we will present other abstractions such as Mutex, Monitor, and Delay.


### Studying an example

Pharo supports the concurrent execution of multiple programs using independent processes \(green threads\). 
These processes are lightweight processes as they share a common memory space. 
Such processes are instances of the class `Process`. 
Note that in operating systems, processes have their own memory and communicate via pipes supporting strong isolation. 
In Pharo, a process is what is usually called a (green) thread or fiber in other languages.
They have their own execution flow but share the same memory space and use concurrent abstractions such as semaphores to synchronize with each other.

### A simple example

Let us start with a simple example.
We will explain all the details in subsequent sections.
The following code creates two processes using the message `fork` sent to a block.
In each process, we enumerate numbers: the first process from 1 to 10 and the second one from 101 to 110.
During each loop step, using the expression `Processor yield`, the current process mentions the scheduler that it can relinquish the CPU to give a chance to other processes with the same priority to get executed.
We say that the active process relinquishes its execution.

```
[ 1 to: 10 do: [ :i |
	i trace; trace: ' '.
	Processor yield ] ] fork.

[ 101 to: 110 do: [ :i |
	i trace; trace: ' '.
	Processor yield ] ] fork
```


The output is the following: 

```
1 101 2 102 3 103 4 104 5 105 6 106 7 107 8 108 9 109 10 110 
```

We see that the two processes run concurrently, each outputting a number at a time and not producing two numbers in a row.
We also see that a process has to explicitly give back the execution control to the scheduler using the expression `Processor yield`. 
We will explain this in more detail in the following.
Let us look at what a process is.

### Process

In  Pharo, a process (green thread) is an object like anything else.
A process is an instance of the class `Process`. 
Pharo follows the Smalltalk naming and from a terminology point of view, this class should be called a `Thread` as in other languages.
It may change in the future.

A process is characterized by three pieces of information: 
- A process has a priority (between 10 lowest and 80 highest). Using this priority, a process will preempt other processes having lower priority and it will be managed by the process scheduler in the group of processes with the same priority as shown in Figure *@SchedulerSimple@*.
- when suspended, a process has a suspendedContext which is a stack reification of the moment of the suspension.
- when runnable, a process refers to one of the scheduler priority lists corresponding to the process' priority. Such a list is also called the run queue to which the process belongs.



### Process lifetime

A process can be in different states depending on its lifetime (**runnable**, **suspended**, **executing**, **waiting**, **terminated**) as shown in Figure *@processStates@*. We look at such states now.

![Process states:  A process (green thread) can be in one of the following states: **runnable**, **suspended**, **executing**, **waiting**, **terminated**.](figures/processStates.pdf width=75&label=processStates)

We define the states in which a process can be:
- **executing** - the process is currently executing.
- **runnable** - the process is scheduled. This process is in one of the priority lists of the scheduler.
- **terminated** - the process has run and finished its execution. It is not managed anymore by the scheduler. It cannot be executed anymore.
- **suspended** - the process is not managed by the scheduler: This process is not in one of the scheduler lists or in a semaphore list. The process can become runnable by sending it the `resume` message.
- **waiting** - the process is waiting on a semaphore waiting list. It is not managed by the scheduler. The process can become runnable when the semaphore releases it.


We will use systematically this vocabulary in the rest of the book.

### Creating processes

Let us now write some code snippets. 

#### Creating a process without scheduling it

We create a process by sending the message `newProcess` to a block defining the computation that should be carried in such process.
This process is not immediately scheduled, it is **suspended**. 
Then later on, we can schedule the process by sending it the message `resume`: the process will become **runnable**.

The following creates a process in a `suspended` state, it is not added to the list of the scheduled processes of the process scheduler.

```
| pr |
pr := [ 1 to: 10 do: [ :i | i traceCr ] ] newProcess.
pr inspect
```


To be executed, this process should be scheduled and added to the list of suspended processes managed by the process scheduler.
This is simply done by sending it the message `resume`.

In the inspector opened by the previous expression, you can execute `self resume` and then the process will be scheduled.
It means that it will be added to the priority list corresponding to the process priority of the process scheduler and that the process scheduler will eventually schedule it.

```
self resume
```


Note that by default the priority of a process created using the message `newProcess` is the active priority: the priority of the active process.

#### Passing arguments to a process

You can also pass arguments to a process with the message `newProcessWith: anArray` as follows:

```
| pr |
pr := [ :max | 
			1 to: max do: [ :i | i crTrace ] ] newProcessWith: #(20).
pr resume
```


The arguments are passed to the corresponding block parameters. It means that in the snippet above, `max` will be bound to `20`. 

#### Suspending and terminating a process

A process can also be temporarily suspended \(i.e., stopped from executing\) using the message `suspend`. 
A suspended processed can be rescheduled using the message `resume` that we saw previously.
We can also terminate a process using the message `terminate`. 
A terminated process cannot be scheduled anymore.
The process scheduler terminates the process once its execution is done. 

```
| pr |
pr := [ :max |
			1 to: max do: [ :i | i crTrace ] ] newProcessWith: #(20).
pr resume.
pr isTerminated
>>> true
```


#### Creating and scheduling in one go

We can also create and schedule a process using a helper method named: `fork`.
It is basically sending the `newProcess` message and a `resume` message to the created process.


```
[ 1 to: 10 do: [ :i | i trace ] ] fork
```


This expression creates an instance of the class `Process` whose priority is the one of the calling process (the active process).
The created process is `runnable`.
It will be executed when the process scheduler schedules it as the current running process and gives it the flow of control.
At this moment the block of this process will be executed.

Here is the definition of the method `fork`.

```
BlockClosure >> fork
	"Create and schedule a Process running the code in the receiver."

	^ self newProcess resume
```



#### Creating a waiting process

As you see in Figure *@processStates@* a process can be in a waiting state.
It means that the process is blocked waiting for a change to happen (usually waiting for a semaphore to be signaled).
This happens when you need to synchronize concurrent processes.
The basic synchronization mechanism is a semaphore and we will cover this deeply in subsequent chapters.

### First look at ProcessorScheduler

Pharo implements time sharing where each process (green thread) has access to the physical processor during a given amount of time. 
This is the responsibility of the `ProcessorScheduler` and its unique instance `Processor` to schedule processes. 

The scheduler maintains _priority lists_, also called _run queues_, of pending processes as well as the currently active process \(See Figure *@SchedulerSimple@*\).  
To get the running process, you can execute: `Processor activeProcess`.
Each time a process is created and scheduled it is added at the end of the run queue corresponding to its priority.
The scheduler will take the first process and executes it until a process of higher priority interrupts it or the process gives back control to the processor using the message `yield`.


![The scheduler knows the currently active process as well as the lists of runnable processes based on their priority.](figures/SchedulerSimple.pdf width=70&label=SchedulerSimple)

### Process priorities


At any time only one process is executing. First of all, the processes are being run according to their priority. This priority can be given to a process with the `priority:` message, or `forkAt:` message sent to a block. By default, the priority of a newly created process is the one of the active process. There are a couple of priorities predefined and can be accessed by sending specific messages to `Processor`. 


For example, the following snippet is run at the same priority as background user tasks.

```
[ 1 to: 10 do: [ :i | i trace ] ] 
	forkAt: Processor userBackgroundPriority
```



The scheduler has process priorities from 10 to 80.  Only some of these are named. 
The programmer is free to use any priority within that range that they see fit.
The following table lists all the predefined priorities together with their numerical value and purpose.



| Priority | Name or selector |  |
| --- | --- | --- |
| 80 | timingPriority |  |
|  | For processes that are dependent on real time. |
|  | For example, Delays \(see later\). |
| 70 | highIOPriority |  |
|  | The priority at which the most time critical input/output |
|  | processes should run. An example is the process handling input from a |
|  | network. |
| 60 | lowIOPriority |  |
|  | The priority at which most input/output processes should run. |
|  | Examples are the process handling input from the user \(keyboard, |
|  | pointing device, etc.\) and the process distributing input from a network. |
| 50 | userInterruptPriority |  |
|  | For user processes desiring immediate service. |
|  | Processes run at this level will preempt the ui process and should, |
|  | therefore, not consume the Processor forever. |
| 40 | userSchedulingPriority |  |
|  | For processes governing normal user interaction. |
|  | The priority at which the ui process runs. |
| 30 | userBackgroundPriority |
|  | For user background processes. |
| 20 | systemBackgroundPriority |
|  | For system background processes. |
|  | Examples are an optimizing compiler or status checker. |
| 10 | lowestPriority |
|  | The lowest possible priority. |


% We generated the table using the following expression
% [[[
% (Processor class organization listAtCategoryNamed: #'priority names' )
% 	collect: [ :each | { each . Processor perform: each}  ] 
% >>> #(#(#lowestPriority 10) #(#timingPriority 80) #(#lowIOPriority 60) #(#highIOPriority 70) 
% #(#systemBackgroundPriority 20) #(#userBackgroundPriority 30) 
% #(#userInterruptPriority 50) #(#userSchedulingPriority 40))
% ]]]

Here is an example showing that how to use such named priorities. 
```
[3 timesRepeat: [3 trace. ' ' trace ]] forkAt: Processor userBackgroundPriority.
[3 timesRepeat: [2 trace. ' ' trace ]] forkAt: Processor userBackgroundPriority + 1.
```


### ProcessScheduler rules


The scheduler knows the currently active process as well as the lists of pending processes based on their priority.
It maintains an array of linked lists per priority as shown in Figure *@SchedulerSimple@*.
It uses the priority lists to manage processes that are runnable in the first-in-first-out way.


There are simple rules that manage process scheduling. We will refine the rules a bit later:
- Processes with higher priority preempt \(interrupt\) lower priority processes if they have to be executed.
- Assuming an ideal world where processes could execute in one shot, processes with the same priority are executed in the same order they were added to the scheduled process list. (See below for a better explanation).



Here is an example showing that process execution is ordered based on priority. 
```
[3 timesRepeat: [3 trace. ' ' trace ]] forkAt: 12.
[3 timesRepeat: [2 trace. ' ' trace ]] forkAt: 13.
[3 timesRepeat: [1 trace. ' ' trace ]] forkAt: 14.
```


The execution outputs: 

```
1 1 1 2 2 2 3 3 3
```


It shows that the process of priority 14 is executed prior to the one of priority 13. 


### Let us trace better what is happening


Let us define a little trace that will show the current process executing code.
It will help to understand what is happening. 
Now when we execute again the snippet above but slightly modified:

```
	| trace |
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) crTrace ].
	[3 timesRepeat: [ trace value: 3 ]] forkAt: 12.
	[3 timesRepeat: [ trace value: 2 ]] forkAt: 13.
	[3 timesRepeat: [ trace value: 1 ]] forkAt: 14.
```


We get the following output, which displays the priority of the executing process. 

```
@14 1
@14 1
@14 1
@13 2
@13 2
@13 2
@12 3
@12 3
@12 3
```



### Yielding the computation

Now we should see how a process relinquishes its execution and lets other processes of the same priority perform their tasks.
Let us start with a small example based on the previous example. 

```
	| trace |
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) crTrace ].
	[3 timesRepeat: [ trace value: 3. Processor yield ]] forkAt: 12.
	[3 timesRepeat: [ trace value: 2. Processor yield ]] forkAt: 13.
	[3 timesRepeat: [ trace value: 1. Processor yield ]] forkAt: 14.
```


Here the result is the same. 

```
@14 1
@14 1
@14 1
@13 2
@13 2
@13 2
@12 3
@12 3
@12 3
```


What you should see is that the message `yield` was sent, but the scheduler rescheduled the process of the highest priority that 
did not finish its execution. This example shows that yielding a process will never allow a process of lower priority to run.

#### Between processes of the same priority

Now we can study what is happening between processes of the same priority. 
We create two processes of the same priority that perform a loop displaying numbers.

```
| p1 p2 |
p1 := [ 1 to: 10 do: [:i| i trace. ' ' trace ] ] fork.
p2 := [ 11 to: 20 do: [:i| i trace. ' ' trace ] ] fork.
```


We obtain the following output:
```
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20
```


This is normal since the processes have the same priority.
They are scheduled and executed one after the other.
`p1` executes and displays its output. 
Then it terminates and `p2` gets the control and executes. 
It displays its output and gets terminated.

During the execution of one of the processes, nothing forces it to relinquish computation.
Therefore it executes until it finishes. 
It means that if a process has an endless loop it will not release the execution 
except if it is preempted by a process of higher priority (see Chapter scheduler's principle).


#### Using yield


We modify the example to introduce an explicit return of control to the process scheduler.

```
| p1 p2 |
p1 := [ 1 to: 10 do: [:i| i trace. ' ' trace. Processor yield ] ] fork.
p2 := [ 11 to: 20 do: [:i| i trace. ' ' trace. Processor yield ] ] fork.
```


We obtain the following trace showing that each process gave back the control to the scheduler after each loop step.
```
1 11 2 12 3 13 4 14 5 15 6 16 7 17 8 18 9 19 10 20 
```


We will come back to yield in future chapters.

#### Summary


Let us revisit what we learned in this chapter.

- Processes with the same priority are executed in the same order they were added to the scheduled process list. In fact, processes within the same priority should collaborate to share the execution amongst themselves. In addition, we should pay attention since a process can be preempted by a process of higher priority, the semantics of the preemption \(i.e., how the preempted process is rescheduled\) has an impact on the process execution order. We will discuss this in-depth in the following chapters.
- Processes should explicitly give back the computation to give a chance to other pending processes of the same priority to execute. The same remark as above works here too.%  Imagine a long process not yielding its execution, this process may be interrupted by a process of higher priority, and depending on the semantics of the preemption this process may not be the one that will continue to be executed.
- A process should use `Processor yield` to give an opportunity to run to the other processes with the same priority. In this case, the yielding process is moved to the end of the list to give a chance to execute all the pending processes \(see below Scheduler's principles\).


### Important API


The process creation API is composed of messages sent to blocks.

- `[ ] newProcess` creates a suspended \(unscheduled\) process whose code is the receiver bloc. The priority is one of the active process.
- `[ ] newProcessWith: anArray`  same as above but pass arguments (defined by an array) to the block.
- `priority:` defines the priority of a process.
- `[ ] fork` creates a newly scheduled process having the same priority as the process that spawns it. It receives a `resume` message so it is added to the queue corresponding to its priority. 
- `[ ] forkAt:` same as above but with the specification of the priority. 
- `ProcessorScheduler yield` releases the execution from the current process and give a chance to processes of the same priority to execute.




### Conclusion

We presented the notion of process \(green thread\) and process scheduler. 
We presented briefly the concurrency model of Pharo: preemptive and collaborative. A process of higher priority can stop the execution of processes of lower ones. 
Processes at the same priority should explicitly return control using the `yield` message.

In the next chapter, we explain semaphores since we will explain how the scheduler uses delays to perform its scheduling. 
