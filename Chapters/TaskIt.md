## TaskIt

In the previous chapters, we presented low-level concepts of concurrent programming. 
It is, however, not really good to sprinkle your code with process forking. This is why we present now TaskIt. TaskIt is a higher-level library to manage concurrent tasks but without resolving to low-level mechanisms. 

TaskIt is a library that eases process usage in Pharo. It provides abstractions to execute and synchronize concurrent tasks, and several pre-built mechanisms that are useful for many application developers. This chapter starts by familiarizing the reader with TaskIt's abstractions, guided by examples and code snippets. In the end, we discuss TaskIt extension points and possible customizations.

### Why TaskIt?
Expressing and managing concurrent computations is a concern of importance to developing applications that scale. 
A web application may want to use different processes for each of its incoming requests. Or it may want to use a "thread pool" in some cases. 
In other cases, a desktop application may want to send computations to a worker to not block the UI thread.

You can use the low-level libraries we presented so far. Now it means that as a developer you will have to take care of the following point: you should pay attention that the processes you created do not:
- create race condition where one process is taking a resource already used by another one,
- create processes with the wrong priority that can impact the performance and behavior of the complete system,
- create a process that is starving and waiting endlessly that a semaphore gets signaled.



These are the basic problems that may arise when processes are used without real care. 
In addition, you should understand where to place critical sections if needed.

This is why TaskIt is interesting. It abstracts low-level mechanisms.
TaskIt's main abstractions are, as the name indicates, tasks. A task is a unit of execution. By splitting the execution of a program into several tasks, TaskIt can run those tasks concurrently, synchronize their access to data, or order even help in ordering and synchronizing their execution.



%Processes in Pharo are implemented as green threads scheduled by the virtual machine, without %depending on the machinery of the underlying operating system. This has several consequences on the usage of concurrency we can do:

%- Processes are cheap to create and schedule. We can create as many of them as we want, and %performance will only degrade if the code executed in those processes does so, which is to be %expected.
%- Processes provide concurrent execution but no real parallelism. Inside Pharo, it does not matter %the number of processes we use. They will be always executed in a single operating system thread, %in a single operating system process.

%Also, besides how expensive it is to create a process, to know how we could organize the processes %in our application, we need to know how to synchronize such processes. For example, maybe we need %to execute two processes concurrently and we want a third one to wait for the completion of the %first two before starting. Or maybe we need to maximize the parallelism of our application while %enforcing the concurrent access to some piece of state. And all these issues require avoiding the %creation of deadlocks.



### Loading

If you want a specific release such as v1.0, you can load the associated tag as follows:

```smalltalk
Metacello new
  baseline: 'TaskIt';
  repository: 'github://pharo-contributions/taskit:v1.0';
  load.
```

Otherwise, if you want the latest development version, load the master branch:

```smalltalk
Metacello new
  baseline: 'TaskIt';
  repository: 'github://pharo-contributions/taskit';
  load.
```


#### Adding it as a Metacello dependency

Add the following in your metacello configuration or baseline specifying the desired version:

```smalltalk
spec
    baseline: 'TaskIt'
    with: [ spec repository: 'github://pharo-contributions/taskit:v1.0' ]
```


### First example

Launching a task is as easy as sending the message `schedule` to a block closure, as it is used in the following first code example:

```smalltalk
[ 1 + 1 ] schedule
```
The selector name `schedule` is chosen in purpose instead of others, such as run, launch, or execute. TaskIt promises that a task will be *eventually* executed, but this is not necessarily right away. In other words, a task is *scheduled* to be executed at some point in time in the future.

This first example is, however, useful to clarify the first two concepts, but it remains too simple. 

We are scheduling a task that is useless, and we cannot even observe its result (*yet*). Let's explore some other code snippets that may help us understand what's going on.

The following code snippet will schedule a task that prints to the `Transcript`. Just evaluating the expression below will make it evident that the task is actually executed. However, a so simple task runs so fast that it's difficult to tell if it's actually running concurrently with our primary process or not.
```smalltalk
[ 'Happened' traceCr ] schedule.
```
The real acid test is to schedule a long-running task. The following example schedules a task that waits for a second before writing to the transcript. While normal synchronous code would block the main thread, you'll notice that this one does not. 
```smalltalk
[ 1 second wait.
'Waited' traceCr ] schedule.
```

