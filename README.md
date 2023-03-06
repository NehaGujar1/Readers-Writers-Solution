# Readers-Writers-Solution
# Readers-Writers Problem

Readers-Writers problem is a classical synchronization problem in OS.
Basic assumptions:
1. Readers and Writers share the same dataset.
2. Readers can't read while writers write and vice versa.
3. Multiple readers can read at the same time.

## Implementation of Semaphore
### Semaphore functions
```
wait(int &s)
{
    while(s<=0);
    s--;
}
signal(int &s)
{
    s++;
}
```

## Solutions to Reader-Writer's Problem
The first and second reader-writer's problems propose solutions that solve the problem, but, there is a chance that the writers may starve in the solution to the first problem and the readers may starve in the solution to the second problem. We include a strave free solution followed by these 2.

### Classical solution (Writers Starve)
This is the solution to the first reader-writer's problem. Here, there is a chance that the writers starve. The global variables (that are shared across all the readers and writers) are as shown below.

#### Global Variables
```
Semaphore mutex = 1;
Semaphore wrt = 1;
int readcnt = 0;
```
#### Psuedocode
```
Reader Process :
    do{
    wait(mutex);
    readcnt++;
    if (readcnt == 1) wait(wrt);
    signal(mutex);
    …
    reading is performed
    …
    wait(mutex);
    readcnt--;
    if (readcnt == 0) signal(wrt);
    signal(mutex);
    } while(true);

Writer Process :
    do{
    wait(wrt);
    …
    writing is performed
    …
    signal(wrt);
    } while(true);
```

In the above case, as it can be clearly seen as long as the readers queue is nonempty writer waits. Thus this leads to starvation.

### Second solution (Readers Starve)
This is the solution to the first reader-writer's problem. Here, there is a chance that the readers starve. The global variables (that are shared across all the readers and writers) are as shown below.

#### Global Variables
```
Semaphore mutex = 1;
Semaphore wrt = 1;
Semaphore writereq = 1;
Semaphore enter = 1;
int readcnt = 0;
```
#### Psuedocode
```
Reader Process :
    do{
    wait(enter);
    wait(mutex);
    readcnt++;
    if (readcnt == 1) wait(wrt);
    if(writereq==0) signal(enter); 
    signal(mutex);
    …
    reading is performed
    …
    wait(mutex);
    readcnt--;
    if (readcnt == 0) signal(wrt);
    signal(mutex);
    }while(true);

Writer Process :
    do{
    signal(writereq);
    wait(wrt);
    …
    writing is performed
    …
    signal(wrt);
    wait(writereq);
    } while(true);
 ```

In the above case, as it can be clearly seen as long as the writers queue is nonempty reader waits. Thus this leads to starvation.

### Starve Free Solution

#### Global Variables
```
Semaphore newEntrant=1;
Semaphore mutex=1;
Semaphore wrt=1
int readcnt=0;
queues: queuewait, readtrue with push_back() and pop_front() standard functionalities
```

#### Psuedocode
```
Reader process:
do {
    wait(newEntrant);
    //pushes process id into the queue
    queuewait.push_back(pid);
    //pushes 1 for read
    readtrue.push_back(1);
    signal(newEntrant);
    wait(mutex);
    //in case a few readers are already reading before letting a new process read we must block it 
    //to check if the next process is a write process and give it priority accordingly
    if(readcnt>0) {
      signal(mutex);
      block(pid);
      wait(mutex);
    }
    readcnt++;
    //as the 1st reader enters no writer should write
    if(readcnt==1) wait(wrt);
    signal(mutex);
    wait(newEntrant);
    //as the read processes are no more waiting we remove them from the queue
    queuewait.pop_front();
    readtrue.pop_front();
    //if the next process in the queue is read we do wakeup call
    if(readtrue.size()>0&&readtrue[0]==1) wakeup([queuewait[0]]);
    signal(newEntrant);
    //read
    wait(mutex);
    readcnt--;
    //as all the readers leave the queue we can let the new writer/reader process execute
    if(readcnt==0) {
      wait(newEntrant);
      if(queuewait.size()>0) wakeup([queuewait[0]]);
      signal(newEntrant);
      signal(wrt);
    }
    signal(mutex);
} while(true)

Writer Process:
do
{
    //stores size
    int size;
    wait(newEntrant);
    //pushes process id into the queue
    queuewait.push_back(pid);
    //pushes 0 for write
    readtrue.push_back(0);
    //stores the size of queuewait immidiately after pushing current process
    size = queuewait.size();
    //current process is blocked if either of previous read or write processes is executing
    if(size!=1) block(pid);
    //critical section
    wait(wrt);
    //write
    signal(wrt);
    //once execution is complete the process is popped 
    queuewait.pop_front();
    readtrue.pop_front();
    //if any process is waiting for execution be it read or write can resume by wakeup call
    if(queuewait.size()>0) wakeup(queuewait[0]);
    signal(newEntrant);
} while (true);
```

#### Logic
- We have basically maintained 2 queues: `queuewait` and `readtrue` which store process id and boolean value (1-read and 0-write). We are ensuring FCFS for the processes which ultimately removes starvation.
- `newEntrant` is used to ensure mutual exclusion of `queuewait` and `readtrue`.
- `mutex` is used for mutual exclusion of `readcnt` and `wrt` for critical sections of read and write processes

##### Logic for the writers
- As a writer enters the queue, we check whether all the previous readers have completed reading or not by checking the value of `wrt` semaphore which ensures mutual exclusion between read and writers also by the same.
- Post one writer completes it's execution (only one writer can write at a time) the next process in the queuewait is released by wakeup call.

##### Logic for the readers
- As a new reader enters, if no reader is previously present then it completes the `Entry section` and waits for signal to enter the `Critical Section` if any writer is executing.
- If any reader is previously present then it is blocked and released after checking whether any writer has requested before the reader if so then it won't be released by reader but later by the writer after it completes its execution else it will ba released and can simultaneously read with other readers.

## Correctness of the starve-free solutions

### Mutual Exclusion
- It allows only one process at a time in the critical section for a resource.
- In strave free solution this is ensured by clever use of semaphores `newEntrant`, `mutex` and `wrt` for `queuewait` and `readtrue` queues, `readcnt` and critical section respectively.

### Progress
- It means that if a process doesn't need to execute into critical section then it should not stop other processes to get into the critical section.
- The structure of the algorithm is such that if one process doesn't enter the critical section it doesn't stop any other process from entering the critical section.

### Bounded Waiting
- It implies that every process must have limited waiting period. It mustn't endlessly wait to enter the critical section.
- Use of FCFS priciple ensures that the process has to wait for finite interval i.e. till the processes that arrived before have completed their execution. (all processes are assumed to have finite execution time).
