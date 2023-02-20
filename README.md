# Reader-Writer-Problem
The Reader-Writer problem is a classic synchronization problem in computer science, which involves multiple processes (or threads) accessing a shared resource. In this problem, multiple readers can access the shared resource simultaneously, but only one writer can access it at a time. The objective is to ensure that the access to the shared resource is controlled in a way that maintains data consistency while allowing maximum concurrency.

The problem arises when there is a shared resource (such as a file or database) that needs to be accessed by multiple processes, some of which only want to read the resource while others want to modify it. Readers only need to access the resource for reading, and they do not modify it, while writers need exclusive access to the resource for writing.

The problem is difficult because allowing concurrent access to the shared resource can result in data inconsistency. For example, if two writers access the shared resource simultaneously, they may overwrite each other's changes, leading to incorrect data. Similarly, if a reader and a writer access the shared resource simultaneously, the reader may see the old data while the writer is in the process of updating the resource, leading to inconsistent data.


## Classical Solution
The classical solution to the reader-writer problem is a synchronization mechanism that uses two variables to track the number of readers and writers currently accessing the shared resource. This solution ensures that writers have exclusive access to the resource, while allowing multiple readers to access the resource concurrently.

The two variables used in the classical solution are:

1. `readers_count`: This variable tracks the number of readers currently accessing the shared resource.

2. `writers_count`: This variable tracks the number of writers currently accessing the shared resource.

The solution involves implementing the following rules:

1. When a writer wants to access the shared resource, it must wait until there are no readers or writers currently accessing the resource. The writer increments the `writers_count` variable to indicate that it wants to access the resource, and then waits until `readers_count` is zero.
2. When a reader wants to access the shared resource, it can do so if there are no writers currently accessing the resource. The reader increments the `readers_count` variable to indicate that it wants to access the resource and then proceeds to access the resource.
3. When a reader is finished accessing the shared resource, it decrements the `readers_count` variable.
4. When a writer is finished accessing the shared resource, it decrements the `writers_count` variable.

Although the classical solution to the reader-writer problem is effective in ensuring mutual exclusion between readers and writers, it has several drawbacks:

1. Starvation: The classical solution can result in starvation of writers if there are always readers accessing the resource. Since readers can access the resource concurrently, they may hold onto the resource for an extended period, preventing writers from accessing it.
2. Writer Priority: In the classical solution, there is no priority given to writers over readers. If a writer is waiting to access the resource, and a new reader arrives, the reader may be allowed to access the resource first, further delaying the writer.
3. Performance: The classical solution can result in decreased performance in situations where there are a large number of readers and writers. Since the solution allows multiple readers to access the resource concurrently, it can lead to a large number of context switches and reduced performance.
4. Deadlock: The classical solution can lead to a deadlock if a writer is waiting to access the resource, and a reader is holding onto the resource indefinitely. In this situation, the writer will never be able to access the resource, resulting in a deadlock.