### Schedule vs. fork
You may be asking yourself what's the difference between the `schedule` and `fork`. From the examples above, they seem to do the same, but they do not. 
In a nutshell, to understand why `schedule` means something different than `fork`, picture that using TaskIt two tasks may execute inside the same process, or in a pool of processes, while `fork` creates a new process every time.

You will find a longer answer in the section below explaining *runners*. In TaskIt, tasks are not directly scheduled in Pharo's global `ProcessScheduler` object as usual `Process` objects are. Instead, a task is scheduled in a task runner. It is the responsibility of the task runner to execute the task.

### All valuables can be Tasks

We have been using block closures so far as tasks. Block closures are a handy way to create a task since they implicitly capture the context: they have access to `self` and other objects in the scope. However, blocks are not always the wisest choice for tasks. Indeed, when a block closure is created, it references the current `context` with all the objects in it and its *sender contexts*, being a potential source of memory leaks.

The good news is that TaskIt tasks can be represented by almost any object. 

A task in TaskIt's domain is **valuable objects** i.e., objects that will do some computation when they receive the `value` message. 

The message `schedule` is just syntax sugar for:

```smalltalk
(TKTTask valuable: [ 1 traceCr ]) schedule.
```

We can then create tasks using message sends or weak message sends:

```smalltalk
TKTTask valuable: (WeakMessageSend receiver: Object new selector: #yourself).
TKTTask valuable: (MessageSend receiver: 1 selector: #+ arguments: { 7 }).
```

Or even create our task object:

```smalltalk
Object << #MyTask
	package: 'MyPackage'.
```
```
MyTask >> value
    ^ 100 factorial
```

and use it as follows:

```smalltalk
TKTTask valuable: MyTask new.
```

### Retrieving a task's result with futures

In TaskIt we differentiate two different kinds of tasks: some tasks are just *scheduled* for execution, they produce some side-effect and no result, and some other tasks will produce (generally) a side-effect-free value. When the result of a task is important, TaskIt provides us with a *future* object. 
A *future* is no other thing than an object that represents the future value of the task's execution.
We can schedule a task with a future by using the `future` message on a block closure, as follows.

```smalltalk
aFuture := [ 2 + 2 ] future.
```

One way to see futures is as placeholders. When the task is finished, it deploys its result into the corresponding future. A future then provides access to its value, but since we cannot know *when* this value will be available, we cannot access it right away. Instead, futures provide an asynchronous way to access its value by using *callbacks*. A callback is an object that will be executed when the task execution is finished.  

In general terms, we do not want to **force** a future to retrieve its value in a synchronous way.
By doing so, we would be going back to the synchronous world, blocking a process' execution and not exploiting concurrency.
Later sections will discuss synchronous (blocking) retrieval of a future's value.

A future can provide two kinds of results: either the task execution was a success or a failure. 
- A success happens when the task completes in a normal way.
- A failure happens when an uncatched exception is raised in the task. 

Because of these distinctions, futures allow the subscription of two different callbacks using the methods `onSuccessDo:` and `onFailureDo:`.

In the example below, we create a future and subscribe to it as a success callback. As soon as the task finishes, the value gets deployed in the future and the callback is called with it.

```smalltalk
aFuture := [ 2 + 2 ] future.
aFuture onSuccessDo: [ :result | result traceCr ].
```
We can also define callbacks that handle a task's failure using the `onFailureDo:` message. If an exception occurs and the task cannot finish its execution as expected, the corresponding exception will be passed as an argument to the failure callback, as in the following example.

```smalltalk
aFuture := [ Error signal ] future.
aFuture onFailureDo: [ :error | error signalerContext sender method selector traceCr ].
```

Futures accept more than one callback. When its associated task is finished, all its callbacks will be *scheduled* for execution. In other words, the only guarantee that callbacks give us is that they will all be eventually executed. However, the future itself cannot guarantee either **when** the callbacks will be executed or **in which order**. 

The following example shows how we can define several successful callbacks for the same future.

```smalltalk
future := [ 2 + 2 ] future.
future onSuccessDo: [ :v | Stdio stdout nextPutAll: v asString; cr; flush. ].
future onSuccessDo: [ :v | 'Finished' traceCr ].
future onSuccessDo: [ :v | [ v factorial traceCr ] schedule ].
future onFailureDo: [ :error | error traceCr ].
```

Callbacks work whether the task is still running or already finished. If the task is running, callbacks are registered and wait for the completion of the task. 
If the task is finished, the callback will be immediately scheduled with the already deployed value. 
See below code examples that illustrate this: we first create a future and subscribe a callback before it is finished, then we wait for its completion and subscribe a second callback afterwards. Both callbacks are scheduled for execution.

