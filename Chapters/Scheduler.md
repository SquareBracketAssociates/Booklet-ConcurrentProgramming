## Scheduler's principles
pr := [ 1 to: 1000000 do: [ :i | i traceCr ] ] forkAt: 10.
pr inspect
>>> true
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
	"Allow the process that the receiver represents to continue. Put 
	the receiver in line to become the activeProcess. Check for a nil 
	suspendedContext, which indicates a previously terminated Process that 
	would cause a vm crash if the resume attempt were permitted"

	suspendedContext ifNil: [ ^ self primitiveFailed ].
	^ self primitiveResume
	"Allow the process that the receiver represents to continue. Put 
	the receiver in line to become the activeProcess. Fail if the receiver is 
	already waiting in a queue (in a Semaphore or ProcessScheduler)."

	<primitive: 87>
	self primitiveFailed
	"Put this process on the scheduler's lists thus allowing it to proceed next time there is
	 a chance for processes of its priority level. It must go to the back of its run queue so
	 as not to preempt any already running processes at this level. If the process's priority
	 is higher than the current process, preempt the current process."
	| proc |
	proc := self stackTop.  "rcvr"
	(objectMemory isContext: (objectMemory fetchPointer: SuspendedContextIndex ofObject: proc)) ifFalse:
		[^self primitiveFail].
	self resume: proc preemptedYieldingIf: preemptionYields
	"Answer a Process running the code in the receiver. The process is not 
	scheduled."

	<primitive: 19>
	^ Process
		forContext:
			[ self value.
			Processor terminateActive ] asContext
		priority: Processor activePriority
	trace := [ :message | ('@{1} {2}' format: { Processor activePriority. message }) crTrace ].
	[3 timesRepeat: [ trace value: 3. Processor yield ]] forkAt: 12.
	[3 timesRepeat: [ trace value: 2. Processor yield ]] forkAt: 13.
	[3 timesRepeat: [ trace value: 1. Processor yield ]] forkAt: 14.
@14 1
@14 1
@13 2
@13 2
@13 2
@12 3
@12 3
@12 3
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
	@20 Process 2a up to signalling semaphore
	@30 Process 1b received signal and terminates
	@20 Process 2b continues and terminates
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
	@20 Process 2a up to signalling semaphore
	@30 Process 1b received signal and terminates
	@20 Process 3a works and terminates
	@20 Process 2b continues and terminates
	"Give other Processes at the current priority a chance to run."

	| semaphore |
	semaphore := Semaphore new.
	[ semaphore signal ] fork.
	semaphore wait
P2 := [11 to: 20 do: [:i| i trace. ' ' trace. Processor yield ]] fork.
yielded := false.
[ yielded := true ] fork.
yielded
>>> false
yielded := false.
[ yielded := true ] fork.
Processor yield.
yielded
>>> true
yielded := false.
p := [ yielded := true ] forkAt: Processor activeProcess priority - 1.
Processor yield.
yielded
>>> false
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
	@40 Original process post-yield
	@30 Process 1a waits for signal on semaphore
	@20 Process 2a signals semaphore
	@30 Process 1b received signal and terminates
	@20 Process 2b continues and terminates
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
	@30 Process 1a waits for signal on semaphore
	@20 Process 2a signals semaphore
	@30 Process 1b received signal and terminates
	@20 Process 2b continues and terminates
	@40 Original process post-delay
	| semaphore |
	<primitive: 167>
	semaphore := Semaphore new.
	[semaphore signal] fork.
	semaphore wait.
yielded := false.
[ yielded := true ] fork.
Processor yield.
yielded
>>> true
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
>>> true
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