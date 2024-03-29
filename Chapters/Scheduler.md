## Scheduler's principles
@chaprinciples

In this chapter, we revisit the way to scheduler works and present some implementation aspects.
In particular, we show how yield is implemented. The Pharo scheduler is a cooperative, preemptive across priorities, non-preemptive within priorities scheduler.
But let us start with the class `Process`.

### Revisiting the class `Process`

A process has the following instance variables:
- priority: holds an integer to represent the priority level of the process.
- suspendedContext: holds the execution context (stack reification) at the moment of the suspension of the process.
- myList: the process scheduler list of processes to which the suspended process belongs to. This list is also called it run queue and it is only for suspended processes.


You can do the following tests to see the state of a process.

The first example opens an inspector in which you can see the state of the executed process. 

```
| pr |
pr := [ 1 to: 1000000 do: [ :i | i traceCr ] ] forkAt: 10.
pr inspect
```

It shows that while the process is executing the expression `self suspendingList` is not nil,
while that when the process terminates, its suspending list is nil.

The second example shows that the process suspendedContext is nil when a process is executing. 
```
Processor activeProcess suspendedContext isNil.
>>> true
```


Now a suspended process suspended context should not be nil, since it should have a stack of the suspended program.
```
([ 1 + 2 ] fork suspend ; suspendedContext) isNotNil
```


#### Implementation details.

The class `Process` is a subclass of the class `Link`.
A link is an element of a linked list \(class `LinkedList`\).
This design is to make sure that processes can be elements in a linked list without wrapping them in a `Link` instance.
Note that this _process_ linked list is tailored for the process scheduler logic.
This _process_ linked list is for internal usage. If you need a linked link, better uses another one if you need one.

#### States

We saw previously the different states a process can be in.
We also saw that semaphores suspend and resume suspended processes.
We revisit the different states of a process by looking at its interaction with the process scheduler and 
semaphores as shown in *@ProcessorStateSchedulerSemaphore@* : 


- **executing** - the process is currently executing.
- **runnable** - the process is scheduled. This process is in one of the priority lists of the scheduler. It may be turned into the executing state by the scheduler.
- **terminated** - the process ran and finished its execution. It is not managed anymore by the scheduler. It cannot be executed anymore.
- **suspended** - the process is not managed by the scheduler: This process is not in one of the scheduler lists or in a semaphore list. The process can become runnable sending it the `resume` message. This state is reached when the process received the message `suspend`.
- **waiting** - the process is waiting on a semaphore waiting list. It is not managed by the scheduler. The process can become runnable when the semaphore releases it.


![Revisiting process \(green thread\) lifecycle and states.](figures/StateSchedulerSemaphore2.pdf label=ProcessorStateSchedulerSemaphore)


### Looking at some core process primitives

It is worth looking at the way `Process` key methods are implemented.

The method `suspend` is a primitive and implemented at the VM level.
Since the process list (`myList`) refers to one of the scheduler priority lists in which it is, 
we see that the message `suspend` effectively remove the process from the scheduler list.

```
Process >> suspend
	"Stop the process that the receiver represents in such a way
	that it can be restarted at a later time (by sending the receiver the
	message resume). If the receiver represents the activeProcess, suspend it.
	Otherwise remove the receiver from the list of waiting processes.
	The return value of this method is the list the receiver was previously on (if any)."

	<primitive: 88>
	| oldList |
	myList ifNil: [ ^ nil ].
	oldList := myList.
	myList := nil.
	oldList remove: self ifAbsent: [ ].
	^ oldList
```


The `resume` method is defined as follows:
```
Process >> resume
	"Allow the process that the receiver represents to continue. Put 
	the receiver in line to become the activeProcess. Check for a nil 
	suspendedContext, which indicates a previously terminated Process that 
	would cause a vm crash if the resume attempt were permitted"

	suspendedContext ifNil: [ ^ self primitiveFailed ].
	^ self primitiveResume
```


```
Process >> primitiveResume
	"Allow the process that the receiver represents to continue. Put 
	the receiver in line to become the activeProcess. Fail if the receiver is 
	already waiting in a queue (in a Semaphore or ProcessScheduler)."

	<primitive: 87>
	self primitiveFailed
```


Looking at the virtual machine definition shows that the resumed process does not preempt processes having the same priority and that would be executing.