## Starve Free Solution
To solve the reader-writer problem, a synchronization mechanism is required that allows multiple readers to access the shared resource simultaneously while ensuring that only one writer can access it at a time. A semaphore can be used to control access to the shared resource. A writer acquires the semaphore and blocks other writers and readers from accessing the resource while it is being modified. Readers can acquire the semaphore to read the resource, but they do not block other readers. Here is the implementation of the same.
```C++
struct runningProcess { //Basically represents the PCB of a given process
	runningProcess* next;
	int processID;
	bool isItActive = true;
	// PCB contains lots of other things too like program counter, registers but
	// these are not required for the implementation
};

class QueueOfBlockedProcesses {
	int size = 0; //current size of the Queue
	int MaxSizeOfQueue = 200; //max size of the queue, any large number could work
	runningProcess* start, end, nullProcess; //2 pointers start and end determining the
	// start and end positons of the queue.
	// initialization of a nullProcess which will be returned when pop is called
	// on a empty Queue
	nullProcess ->next = nullptr;
	nullProcess ->processID = -1;
	nullProcess ->isItActive = false;

	public: 
		void push (int PID) {
			if (size == MaxSizeOfQueue) {
				return;
			}
			runningProcess* P;
			P ->processID = PID;
			P ->isItActive = false;
			// after pushing the process in the Queue, the state of 
			// the process is set to inactive
			size++;
			if (size == 1) {
				start = P;
				end = P;
				return;
			}
			end ->next = P; // updating the end pointer
			end = P;
		}

		runningProcess* pop() {
			if (size == 0) {
				return nullProcess;
			}
			size--;
			runningProcess* X = front; // updating the front pointer
			front = front ->next;
			return X;
		}
};

struct semaphore() {
	int value;
	QueueOfBlockedProcesses* Q = new QueueOfBlockedProcesses();
	// Each semaphore is associated with a Queue of blocked processes

	semaphore(int x) {
		value = x;
	}
};
```
Initialization of the semaphores will be done as following
```C++
// Initializing and declaring semaphores
semaphore* mutexForEntry = new semaphore(1);
//this semaphore ensures that readers and writers have equal priority
semaphore* mutexForExit = new semaphore(1);
// after reader's role in critical section is completed, it checks whether a writer is waiting
// and the last reader signals the writer that the critical section is now available 
semaphore* mutexToWrite = new semaphore(0);
// used to write in the critical section

bool isWriterWaiting = false;
int numberOfEntries = 0, numberOfExits = 0;
```
The proposed solution to the reader-writer problem involves a Writer notifying Readers of its need to access a shared resource. While the Writer is waiting for access, no new Readers can begin accessing the resource. As each Reader completes its access to the resource, it checks if there is a Writer waiting. If so, the last Reader to leave the resource signals the Writer that it is safe to proceed. Once the Writer has completed its access to the resource, it signals waiting Readers that it has finished, allowing them to access the resource again. This solution ensures that Writers have exclusive access to the resource while minimizing the waiting time for Readers.
```C++
void StarveFreeReaderProcess(int processID) {
	wait(mutexForEntry, processID);
	numberOfEntries++;
	signalProcess(mutexForEntry);
	/**

	CRITICAL SECTION

	**/
	wait(mutexForExit, processID);
	numberOfExits++;
	// after the reader has finshed its work in the critical section, it increments
	// the count of exits and if no other readers are currently active in the
	// critical section and there is/are writers waiting, then it signals the writer
	// to start its work in the critical section

	if (isWriterWaiting && numberOfEntries == numberOfExits) {
		signalProcess(mutexToWrite);
	}

	signalProcess(mutexForExit);
}

void StarveFreeWriterProcess(int processID) {
	wait(mutexForEntry, processID);

	wait(mutexForExit, processID);
	// to make sure that numberOfExits do not change
	// while this executes

	if (numberOfExits == numberOfEntries) {
		signalProcess(mutexForExit);
		// if there is no reader in the critical section, then writer enters
		// the critical section
	}else {
		//if readers are present in the critical section,
		// writer waits
		isWriterWaiting = true;
		signalProcess(mutexForExit);
		wait(mutexToWrite, processID);
		isWriterWaiting = false;
	}
	/**

	CRITICAL SECTION

	**/
	signalProcess(mutexForEntry);
}
```
This solution allows multiple reader processes to enter a critical section and read data at the same time, while only one writer process is allowed to enter and modify data. When a reader process wants to enter the critical section, it acquires a semaphore, increases the count of the number of readers, and enters the critical section. If another reader process comes up, it does the same thing and enters the critical section as well. If a writer process comes up, it gets queued and has to wait until there are no more readers in the critical section. Once it enters the critical section, it can modify the data as needed.

When a reader process finishes, it decreases the count of the number of readers, and signals the semaphore. If there are no more reader processes in the critical section or a writer process is waiting to enter, the reader process signals the writer semaphore to let the writer process enter the critical section. When a writer process finishes, it releases the semaphore so that another process can enter.

This solution ensures that both readers and writers have access to the critical section without starving each other, and only requires one semaphore access for each reader process entering the critical section, making it faster.
