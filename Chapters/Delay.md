
delay := Delay forSeconds: 3.
[ 1 to: 10 do: [:i |
  Transcript show: i printString ; cr.
  delay wait ] ] fork
	'ping' crTrace. 
	(Delay forMilliseconds: 300) wait ] 
	] forkAt: Processor userBackgroundPriority.
	  
[ 10 timesRepeat: [ 
	'PONG' crTrace. 
	(Delay forMilliseconds: 100) wait]
	] forkAt: Processor userBackgroundPriority.
	"Return a new Delay for the given number of milliseconds. 
	Sending 'wait' to this Delay will cause the sender's process to be 
	suspended for approximately that length of time."

	^ self new setDelay: aNumber forSemaphore: Semaphore new
	"Private! Initialize this delay to signal the given semaphore after the given number of milliseconds."

	millisecondDelayDuration := milliseconds asInteger.
	millisecondDelayDuration < 0 ifTrue: [self error: 'delay times cannot be negative'].
	delaySemaphore := aSemaphore.
	beingWaitedOn := false.
	"Schedule this Delay, then wait on its semaphore. The current process will be suspended for the amount of time specified when this Delay was created."

	self schedule.
	[ delaySemaphore wait ] ifCurtailed: [ self unschedule ]
	"The delay time has elapsed; signal the waiting process."

	beingWaitedOn := false.
	delaySemaphore signal.