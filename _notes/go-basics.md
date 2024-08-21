---
layout: default
title: Go Basics
---

# Go Basics

## Basic Syntax

### Type Conversion
- float to int ← truncate
- int to string
  - string(int) - ascii table
  - strconv.Itoa(int)
- string to int
  - strconv.Atoi(string)

### String Concat
- Can concat with string type only, if concat int need to convert int to string first
- Using +=
  - time complexity O(n)
  - space complexity O(n^2)
    - since each time has to copy the previous string to new space
- Using strings.Builder.WriteByte
  - time complexity also O(n)
    - but it's way faster than +=
  - reason: a bigger byte array assigned for storing the additional characters, it will be allocated and copied to new byte array with double size if the current byte array is full
  - only until string() is called, there is no string created to store the string form. Just created a 'string type pointer'

### Compilation
- Only need the executable to run the program so it's not need to install interpreter in client machine to run the program

### Memory Usage Compared to Java
1. Memory Management:
   - Java: Uses automatic garbage collection, which can lead to higher memory overhead. Java stores all objects on heap, even small ones. Primitive data type variables and function calls stored on stack.
   - Go: Uses a combination of stack allocation and garbage collection, often resulting in lower memory usage. Go's compiler performs Escape analysis to decide if variable can be allocated on stack or heap.

2. Object Model:
   - Java: Everything (except primitives) is an object, which adds memory overhead.
   - Go: Has value types and doesn't require everything to be an object.

3. Runtime:
   - Java: JVM (Java Virtual Machine) requires additional memory.
   - Go: Compiles to native code, avoiding the need for a VM.

4. Data Structures:
   - Java: Often uses more memory-intensive data structures by default.
   - Go: Tends to use more memory-efficient structures. Go runtime is used to perform GC during runtime

5. Type System:
   - Java: Uses reference types for non-primitives, adding indirection.
   - Go: Uses value types more extensively, reducing indirection.

### Constant
- Cannot be more complex types like slices, maps and structs
- Cannot assign the value that can only be computed during runtime e.g., time.Now()

```go
timevar := time.Now()
const test = timevar.Truncate(time.Hour)
//can't compile too
```

### Runes
- byte: 8bits
- rune: 32bits, although in Go string it's byte array, the number of rune = number of character, and len(string) isn't always equal to number of character. Each character costs 1 rune, so it can support emoji and Chinese characters and all other characters