```
InterpreterPrimitives >> primitiveResume
	"Put this process on the scheduler's lists thus allowing it to proceed next time there is
	 a chance for processes of its priority level. It must go to the back of its run queue so
	 as not to preempt any already running processes at this level. If the process's priority
	 is higher than the current process, preempt the current process."
	| proc |
	proc := self stackTop.  "rcvr"
	(objectMemory isContext: (objectMemory fetchPointer: SuspendedContextIndex ofObject: proc)) ifFalse:
		[^self primitiveFail].
	self resume: proc preemptedYieldingIf: preemptionYields
```


Now we can have a look at the implementation of `newProcess`.
The method `newProcess` creates a process by reifying a new block as a stack representation.
The responsibility of this new block is to execute the receiver and terminates the process.

```
BlockClosure >> newProcess
	"Answer a Process running the code in the receiver. The process is not 
	scheduled."

	<primitive: 19>
	^ Process
		forContext:
			[ self value.
			Processor terminateActive ] asContext
		priority: Processor activePriority
```


### Priorities

A runnable process has a priority. 
It is always executed before a process of an inferior priority.
Remember the examples of previous chapters:

```
	| trace |
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) crTrace ].
	[3 timesRepeat: [ trace value: 3. Processor yield ]] forkAt: 12.
	[3 timesRepeat: [ trace value: 2. Processor yield ]] forkAt: 13.
	[3 timesRepeat: [ trace value: 1. Processor yield ]] forkAt: 14.
```


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


This code snippet shows that even if processes relinquish execution (via a message `yield`), the processes of lower priority are not scheduled before the process of higher priority got terminated.

!!note In the case of a higher priority level process preempting a process of lower priority, when the preempting process releases the control, the question is then what is the next process to resume: the interrupted one or another one? Currently in Pharo, the interrupted process is put at the end of the waiting queue, while an alternative is to resume the interrupted process to give it a chance to continue its task.

### signal and preemption


In the previous chapter, we presented this example:

```
	| trace semaphore p1 p2 |
	semaphore := Semaphore new.
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) crTrace ].
	p1 := [
	   trace value: 'Process 1a waits for signal on semaphore'. 
	   semaphore wait.
	   trace value: 'Process 1b received signal and terminates' ] forkAt: 30.
	p2 := [
	   trace value: 'Process 2a up to signaling semaphore'. 
	   semaphore signal.
	   trace value: 'Process 2b continues and terminates' ] forkAt: 20.
```


Here the higher priority process (`p1`) produces the trace and waits on the semaphore. 
`p2` is then executed: it produces a trace, then signals the semaphore. 
This signal reschedules `p1` and since it is of higher priority, it preempts \(`p2`\) and it terminates. 
Then `p2` terminates.

```
	@30 Process 1a waits for signal on semaphore
	@20 Process 2a up to signaling semaphore
	@30 Process 1b received signal and terminates
	@20 Process 2b continues and terminates
```


Now we add a second process of lower priority to understand what may happen on preemption.

```
	| trace semaphore p1 p2 p3 |
	semaphore := Semaphore new.
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) crTrace ].
	p1 := [
	   trace value: 'Process 1a waits for signal on semaphore'. 
	   semaphore wait.
	   trace value: 'Process 1b received signal and terminates' ] forkAt: 30.
	p2 := [
	   trace value: 'Process 2a up to signalling semaphore'. 
	   semaphore signal.
	   trace value: 'Process 2b continues and terminates' ] forkAt: 20.
   p3 := [
	   trace value: 'Process 3a works and terminates'. ] forkAt: 20.
```


Here is the produced trace. What is interesting to see is that `p2` is preempted by `p1` as soon as it is signaling the semaphore.
Then `p1` terminates and the scheduler does not schedule `p2` but `p3`. 

```
	@30 Process 1a waits for signal on semaphore
	@20 Process 2a up to signalling semaphore
	@30 Process 1b received signal and terminates
	@20 Process 3a works and terminates
	@20 Process 2b continues and terminates
```


This behavior can be surprising.
In fact the Pharo virtual machine offers two possibilities as we will show later.
In one, when a preempting process terminates, the preempted process is managed as if an implicit yield happened, moving the preempted process to the end of its run queue on preemption return and scheduling the following pending process.
In another one, when a preempting process terminates, the preempted process is the one that get scheduled \(it does not move at the end of the pending list\).
By default, Pharo uses the first semantics.