```smalltalk
future := [ 1 second wait. 2 + 2 ] future.
future onSuccessDo: [ :v | v traceCr ].

2 seconds wait.
future onSuccessDo: [ :v | v traceCr ].
```

### Task runners: Controlling how tasks are executed 

So far, we created and executed tasks without caring too much about the form they were executed. Indeed, we knew that they were run concurrently because they were non-blocking. We also said already that the difference between a `schedule` message and a `fork` message is that scheduled messages are run by a **task runner**.

A task runner is an object in charge of executing tasks *eventually*. Indeed, the main API of a task runner is the `schedule:` message that allows us to tell the task runner to schedule a task.

```smalltalk
aRunner := TKTNewProcessTaskRunner new.
aRunner schedule: [ 1 + 1 ]
```

A nice extension built on top of the schedule is the  `future:` message that allows us to schedule a task but obtain a future of its eventual execution.

```smalltalk
aRunner := TKTNewProcessTaskRunner new.
future := aRunner future: [ 1 + 1 ]
```

Indeed, the messages `schedule` and `future` we have learned before are only syntax-sugar extensions that call these respective ones on a default task runner. This section discusses several useful task runners already provided by TaskIt.

### New process task runner

A new process task runner, an instance of `TKTNewProcessTaskRunner`, is a task runner that runs each task in a new separate Pharo process. 

```smalltalk
aRunner := TKTNewProcessTaskRunner new.
aRunner schedule: [ 1 second wait. 'test' traceCr ].
```
Moreover, since new processes are created to manage each task, scheduling two different tasks will be executed concurrently. For example, in the code snippet below, we schedule twice a task that prints the identity hash of the current process.

```smalltalk
aRunner := TKTNewProcessTaskRunner new.
task := [ 10 timesRepeat: [ 10 milliSeconds wait.
				('Hello from: ', Processor activeProcess identityHash asString) traceCr ] ].
aRunner schedule: task.
aRunner schedule: task.
```

The generated output will look something like this:

```
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
```

First, you'll see that a different process is being used to execute each task. Also, their execution is concurrent, as we can see the messages interleaved.

### Local Process Task Runner

The local process runner, an instance of `TKTLocalProcessTaskRunner`, is a task runner that executes a task in the caller process. In other words, this task runner does not run concurrently. Executing the following piece of code:

```smalltalk
aRunner := TKTLocalProcessTaskRunner new.
future := aRunner schedule: [ 1 second wait ].
```
is equivalent to the following piece of code:
```smalltalk
[ 1 second wait ] value.
```
or even:
```smalltalk
1 second wait.
```

While this runner may seem naive, it may also come in handy to control and debug task executions. Besides, the power of task runners is that they offer a polymorphic API to execute tasks.

### The Worker Runner

The worker runner, and instance of `TKTWorker`, is a task runner that uses a single process to execute tasks from a queue. The worker's single process removes one-by-one the tasks from the queue and executes them sequentially. Then, scheduling a task for a worker means adding the task to the queue.

A worker manages the life cycle of its process and provides the messages `start` and `stop` to control when the worker thread begins and end.

```smalltalk
worker := TKTWorker new.
worker start.
worker schedule: [ 1 + 5 ].
worker stop.
```

Using workers, we can control the amount of live processes and how tasks are distributed amongst them. For example, in the following example, three tasks are executed sequentially in a single separate process while still allowing us to use an asynchronous style of programming.

```smalltalk
worker := TKTWorker new start.
future1 := worker future: [ 2 + 2 ].
future2 := worker future: [ 3 + 3 ].
future3 := worker future: [ 1 + 1 ].
1 second wait.
worker stop.
```

Workers can be combined into *worker pools*.


### The Worker pool

A TaskIt worker pool is a pool of worker runners, equivalent to a ThreadPool from other programming languages. Its main purpose is to provide several worker runners and decouple us from the management of threads/processes. 
A worker pool is a runner in the sense we use the `schedule:` message to schedule tasks in it. 

In TaskIt, we count two kinds of worker pools: 

 * TKTWorkerPool  
 * TKTCommonQueueWorkerPool


#### TKTWorkerPool

Internally, all runners inside a TKTWorkerPool pool have a task queue. This pool counts a worker in charge of scheduling tasks into one of the available workers, taking into account the workload of each worker. 

Different applications may have different concurrency needs; thus, TaskIt worker pools do not provide a default amount of workers. 

