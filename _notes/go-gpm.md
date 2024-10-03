---
layout: default
title: GO GPM model
---

# Go GPM   
   
### Definition of G, P, M   
- G : Goroutine, light weight thread created and managed by go runtime   
- P : Processor, setting up the goroutine stack and states inside goruntime internal memory for executing the goroutine, and assigning available M to this G. The stack size default to be 2KB.   
- M : OS thread, kernel thread created by go runtime. M execute the machine code of G scheduled by P, using the context provided by P.   
   
   
GPM scheduler model   
![image.png](https://ymytheresa.github.io/go4go/assets/image1.png)    
![image.png](https://ymytheresa.github.io/go4go/assets/image_m1.png)    
- Hands Off Mechanism   
    - When a G is being blocked (eg waiting for io response), the G and M is separated from P. P will assign new G and available M or create new M.   
    - Noted that the states are always stored in go runtime internal memory. No data transferred happened.   
- Preemption   
    - The idea is to prevent starvation of goroutine by limiting goroutine occupying cpu for too long.   
    - There is a sysmon() that wakes up periodically to check if any P is idle or any goroutine still running.   
        - P is idle - assign another goroutine / releasing P back to the OS if nth to do   
        - Goroutine running > 10ms might be preempted and alloc another goroutine to be executed.   
            - 10ms?   
            - sysmon() wakes up periodically, starting by 20 microsecond. If there are no active goroutine then it will double the durable to at most 10ms.    
                - if wakes up at 20microsecond and found goroutine still running it wont preempt it, the threshold is at 10ms.   
- Global G Queue   
    - When an M attempts to steal goroutine from other P local queue but cant find any, it can retrieve goroutine from global G queue   
- Global Queue : goroutines to be executed   
    - go func() Scheduling process   
        - when go func() is executed, this newly created goroutine will first attempt to store in local P queue first, if all local P queues are full it will be stored in the global queue   
- Local queue of P  : Local queue of goroutines that to be executed by its P   
- P : Processor. The number of P is set in $GOMAXPRO   
    - it means how many processor can execute goroutine simutaneously. Theoretically we can set this number to above #cores, but it does not make sense since it will induce P creation and destructions and so. Go runtime will set the amount of P to GOMAXPRO > #core ? #core ; GOMAXPRO   
- M : Kernel thread that executing G that assigned by P.   
    - Amount of M : when program starts, max is set to be 10000. But can be disregarded using SetMaxThreads()   
    - Assignment of P (the goal is to reduce the creation and termination of M, try to reuse the M)   
        - Work stealing mechanism   
            - when local P queue is empty, P will steal goroutine from other local P's queue and assign to M   
   
   
Initial M0 and G0   
![image.png](https://ymytheresa.github.io/go4go/assets/image_u1.png)    
- When program launches, M0 is created to execute initialization operations and launching the first G0. After launching the first G0, M0 functions like any other M   
- Each time a M is started, the first goroutine created is G0, so each M has its own G0. G0 is solely dedicated to scheduling tasks and doesnt point to any executable function. During scheduling or system called, M switches from any G to G0 and scheduling is performed through G0. M0's G0 is placed in global space.   
![image.png](https://ymytheresa.github.io/go4go/assets/image_s1.png)    
   
   
![image.png](https://ymytheresa.github.io/go4go/assets/image_e1.png)    
 --- 
### Scenario 1: G1 Creates G2   
When Goroutine G1 creates G2, G2 is prioritized and added to the local queue of Processor P1. This prioritization is due to the locality principle, which keeps related tasks close together in memory, making context switching faster and more efficient.   
### Scenario 2: G1 Execution Completed   
After G1 finishes executing, the machine (M1) that was running it switches to G2, which is in P1's local queue. This demonstrates the reusability of machines in the GPM model, as M1 can immediately take on another task without waiting for new resources.   
### Scenario 3: G2 Creates Excessive Gs   
If G2 tries to create 6 new Goroutines but the local queue can only hold 4, only the first 4 (G3, G4, G5, G6) are added to the queue. The excess Goroutines (G7, G8) are not added to the local queue. This scenario highlights that when too many Goroutines are created, the local queue can become full, and the order of execution may be disrupted since G7 and G8 will need to be handled differently in the next scenario.   
### Scenario4: G2 Locally Full, Creating G7   
When G2 attempts to create G7 and finds that P1's local queue is full, it triggers a load balancing algorithm. This algorithm moves the first half of the Goroutines (G3, G4) from P1's local queue to the global queue, along with the newly created G7. As a result, the order of the Goroutines is disrupted: G3 and G4 are now in the global queue, while G5 and G6 remain in P1's local queue. This scenario illustrates how the system manages overflow and maintains efficiency, but it also shows that the sequence of execution can be affected.   
### Scenario 5: G2 Local Queue Not Full, Creating G8   
When G2 creates G8 and the local queue of P1 is not full, G8 is added to the local queue. While G8 is placed at the end of the queue, it is considered a "top priority" in the sense that it is the most recent task created and will be executed as soon as the currently running Goroutine (G2) completes. However, it will indeed be the last to be executed in the order of the queue. The existing Goroutines already in the queue will still be executed first, so after G2 finishes, the next scheduled Goroutine will be G5, followed by G8.   
### Scenario 6: Waking Up a Sleeping M   
In the GPM model, when G2 creates a new Goroutine, it can wake up idle machines (M) that are in a sleeping state. This means that if there are surplus machines from previous operations, they can be activated to handle new tasks, ensuring that resources are utilized effectively.   
### Scenario 7: M2 Awoken Fetches a Batch of Gs from the Global Queue   
When M2 wakes up, it attempts to fetch a batch of Goroutines from the global queue (GQ) to place them in its local queue (LQ). The number of Goroutines fetched is determined by a formula that balances the load, ensuring that not too many are taken at once. This helps maintain efficiency and prevents overwhelming any single processor.   
### Scenario 8: M2 Steals from M1   
If M2 is idle and there are no tasks in its local queue, it can "steal" Goroutines from another processor (P1) that has tasks. This allows M2 to continue working and prevents idle resources, ensuring that all available Goroutines are executed efficiently.   
### Scenario 9: Maximum Limit of Spinning Threads   
The GOMAXPROCS variable sets a limit on how many processors can be active at once. If there are more machines than tasks, the extra machines go idle (spinning). This is done to prevent wasting CPU resources while still keeping machines ready to run new Goroutines as they become available.   
### Scenario 10: Blocking System Call in G   
If a Goroutine (G8) makes a blocking system call, it releases its machine (M2) to work on other tasks while waiting. This allows the system to continue processing other Goroutines without being held up by the blocking call.   
### Scenario 11: G undergoes a non-blocking system call   
After G8 finishes a non-blocking call, M2 tries to reacquire its previously bound processor (P2). If P2 is busy with another machine (M5), M2 cannot bind to it. If no idle processors are available, G8 is marked as runnable and added to the global queue for later execution. M2 then enters a sleeping state, waiting for a processor to become available.   