### Understanding `yield`

As we mentioned in the first chapter, Pharo's concurrency model is preemptive between processes of different priorities and collaborative among processes of the same priority.
We detail how the collaboration occurs: a process has to explicitly give back its execution. 
As we show in the previous chapter, it does it by sending the message `yield` to the scheduler. 

Now let us study the implementation of the method `yield` itself. 
It is really elegant. 
It creates a process whose execution will signal a semaphore and the current process will wait on such a semaphore until the created process is scheduled by the processor. Since 


```
ProcessScheduler >> yield
	"Give other Processes at the current priority a chance to run."

	| semaphore |
	semaphore := Semaphore new.
	[ semaphore signal ] fork.
	semaphore wait
```


Note that this implementation implies that 

The `yield` method does the following: 
1. The fork creates a new process. It adds it to the end of the active process's run queue (because fork creates a process whose priority is the same as the active process).
1. The message `wait` in `semaphore wait` removes the active process from its run queue and adds it to the semaphore list of waiting processes, so the active process is now not runnable anymore but waiting on the semaphore.
1. This allows the next process in the run queue to run, and eventually
1. allows the newly forked process to run, and
1. the signal in `semaphore signal` removes the process from the semaphore and adds it to the back of the run queue, so
1. all processes at the same priority level as the process that sent the message `yield` have run.



### `yield` illustrated


`yield` only facilitates other processes having the same priority getting a chance to run.
It doesn't put the current process to sleep, it just moves the process to the back of its priority run queue.
It gets to run again before any lower-priority process gets a chance to run. Yielding will never allow a lower-priority process to run.

Figure *@yield@* illustrates the execution of the two following processes yielding their computation.

```
P1 := [1 to: 10 do: [:i| i trace. ' ' trace. Processor yield ]] fork.
P2 := [11 to: 20 do: [:i| i trace. ' ' trace. Processor yield ]] fork.
```


Here is the output
```
1 11 2 12 3 13 4 14 5 15 6 16 7 17 8 18 9 19 10 20 
```


Here are the steps:
1. Processes P1 and P2 are scheduled and in the list \(run queue\) of the processor.
1. P1 becomes first active, it writes 1 and sends the message `yield`.
1. The execution of `yield` in P1 creates a Semaphore S1, a new process Py1 is added to the processor list after P2. P1 is added to S1's waiting list.
1. P2 is active, it writes 11 and sends the message `yield`.
1. The execution of `yield` in P2 creates a Semaphore S2, a new process Py2 is added to the processor list after Py1. P2 is added to S2's waiting list.
1. Py1 is active. S1 is signalled. Py1 finishes and is terminated.
1. P1 is scheduled. It moves from semaphore pending list to processor list after Py2. 
1. Py2 is active. S2 is signalled. Py2 finishes and is terminated.


![Sequences of actions caused by two processes yielding the control to the process scheduler.](figures/yield.pdf width=60&label=yield)


### Considering UI processes 

We saw that the message `signal` does not transfer execution unless the waiting process that received the signal has a higher priority.
It just makes the waiting process runnable, and the highest priority runnable process is the one that is run.
This respects the preemption semantics between processes of different priorities.

The following code snippet returns false since the forked process got the priority than the current process and the 
current process continued its execution until the end. 
Therefore the `yielded` did not get a chance to be modified. 

```
| yielded |
yielded := false.
[ yielded := true ] fork.
yielded
>>> false
```

Now let us imagine that would return true.
```
| yielded |
yielded := false.
[ yielded := true ] fork.
Processor yield.
yielded
>>> true
```


This expression returns true because `fork` creates a process with the same priority and the `Processor yield` expression
allows the forked process to execute. 


Now let us change the priority of the forked process to be lower than the active one (here the active one is the UI process).
The current process yields the computation but since the forked process is of lower priority, the current process will be executed before
the forked one.
```
| yielded |
yielded := false.
p := [ yielded := true ] forkAt: Processor activeProcess priority - 1.
Processor yield.
yielded
>>> false
```



The following illustrates this point using the UI process.
Indeed when you execute interactively a code snippet, the execution happens in the UI process \(also called UI thread\) 
with a priority of 40.