Before using a pool, we need to specify the maximum number of workers using the `poolMaxSize:` message. A worker pool will create new workers on demand.


```smalltalk
pool := TKTWorkerPool new.
pool poolMaxSize: 5.
```
TaskIt worker pools internally use an extra worker to synchronise the access to its task queue. Because of this, a worker pool has to be manually started using the `start` message before scheduled messages start to be executed.

```smalltalk
pool := TKTWorkerPool new.
pool poolMaxSize: 5.
pool start.
pool schedule: [ 1 traceCr ].
```

Once we are done with the worker pool, we can stop it by sending it the `stop` message.
```smalltalk
pool stop.
```


#### TKTCommonQueueWorkerPool

Internally, all runners inside a TKTCommonQueueWorkerPool pool share a common queue. This pool counts with a watchdog that is in charge of ensuring that all the workers are alive, and in charge of reducing the number of workers when the load of work goes down. 


Different applications may have different concurrency needs; thus, TaskIt worker pools do not provide a default amount of workers. Before using a pool, we need to specify the maximum number of workers in the pool using the `poolMaxSize:` message. A worker pool will create new workers on demand.

```smalltalk
pool := TKTCommonQueueWorkerPool new.
pool poolMaxSize: 5.
```
TaskIt worker pools internally use an extra worker to synchronise the access to its task queue. Because of this, a worker pool has to be manually started using the `start` message before scheduled messages start to be executed.

```smalltalk
pool := TKTCommonQueueWorkerPool new.
pool poolMaxSize: 5.
pool start.
pool schedule: [ 1 traceCr ].
```

Once we are done with the worker pool, we can stop it by sending it the `stop` message.
```smalltalk
pool stop.
```

### Managing runner exceptions

As we stated before, in TaskIt, the result of a task can be interesting for us or not. In case we do not need a task's result, we will schedule it using the `schedule` or `schedule:` messages. This is a kind of fire-and-forget way of executing tasks. On the other hand, if the result of a task execution interests us we can get a future on it using the `future` and `future:` messages. These two ways to execute tasks require different ways to handle exceptions during task execution.

First, when an exception occurs during a task execution with an associated future, the exception is forwarded to the future. In the future, we can subscribe to a failure callback using the `onFailureDo:` message to manage the exception accordingly.

However, on a fire-and-forget kind of schedule, the execution and results of a task are no anymore under our control. If an exception happens in this case, it is the responsibility of the task runner to catch the exception and manage it gracefully. For this, each task runner is configured with an exception handler in charge of it. TaskIt exception handler classes are subclasses of the abstract `TKTExceptionHandler` that defines a `handleException:` method. Subclasses need to override the `handleException:` method to define how to manage exceptions.

TaskIt provides by default a `TKTDebuggerExceptionHandler`, accessible from the configuration `TKTConfiguration errorHandler`, that will open a debugger on the raised exception. The `handleException:` method is defined as follows:

```smalltalk
handleException: anError 
	anError debug
```

Changing a runner's exception handler can be done by sending it the `exceptionHandler:` message as follows:

```smalltalk
aRunner exceptionHandler: TKTDebuggerExceptionHandler new.
```

### Task Timeout

In TaskIt, tasks can be optionally scheduled with a timeout. A task's timeout limits the execution of a task to a window of time. If the task tries to run longer than the specified time, the task is cancelled automatically. This behaviour is desirable because a long-running task may be a hint towards a problem, or it can just affect the responsiveness of our application.

A task's timeout can be provided while scheduling a task in a runner using the `schedule:timeout:` message, follows: 
```smalltalk
aRunner schedule: [1 second wait] timeout: 50 milliSeconds.
```
If the task surpasses the timeout, the scheduled task will be cancelled with an exception.

A task's timeout must not be confused with a future's synchronous access timeout (*explained below*). The task timeout governs the task execution, while a future's timeout governs only the access to the future value. If a future time out while accessing its value, the task will continue its execution normally.


### Where do tasks and callbacks run by default?

As we stated before, the messages #schedule and #future will schedule a task implicitly in a *default* task runner. To be more precise, it is not a default task runner but the **current task runner** that is used. In other words, task scheduling is context sensitive: if task A is being executed by a task runner R, new tasks scheduled by A are implicitly scheduled by R. The only exception to this is when there is no such task runner, i.e., when the task is scheduled from, for example, a workspace. In that case, a default task runner is chosen for scheduling.

