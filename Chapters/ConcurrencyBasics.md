 
 
 
## About synchronisation 
 
 
When multiple threads share and modify the same resources we can easily end up in  
broken state.  
 
### Motivation 
 
Let us imagine that two threads are accessing an account to redraw money. 
When the threads are not synchronised you may end up to the following situation 
that one thread access information while the other thread is actually modifying.  
 
Here we see that we redraw 1000 and 200 but since the thread B reads before  
the other thread finished to commit its changes, we got desynchronised. 
 
 
| Thread A: account debit: 1000 | Thread B: account debit: 200 |  | 
| --- | --- | --- | 
| Reading: account value -> 3000 |  |  | 
| account debit: 1000 | Reading: account value = 3000 |  | 
| account value -> 2000 | account debit: 200 |  | 
|  | account value -> 2800 | 
 
The solution is to make sure that a thread cannot access a resources while another one is modifying it. Basically we want that all the threads sharing a resources are mutually exclusive.  
 
When several access a shared resources, only one gets the resources, the other threads got suspended, waiting for the first thread to have finished and release the resources. 
 
 
### Using a semaphore 
 
 
As we already saw, we can use a semaphore to control the execution of several threads.  
 
Here we want to make sure that we can do 10 debit and 10 deposit of the same amount and that we get the same amount at the end.  
 
``` 
| lock counter |
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
	] fork 
``` 
 
 
``` 
2900
3000
2900
3000 
``` 
 
 
 
Notice the pattern, the thread are not symmetrical.  
The first one will first wait that the resources is accessible and perform his work  
and signals that he finished.  
The second one will work and signal and wait to perform the next iteration. 
 
 
The same problem can be solved in a more robust wait using Mutex and critical sections 
as we see present in the following section. 
 
### Using a Mutex 
 
 
A Mutex \(MUTual EXclusion\) is an object to protect a share resources. 
A mutex can be used when two or more processes need to access a shared resource concurrently. 
A Mutex grants ownership to a single process and will suspend any other process trying to aquire the mutex while in use. 
Waiting processes are granted access to the mutex in the order the access was requested. 
An instance of the class `Mutex` will make sure that only one thread of control can be executed simultaneously on a given portion of code using the message `critical:`. 
 
In the following example the expressions `Processor yield` ensures that thread of the same priority can get a chance to be executed.  
 
``` 
| lock counter |
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
	] fork 
``` 
 
 
``` 
3100
3000
3100
3000 
``` 
 
 
#### Nested critical sections. 
 
In addition a Mutex is also more robust  to nested critical calls than a semaphore. 
For example the following snippet will not deadlock, while a semaphore will. This is why a mutex is also called a recursionLock. 
 
``` 
|mutex|
mutex := Mutex new. 
mutex critical: [ mutex critical: [ 'Nested passes!' crTrace] ] 
``` 
 
 
The same code gets blocked on a deadlock with a semaphore. 
A Mutex and a semaphore are not interchangeable from this perspective. 
 
 
 
 
 
 
## Some examples of semaphores at work 
 
Semaphores are low-level concurrency abstractions.  
In this chapter, we present some abstractions built on top of semaphores: `Promise`, `SharedQueue`, and discuss Rendez-vous. 
 
### Promise 
 
Sometimes we have a computation that can take times. We would like to have the possibility not be blocked waiting for it especially if we do not need immediately.  
Of course there is no magic and we accept to only wait when we need the result of the computation. 
We would like a promise that we will get the result in the future. 
In the literature, such abstraction is called a promise or a future. 
Let us implement a simple promise mechanism: our implementation will not manage errors that could happen during the promise execution.  
The idea behind the implementation is to design a block that 
1. returns a promise and will get access to the block execution value 
1. executes the block in a separated thread. 
 
 
### Illustration 
 
 
For example,  `[ 1 + 2 ] promise` returns a promise, and executes `1 + 2` in a different thread.  
When the user wants to know the value of the promise it sends the message `value` to the promise: 
if the value has been computed, it is handed in, else it is blocked waiting for the result to be computed. 
 
The implementation uses a semaphore to protect the computed value, it means that the requesting process will  
wait for the semaphore until the value is available, but the user of the promise will only be blocked when it requests the value  
of the promise \(not on promise creation\). 
 
The following snippet shows that even if the promise contains an endless loop, it is only looping forever when the promise value is requested - the variable `executed` is true and the program loops forever. 
 
``` 
	| executed promise |
	executed := false. 
	promise := [ endless loops ] promise. 
	executed := true. 
	promise value 
``` 
 
 
### Promise implementation 
 
Let us write some tests: 
First we checks that a promise does not have value when it is only created.  
 
