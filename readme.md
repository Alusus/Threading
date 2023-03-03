# Threading
[[عربي]](readme.ar.md)

Provides threading and thread synchronization functionality.

## Adding to the Project

Use the following lines:

```
import "Apm";
Apm.importFile("Alusus/Threading");
```

## Functions

### createThread
```
    func createThread(
      pthread: ptr[Thread], attr: ptr[ThreadAttributes], startRoutine: ptr[func (ptr): ptr], arg: ptr
    ): Int;
```
A function to create a thread. This function is equivalent to Posix `pthread_create`.

Arguments:

* `pthread`: A pointer to a variable of type Thread, in which the result is stored.
* `attr`: The attributes we want for the thread. Passing null indicates default attributes.
* `startRoutine`: A pointer to a function that will be executed on this thread.
* `arg`: An arbitrary pointer to pass to the function in the preceding argument. This is used to pass user data to the
  thread.

Return Value:

Returns 0 on success, an error code otherwise.

### joinThread
```
    func joinThread(pthread: ptr[Thread], retval: ptr[ptr]): Int;
```
Waits for a thread to finish execution. This function is equivalent to Posix `pthread_join`.

Arguments:

* `pthread`: A pointer to the thread to wait for.
* `retval`: A pointer to store the result of executing the given thread's function. This will contain the value
  returned by the thread function.

Return Value:

Returns 0 on success, an error code otherwise.

### initMutex
```
    func initMutex(mutex: ptr[Mutex], attrs: ptr[MutexAttributes]): Int;
```
Initializes a mutex lock. A mutex must be initialized by this function before it can be used. This function is
equivalent to Posix `pthread_mutex_init`.

Arguments:

* `mutex`: A pointer to the Mutex object.
* `attrs`: Attributes for this mutex. Passing null indicates default attributes.

Return Value:

Returns 0 on success, an error code otherwise.

### lockMutex
```
    func lockMutex(mutex: ptr[Mutex]): Int;
```
Locks the mutex object. If the mutex is locked by another thread the current thread will be paused until the mutex is
available. This function is equivalent to Posix `pthread_mutex_lock`.

Arguments:

* `mutex`: A pointer to the Mutex object.

Return Value:

Returns 0 on success, an error code otherwise.

### tryLockMutex
```
    func tryLockMutex(mutex: ptr[Mutex]): Int;
```
Locks the mutex object. If the mutex is locked by another thread the function returns immediately with an error code.
This function is equivalent to Posix `pthread_mutex_trylock`.

Arguments:

* `mutex`: A pointer to the Mutex object.

Return Value

Returns 0 on success, an error code otherwise.

### unlockMutex
```
    func unlockMutex(mutex: ptr[Mutex]): Int;
```
Unlocks a mutex. This function is equivalent to Posix `pthread_mutex_unlock`.

Arguments:

* `mutex`: A pointer to the Mutex object.

Return Value

Returns 0 on success, an error code otherwise.

### initCond
```
    func initCond(cond: ptr[Cond], attrs: ptr[CondAttributes]): Int;
```
Initializes a condition object. A condition must be initialized by this function before it can be used.
This function is equivalent to Posix `pthread_cond_init`.

Arguments:

* `cond`: A pointer to the Cond object.
* `attrs`: Attributes for this condition. Passing null indicates default attributes.

Return Value:

Returns 0 on success, an error code otherwise.

### signalCond
```
    func signalCond(cond: ptr[Cond]): Int;
```
Sends a signal that a condition is met. This will unlock one thread that is waiting on this condition. If multiple
threads are waiting for the same condition only one of them will be released per call. Before calling this function
the calling thread must first lock a mutex (the same mutex used by the thread in `waitCond`) and must release the
mutex after the call. The thread calling `waitCond` will only be released after the mutex is released since the 
`waitCond` will attempt to lock the mutex before returning.
This is equivalent to Posix `pthread_cond_signal`.

Arguments:

* `cond`: A pointer to the condition object to signal.

Return Value:

Returns 0 on success, an error code otherwise.

### waitCond
```
    func waitCond(cond: ptr[Cond], mutex: ptr[Mutex]): Int;
```
Waits for a condition to be met. This takes a mutex in addition to the condition to wait for, and it requires that the
mutex is locked before the call. The function will atomically block the thread and release the mutex. When the condition
is signaled the function will atomically re-lock the mutex and release the calling thread.
This function is equivalent to Posix `pthread_cond_wait`.

Arguments:

* `cond`: A pointer to the Cond object to wait on.
* `mutex`: A pointer to a Mutex object. This must be locked before calling this function.

Return Value:

Returns 0 on success, an error code otherwise.

## Types

```
class ThreadAttributes {
    def flags: Int;
    def stackSize: Int;
    def contentionScope: Int;
    def inheritSched: Int;
    def detachState: Int;
    def sched: Int;
    def param: SchedParam;
    def startTime: TimeSpec;
    def deadLine: TimeSpec;
    def period: TimeSpec;
}

class SchedParam {
    def schedPriority: Int;
}

class TimeSpec {
    def tvSec: ArchInt;
    def tvNsec: Int[64];
}

class MutexAttributes {
    def pshared: Int = 0;
    def kind: Int = 0;
    def protocol: Int = 0;
    def robustness: Int = 0;
}

class CondAttributes {
    def dummy: Int = 0;
}
```

## Example

```
import "Srl/Console";
import "Apm";
Apm.importFile("Alusus/Threading");

def mutex: Threading.Mutex;
def sum: Int = 0;
def counter: Int = 0;

func totalSum {
    use Threading;

    initMutex(mutex~ptr, 0);

    def NTHREADS: 10;
    def threads: array[Thread, NTHREADS];
    def i: Int;

    // Create the threads.
    for i = 0, i < NTHREADS, i++ {
        createThread(threads(i)~ptr, 0, calculateSum~ptr, 0);
    }

    // Wait for the threads.
    for j = 0, j < NTHREADS, j++ {
        joinThread(threads(j), 0);
    }

    // Print output
    Srl.Console.print("The result is: %d\n", sum); // 499500
}

func calculateSum(p: ptr): ptr {
    use Threading;

    // Each thread calculates the sum of 100 numbers.
    // The first thread will compute the sum of the numbers from 0 to 100
    // and so on for the rest.

    // Sync with other threads before accessing the global var.
    lockMutex(mutex~ptr);
    def st: Int = counter*100;
    counter++;
    unlockMutex(mutex~ptr);

    def i: Int;
    for i = st, i < st + 100, i++ {
        // Sync with other threads before accessing the global var.
        lockMutex(mutex~ptr);
        sum += i;
        unlockMutex(mutex~ptr);
    }
    return 0;
}

totalSum();
```