Note: In the current version of TaskIt (v1.0), the default task runner is the global worker pool that can be explicitly accessed by evaluating the following expression `TKTConfiguration runner`.

Something similar happens with callbacks. Before, we said that callbacks are eventually and concurrently executed. This happens because callbacks are scheduled as normal tasks after a task's execution. This scheduling follows the rules from above: callbacks will be scheduled in the task runner where its task was executed.

### Advanced Futures: combinators

Futures are a nice asynchronous way to obtain the results of our eventually executed tasks. However, as we do not know when tasks will finish, processing those results will be another asynchronous task that needs to start as soon as the first one finishes. To simplify the task of future management, TaskIt's futures come along with some combinators.

- **The `collect:` combinator**

The `collect:` combinator does, as its name says, the same as the collection's API: it transforms a result using a transformation.

```smalltalk
future := [ 2 + 3 ] future.
(future collect: [ :number | number factorial ])
    onSuccessDo: [ :result | result traceCr ].
```

The `collect:` combinator returns a new future whose value will result from transforming the first future's value.

- **The `select:` combinator**

The `select:` combinator does, as its name says, the same as the collection's API: it filters a result satisfying a condition.

```smalltalk
future := [ 2 + 3 ] future.
(future select: [ :number | number even ])
    onSuccessDo: [ :result | result traceCr ];
    onFailureDo: [ :error | error traceCr ].
```

The `select:` combinator returns a new future whose result is the result of the first future if it satisfies the condition. Otherwise, its value will be a `NotFound` exception.

- **The `flatCollect:`combinator**

The `flatCollect:` combinator is similar to the `collect:` combinator, transforming the result of the first future using the given transformation block. However, `flatCollect:` excepts as the result of its transformation block a future.

```smalltalk
future := [ 2 + 3 ] future.
(future flatCollect: [ :number | [ number factorial ] future ])
    onSuccessDo: [ :result | result traceCr ].
```
The `flatCollect:` combinator returns a new future whose value will be the result of the future yielded by the transformation.

- **The `zip:`combinator**

The `zip:` combinator combines two futures into a single future that returns an array with both results.

```smalltalk
future1 := [ 2 + 3 ] future.
future2 := [ 18 factorial ] future.
(future1 zip: future2)
    onSuccessDo: [ :result | result traceCr ].
```
`zip:` works only on success: the resulting future will fail if any of the future is also a failure.

- **The `on:do:`combinator**

The `on:do:` allows us to transform a future that fails with an exception into a future with a result.

```smalltalk
future := [ Error signal ] future
    on: Error do: [ :error | 5 ].
future onSuccessDo: [ :result | result traceCr ].
```

- **The `fallbackTo:` combinator**

The `fallbackTo:` combinator combines two futures in a way such that if the first future fails, it is the second one that will be considered.

```smalltalk
failFuture := [ Error signal ] future.
successFuture := [ 1 + 1 ] future.
(failFuture fallbackTo: successFuture)
    onSuccessDo: [ :result | result traceCr ].
```

In other words, `fallbackTo:` produces a new future whose value is the first's future value if success, or it is the second future's value otherwise. 

- **The `firstCompleteOf:` combinator**

The `firstCompleteOf:` combinator combines two futures resulting in a new future whose value is the value of the future that finishes first, whether a success or a failure.

```smalltalk
failFuture := [ 1 second wait. Error signal ] future.
successFuture := [ 1 second wait. 1 + 1 ] future.
(failFuture firstCompleteOf: successFuture)
    onSuccessDo: [ :result | result traceCr ];
    onFailureDo: [ :error | error traceCr ].
```

In other words, `fallbackTo:` produces a new future whose value is the first's future value if success, or it is the second future's value otherwise.

- **The `andThen:` combinator**

The `andThen:` combinator allows one to chain several futures to a single future's value. All futures chained using the `andThen:` combinator are guaranteed to be executed sequentially (in contrast to normal callbacks), and all of them will receive as value the value of the first future (instead of its preceding future).

```smalltalk
([ 1 + 1 ] future
    andThen: [ :result | result traceCr ])
    andThen: [ :result | Stdio stdout nextPutAll: result ]. 
```

This combinator is meant to enforce the order of execution of several actions, and this it is mostly for side-effect purposes where we want to guarantee such order.

### Synchronous access

