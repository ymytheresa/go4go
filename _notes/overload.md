---
layout: default
title: Overload
---

# overload   
- go   
    - does not support function overload   
    - approach using empty [Go basics](go-basics.md) 's Empty Interface   
        ```
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
        - Empty interface : interface type with no methods   
            - since every type implements at least zero methods, every type satisfies the empty interface. This means interface{} can hold values of any type   
        - a.(int) is a type assertion, asserting that a contains an int and tries to extract that int   
            - if type assertion failed and error not handled properly, will cause panic   
