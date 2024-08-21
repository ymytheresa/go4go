---
layout: page
title: Closure
---

# closure   
Closures are a powerful feature in Go that allow functions to capture and access variables from their surrounding lexical scope. They're particularly useful for creating function factories, callbacks, and maintaining state. Here's an explanation of how closures can be used in Go, along with some examples:   
1. Basic closure:   
   
```
func main() {
    x := 10
    f := func() {
        fmt.Println(x)  // Captures x from outer scope
    }
    f()  // Prints: 10
}

```
1. Function factories:
Closures can be used to create functions with pre-set parameters.   
   
```
func adder(x int) func(int) int {
    return func(y int) int {
        return x + y  // x is captured from the outer function
    }
}

func main() {
    add5 := adder(5)
    fmt.Println(add5(3))  // Prints: 8
    fmt.Println(add5(7))  // Prints: 12
}

```
1. Maintaining state:
Closures can be used to create stateful functions.   
   
```
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

func main() {
    c := counter()
    fmt.Println(c())  // 1
    fmt.Println(c())  // 2
    fmt.Println(c())  // 3
}

```
1. Callbacks:
Closures are often used as callbacks, particularly in goroutines.   
   
```
func doWork(done chan bool) {
    go func() {
        // Do some work
        fmt.Println("Work done!")
        done <- true
    }()
}

func main() {
    done := make(chan bool)
    doWork(done)
    <-done  // Wait for work to complete
}

```
1. Modifying captured variables:
Closures can modify variables from the outer scope.   
   
```
func main() {
    x := 0
    increment := func() int {
        x++
        return x
    }
    fmt.Println(increment())  // 1
    fmt.Println(increment())  // 2
    fmt.Println(x)            // 2
}

```
1. Implementing middleware:
Closures are useful for creating middleware in web applications.   
   
```
func logging(f http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("Request received")
        f(w, r)
    }
}

func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, World!")
}

func main() {
    http.HandleFunc("/", logging(hello))
    http.ListenAndServe(":8080", nil)
}

```
Closures in Go provide a clean way to encapsulate behavior and state. They're particularly useful when you need to create functions dynamically, maintain private state, or pass behavior as an argument to another function.   