Sometimes, although we do not recommend it, you will need or want to access the value of a task synchronously: that is, to wait for it. We do not recommend waiting for a task because of several reasons:
  - sometimes, you do not know how much a task will last and therefore, the waiting can kill your application's responsiveness
  - also, it will block your current process until the waiting is finished
  - you come back to the synchronous world, killing ultimately the purpose of using TaskIt :)

However, since experienced users may still need this feature, TaskIt futures provide three different messages to access synchronously its result: `isFinished`, `waitForCompletion:` and `synchronizeTimeout:`.

`isFinished` is a testing method that we can use to test if the corresponding future is still finished or not. The following piece of code shows how we could implement an active wait on a future:

```smalltalk
future := [1 second wait] future.
[future isFinished] whileFalse: [50 milliSeconds wait].
```

An alternative version for this code that does not require an active wait is the message `waitForCompletion:`. `waitForCompletion:` expects a duration as an argument that he will use as a timeout. This method will block the execution until the task finishes or the timeout expires, whatever comes first. If the task does not finish by the timeout, a `TKTTimeoutException` will be raised.

```smalltalk
future := [1 second wait] future.
future waitForCompletion: 2 seconds.

future := [1 second wait] future.
[future waitForCompletion: 50 milliSeconds] on: TKTTimeoutException do: [ :error | error traceCr ].
```

Finally, to retrieve the future's result, futures understand the `synchronizeTimeout:` message, which receives a duration as an argument as its timeout. The result is returned if a successful value is available by the timeout. Suppose the timeout finishes the task with a failure; an `UnhandledError` exception is raised, wrapping the original exception. Otherwise, if the task is not finished by the timeout, a `TKTTimeoutException` is raised.

```smalltalk
future := [1 second wait. 42] future.
(future synchronizeTimeout: 2 seconds) traceCr.

future := [ self error ] future.
[ future synchronizeTimeout: 2 seconds ] on: Error do: [ :error | error traceCr ].

future := [ 5 seconds wait ] future.
[ future synchronizeTimeout: 1 seconds ] on: TKTTimeoutException do: [ :error | error traceCr ].
```

## Services

TaskIt offers a package of implementing services. A service is a process that executes a task over and over again. You can think about a web server or a database server that needs to be up and running and listening to new connections all the time.

Each TaskIt service may define a `setUp`, a `tearDown`, and a `stepService`. `setUp` is run when a service is being started, `shutDown` is run when the service is being shut down, and `stepService` is the main service action that will be executed repeatedly.

Creating a new service is as easy as creating a subclass of `TKTService`. For example, let's create a service that watches the existence of a file. If the file does not exist it will log it to the transcript. It will also log when the service starts and stops to the transcript.

```smalltalk
TKTService << #TKTFileWatcher
  slots: {#file};
  package: 'TaskItServices-Tests'
```

Hooking on the service's `setUp` and `tearDown` is as easy as overriding such methods:

```smalltalk
TKTFileWatcher >> setUp
  super setUp.
  Transcript show: 'File watcher started'.
```
```
TKTFileWatcher >> tearDown
  super tearDown.
  Transcript show: 'File watcher finished'.
```

Finally, setting the watcher action is as easy as overriding the `stepService` message.

```smalltalk
TKTFileWatcher >> stepService
  1 second wait.
  file asFileReference exists
    ifFalse: [ Transcript show: 'file does not exist!' ]
```

Making the service work requires yet an additional method: the service name. Each service should provide a unique name through the `name` method. TaskIt verifies that service names are unique and prevents the starting of two services with the same name.

```smalltalk
TKTFileWatcher >> name
  ^ 'Watcher file: ', file asString
```

Once your service is finished, starting it is as easy as sending it the `start` message.

```smalltalk
watcher := TKTFileWatcher new.
watcher file: 'temp.txt'.
watcher start.
```

Requesting the stop of service is done by sending it the `stop` message. 
Know that sending the `stop` message will not stop the service right away. 
It will actually request it to stop, which will schedule the teardown of the service and kill its process after that. 

```smalltalk
watcher stop.
```

Stopping the process in an unsafe way is also supported by sending it the `kill` message. 
Killing a service will stop it right away, interrupting whatever task it was executing.

```smalltalk
watcher kill.
```

### Creating services with blocks

Additionally, TaskIt provides an alternative means to create services through blocks (or valuables) using `TKTParameterizableService`. An alternative implementation of the file watcher could be done as follows.

```smalltalk
service := TKTParameterizableService new.
service name: 'Generic watcher service'.
service onSetUpDo: [ Transcript show: 'File watcher started' ].
service onTearDownDo: [ Transcript show: 'File watcher finished' ].
service step: [
  'temp.txt' asFileReference exists
    ifFalse: [ Transcript show: 'file does not exist!' ] ].

service start.
```

