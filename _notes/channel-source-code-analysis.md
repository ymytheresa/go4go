# channel underlying implementation with source code   

- Creating channel - make()   
    - make() handles the actual creation and initialization of the channel   
    - 1 Parameter validation   
        - function checks if the parameters (element type and buffer size) are valid   
    - 2 Memory allocation   
        - Unbuffered channel   
            - allocate just hchan structure   
        - Buffered channel   
            - allocate hchan structure and the buffer  
            - buffer : ring buffer   
    - 3 Initialize hchan structure   
        - buf : pointer to buffer   
        - qcount : number of element in the buffer   
        - elemsize : size of each element   
        - elemtype : type of each element   
        - dataqsiz : size of the buffer   
        - sendx : send index   
        - recvx : receive index   
        - sendq : go routine queue, waiting for send   
        - recvq : go routine queue, waiting for receive   
   
    ```
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
    - Further explanations on waitq   
        - Channel blocking simple explanation   
            - send goroutine being blocked    
                - buffer full   
                - or no receiver   
            - receive goroutine being blocked   
                - empty buffer   
                - or no sender   
        - Channel blocking with explanation of wait queues (FIFO)   
            - recvq : hold goroutines that are waiting to receive data from buffer   
                - if buffer is empty, the goroutine which tries to receive data,  will be added to recvq, and be blocked until data is available   
            - sendq : hold goroutines that are waiting to send data to buffer   
                - if buffer is full, the goroutine which tries to send data, will be added to sendq and be blocked until space is available   
- Sending data - chansend()   
    - Check if channel is closed   
        ```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // Lock the channel to ensure thread safety
    lock(&c.lock)

    // Check if the channel is closed
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

```
    - Scenario 1: If there is receiver waiting, send data to it   
        ```
if sg := c.recvq.dequeue(); sg != nil {
	send(c, sg, ep, func() { unlock(&c.lock) }, 3)
	return true
}
```
        - send function   
            - copies the data from sender(ep) to the memory location where the receiver is expecting the data (sg.elem)   
            - both sendx and recvx will be incremented.   
    - Scenario 2: ringbuffer isnt full, there is no receiver.   
        ```
if c.qcount < c.dataqsiz {
	//the current sendx is the index of ringbuffer where we want to store the data
	qp := chanbuf(c, c.sendx) // mem location of ringbuffer[sendx]
	typedmemmove(c.elemtype, qp, ep) //copy from sender ep to memory location qp
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0 //ringbuffer
	}
	c.qcount++
	unlock(&c.lock) //we locked and now unlock
	return true
}
```
    - Scenario 3: ringbuffer is full &&  send/receive operation will return immediately if it cannot proceed instead of wait until it can proceed.
scenario that if the ringbuffer is full and a send operation on a full buffer will return false without waiting.so it indicates the send/receive operation could not be completed.   
    - it is not common. but it will happened when we are using select.   
        ```
if !block {
	unlock(&c.lock)
	return false
}
```
    - Scenario 4: ringbuffer is full and send operation is blocked   
        ```
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
mysg.elem = ep //data to be sent
mysg.waitlink = nil
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.waiting = mysg
gp.param = nil
c.sendq.enqueue(mysg) //enqueue the sender
gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
```
        - adding current goroutine to sendq and block it until its being waken up by recv (there is receiver), and allowing other goroutines to run   
        - Cont, ringbuffer isnt full now and the above send operation is resumed. BUT. The data transfer isnt continued in here. The process is done at recv() (will cover later) and it will wake up this blocked goroutine to continue the following code.   
        - the remaining part of code is to clean up the waiting states and return true indicating the data transfer is done / data is sent.   
   
        ```
if mysg != gp.waiting {
	throw("G waiting list is corrupted")
}
gp.waiting = nilin
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
           
- Receiving value from channel - chanrecv()   
    - chanrecv1 - v := c ←, chanrecv2 - v, ok := c ←   
    - Check if channel closed   
        ```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

```
    - Then lock the channel   
        ```
lock(&c.lock)
```
    - Scenario 1: If the channel is closed and empty, return immediately   
        ```
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
    - Scenario 2: If there is a sender waiting, directly receive the data from the sender   
        ```
if sg := c.sendq.dequeue(); sg != nil {
	recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
	return true, true
}
```
        - recv() - (include the above mentioned when we goready the blocked send goroutine)   
            ```
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // ... (data transfer logic)
    gp := sg.g //sender goroutine
    unlockf()
    gp.param = unsafe.Pointer(sg)
    goready(gp, skip+1) // Wake up the blocked sender
}
```
            - both sendx and recvx will be incremented   
    - Scenario 3: If ring buffer isnt empty, read data from buffer   
        ```
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
    - Scenario 4: If ring buffer is empty and non-blocking (eg buffer is empty and read operation immediately returns false without waiting) (again, normally triggered by select)   
        ```
if !block {
	unlock(&c.lock)
	return false, false
}

```
    - Scenario 5: Ring buffer is empty and blocked   
        ```
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
c.recvq.enqueue(mysg) //enqueue to recvq
gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
```
        - adding current goroutine to recvq and block it until its being waken up by send (there is sender), and allowing other goroutines to run   
        - Cont, ringbuffer is not empty now and the above recv operation is resumed. BUT. The data transfer isnt continued in here. The process is done at send() and it will wake up this blocked goroutine to continue the following code.   
        - the remaining part of code is to clean up the waiting states and return true indicating the data transfer is done / data is received   
   
        ```
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
   
       
   
   
