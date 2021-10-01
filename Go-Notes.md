# Reference vs Value Types
| Value Types (Use pointer) | Reference Types (no pointer needed) |
| ------------------------- | ----------------------------------- |
| int                       | slices                              |
| float                     | maps                                |
| string                    | channels                            |
| bool                      | pointers                            |
| struct                    | functions                           |

# Differences between Maps & Structs

### Map
 - All keys must be same type
 - All values must be same type
 - Keys are indexed - we can iterate over them
 - Use to represent a collection of related properties
 - Don't need to know all the keys at compile time
 - Reference Type!

### Struct
- Values can be of different types
- Keys don't support indexing
- You have to know all the different fields at compile time
- Use to represent a "thing" with a lot of different properties
- Value Type!

# Memory: Stack vs Heap

-   Go’s runtime creates 1 stack per goroutine.
-   Go’s runtime does not clean the stack every time a goroutine exits, it just marks it as invalid so it’s available for other programs or routine to claim it.
-   The Go runtime is observant enough to know that a reference to the salutation variable is still being held, and therefore will transfer the memory to the heap so that the goroutines can continue to access it. This in known as **Escape Analysis.**
-   The rule of thumb for memory allocation

Sharing down **typically** stays on the stackSharing up **typically** escapes to the heap

Only the compiler knows for sure when typically is not typically

To get accurate information build your program with gcflags

`go build -gcflags=”-m -l” program.go`

-   When does a variable escape to the heap?

- If it could possibly be referenced after the function returns- When a value is too big for the stack- When the compiler doesn’t know the size in compile time

-   Don’t do premature optimisations, relay on data and find problems before fixing them. Use profilers and analysis tools to find the root.

# Goroutines

-   Every Go program has at least one goroutine: the _main goroutine_, which is automatically created and started when the process begins.
-   A goroutine is a function that is running concurrently. Notice the difference between concurrence and parallelism.
-   They’re not OS threads, they’re a higher level of abstraction known as coroutines. Coroutines are simply concurrent subroutines that are nonpreemptive (they cannot be interrupted).
-   Go follows a fork-join model for launching and waiting for goroutines.
-   **Leak prevention**: The way to successfully mitigate this is to establish a signal between the parent goroutine and its children that allows the parent to signal cancellation to its children. By convention, this signal is usually a read-only channel named _done_. The parent goroutine passes this channel to the child goroutine and then closes the channel when it wants to cancel the child goroutine.
-   If a goroutine is responsible for creating a goroutine, it is also responsible for ensuring it can stop the goroutine.
-   For better error handling inside goroutines create a struct that wraps both possible result and possible error. Return a channel of this struct.

# Sync package

**WaitGroup**

Use it when you have to wait for a set of concurrent operations to complete when you either don’t care about the result of the concurrent operation, or you have other means of collecting their results.

**Mutex and RWMutex**

Locks a piece of code so only one goroutine can access it. The best practice is to lock and on the next line unlock with a defer statement.

A `RWMutex` gives you fine-grained control on when read or write privileges are given.

# Channels

-   Channels serve as a conduit for a stream of information; values may be passed along the channel, and then read out downstream
-   Bidirectional or unidirectional
-   Channels in Go are blocking. This means that any goroutine that attempts to write to a channel that is full will wait until the channel has been emptied, and any goroutine that attempts to read from a channel that is empty will wait until at least one item is placed on it.
-   Reading sends two values, value and ok. If the channel is closed then value is the default value for the channel’s type and ok is false. Range over channel checks the ok value.
-   Creation of a channel should probably be tightly coupled to goroutines that will be performing writes on it so that we can reason about its behavior and performance more easily.
-   The goroutine that owns a channel should: Instantiate the channel, Perform writes, or pass ownership to another goroutine, close the channel, encapsulate the previous three things and expose them via a reader channel.

# Select and for-select statements

-   In the select statement the evaluation of branches is done simultaneously, the first one that meets the condition is executed. If there are multiple options, go’s runtime does pseudorandom selection
-   Use <- time.after(n*time.Second) for timing out if none of the branches of the select statement become ready
-   Default clause in a select statement is executed if none is ready. Mixed with for loop to create fallthrough and check for other options every time