### ActIt

TaskIt we are also providing actors, leveraging the whole Taskit implementation, and adding some extra features. Our implementation is inspired by "Actalk: a Testbed for Classifying and Designing Actor Languages in the Smalltalk-80 Environment", but adapted to the new Pharo state-full traits.


#### Actors

The actor's model proposes to provide an interface to interact with a process. Allowing the user of a process to ask for service to a process, by message sending.

To achieve the same kind of behaviour in Pharo, we subordinate a process to expose and serve the behaviour of an existing object. 

#### How to use it

To achieve this, we propose the trait `TKTActorBehaviour`, which is responsible for extending a class by adding the message actor. 

This actor message will return an instance of the class `TKTActor`, which will act as a proxy (managed by `doesnotUnderstand:` message) to the object but transform the calls into tasks to be executed sequentially.

Each method sent to the actor will return a *future*. 

To make your domain object become an actor, add the usage of the trait `TKTActorBehaviour` as follows:

```smalltalk
Object << #MyDomainObject
	uses: TKTActorBehaviour;
	slots: {value};
	package: 'MyDomainObjectPack'

myObject := MyDomainObject new. 
myObject setValue: 2.

self assert: myObject getValue equals: 2.

myActor := myObject actor.
self assert: (myActor getValue isKindOf: TKTFuture).
self assert: (myActor getValue synchronizeTimeout: 1 second) equals: myObject getValue. 

```

### How to act

To add this trait is not enough to make your Object into an Actor. 
You have to keep in mind that any time that you use `smalltalk self` in your object, you are doing a synchronous call. 
That each time that you give your object's reference by parameter instead of the actor's reference, your object will work as a classic object as well.
  
For allowing the user to do async calls to self, the trait provides de property `smalltalk aself` (Async-self). 
  
Remind also that even when actors provide a nice way to avoid simple semaphores, they do not entirely avoid deadlocks since the interaction between actors is possible, desirable, and 
non-regulated.
  


### Process dashboard 

TaskIt provides a process dashboard based on announcements. 

To access this dashboard, go to World menu > TaskIt > Process dashboard.

The window has two tabs.

#### TaskIt tab 
The first one shows the processes launched by TaskIt looks like 

The showed table has six fields. 
- \# ordinal number. Just for easing the reading.
- Name: The name of the task. If no name was given, it generates a name based on the related objects. 
- Sending: The selector of the method that executes the task. If the task is based on a block, it will be #value. 
- To: The receiver of the message that executes the task. 
- With: The arguments of the message send that executes the task
- State: [Running|NotRunning].

Some of those fields are attached to some contextual menus. 

Do right-click on top of the name of the process to interact with the process.

The options given are:
- Inspect the process: It opens an inspector showing the related TaskIt process.
- Suspend|Resume the process: It pauses or resumes the selected process. 
- Cancel the process: It cancels the process execution.  

Do right-click on the top of the message selector to interact with the selector method.

The options given are:
- Method. This option browses the method executed by the task.
- Implementors. This option browses all the implementors of this selector. 


Finally, do right-click on the top of the receiver to interact with it

The option given is
-  Inspect the receiver. It inspects the receiver of the message.

####System tab

Finally, to allow the user to use just one interface. There is a second tab that shows the processes that were not spawned by TaskIt. 

#### About on announcements 
  
The TaskIt browser is based on announcements. 
This fact allows the interface to be dynamic, always having fresh information, without needing a pulling process, as in the native process browser. 

### Debugger

TaskIt comes with a debugger extension for Pharo that can be installed by loading the 'debug' group of the baseline (the debugger is not loaded by any other group):

```smalltalk
Metacello new
  baseline: 'TaskIt';
  repository: 'github://pharo-contributions/taskit';
  load: 'debug'.
```

After installation, the TaskIt debugger extension will automatically be available to processes associated with a task or future. You can manually enable or disable the debugger extension by evaluating `TKTDebugger enable` or `TKTDebugger disable`.

The TaskIt debugger shows an augmented stack, in which the process that represents the task or future is at the top, and the process that created the task or future is at the bottom (recursively for tasks and futures created from other tasks and futures). The following visualisation shows one future process (top) with frames `1` and `2` and the corresponding creator process (frames `3` and `4`):

