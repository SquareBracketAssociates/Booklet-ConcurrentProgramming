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






### SharedQueue: a nice semaphore example

A `SharedQueue` is a FIFO \(first in first out\) structure. 
It is often used when a structure can be used by multiple processes that may access the same structure. 
The implementation of a `SharedQueue` uses semaphores to protect its internal queue from concurrent accesses: in particular, a read should not happen when 
a write is under execution. 
Similarly two reads would not read elements in the correct order. 

The definition in Pharo core is different because based on `Monitor`. 
A monitor is a more advanced abstraction to manage concurrency situations. 

Let us look at a possible definition.
We define a class with the following instance variables: a contents holding the elements of the queue, a read and write position and two semaphores for reading and writing control.

```
Object subclass: #SharedQueue
	instanceVariableNames: 'contentsArray readPosition writePosition accessProtect readSynch ' 
	package: 'Collections-Sequenceable'
```


`accessProtect` is a mutual exclusion semaphore used to synchronise write operations while `readSync` is a semahore used for synchronizing read operations. 
These variables are instantiated in the `initialize` method as follows: 

```
SharedQueue >> initialize
	super initialize. 
	accessProtect := Semaphore forMutualExclusion.
	readSynch := Semaphore new
```


These two semaphores are used in the methods to access \(`next`\) and add elements \(`nextPut:`\). 
The idea is that a read should be blocked when there is no element and adding an element will enable reading.
In addition any modification of the internal elements should happen within one single process at the same time.


```
SharedQueue >> next
	| value |
	readSynch wait.
	accessProtect
		critical: [ 
			readPosition = writePosition
				ifTrue: [ self error: 'Error in SharedQueue synchronization'.
					value := nil ]
				ifFalse: [ value := contentsArray at: readPosition.
					contentsArray at: readPosition put: nil.
					readPosition := readPosition + 1 ]].
	^ value
```


In the method `next` used to access elements, the semaphore `readSynch` guards the beginning of the method \(line 3\). 
If a process sends the `next` message when the queue is empty, the process will be suspended and placed in the waiting list of
the semaphore `readSync`. Only the addition of a new element will make this process executable \(as shown in the `nextPut:` method below\).
The critiical section managed by the `accessProtect` semaphore \(lines 4 to 10\) ensures that queue elements cannot be interrupted by another process that could be make the queue inconsistent.

In the method `nextPut:`, the critical section \(lines 3 to 6\) protects the contents of the queue. After such a critical section, the `readSyn` semaphore is signalled. 
This is makes sure that the waiting read processes can now work.  
Again the modification of the internal queue is ensured to not be transversed by different processes. 

```
SharedQueue >> nextPut: value 
	accessProtect
		critical: [ 
			writePosition > contentsArray size
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



%  Local Variables:
%  eval: (flyspell-mode -1)
%  End: