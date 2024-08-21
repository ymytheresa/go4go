---
layout: page
title: Escape analysis
---

# Escape anlysis   
- If the compiler can prove a variable doesn't escape its function, it can be allocated on the stack.   
- If the variable might outlive the function or its lifetime can't be determined, it's allocated on the heap.   
   
   
Example 1: Variable doesn't escape (stack allocation)   
```
func createPoint() [2]int {
    point := [2]int{1, 2}
    return point
}

func main() {
    p := createPoint()
    fmt.Println(p)
}

```
In this case, `point` doesn't escape:   
- The `[2]int` array is a value type, not a reference type.   
- When `point` is returned, a copy is made.   
- The compiler can allocate `point` on the stack.   
   
Example 2: Variable escapes (heap allocation)   
```
func createSlice() []int {
    slice := make([]int, 3)
    return slice
}

func main() {
    s := createSlice()
    fmt.Println(s)
}

```
Here, `slice` escapes to the heap:   
- [ ] `int` is a slice, which is a reference type.   
- The underlying array of the slice needs to survive after `createSlice` returns.   
- The compiler will allocate `slice` on the heap.   
   
Example 3: Escaping due to uncertainty   
```
func mayEscape(b bool) *int {
    x := 5
    if b {
        return &x
    }
    return nil
}

func main() {
    p := mayEscape(true)
    fmt.Println(p)
}

```
In this case, `x` will be allocated on the heap:   
- The compiler can't be certain at compile time whether `&x` will be returned.   
- To be safe, it allocates `x` on the heap to ensure it's available if its address is returned.   
   
These examples demonstrate how the Go compiler decides between stack and heap allocation based on its analysis of variable lifetime and usage. The compiler's goal is to optimize performance while ensuring correct behavior, even if that means being conservative and using heap allocation when in doubt.   