```
-------------------
|     frame 1     |
-------------------
|     frame 2     |
-------------------
-------------------
|     frame 3     |
-------------------
|     frame 4     |
-------------------
```
The implementation and conception of this debugger extension can be found in Max Leske's Master's thesis entitled ["Improving live debugging of concurrent threads"](http://scg.unibe.ch/scgbib?query=Lesk16a&display=abstract).


### Configuration

TaskIt bases its general configuration on the idea of profiles. 
A profile defines some major features needed by the library to work properly.

#### Profiles.

The class `TKTProfile` defines default profiles on the class side.

```smalltalk
defaultProfile
	^ #development
```
```
development
	<profile: #development>
	^ TKTProfile
		on:
			{(#debugging -> true).
			(#runner -> TKTCommonQueueWorkerPool createDefault).
			(#poolWorkerProcess -> TKTDebuggWorkerProcess).
			(#process -> TKTRawProcess).
			(#errorHandler -> TKTDebuggerExceptionHandler).
			(#processProvider -> TKTTaskItProcessProvider new).
			(#serviceManager -> TKTServiceManager new)} asDictionary
```

```
production
	<profile: #production>
	^ TKTProfile
		on:
			{
			(#debugging -> false) .
			(#runner -> TKTCommonQueueWorkerPool createDefault).
			(#poolWorkerProcess -> TKTWorkerProcess).
			(#process -> Process).
			(#errorHandler -> TKTExceptionHandler).
			(#processProvider -> TKTPharoProcessProvider new).
			(#serviceManager -> TKTServiceManager new)} asDictionary
```

```
test
	<profile: #test>
	^ TKTProfile
		on:
			{(#debugging -> false).
			(#runner -> TKTCommonQueueWorkerPool createDefault).
			(#poolWorkerProcess -> TKTWorkerProcess).
			(#process -> Process).
			(#errorHandler -> TKTExceptionHandler).
			(#processProvider -> TKTTaskItProcessProvider new).
			(#serviceManager -> TKTServiceManager new)} asDictionary
```



#### Modifying the running profile

There are three ways of modifying the running profile.

**The first** one and simpler, is to go to the *settings browser* and choose the available profile in the section 'TaskIt execution profile.' 
In this combo box you will find all the predefined profiles. 

 **The second** way is to use code:
  
```smalltalk
TKTConfiguration profileNamed: #development 
```
The method `profileNamed: aProfile` receives as a parameter name of a predefined profile. This way is handy for automating behaviour. 

**The third** one finally is to manually build your own profile and set it up, again by code 	


```smalltalk
profile := TKTProfile new. 
	... 
configure 
	...
TKTConfiguration profile: profile.
```

#### Defining a new predefined-profile

To add a new profile is pretty easy, and so far, pretty static

For adding a new profile, you have only to define a new method in the class side of `TKTProfiles` by adding the pragma `<profile:#profileName>`.

This method should return an instance of TKTProfile, or polymorphic to it. 

Since some configurations may not be compatible (since the debugging mode has some specific restrictions), a check of the sanity of the configuration is done during the activation of the profile. Therefore, it is expected to have exceptions with some configurations. 

#### Modifying an existing predefined-profile

As to adding a new profile, everything is on the code. You just have to address the method related to the profile that you want to modify and modify it.
If the modified profile is the one on usage, the changes will have no effect up to the next time you actively set this profile. 

#### Using a specific profile during specific computations

At some points, you may need to switch the working profile, or part of it, not for all the images but for some specific computation.
We have defined different methods to allow you to achieve this feature by code. 

```smalltalk
TKTConfiguration>> profileNamed: aProfileName during: aBlock      	
 	"Uses a predefined profile, during the execution of the given block "
```

```
 profile: aProfile during: aBlock					
 	"Uses a profile, during the execution of the given block "
```

```
 errorHandler: anErrorHandler during: aBlock		
 	"Uses a given errorHandler, during the execution of the given block "
```
 poolWorkerProcess: anObject during: aBlock			
	"Uses a given Pool-Worker process, during the execution of the given block "
```

```
process: anObject during: aBlock					
	"Uses a given process, during the execution of the given block "
```

```
processProvider: aProcessProvider during: aBlock	
	" Uses a given Process provider, during the execution of the given block "
```

```
serviceManager: aManager during: aBlock			
	" Uses a given Service manager, during the execution of the given block "
```

An example of usage

```smalltalk
future := TKTConfiguration>>profileNamed: #test during: [ [2 + 2 ] future ]
```



### Conclusion