```
	| trace semaphore p1 p2 |
	semaphore := Semaphore new.
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) traceCr ].
	p1 := [
		trace value: 'Process 1a waits for signal on semaphore'. 
		semaphore wait.
		trace value: 'Process 1b received signal and terminates' ] forkAt: 30.
	p2 := [
		trace value: 'Process 2a signals semaphore'. 
		semaphore signal.
		trace value: 'Process 2b continues and terminates' ] forkAt: 20.
	trace value: 'Original process pre-yield'.
	Processor yield.
	trace value: 'Original process post-yield'. 
```


The following traces shows that `Processor yield` does not change the execution of 
higher-priority processes. Here the UI thread is executed prior to the other and yielding does not execute 
processes of lower priorities.

```
	@40 Original process pre-yield
	@40 Original process post-yield
	@30 Process 1a waits for signal on semaphore
	@20 Process 2a signals semaphore
	@30 Process 1b received signal and terminates
	@20 Process 2b continues and terminates
```



Now if we make the UI thread wait for small enough time (but long enough that the other processes get executed), then 
the other processes are run since the UI process is not runnable but waiting. 

```
	| trace semaphore p1 p2 |
	semaphore := Semaphore new.
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) traceCr ].
	p1 := [
		trace value: 'Process 1a waits for signal on semaphore'. 
		semaphore wait.
		trace value: 'Process 1b received signal' ] forkAt: 30.
	p2 := [
		trace value: 'Process 2a signals semaphore'. 
		semaphore signal.
		trace value: 'Process 2b continues' ] forkAt: 20.
	
	trace value: 'Original process pre-delay'.
	1 milliSecond wait.
	trace value: 'Original process post-delay'.   
```


```
	@40 Original process pre-delay
	@30 Process 1a waits for signal on semaphore
	@20 Process 2a signals semaphore
	@30 Process 1b received signal and terminates
	@20 Process 2b continues and terminates
	@40 Original process post-delay
```


Yielding will never allow a lower-priority process to run.  
For a lower-priority process to run, the current process needs to suspend itself \(with the way to get woken up later\) rather than yield.




### About the primitive in yield method

If you look at the exact definition of the `yield` message in Pharo, you can see that it contains an annotation mentioning that this is primitive. The primitive is an optimization.

```
ProcessScheduler >> yield
	| semaphore |
	<primitive: 167>
	semaphore := Semaphore new.
	[semaphore signal] fork.
	semaphore wait.
```


When this method is executed, either the primitive puts the calling process to the back of its run queue, or (if the primitive is not implemented), it performs what we explained earlier and that is illustrated by Figure *@yield@*.

Note that all the primitive does is circumvent having to create a semaphore, to create, to schedule a process, and to signal and to wait to move a process to the back of its run queue. This is worthwhile because most of the time a process's run queue is empty, it is the only runnable process at that priority.

```
| yielded |
yielded := false.
[ yielded := true ] fork.
Processor yield.
yielded
>>> true
```


In the previous snippet, the expression `Processor yield` gives a chance for the created process to run. Note that the example does not show precisely if the UI thread was executed first after the `yield` and that in its logic it yields periodically to let lower processes run or if it was just put at the end of its run queue.


Here is the code of the primitive: if the run queue of the active process priority is empty nothing happens, else the active process is added as the last item in the run queue corresponding to its priority, and the highest priority process is run.

```
InterpreterPrimitives >> primitiveYield
	"Primitively do the equivalent of Process>yield, avoiding the overhead of a fork and a wait in the standard implementation."

	| scheduler activeProc priority processLists processList |
	scheduler := self schedulerPointer.
	activeProc := objectMemory fetchPointer: ActiveProcessIndex ofObject: scheduler.
	priority := self quickFetchInteger: PriorityIndex ofObject: activeProc.
	processLists := objectMemory fetchPointer: ProcessListsIndex ofObject: scheduler.
	processList := objectMemory fetchPointer: priority - 1 ofObject: processLists.

	(self isEmptyList: processList) ifFalse:
		[self addLastLink: activeProc toList: processList.
		 self transferTo: self wakeHighestPriority]
```


### About processPreemption settings