``` 
testPromiseCreation
	| promise |
	promise := [ 1 + 2 ] promise.
	self deny: promise hasValue.
	self deny: promise equals: 3 
``` 
 
 
The second test, create a promise and shows that when its value is requested its value is returned.  
 
``` 
testPromise
	| promise |
	promise := [ 1 + 2 ] promise.
	self assert: promise value equals: 3 
``` 
 
 
It is difficult to test that a program will be blocked until the value is present, since it will block 
the test runner thread itself. 
What we can do is to make the promise execution waits on a semaphore before computing a value  
and to create a second thread that waits for a couple of seconds and signals semaphore. 
This way we can check that the execution is happening or not.   
 
``` 
testPromiseBlockingAndUnblocking

	| controllingPromiseSemaphore promise |
	controllingPromiseSemaphore := Semaphore new.
	
	[ (Delay forSeconds: 2) wait.
	controllingPromiseSemaphore signal ] fork.
	
	promise := [ controllingPromiseSemaphore wait. 
				1 + 3 ] promise. 			
	self deny: promise hasValue.
	
	(Delay forSeconds: 5) wait. 
	self assert: promise hasValue.
	self assert: promise value equals: 4 
``` 
 
 
We have in total three threads: One thread created by the promise that is waiting on the controlling semaphore. 
One thread executing the controlling semaphore and one thread executing the test itself.  
When the test is executed, two threads are spawned and the test will first check that the promise has not been executed 
and wait more time than the thread controlling semaphore: this thread is waiting some seconds to make sure that  
the test can execute the first assertion, then it signals the controlling semaphore.  
When this semaphore is signalled, the promise execution thread is scheduled and will be executed. 
 
### Implementation 
 
 
We define two methods on the `BlockClosure` class: `promise` and `promiseAt:`. 
 
``` 
BlockClosure >> promise
	^ self promiseAt: Processor activePriority 
``` 
 
 
`promiseAt:` creates and return a promise object. In addition, in a separate process, it stores the value of the block itself in the promise.  
 
``` 
BlockClosure >> promiseAt: aPriority
	"Answer a promise that represents the result of the receiver execution
	at the given priority."
	
	| promise |
	promise := Promise new.
	[ promise value: self value ] forkAt: aPriority.
	^ promise 
``` 
 
 
We create a class with a semaphore protecting the computed value, a value and a boolean that lets us know the state of the promise.  
 
``` 
Object subclass: #Promise
	instanceVariableNames: 'valueProtectingSemaphore value hasValue'
	classVariableNames: ''
	package: 'Promise' 
``` 
 
 
We initialize by simply creating a semaphore and setting that the value has not be computed. 
 
``` 
Promise >> initialize
	super initialize.
	valueProtectingSemaphore := Semaphore new.
	hasValue := false 
``` 
 
 
We provide on simple testing method to know the state of the promise.  
 
``` 
Promise >> hasValue
	^ hasValue 
``` 
 
 
 
Nwo the method `value` wait on the protecting semaphore.  
Once it is executing, it means that the promise has computed its value, so it should not block anymore. 
This is why it signals the protecting semaphore before returning the value. 
 
``` 
Promise >> value
	"Wait for a value and once it is available returns it"
	
	valueProtectingSemaphore wait.
	valueProtectingSemaphore signal. "To allow multiple requests for the value."
	^ value  
``` 
 
 
Finally the method ` value:`  stores the value, set that the value has been computed and signal the protecting semaphore that the value is available. Note that such method should not be directly use but should only be invoked by a block closure.  
 
``` 
Promise >> value: resultValue

	value := resultValue.
	hasValue := true.
	valueProtectingSemaphore signal 
``` 
 
 
 
 
 
 
### ShareQueue: a nice semaphore example 
 
A `SharedQueue` is a FIFO \(first in first out\) structure. It uses semaphores to protect from concurrent accesses.  
A `SharedQueue` is often used when a structure can be used by multiple processes that may access the same structure.  
 
 The definition in Pharo is different because based on `Monitor`. A monitor is a more advanced 
abstraction to manage concurrency situations.  
 
Let us look at a possible definitionis the following one. 
 
``` 
Object subclass: #SharedQueue
	instanceVariableNames: 'contentsArray readPosition writePosition accessProtect readSynch ' 
	package: 'Collections-Sequenceable' 
``` 
 
 
 
`accessProtect` est un sémaphore d’exclusion mutuelle pour l’écriture, tandis que readSync est utilisé pour la synchronisation en lecture. Ces variables sont instanciées par la méthode d’initialisation de la façon suivante : 
 
``` 
accessProtect := Semaphore forMutualExclusion.
readSynch := Semaphore new 
``` 
 
 
Ces deux sémaphores sont utilisés dans les méthodes d’accès et d’ajouts d’éléments \(voir figure 6- 5\). 
 
