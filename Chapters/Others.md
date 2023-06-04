
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