![iota example](https://ymytheresa.github.io/go4go/assets/image_1y.png)
![image.png](/assets/image_1y.png)

- Rune vs byte
  - `r := []rune(word)` ← expensive operation to convert string to rune slice. Cost O(N), N is number of byte. Use `for range` to get rune at index is less expensive.
- Parsing string using for… range, each value is a rune

```go
import "unicode"

func isValidPassword(password string) bool {
    up, dig := false, false
    for _, c := range password {
        if unicode.IsUpper(c) {
            up = true
            continue
        }
        if unicode.IsDigit(c) {
            dig = true
            continue
        }
    }

    return (up && dig) && (up == true)
}
```

### Passing Variables by Value
- Variables in Go are passed by value (except for a few data types we haven't covered yet). "Pass by value" means that when a variable is passed into a function, that function receives a copy of the variable. The function is unable to mutate the caller's original data.

### Return Value
- Ignoring return value
  - `x, _ = call(a, b)`
    - Ignored the second returned value, since if a declared variable not being used we will get compilation error.
- Named return values
  - Used only in short function, if decided to do naked return

```go
func getCoords() (x, y int) {
  // x and y are initialized with zero values
  x -= 10
  y += 10
  
  return // automatically returns x and y, -10, 10
}
```

- Good for documenting in longer functions with many return values - explicit return

```go
func calculator(a, b int) (mul, div int, err error) {
    if b == 0 {
      return 0, 0, errors.New("Can't divide by zero")
    }
    mul = a * b
    div = a / b
    return mul, div, nil
}
```

(nil = zero value of an error)

### Defer
- Will execute the line of code just before any return of the function

### Variable Scope
- The scope is within block, aka {}

### Closure (stateful function to my understanding)
- Doesn't just return the internal state - it maintains and can modify that state over multiple invocations.

```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

func main() {
    increment := counter()
    fmt.Println(increment()) // 1
    fmt.Println(increment()) // 2
    fmt.Println(increment()) // 3

    newIncrement := counter()
    fmt.Println(newIncrement()) // 1
}
```

### Functions
- Can't overload (same function name diff parameter)
  - Does not support function overload
  - Approach using empty interface

```go
type Calculator struct{}

func (c Calculator) Add(a, b interface{}) float64 { // any type a,b will be accepted
    switch a.(type) {
    case int:
        return float64(a.(int) + b.(int))
    case float64:
        return a.(float64) + b.(float64)
    default:
        return 0
    }
}
```

- Empty interface: interface type with no methods
  - Since every type implements at least zero methods, every type satisfies the empty interface. This means interface{} can hold values of any type
- a.(int) is a type assertion, asserting that a contains an int and tries to extract that int
  - If type assertion failed and error not handled properly, will cause panic

#### Function as Values (passing function as parameter to another function)

```go
func add(x, y int) int {
    return x + y
}

func mul(x, y int) int {
    return x * y
}

func aggregate(a, b, c int, arithmetic func(int, int) int) int {
  firstSum := arithmetic(a, b)
  secondSum := arithmetic(firstSum, c)
  return secondSum
}

func main() {
    sum := aggregate(2, 3, 4, add)
    // sum is 9
    product := aggregate(2, 3, 4, mul)
    // product is 24
}
```

#### Anonymous Functions

```go
func aggregate(a, b, c int, arithmetic func(int, int) int) int {
    firstSum := arithmetic(a, b)
    secondSum := arithmetic(firstSum, c)
    return secondSum
}

func main() {
    // Using an anonymous function for addition
    sum := aggregate(2, 3, 4, func(x, y int) int {
        return x + y
    })
    fmt.Println("Sum:", sum) // Output: Sum: 9

    // Using an anonymous function for multiplication
    product := aggregate(2, 3, 4, func(x, y int) int {
        return x * y
    })
    fmt.Println("Product:", product) // Output: Product: 24

    // You can also assign anonymous functions to variables
    subtract := func(x, y int) int {
        return x - y
    }

    difference := aggregate(10, 3, 2, subtract)
    fmt.Println("Difference:", difference) // Output: Difference: 5
}
```

## Struct
- Nested vs Embedded

![image.png](/assets/image_d.png)

- Anonymous struct (should use only if it's only being used once)
  - The design intention is to prevent us from reusing a struct definition we never intended to reuse
- Receiver: only way to add function for struct that can call by struct.function()
  - `func (p Person) getName() string {return p.Name}`
- Memory alignment
  - (Would be useful for ultimate performance - once had a server in production that held *a lot* of structs in memory. Like *hundreds of thousands* in a list. When I re-ordered the fields in the struct, the memory usage of the program dropped by over 2 gigabytes!)
  - Data alignment refers to storing data at memory addresses divisible by the size of the data type that fits into the word size of CPU
    - 32 bits CPU - word size 4 bytes
    - 64 bits CPU - word size 8 bytes

![image.png](/assets/image_1o.png)

![image.png](/assets/image_1s.png)

  - Go typically uses 4 bytes alignment boundary with the exception of 64bit type e.g., int64, and slice
    - slice: 8 bytes pointer, 8 bytes length (int64) and 8 bytes capacity (int64)
    - if length exceeds int64 will cause compilation time /runtime error
  - Why CPU architecture matters here since CPU architecture dictates the alignment rules.
- When data is properly aligned, struct that can be read through single memory read might require multiple memory read, causing slower execution too.
- Empty struct costs 0 bytes!

## Interface
- As long as struct that implements all the function (same function signature) it's considered as implementing that interface, so we don't have to indicate it's implementing the interface explicitly

```go
package main

import "fmt"

func explainType(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("This is an int. Value: %d, Type: %T\n", v, v)
    case string:
        fmt.Printf("This is a string. Value: %s, Type: %T\n", v, v)
    case bool:
        fmt.Printf("This is a bool. Value: %t, Type: %T\n", v, v)
    default:
        fmt.Printf("Unknown type. Type: %T\n", v)
    }
}
//we can't use i. to get the value since i is interface variable that it can't
//use the functions and attributes as stated in struct

func main() {
    explainType(42)     //This is an int. Value: 42, Type: int
    explainType("hello")    //This is a string. Value: hello, Type: string
    explainType(true)   //This is a bool. Value: true, Type: bool
    explainType(3.14)   //Unknown type. Type: float64
}
```

- There is no overload concept in Go

```go
type shape interface {
  area() float64
}

type Circle struct {
    radius float64
}

func (c Circle) area() float64{
    return math.pi * c.radius * c.radius
}
//hence Circle is implementing shape interface

type Square struct {
    length int
}
func (s Square) area() int{
    return s.length * s.length
}
//hence Square isn't implementing shape interface
```

- But we will not say Circle is type of shape like in Java?
  - But if a function taking interface type variable, we can pass variables that of types that implementing that interface e.g., func(s shape) ← we can pass circle inside
- One struct might therefore implement various interface and that's ok
- Nested interface

```go
type car interface {
    Color() string
    Speed() int
    IsFiretruck() bool // should not add a flag to check if it's firetruck, just use the nested interface such that we know if it's a car also a firetruck
}

//instead
type firetruck interface {
    car //inherit the required methods from car interface
    HoseLength() int
}
```

- Can group a bunch of type into one interface

```go
// Ordered is a type constraint that matches any ordered type.
// An ordered type is one that supports the <, <=, >, and >= operators.
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64 |
        ~string
}
```

- ~int the ~ is a tilde operator

![image.png](/assets/image_k.png)

## Error
- As long as struct has Error() string, it's implementing error interface
  - That we can create function func xxx(yy,zz) error {return aa.Error()}
- Using errors package, we could have just
  ```go
  var err error := errors.New("error msg")
  ```
- If `panic(err)` is called, then the program crashes and print stack trace, until it reaches a function that `defer recover()` TRULY BAD WAY TO HANDLE ERROR

![image.png](/assets/image_t.png)

- If want to exit the program, just use `log.Fatal` to print a message and exit the program

## Loop
- `for i := 0; i < sth; i++{}`
- No `while`
  - Use `for i := 0;; i++{}`
  - `for var < 5 {}`

## Slice & Array
- Array - fixed size hence fewer people using it
  - `s := [3]int{1,2,3} or s :=[...]int{1,2,3}`
- Slice - more flexible but underlying still array
  - `s := []int{1,2,3}`
  - Dynamic resizing
    - Append
      ```go
      //add element to slice
      slice = append(slice, element)

      //add same dimension of slice2 to slice
      slice = append(slice, slice2...)
      ```
    - When a slice needs to grow beyond its current capacity, Go typically doubles the capacity of the slice. This amortized approach ensures that appending to a slice has an average O(1) time complexity.
    - Very interesting resizing mechanism that if use append() once only, the max cap will be allocated with N+1 only
      - Else if use append(), if it's over the previous cap, the cap will be 2X of previous cap

![image.png](/assets/image_22.png)

  - When it's resized then new array is created. This leads to some potential bug if we write code `sliceB := append(sliceA, 10)`
    - Why?
    - If after append it's not resized, then sliceB === sliceA (same address)
      - It means if sliceB or sliceA do any modification, it's changing another side too.
    - If after append it's resized, then sliceB will have different address than sliceA.
    - Such inconsistent coding practice is not suggested!
    - Should always `slice = append(slice, 10)`
    - And Go doesn't have end-of-array character that

```go
i: [0 0 0], len: 3, cap: 8, addr: 0x14000114040
j: [0 0 0 4], len: 4, cap: 8, addr: 0x14000114040
i after appending to j: [0 0 0], addr: 0x14000114040
```

- i and j share same underlying array but they are having different length. How? Actually each slice in Go is a struct
  - Slice Header:
  - Each slice variable in Go is actually a small struct (called a slice header) containing:
    a. A pointer to the underlying array
    b. The length of the slice
    c. The capacity of the slice
- When slice an existing slice (taking partial of the slice), they are editing the underlying same pointed memory!

```go
arrayname[lowIndex:highIndex]
arrayname[lowIndex:]
arrayname[:highIndex]
arrayname[:]
```

- Where `lowIndex` is inclusive and `highIndex` is exclusive.

```go
func main() {
    arr := [5]int{1, 2, 3, 4, 5} // Create an array
    slice := arr[1:4]            // Create a slice from the array

    fmt.Println("Original Array:", arr)
    fmt.Println("Original Slice:", slice)

    // Modify the slice
    slice[0] = 99

    fmt.Println("Modified Slice:", slice)
    fmt.Println("Modified Array:", arr)
}
```

- Init can use `make([]T, len, cap)`
  - cap argument can be omitted and defaults to len
    - make / []int create, the original cap = len
- Iterate

```go
for idx, val := range messages {
    costs[idx] = 0.01 * float64(len(val))    
}
//can put _ in idx/val to ignore it
```

```go
func sum(nums ...int) int { //take nums as slice
    result := 0
    for _, v := range nums {
        result += v
    }
    return result
}
```

- 2D slice

```go
func createMatrix(rows, cols int) [][]int {
    matrix := make([][]int, rows)
    for i := range rows {
        matrix[i] = make([]int, cols)
        for j := range cols {
            matrix[i][j] = i*j
        }
    } 
    return matrix
}
```

- Like slices, maps are also passed by reference into functions. This means that when a map is passed into a function we write, we *can* make changes to the original, we don't have a copy.
- But slice and array have different function signatures, never be the same

![image.png](/assets/image_b.png)

## Map
- `newMap := make(map[string]int)`

![image.png](/assets/image_j.png)

- Value can be any type
- Key can only be types that are comparable
  - boolean, numeric, string, pointer, channel, and interface types, and structs or arrays that contain only those types
    - structs are comparable as long as they contain only the above types

```go
type Key struct {
    Path, Country string
}
hits := make(map[Key]int)

hits[Key{"/", "vn"}]++
n := hits[Key{"/ref/spec", "ch"}]
```

- Utilize the `strings.Fields()` for splitting by whitespace and `strings.ToLower()` and separate

## Pointers
- Reference

![image.png](/assets/image_2c.png)

- But Go auto dereferences for struct pointers, unlike simple types that we have to dereference ourselves

```go
func getMessageText(a *Analytics, msg Message){
    a.MessagesTotal++
    a.callMethod(likeThis)
}
```

- Simple example, replace some string value to another some string value

```go
import (
    "strings"
)

func removeProfanity(message *string) {
    messageVal := *message
    messageVal = strings.ReplaceAll(messageVal, "fubb", "****")
    messageVal = strings.ReplaceAll(messageVal, "shiz", "****")
    messageVal = strings.ReplaceAll(messageVal, "witch", "*****")
    *message = messageVal
}
```

- Pointer that pointing to nothing / pointer == nil
  - If we try to dereference it, it will PANIC
  - Always do nil pointer check
- Pointer Receiver
  - Function that can change the attribute's value of the struct object that calls that function (without creating new struct object and returning)
  - Pointer receiver is more common than value receiver actually, and we don't have to create a pointer and assign it with struct object address then finally call the function, Go has auto done the address part for us

![image.png](/assets/image_w.png)

- Optimization using pointer over copying value?
  - Values are stored on stack while pointers are stored on heap, so values have faster accessing speed than pointers unless the value to be copied is so huge. Once the value becomes large enough that copying is the greater problem, it can be worth using a pointer to avoid copying. That value will probably go to the heap, so the gain from avoiding copying needs to be greater than the loss from moving to the heap.
  - Should only use pointers when need a shared reference to a value

## Pass by reference vs Pass by value table
# Go Types: Final Corrected Passing Behavior
|                                Type | Typical Function Signature | Passed By |                                                                                              Behavior |
|:------------------------------------|:---------------------------|:----------|:------------------------------------------------------------------------------------------------------|
|                                bool |             `func(b bool)` |     Value |                                                                       A copy of the boolean is passed |
|      int, int8, int16, int32, int64 |              `func(i int)` |     Value |                                                                       A copy of the integer is passed |
| uint, uint8, uint16, uint32, uint64 |             `func(u uint)` |     Value |                                                              A copy of the unsigned integer is passed |
|                    float32, float64 |          `func(f float64)` |     Value |                                                                         A copy of the float is passed |
|               complex64, complex128 |       `func(c complex128)` |     Value |                                                                A copy of the complex number is passed |
|                              string |           `func(s string)` |     Value |                                                     A copy of the string header is passed (immutable) |
|                               array |         `func(arr [5]int)` |     Value |                                                                  A copy of the entire array is passed |
|                              struct |         `func(s MyStruct)` |     Value |                                                                 A copy of the entire struct is passed |
|                             pointer |         `func(p *MyType)` |     Value |                                                               A copy of the pointer address is passed |
|                               slice |            `func(s []int)` | Reference |                                           A copy of the slice header is passed, acting as a reference |
|                                 map |   `func(m map[string]int)` | Reference |                                             A copy of the map header is passed, acting as a reference |
|                             channel |        `func(ch chan int)` | Reference | A copy of the channel header is passed, acting as a reference. so `func(ch *chan int)` is not needed |
|                           interface |      `func(i interface{})` | Reference |                                       A copy of the interface header is passed, acting as a reference |
|                            function |           `func(f func())` | Reference |                                       A copy of the function pointer is passed, acting as a reference |

## Key Points:
1. While Go uses pass-by-value semantics internally, slices, maps, channels, interfaces, and functions behave as if they are passed by reference.
2. Modifying the contents of a slice, map, or channel inside a function will affect the original data.
3. Replacing an entire slice, map, or channel inside a function will not affect the original variable unless a pointer to it is passed.
4. For basic types, arrays, and structs, pass a pointer if you want to modify the original value in the function.

This table now correctly reflects the practical behavior of Go types in function calls, distinguishing between those that act like values and those that act like references.

## Package
- `package main` is special that only this is compiled to executable, and it needs `func main()` as entry point. Other packages are all libraries that have no entry point
- All `.go` files in a single directory must all belong to the same package. If they don't, an error will be thrown by the compiler. This is true for main and library packages alike.

## Module
- 1 repo
  - (generally) 1 module
    - many packages
      - 1 package
        - 1 .go
    - Say this is at root level
we need to have a `go.mod` file

![image.png](/assets/image_m.png)

  - If we want to import a package from
    - the module `github.com/google/go-cmp` contains a package in the directory `cmp/`.
    - import path is `github.com/google/go-cmp/cmp`

![image.png](/assets/image_a.png)

- `go install` at the root level
  - It will install the executable to a global program at machine!!

![image.png](/assets/image_c.png)

  - Uninstall by `rm $(which hellogo)`
- Local different modules

![image.png](/assets/image_r.png)

  - In Go, only identifiers (like function names) that start with a capital letter are exported and can be used outside of their package.
- Remote package
  - Just import in main.go e.g., `import tinytime "github.com/wagslane/go-tinytime"`
  - Then in that directory `go mod init example.com/theresa/datetest`
  - Then in that directory `go get github.com/wagslane/go-tinytime`
    - Then the needed binaries will be downloaded and go.mod will include that path
  - Then `go build` and `./datetest`
- Should not export code from main package

## Concurrency / Channels
- Add keyword `go` in front of a function means this function will be executed concurrently
- Go routine

![image.png](/assets/image_u.png)

  - email sent ← main thread keep running while the go routine running too
- Go channel: thread safe queue for value assignment.

```go
func checkEmailAge(emails [3]email) [3]bool {
    isOldChan := make(chan bool)

    go sendIsOld(isOldChan, emails)

    isOld := [3]bool{}
    isOld[0] = <-isOldChan
    isOld[1] = <-isOldChan
    isOld[2] = <-isOldChan
    return isOld
}

// don't touch below this line

func sendIsOld(isOldChan chan<- bool, emails [3]email) {
    for _, e := range emails {
        if e.date.Before(time.Date(2020, 0, 0, 0, 0, 0, 0, time.UTC)) {
            isOldChan <- true
            continue
        }
        isOldChan <- false
    }
}
```

  - Main thread: checkEmailAge
    - isOld[0] ← isOldChan
      - Value receiving instruction. This instruction is blocked until there is a value assigned to the channel.
  - Go routine / second thread: sendIsOld
    - isOldChan ← true
      - Value assignment instruction. After this instruction, if there is no receiver to receive the value, this thread will be blocked
- Another example with empty struct
  - Empty struct?
    - `func waitForDBs(numDBs int, dbChan chan struct{}) {`
    - We use empty struct when we don't need to have data transfer but we do care if there is a value passed to the channel.
    - `<-dbChan` would be blocked until something has passed to the channel and then will remove / pop the single item from the channel, then continue the rest of the code
    - (Goal of the following code is that when, say numDbs=3, that only if there are 3 values passed to the channel then we end the waitForDbs function)

```go
func waitForDBs(numDBs int, dbChan chan struct{}) {
    for i := 0; i < numDBs; i++ {
        <-dbChan
    }
}

// don't touch below this line

func getDBsChannel(numDBs int) (chan struct{}, *int) {
    count := 0
    ch := make(chan struct{})

    go func() {
        for i := 0; i < numDBs; i++ {
            ch <- struct{}{}
            fmt.Printf("Database %v is online\n", i+1)
            count++
        }
    }()

    return ch, &count
}
```

- The above examples are showing no buffered channel - that only if there is a receiver else we are blocked. But buffered channel means only if we queued the predefined capacity then we get blocked.
  - `ch := make(chan int, 100)`
  - When the channel has any space (i.e., smaller than the capacity) then we won't be blocked.
- Closing a channel
  - `close(ch)`
  - Can check if a channel is closed and emptied

![image.png](/assets/image_15.png)

  - Sending on a closed channel will cause PANIC! (panic on that main routine / goroutine)
  - Closing isn't necessary. There's nothing wrong with leaving channels open, they'll still be garbage collected if they're unused. You should close channels to indicate explicitly to a receiver that nothing else is going to come across.
- `range` blocking at each iteration if nothing new is there) and will exit only when the channel is closed.
  - But also true that using range to get the value from the channel will also act as value receiver that will unblock the value assignment code.
- `select` & `default`
  - Whichever channel got value passed first then we will pick that channel
  - Default means when there is no value passed to any channel then will trigger this block

```go
func saveBackups(snapshotTicker, saveAfter <-chan time.Time, logChan chan string) {
    for {
        select {
        case <-snapshotTicker:
            takeSnapshot(logChan)
        case <-saveAfter:
            saveSnapshot(logChan)
            return
        default:
            waitForData(logChan)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

// don't touch below this line

func takeSnapshot(logChan chan string) {
    logChan <- "Taking a backup snapshot..."
}

func saveSnapshot(logChan chan string) {
    logChan <- "All backups saved!"
    close(logChan)
}

func waitForData(logChan chan string) {
    logChan <- "Nothing to do, waiting..."
}
```

  - `time.Tick()`

![image.png](/assets/image_v.png)

  - Ticker is like periodically sending current timestamp to a channel
- Some short points about nil channel

![image.png](/assets/image_1b.png)

- Send to nil channel will actually cause panic
- Read from nil channel will cause deadlock

![image.png](/assets/image_7.png)

- Pretty good wrap up actually

## Mutex
- Protect a block of code

```go
func protected(){
    mu.Lock()
    defer mu.Unlock()
}

for i := 1; i <= 10; i++ {
    go func() {
        mu.Lock()
        defer mu.Unlock()        
        // Some operation
        // the rest of the function is protected
        // only after the first thread which acquired the lock has unlocked it, other nine goroutine calls to `mu.Lock()` will block
    }()
}
```

- `sync.RWMutex`
  - **RLock**: This method acquires a read lock. Multiple goroutines can call `RLock` simultaneously, allowing them to read the shared resource without blocking each other. However, while any goroutine holds a read lock, no goroutine can acquire a write lock.
  - **RUnlock**: This method releases a read lock that was previously acquired by `RLock`. It does not affect other goroutines that may also hold read locks.
  - Blocking Behavior
    - **RLock and Write Lock (Lock)**: When a goroutine has acquired a read lock using `RLock`, any attempt by another goroutine to acquire a write lock using `Lock` will be blocked until all read locks are released. This is crucial for maintaining data consistency because it ensures that no writes can occur while reads are happening.
    - **Lock and RLock**: Conversely, if a goroutine has acquired a write lock using `Lock`, no other goroutine can acquire either a read lock or another write lock until the write lock is released. This prevents any concurrent access to the resource while it is being modified.

```go
import (
	"sync"
	"time"
)

type safeCounter struct {
	counts map[string]int
	mu     *sync.RWMutex
}

func (sc safeCounter) inc(key string) {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	sc.slowIncrement(key)
}

func (sc safeCounter) val(key string) int {
	sc.mu.RLock()
	defer sc.mu.RUnlock()
	return sc.counts[key]
}

// don't touch below this line

func (sc safeCounter) slowIncrement(key string) {
	tempCounter := sc.counts[key]
	time.Sleep(time.Microsecond)
	tempCounter++
	sc.counts[key] = tempCounter
}
```

- Generics   
    ```go
    func ZeroValue[T any]() T {
        return *new(T) //return empty value of any type, can do for *new(int) if function returning int
    }
    ```
    - want some control of what types we can put as generics, so theres some constraints.    
        - Interface as contrains   
   
        ```go
        type stringer interface {
            String() string
        }

        func concat[T stringer](vals []T) string {
            result := ""
            for _, val := range vals {
                // this is where the .String() method
                // is used. That's why we need a more specific
                // constraint instead of the any constraint
                result += val.String()
            }
            return result
        }
        ```
    - reminder : new interface declaration method in go   
        ```go
        // Ordered is a type constraint that matches any ordered type.
        // An ordered type is one that supports the <, <=, >, and >= operators.
        type Ordered interface {
            ~int | ~int8 | ~int16 | ~int32 | ~int64 |
                ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
                ~float32 | ~float64 |
                ~string
        }
        ```
* Now we want to further constrain that an interface can only be implemented for certain types (of course from function level we can restrict it but just to provide more type safety and enables better compile-time checks to prevent certain runtime errors) e.g., 'Store interface' can only sell 'Product type' and 'Service type', other types like 'Human type' are not suitable for store.

```go
//generics
type Store[P Product] interface {
    Sell(P)
}

type Product interface {
    Price() float64
    Name() string
}
```

```go
//non generics
type store interface {
    Sell(p product)
}

type product interface {
    Price() float64
    Name() string
}
```

It's true that the two Sell functions are doing the same thing but the difference is:

```go
// Book is of Product type

// Non-generic usage
var s store = Bookstore{}
book := Book{"The Go Programming Language", 29.99}
s.Sell(book)

// Generic usage
var es Store[Book] = EBookstore[Book]{}
ebook := Book{"Effective Go", 19.99}
es.Sell(ebook)

nonP := NonProduct{23}
es.Sell(nonP) //Compilation error: cannot use nonP (variable of type NonProduct) as Book value in argument to es.Sell
```

## No enums → `= iota`

![image.png](/assets/image_h.png)

`iota` is a pre-declared identifier that represents successive untyped integer constants, starts at 0 in each const block and increments by 1 for each constant declaration. It resets to 0 when a new const block begins.

![image.png](/assets/image_y.png)

## Generics

```go
func ZeroValue[T any]() T {
    return *new(T) //return empty value of any type, can do for *new(int) if function returning int
}
```

If we want some control over what types we can put as generics, there are some constraints.

### Interface as constraints

```go
type stringer interface {
    String() string
}

func concat[T stringer](vals []T) string {
    result := ""
    for _, val := range vals {
        // this is where the .String() method
        // is used. That's why we need a more specific
        // constraint instead of the any constraint
        result += val.String()
    }
    return result
}
```

Reminder: new interface declaration method in Go:

```go
// Ordered is a type constraint that matches any ordered type.
// An ordered type is one that supports the <, <=, >, and >= operators.
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64 |
        ~string
}
```

This concludes the section on generics and interface constraints in Go. The use of generics allows for more flexible and type-safe code, while constraints provide a way to limit the types that can be used with generic functions or types.