``` 
SharedQueue >> next
	| value |
	readSynch wait.
	accessProtect
		critical: [readPosition = writePosition
				ifTrue: [self error: 'Error in SharedQueue synchronization'.
					Value := nil]
				ifFalse: [value := contentsArray at: readPosition.
					contentsArray at: readPosition put: nil.
					readPosition := readPosition + 1]].
	^ value 
``` 
 
 
Dans la méthode d’accès, next, le sémaphore de synchronisation en lecture « garde » l’entrée de la méthode \(ligne 3\). Si un processus envoie le message next alors que la file est vide, il sera suspendu et placé dans la file d’attente du sémaphore readSync par la méthode wait. Seul l’ajout d’un nouvel élément pourra le rendre à nouveau actif. La section critique gérée par le sémaphore accessProtect \(lignes 4 à 10\) garantit que la portion de code contenue dans le bloc est exécutée sans qu’elle puisse être interrompue par un autre sémaphore, ce qui rendrait l’état de la file inconsistant. 
 
Dans la méthode d’ajout d’un élément, `nextPut:`, la section critique \(lignes 3 à 6\) protège l’écriture, après laquelle le sémaphore `readSync` est _signalée_, ce qui rendra actif les processus en attente de données. 
 
 
``` 
SharedQueue >> nextPut: value 
	accessProtect
		critical: [ writePosition > contentsArray size
				ifTrue: [self makeRoomAtEnd].
			contentsArray at: writePosition put: value.
			writePosition := writePosition + 1].
			readSynch signal.
			^ value 
``` 
 
 
### About Rendez-vous 
 
 
As we saw, using `wait` and `signal` we can  make sure that two programs running in separate threads can be executed one after the other in order. 
 
The following example is freely inspired from "The little book of semaphores book. 
Imagine that we want to have one process reading from file and another process displaying the read contents.  
Obviously we would like to ensure that the reading happens before the display.  
We can enforce such order by using `signal` and `wait` as following 
 
``` 
| readingIsDone read file |
file := FileSystem workingDirectory / 'oneLineBuffer'.
file writeStreamDo: [ :s| s << 'Pharo is cool' ; cr ].
readingIsDone := Semaphore new. 
[
'Reading line' crTrace.
read := file readStream upTo: Character cr.
readingIsDone signal.
] fork.
[ 
readingIsDone wait.	
'Displaying line' crTrace.
read crTrace.
] fork. 
``` 
 
 
Here is the output 
 
``` 
'Reading line'
'Displaying line'
'Pharo is cool' 
``` 
 
 
 
#### Rendez-vous 
 
 
Now a question is how can be generalize such a behavior so that we can have two programs that work freely to a point  
where a part of the other has been performed.  
  
For example imagine that we have two prisoners that to escape have to pass a barrier together \(their order is irrelevant but they should do it consecutively\) and that before that they have to run to the barrier.  
 
The following output is not permitted.  
``` 
'a running to the barrier'
'a jumping over the barrier'
'b running to the barrier'
'b jumping over the barrier' 
``` 
 
 
``` 
'b running to the barrier'
'b jumping over the barrier'
'a running to the barrier'
'a jumping over the barrier' 
``` 
 
 
The following cases are permitted.  
``` 
'a running to the barrier'
'b running to the barrier'
'b jumping over the barrier'
'a jumping over the barrier' 
``` 
 
 
``` 
'a running to the barrier'
'b running to the barrier'
'a jumping over the barrier'
'b jumping over the barrier' 
``` 
 
 
``` 
'b running to the barrier'	
'a running to the barrier'
'b jumping over the barrier'
'a jumping over the barrier' 
``` 
 
 
``` 
'b running to the barrier'	
'a running to the barrier'
'a jumping over the barrier'
'b jumping over the barrier' 
``` 
 
 
Here is a code without any synchronisation. We randomly shuffle an array with two blocks and execute them.  
It produces the non permitted output.  
 
``` 
{  
['a running to the barrier' crTrace.
'a jumping over the barrier' crTrace ] 
.
[ 'b running to the barrier' crTrace.
'b jumping over the barrier' crTrace ] 
} shuffled do: [ :each | each fork ] 
``` 
 
 
 
Here is a possible solution using two semaphores.  
 
``` 
| aAtBarrier bAtBarrier |
aAtBarrier := Semaphore new.
bAtBarrier := Semaphore new.
{[ 'a running to the barrier' crTrace.
aAtBarrier signal. 
bAtBarrier wait.
'a jumping over the barrier' crTrace ] 
.
[ 'b running to the barrier' crTrace.
bAtBarrier signal. 
aAtBarrier wait.
'b jumping over the barrier' crTrace ] 
} shuffled do: [ :each | each fork ] 
```


### Conclusion 
 
We presented the key elements of basic concurrent programming in Pharo and some implementation details. 
