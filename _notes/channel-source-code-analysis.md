---
layout: default
title: Channel Underlying Implementation with Source Code
---


# Channel Underlying Implementation with Source Code

- **Creating Channel - make()**
    - `make()` handles the actual creation and initialization of the channel
    - **1. Parameter Validation**
        - Function checks if the parameters (element type and buffer size) are valid
    - **2. Memory Allocation**
        - **Unbuffered Channel**
            - Allocate just `hchan` structure
        - **Buffered Channel**
            - Allocate `hchan` structure and the buffer
            - Buffer: ring buffer
    - **3. Initialize `hchan` Structure**
        - `buf`: pointer to buffer
        - `qcount`: number of elements in the buffer
        - `elemsize`: size of each element
        - `elemtype`: type of each element
        - `dataqsiz`: size of the buffer
        - `sendx`: send index
        - `recvx`: receive index
        - `sendq`: goroutine queue, waiting for send
        - `recvq`: goroutine queue, waiting for receive

    ```go
    package main

    import "fmt"

    func main() {
        // Create an unbuffered channel
        unbuffered := make(chan int)
        fmt.Println("Unbuffered Channel:", unbuffered)

        // Create a buffered channel with a capacity of 3
        buffered := make(chan int, 3)
        fmt.Println("Buffered Channel:", buffered)
    }
    ```

    - **Further Explanations on Waitq**
        - **Channel Blocking Simple Explanation**
            - **Send Goroutine Being Blocked**
                - Buffer full
                - Or no receiver
            - **Receive Goroutine Being Blocked**
                - Empty buffer
                - Or no sender
        - **Channel Blocking with Explanation of Wait Queues (FIFO)**
            - `recvq`: hold goroutines that are waiting to receive data from buffer
                - If buffer is empty, the goroutine which tries to receive data will be added to `recvq` and be blocked until data is available
            - `sendq`: hold goroutines that are waiting to send data to buffer
                - If buffer is full, the goroutine which tries to send data will be added to `sendq` and be blocked until space is available

- **Sending Data - chansend()**
    - Check if channel is closed
    ```go
    func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
        // Lock the channel to ensure thread safety
        lock(&c.lock)

        // Check if the channel is closed
        if c.closed != 0 {
            unlock(&c.lock)
            panic(plainError("send on closed channel"))
        }
    ```
    - **Scenario 1: If there is a receiver waiting, send data to it**
    ```go
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    ```
        - **send function**
            - Copies the data from sender (`ep`) to the memory location where the receiver is expecting the data (`sg.elem`)
            - Both `sendx` and `recvx` will be incremented
    - **Scenario 2: Ring buffer isn't full, there is no receiver**
    ```go
    if c.qcount < c.dataqsiz {
        // The current sendx is the index of ring buffer where we want to store the data
        qp := chanbuf(c, c.sendx) // Mem location of ring buffer[sendx]
        typedmemmove(c.elemtype, qp, ep) // Copy from sender ep to memory location qp
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0 // Ring buffer
        }
        c.qcount++
        unlock(&c.lock) // We locked and now unlock
        return true
    }
    ```
    - **Scenario 3: Ring buffer is full && send/receive operation will return immediately if it cannot proceed instead of waiting until it can proceed**
        - It is not common, but it will happen when we are using `select`
    ```go
    if !block {
        unlock(&c.lock)
        return false
    }
    ```
    - **Scenario 4: Ring buffer is full and send operation is blocked**
    ```go
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    mysg.elem = ep // Data to be sent
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg) // Enqueue the sender
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    ```
        - Adding current goroutine to `sendq` and block it until it's being woken up by `recv` (there is a receiver), and allowing other goroutines to run
        - The remaining part of the code is to clean up the waiting states and return true indicating the data transfer is done / data is sent
    ```go
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    gp.activeStackChans = false
    if gp.param == nil {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true
    }
    ```

- **Receiving Value from Channel - chanrecv()**
    - `chanrecv1` - `v := <-c`, `chanrecv2` - `v, ok := <-c`
    - Check if channel closed
    ```go
    func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
        if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
            c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
            atomic.Load(&c.closed) == 0 {
            return
        }
    ```
    - Then lock the channel
    ```go
    lock(&c.lock)
    ```
    - **Scenario 1: If the channel is closed and empty, return immediately**
    ```go
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
    ```
    - **Scenario 2: If there is a sender waiting, directly receive the data from the sender**
    ```go
    if sg := c.sendq.dequeue(); sg != nil {
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }
    ```
        - **recv()** - (include the above mentioned when we `goready` the blocked send goroutine)
    ```go
    func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
        // ... (data transfer logic)
        gp := sg.g // Sender goroutine
        unlockf()
        gp.param = unsafe.Pointer(sg)
        goready(gp, skip+1) // Wake up the blocked sender
    }
    ```
        - Both `sendx` and `recvx` will be incremented
    - **Scenario 3: If ring buffer isn't empty, read data from buffer**
    ```go
    if c.qcount > 0 {
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    ```
    - **Scenario 4: If ring buffer is empty and non-blocking (e.g., buffer is empty and read operation immediately returns false without waiting) (again, normally triggered by `select`)**
    ```go
    if !block {
        unlock(&c.lock)
        return false, false
    }
    ```
    - **Scenario 5: Ring buffer is empty and blocked**
    ```go
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.recvq.enqueue(mysg) // Enqueue to recvq
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
    ```
        - Adding current goroutine to `recvq` and block it until it's being woken up by `send` (there is a sender), and allowing other goroutines to run
        - The remaining part of the code is to clean up the waiting states and return true indicating the data transfer is done / data is received
    ```go
    // After being unblocked
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    gp.activeStackChans = false
    if gp.param == nil {
        if c.closed == 0 {
            throw("chanrecv: spurious wakeup")
        }
        panic(plainError("receive on closed channel"))
    }
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, true
    }
    ```