Now we will discuss a bit the settings of the VM regarding process preemption: What exactly happens when a process is 
preempted by a process of a higher priority, and which process is scheduled after the execution of a `yield` message.
The following is based on an answer from E. Miranda on the VM mailing list.

The virtual machine has a setting to change the behavior of process preemption and especially which process gets
resumed once the preempting process terminates. 

In Pharo the setting is true. It means that the interrupted process will be added to the end of the queue and it gives other 
processes a chance to execute themselves without having to have an explicit `yield`.

```
Smalltalk vm processPreemptionYields
>>> true
```


If  `Smalltalk vm processPreemptionYields` returns false then when preempted by a higher-priority process, the current process stays at the head of its run queue. 
It means that it will be the first one of this priority to be resumed.

Note that when a process waits on a semaphore, it is removed from its run queue.
When a process resumes, it always gets added to the back of its run queue.
The `processPreemptionYields` setting does not change anything.


### Comparing the two semantics


The two following examples show the difference between the two semantics that can be controlled by the `processPreemptionYields` setting.


#### First example: two equal processes


- Step 1. First, we create two processes at a lower priority than the active process and at a priority where there are no other processes. The first expression will find an empty priority level at a priority lower than the active process.
- Step 2. Then create two processes at that priority and check that their order in the list is the same as the order in which they were created.
- Step 3. Set the boolean to indicate that this point was reached and block a delay, allowing the processes to run to termination. Check that the processes have indeed terminated.


```
| run priority process1 process2 |
run := true.
"step1"
priority := Processor activePriority - 1.
[(Processor waitingProcessesAt: priority) isEmpty] whileFalse:
	[priority := priority - 1].
"step2"
process1 := [[run] whileTrue] forkAt: priority.
process2 := [[run] whileTrue] forkAt: priority.
self assert: (Processor waitingProcessesAt: priority) first == process1.
self assert: (Processor waitingProcessesAt: priority) last == process2.
"step3"
run := false.
(Delay forMilliseconds: 50) wait.
self assert: (Processor waitingProcessesAt: priority) isEmpty
```


### Second example: preempting P1

The steps 1 and 2 are identical. 
Now let's preempt `process1` while it is running, by waiting on a delay without setting run to false:

```
| run priority process1 process2 |
run := true.
"step1"
priority := Processor activePriority - 1.
[(Processor waitingProcessesAt: priority) isEmpty] whileFalse: 
	[priority := priority - 1].
"step2"
process1 := [[run] whileTrue] forkAt: priority.
process2 := [[run] whileTrue] forkAt: priority.
self assert: (Processor waitingProcessesAt: priority) first == process1.
self assert: (Processor waitingProcessesAt: priority) last == process2.

"Now block on a delay, allowing the first one to run, spinning in its loop.
When the delay ends the current process (the one executing the code snippet) 
will preempt process1, because process1 is at a lower priority."

(Delay forMilliseconds: 50) wait.

Smalltalk vm processPreemptionYields
	ifTrue: 
		"If process preemption yields, process1 will get sent to the back of the run 
		queue (give a chance to other processes to execute without explicitly yielding a process)"
		[ self assert: (Processor waitingProcessesAt: priority) first == process2.
		self assert: (Processor waitingProcessesAt: priority) last == process1 ]
	ifFalse: "If process preemption doesn't yield, the processes retain their order 
		(process must explicit collaborate using yield to pass control among them."
		[ self assert: (Processor waitingProcessesAt: priority) first == process1.
		 self assert: (Processor waitingProcessesAt: priority) last == process2 ].

"step3"
run := false.
(Delay forMilliseconds: 50) wait.
"Check that they have indeed terminated"
self assert: (Processor waitingProcessesAt: priority) isEmpty
```


Run the above after trying both `Smalltalk vm processPreemptionYields: false` and `Smalltalk processPreemptionYields: true`.

What the setting controls is what happens when a process is preempted by a higher-priority process. 
The `processPreemptionYields = true` does an implicit yield of the preempted process. 
It changes the order of the run queue by putting the preempted process at the end of the run queue letting a chance for other processes
to execute. 

### Conclusion

This chapter presents some advanced parts of the scheduler and we hope that it gives a better picture of the scheduling behavior
and in particular, the preemption of the currently running process by a process of higher priority as well as the way yielding the control is implemented. 


