
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
3000
2900
3000
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
3000
3100
3000
mutex := Mutex new. 
mutex critical: [ mutex critical: [ 'Nested passes!' crTrace] ]
	executed := false. 
	promise := [ endless loops ] promise. 
	executed := true. 
	promise value
	| promise |
	promise := [ 1 + 2 ] promise.
	self deny: promise hasValue.
	self deny: promise equals: 3
	| promise |
	promise := [ 1 + 2 ] promise.
	self assert: promise value equals: 3

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
	^ self promiseAt: Processor activePriority
	"Answer a promise that represents the result of the receiver execution
	at the given priority."
	
	| promise |
	promise := Promise new.
	[ promise value: self value ] forkAt: aPriority.
	^ promise
	instanceVariableNames: 'valueProtectingSemaphore value hasValue'
	classVariableNames: ''
	package: 'Promise'
	super initialize.
	valueProtectingSemaphore := Semaphore new.
	hasValue := false
	^ hasValue
	"Wait for a value and once it is available returns it"
	
	valueProtectingSemaphore wait.
	valueProtectingSemaphore signal. "To allow multiple requests for the value."
	^ value 

	value := resultValue.
	hasValue := true.
	valueProtectingSemaphore signal
	instanceVariableNames: 'contentsArray readPosition writePosition accessProtect readSynch ' 
	package: 'Collections-Sequenceable'
readSynch := Semaphore new
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
	accessProtect
		critical: [ writePosition > contentsArray size
				ifTrue: [self makeRoomAtEnd].
			contentsArray at: writePosition put: value.
			writePosition := writePosition + 1].
			readSynch signal.
			^ value
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
'Displaying line'
'Pharo is cool'
'a jumping over the barrier'
'b running to the barrier'
'b jumping over the barrier'
'b jumping over the barrier'
'a running to the barrier'
'a jumping over the barrier'
'b running to the barrier'
'b jumping over the barrier'
'a jumping over the barrier'
'b running to the barrier'
'a jumping over the barrier'
'b jumping over the barrier'
'a running to the barrier'
'b jumping over the barrier'
'a jumping over the barrier'
'a running to the barrier'
'a jumping over the barrier'
'b jumping over the barrier'
['a running to the barrier' crTrace.
'a jumping over the barrier' crTrace ] 
.
[ 'b running to the barrier' crTrace.
'b jumping over the barrier' crTrace ] 
} shuffled do: [ :each | each fork ]
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