```go
for { select { case <-done:
  return default: // Do non-preemptable work }}
```

# Pipelines

When constructing pipelines use a generator function to convert input to channel

```go
func IntGenerator(done <-chan interface{}, integers ...int) <-chan int {
  intStream := make(chan int)

  go func() {
    defer close(intStream)
    for _, i := range integers {
     select {
       case <-done:
        return
       case intStream <- i:
     }
    }
  }()
 return intStream
}
```

Fan-out, fan-in: reuse a single stage of our pipeline on multiple goroutines in an attempt to parallelise pulls from an upstream stage.

When to fan-out?

• It doesn’t rely on values that the stage had calculated before.

• It takes a long time to run.

# Slices, declaration vs initialization

```go
var arr []string // declares slice, nil value
arr := make([]string, 3) //declares and initializes slice [“”,””,””]`
```
# Interfaces

We know that...
  - Every value has a type
  - Every function has to specify the type of its arguments

So does that mean...
  - Every function we ever write has to be rewritten to accommodate different types even if the logic in it is identical?

instantiates a new type that can be accessed by any other type with a function that has the same name bit only if they return the same type

multiple types with the same function, but specific to that type can use the interface to extend

interfaces define a method/function set - what functions and return types it should have

### Concrete types
Can create a value out of this type
- map
- struct
- int
- string

### Interface type
Can't create a value out of this type
- interface

Interfaces are not generic types
Interfaces are `implicit`
  - don't have to declare that our custom type satisfies some interface
  - Helps with the DRY code battle
  - Hard to keep track of which types implement which interfaces
Interfaces are a "contract" to help us manage types - Garbage in >> garbage out!
  - if our custom types implementation of a function is broken - an interface will not help
Interfaces are tough. Step #1: Understand how to read them
  - understand how to read interfaces in the standard lib. Writing your own is tough and requires experience
  - Not a requirement of the language, no need to write custom interfaces


## Compile time check

Interface face implementation is done implicitly and checked in runtime. If you do not conform to an interface then an error will raise in production.

Add this line to do a compile time check of your interface implementation, will fail to compile if for example `*Handler` ever stops matching the `http.Handler` interface.

`var _ http.Handler = (*Handler)(nil)`

## Receivers matter

```go
package main

import (
  “fmt”
)

type Animal interface {
   Speak() string
}

type Dog struct {

}

func (d Dog) Speak() string {
  return “Woof!”
}

type Cat struct {}

func (c *Cat) Speak() string {
  return “Meow!”
}

func main() {
  animals := []Animal{Dog{}, Cat{}}

  for _, animal := range animals { fmt.Println(animal.Speak())
  }
}

// Output./prog.go:26:32: cannot use Cat literal (type Cat) as type Animal in slice literal:Cat does not implement Animal (Speak method has pointer receiver
```

## Interface segregation principle

_A great rule of thumb for Go is_ **_accept interfaces, return structs_**_.
–Jack Lindamood_

## Resources

-   [Memory management in Go](https://deepu.tech/memory-management-in-golang/)
-   [Understanding Allocations: the Stack and the Heap — GopherCon SG 2019 (Jacob Walker)](https://www.youtube.com/watch?v=ZMZpH4yT7M0&list=WL&index=14&t=14s&ab_channel=SingaporeGophers)
-   [Golang UK Conference 2016 — Dave Cheney — SOLID Go Design](https://www.youtube.com/watch?v=zzAdEt3xZ1M&ab_channel=GopherConUK)
-   [MapReduce in Go](https://appliedgo.net/mapreduce/)
-   [Concurrency in Go, Katherine Cox-Buday](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/)
-   [https://github.com/uber-go/guide/blob/master/style.md](https://github.com/uber-go/guide/blob/master/style.md)
-   [https://github.com/golang-standards/project-layout](https://github.com/golang-standards/project-layout)
-   [https://github.com/mastanca/go-concurrent-utils](https://github.com/mastanca/go-concurrent-utils)
