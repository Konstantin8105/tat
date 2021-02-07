# Type of type

In according to question [template](https://github.com/golang/proposal/blob/master/go2-language-changes.md):
* I create enought Go code and I am not a novice Go programmer.
* Another programming languages in my experience: C, C++, Java.
* In my point of view, that design - add new think: *variables have type the type*.
* Yes, my design is look more native for Go in my point of view. But I see a few nice designs or exampes and I add links in my text.
* Main goal - try to avoid strange too long type clarification `func New[Node NodeConstraint[Edge], Edge EdgeConstraint[Node]] (nodes []Node) *Graph[Node, Edge] ` [code](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md).



## Minimal example

```go
package main

import "fmt"

// T is slice of types.
var T = []type{int, float32}

// summ resurn summary of values
func summ(vs ...T) T {
    var res T
    for _, v := range vs{
        res += v
    }
    return res
}

func main() {
    fmt.Printf("%d\n", summ[T:int](1,2,3,4))                  // show "10"
    fmt.Printf("%T\n", summ[T:int])                           // show "func (...int) int"
    fmt.Printf("%.2f\n", summ[T:float32](1.0, 2.0, 3.0, 4.0)) // show "10.00"
    //                       -----------
    //                      specific type

    fmt.Printf("%d\n", summ(1,2,3,4))          // Error: type T is undefined
    fmt.Printf("%d\n", summ[T:int32](1,2,3,4)) // Error: type `int32` is not in types slice T
}
```

For present language design:

* [int](https://play.golang.org/p/ppF1FDjBAqh)
* [float32](https://play.golang.org/p/R6BvhLTbgGN)


## Types slice manipulation

Types slice:
* always slice of types
* slice is limited
* no `any`
* name of variable accepted for [export](https://golang.org/ref/spec#Exported_identifiers)

### Acceptable types slice initialization

``` go
// Empty is empty slice of types.
// Exported.
var Empty []type

// ut is slice of unsigned types.
// Not exported.
var ut = []type{uint, uint8, uint16, uint32, uint64}

// num is slice of types from some external package `types`
var num = types.Numbers
```

### Not acceptable slice of types

```go
var (
    One  type = int // Error : type `type` is not slice of type `[]type`.
    Many []type = _ // Error : slice of types is not limited.
)

type A = One // Error : type `One` is not a type
```

### Useful operations

Create a copy of types slice
```go
var (
    N  = []type{float32, float64}
    NC []type
)

func init() {
    NC = N
}
```

Append new type
```go
var N []type

func init() {
    N = append(N, float64)
}
```

Append types slice to types slice
```go
var (
    floats = []type{float32,float64}
    ints  = []type{int,int32,int64}
    mix   []type
)

func init() {
    mix = append(mix, floats...)
    mix = append(mix, ints...)
}
```

Append new type to external package
```go
package main

import "types"

func init() {
    types.Numbers = append(types.Numbers, float64)
}
```

Remove type from external package
```go
package main

import "types"

func init() {
    for i,t := range types.Numbers{
        if t.(type) == uint32 {
            types.Numbers = append(types.Numbers[:i], types.Numbers[i+1:]...)
            break
        }
    }
}
```

### Scope

Type slice is cannot by changed in any functions, methods, except 
function `func init() {...}`. So, we can create a tests for avoid
some types in types slice.

```go
package main

import "fmt"

var floats =[]type{float32, float64}

func main() {
    floats = append(float32, int) // Error: cannot append types slice
    floats[0] = int               // Error: cannot change types slice

	floats[0],floats[1] = floats[1], floats[0] // Error: cannot change types slice

	fmt.Println(floats) // Show : "[]type{float32, float64}"
	for i, v := range floats{
		fmt.Println(v) // Show correctly types slice per line
	}
}
```

### Generic prototype with one types slice

**Examlpe 1**

External package `tools` with empty types slice

```go
package tools

var Numeric []type // empty types slice

// Min return minimal of 2 number with same type
func Min(a, b Numeric) Numeric {
    if a < b {
        return a
    }
    return b
}
```
Use package `tools`:

```go
package main

import "fmt"
import "tools"

func init() {
    tools.Numeric = append(tools.Numeric, int, float32)
}

func main() {
    fmt.Println(tools.Min[Numeric: int](1,2))          // show: 1
    fmt.Println(tools.Min[Numeric: float32](1.2,-2.1)) // show: -2.1

    Numeric := 12 // initialization variable `Numeric` with same name as types slice
    fmt.Println(tools.Min[Numeric: float32](1.2,-2.1)) // Error: slice `tools.Min` index `float32` is not valid. Function `tools.Min` is not slice.
}
```

**Example 2**

Example based on code from https://blog.golang.org/why-generics.

```go
package tree

var E []type

type Tree struct {
    root    *node
    compare func(E, E) int
}

type node struct {
    val         E
    left, right *node
}

func New(cmp func(E, E) int) *Tree {
    return &Tree{compare: cmp}
}

func (t *Tree) find(v E) **node { ... }
func (t *Tree) Contains(v E) bool { ... }
func (t *Tree) Insert(v E) bool { ... }
```

Using:

```go
package main

import "tree"

func init() {
    tree.E = append(tree.E, int, uint) // registration
}

func main(){
    t := tree.New[tree.E: int](func(a, b int) int { return a - b })

	var t2 tree.E // type of variable will be clarify later
	t2 = tree.New[tree.E: uint](func(a, b uint) uint { return a - b })
	// t2 = tree.New[tree.E: int] (...) Error: t2 have type tree[E: uint] already

    var (
        // For create a pointer of generic type
        // type added like in variable struct initialization
        v   = tree.Tree[E: float64]
        pt1  *tree.Tree[E: int]
        pt2 = new(tree.Tree[E: int])
    )

    // ...
}
```

**Example 3**

Example based on code from 
https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md#syntax
```go
// This defines `T` as a type parameter for the parameterized type `List`.
var T = []type{int, float32}

type List struct {
    element T
    next *List[T]
}

type ListInt List[T:int]
var v1 List[T:float32]
var v2 List[T: ListInt]
var v3 List[T: List] // Error: List[T] is undefined

type S struct { f T }
func L(n int, e T) interface{} {
    if n == 0 {
        return e
    }
    // return L(n-1, S{f:e}) // Error: type T of struct S is undefined
    return L(n-1, S[T:T]{f:e})
}


// Example of using: 
// f := Counter[T:int]()
func Counter() func() T {
	var c T
	return func() T {
		c++
		return c
	}
}
```


### Generic prototype with few types slice

**Example 1**

External package `tools` with empty types slice

```go
package tools

var Num1 []type // empty types slice
var Num2 []type // empty types slice
var Num3 []type // empty types slice

// Summ is summary of two value with different types
func Summ(a Num1, b Num2) Num3 {
	return Num3(a) + Num3(b)
}
```

```go
package main

import "fmt"
import "tools"

func init() {
    tools.Num1 = append(tools.Num1, int, float32, float64)
    tools.Num2 = tools.Num1
    tools.Num3 = tools.Num1
}

func main() {
    fmt.Println(tools.Summ[Num1: int, Num2: float32, Num3: float64](1,2.1))    // show: 3.1
    fmt.Println(tools.Summ[Num1: float64, Num2: float32, Num3: int](3.14,2.7)) // show: 5
}
```

**Example 2**

Swap types
```go
type T, T2 []type

type S struct{t T, t2 T2}

// SwapTypes have argument and return type `struct S`, but that
// struct have different types
func SwapTypes(s S) S {
	return S[T:T2, T2:T]{t: T2(s.t), t2: T(s.t2)}
}
```

**Example 3**

Example based on code from 
https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md#syntax
```go
var T []type
var T2 []type
type (
    MyMap  map[T]T2     // Error: type T, T2 is undefined
    MyMap2 map[T:int]T2 // Error: type T2 is undefined
)
```

```go
package hashmap

var K, V []type

type bucket struct {
	next *bucket
	key K
	val V
}

type Hashfn func(K) uint
type Eqfn func(K, K) bool

type Hashmap struct {
	hashfn Hashfn[K]
	eqfn Eqfn[K]
	buckets []bucket[K, V]
	entries int
}

// Example of using:
// h := hashmap.New[K:int, V:string](hashfn, eqfn)
func New(hashfn Hashfn, eqfn Eqfn) *Hashmap {
	// Type clarifications
	return &Hashmap[K:K, V:V]{hashfn, eqfn, make([]bucket, 16), 0}
}

func (p *Hashmap) Lookup(key K) (val V, found bool) {
	h := p.hashfn(key) % len(p.buckets)
	for b := p.buckets[h]; b != nil; b = b.next {
		if p.eqfn(key, b.key) {
			return b.val, true
		}
	}
	return
}
```

**Example 4**

Example with 3 types slices in one struct.
In struct structure used 2 types slices and 1 in struct method.

```go
var T1,T2,T3 []type
type S struct{ t1 T1; t2 T2}
func (s S) Summ() T3 {
	return T3(s.t1) + T3(s.t2)
}
// example of using: 
// v := s[T1: int, T2: float32, T3: float64]{t1: 1, t2: 3.14}
// fmt.Println("%2f\n", v.Summ()) // return type is float64
```

### Useful generics

```go
package sliceoperation

var T []type // empty types slice

func Reverse(s []T) {
    first := 0
    last := len(s) - 1
    for first < last {
        s[first], s[last] = s[last], s[first]
        first++
        last--
    }
}

func Sum(x []T) T {
	var s T
	for _, v := range x {
		s += v
	}
	return s
}
```


### Using in go package `math`

Code based on [function math.Pow](https://golang.org/src/math/pow.go?s=1186:1216#L28)

```go
package math

import powInt "https://github.com/Konstantin8105/pow"

var num = []type{float64, int}

// Pow returns x**y, the base-x exponential of y.
func Pow(x float64, y num) float64 {
    // Typical solution for switching by type:
    // switch v := y.(type) {
    //     case float64:
    //         return pow(x,v)
    //     case int:
    //         return powInt.En(x,v) // Just for example
    // }
    // 
    // But I think about switching by types slice `num`
	switch num {
         case float64:
             return pow(x,y)
         case int:
             return powInt.En(x,y)
	}
	panic("Unacceptable type")
}
```

### Types for-range

I do not know - is it useful?
```go
var nums = []type{float64, int}

func twice(v nums) nums{
	return v * 2
}

func main() {
    for _,num := range nums {
        fmt.Printf("%T\n", num)
        v := num(3.1415)
        fmt.Printf("%v. twice = %v",v, twice(v))
    }
}
```

or may be for tests

```go
func Test(t *testing.T){
    for _,num := range nums {
        if num == []float64 {
            t.Fatalf("not acceptable test")
	    }
    }
}
```


### Pointers in types slice

```go
var pnts = []type{*int, *float32}

func twice(v pnts) error {
    if v == nil {
        return fmt.Errorf("nil")
    }
    *v *= 2
    return nil
}
```





## Generic from exist code

### Replace on external package level

We have package with math operations [sparse](https://github.com/Konstantin8105/sparse) 
and we want to use that package but with changing type from `float64` to `float32`.

```go
package main

// package with typical slices of types
import (
    "go/generic/types"
    "go/math/big"
    "github.com/Konstantin8105/sparse"
)

var Calctype = []type{float32, float64, complex64, complex128, big.NewFloat(0.0).SetPrec(45)}
//                                      ----------------------------------------------------
//                                                it will be great

func init() {
    types.ReplaceAllPkg("Konstantin8105/sparse", float64, Calctype)
    // Now, package fully acceptable with float32 and float64
}
```

As we see - it is easy for adding new types, for example in future `float128` and other.

### Replace on function level of external package

For example, we found function:
```go
package mymath

// Mult is naive matrix multiplication
func Mult(A,B,C [][] float64) {
    n := len(A)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				C[i][j] += A[i][k] * B[k][j]
			}
		}
	}
}
```

But we want use with another types in our code:

```go
package

import "fmt"
import "Konstantin8105/mymath"
import "go/generic/types"

var Calctype = []type{float32, float64}

_ = Mult(A,B,C Calctype) // Acceptable for see a new function

func init() {
    types.ReplaceAllFunc("Konstantin8105/mymath.Mult", float64, Calctype, Mult)
    // Now, function acceptable with float32 and float64
    // where:
    //  "Konstantin8105/mymath/Mult" - location of function
    //  float64                      - replace from
    //  Calctype                     - replace to
    //  Mult                         - name of new function in that package
}

func main() {
    A64 := [][]float64{{2}}
    mymath.Mult(A64,A64,A64)
    fmt.Println(A64)
    Mult[Calctype: float64](A64,A64,A64)
    fmt.Println(A64)

    A32 := [][]float32{{2}}
    // mymath.Mult(A32,A32,A32) // Not acceptable, because function mymath.Mult work only with float64
    Mult[Calctype: float32](A32,A32,A32) // Acceptable. Function generated automatically. See function init.
    fmt.Println(A32)
}
```
