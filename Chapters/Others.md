## About synchronisationWhen multiple threads share and modify the same resources we can easily end up in broken state. ### MotivationLet us imagine that two threads are accessing an account to redraw money.When the threads are not synchronised you may end up to the following situationthat one thread access information while the other thread is actually modifying. Here we see that we redraw 1000 and 200 but since the thread B reads before the other thread finished to commit its changes, we got desynchronised.| Thread A: account debit: 1000 | Thread B: account debit: 200 |  || --- | --- | --- || Reading: account value -> 3000 |  |  || account debit: 1000 | Reading: account value = 3000 |  || account value -> 2000 | account debit: 200 |  ||  | account value -> 2800 |The solution is to make sure that a thread cannot access a resources while another one is modifying it. Basically we want that all the threads sharing a resources are mutually exclusive. When several access a shared resources, only one gets the resources, the other threads got suspended, waiting for the first thread to have finished and release the resources.### Using a semaphoreAs we already saw, we can use a semaphore to control the execution of several threads. Here we want to make sure that we can do 10 debit and 10 deposit of the same amount and that we get the same amount at the end. ```| lock counter |
lock := Semaphore new.
counter := 3000.
[ 10 timesRepeat: [
	lock wait.
	counter := counter + 100.
	counter crTrace.
	lock signal ]
	] fork.
	
[ 10 timesRepeat: [
	counter := counter - 100.
	counter crTrace. 
	lock signal. 
	lock wait ]
	] fork``````2900
3000
2900
3000```Notice the pattern, the thread are not symmetrical. The first one will first wait that the resources is accessible and perform his work and signals that he finished. The second one will work and signal and wait to perform the next iteration.The same problem can be solved in a more robust wait using Mutex and critical sectionsas we see present in the following section.### Using a Mutex```| lock counter |
lock := Mutex new.
counter := 3000.
[10 timesRepeat: [ 
	Processor yield.
	lock critical: [ counter := counter + 100.
						counter crTrace ] ]
	] fork.

[10 timesRepeat: [ 
	Processor yield.
	lock critical: [ counter := counter - 100.
					counter crTrace ] ]
	] fork``````3100
3000
3100
3000```