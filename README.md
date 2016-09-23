### Golang Tripping Hazards
---
##### Every time I learn something new, I always end up finding things that I really wish someone had told me at the beginning. If you're starting to learn Go, perhaps you'll find these tidbits useful.

We hope that by sharing this list we can save new Go developers some time hitting their head against the keyboard. If you'd like to contribute something to the list, please send a Pull request.

Compiled by Dan Miles: [Twitter](https://twitter.com/daniel_t_miles), [GitHub](https://github.com/danieltmiles)

If you'd like to see some of [Monsoon Commerce's](https://www.monsooncommerce.com/) Open Source projects we've released in Go, check out:

1. https://github.com/monsooncommerce/mockstomp We use it to mock a connection to a STOMP-compatable message broker for unit testing purposes

2. https://github.com/monsooncommerce/log A logging library (yes, another one) that allows for networked-logging to a STOMP-compatable message broker

3. https://github.com/monsooncommerce/stompWriter A short and simple Writer implementation we use with the logging library to write to STOMP-compatable message brokers

4. https://github.com/monsooncommerce/mockwriter A way to mock-up a writer for the logger when unit testing.

5. https://github.com/monsooncommerce/cryptoutil A simple wrapper to simplify some of Go's built-in cryptographic features (right now it only supports AES encrypt/decrypt). It has been pointed out to me that https://godoc.org/golang.org/x/crypto/nacl/secretbox is the more canonical solution, but feel free to use this, too.

---
**The defer statement**: A reserved word in Go that causes a function to execute as the last thing that happens when a function returns. You can stack as many as you want and they execute in the reverse order that they were declared. It looks something like this:
```go
func myFunction(intArg int, stringArg string) (int, error) {
	networkConn := openNetworkConnection()
	defer networkConn.Close()
	if intArg < 0 {
		errMsg := fmt.Sprintf("myFunction can only deal with positive integers, invalid argument: %v", intArg)
		return 0, errors.New(errMsg) // right after this, your deferred networkConn.Close() will execute
	}
	return intArg, nil // right after this, your deferred networkConn.Close() will execute
}
```
However, there's a complication. If the function you name in your defer statement takes arguments, and if those arguments are the return values of other functions, the argument-functions execute at the time you declare the defer, and the deferred function itself executes at function exit:
```go
func myFunction(intArg int, stringArg string) (int, error) {
	networkConn := openNetworkConnection()
	//determineHowConnShouldBeClosed() executes NOW, its return-value is stored as an
	// argument to closeMyNetworkConnection when it finally gets run.
	defer closeMyNetworkConnection(networkConn, determineHowConnShouldBeClosed())
	if intArg < 0 {
		errMsg := fmt.Sprintf("myFunction can only deal with positive integers, invalid argument: %v", intArg)
		return 0, errors.New(errMsg) // right after this, your deferred networkConn.Close() will execute
	}
	return intArg, nil // right after this, your deferred networkConn.Close() will execute
}
```

---
**Interfaces are always pointers**: Consider the following example:
```go
package main

import (
	"fmt"
)

type Fooer interface {
	Foo() int
}

type ImplementsFooer struct{
	DataMember int
}

// provide the function foo to make your ImplementsFooer struct
// implement Fooer. Note that this function has a non-pointer
// receiver to an ImplementsFooer instance, we'll get to that
// later
func (f ImplementsFooer) Foo() int {
	return f.DataMember
}

func WantsPointerToFooer(f *Fooer) {
	fmt.Printf("I got a fooer, result of Foo() is %v\n", f.Foo())
}

func main(){
	var myInstance Fooer // declare as type Fooer
	myInstance = ImplementsFooer{DataMember: 5}
	WantsPointerToFooer(&myInstance)
}
```
It doesn't compile: *prog.go:24: f.Foo undefined (type *Fooer is pointer to interface, not interface).* This is because *myInstance* is of type interface, which is already a pointer. *WantsPointerToFooer* should actually accept *(f Fooer)* and you'll still get the pass-by-reference performance you're looking for.

---


**Pointer recievers, non-pointer receivers and the implementation of interfaces**

As you program your way through some of your first go projects, you'll likely discover, as I did, that you can add instance-methods to structs. In this way, you can get some object-like stuff going on. You might do it like this in python:
```python
class Foo:
    data_member = 1
    def increment_data_member(self):
        self.data_member += 1
```
and in Go, it looks like this:
```go
type Foo struct {
	DataMember int
}
func (f *Foo) IncrementDataMember() {
	f.DataMember++
}
```

Notice that the method, IncrementDataMember is made a "part" of the Foo struct with the "reciever" between the *func* and the *IncremenentDataMember*? That receiver is a pointer-receiver and if you're like me, you just internalized that as the way things are, but it goes a little deeper.

Pointers and the base type they point to have distinct method sets in Go, and this affects how you implement an interface. Let's use our implementation of the `Fooer` interface in the previous example:

```go
type Fooer interface {
  Foo() int
}

type ImplementsFooer struct{
  DataMember int
}

// this receiver is NOT a pointer
func (f ImplementsFooer) Foo() int {
  return f.DataMember
}
```

`ImplementsFooer` and the pointer to that type, `*ImplementsFooer` are _distinct types_ in Go, and in this example only one of them implements the `Fooer` interface.

Given that information, please look at this sample program:

```go
package main
import "fmt"

type Fooer interface {
  Foo() int
}

type ImplementsFooer struct{
  DataMember int
}

// implement the Fooer interface
func (f *ImplementsFooer) Foo() int {
  return f.DataMember
}

func main() {
  implFoo := ImplementsFooer{35}
  printFoo(implFoo)
}

// this function requires a 
// type that implements Fooer
func printFoo(f Fooer) {
  fmt.Println(f.Foo())
}
```

This program will not compile. The compiler will give you an error when you try to compile it:

```
cannot use implFoo (type ImplementsFooer) as type Fooer in argument to printFoo:
  ImplementsFooer does not implement Fooer (Foo method has pointer receiver)
```

In this case, the `*ImplementsFooer` type implements the interface, but the `ImplementsFooer` type does not – structs and a pointer to that struct are have two distinct method sets in Go.

The simple fix for this example program is to pass a pointer into the `printFoo` function, like so:

```go
func main() {
  implFoo := ImplementsFooer{35}
  printFoo(&implFoo)
}
```

---
**The := symbol**: It's not an emoticon it's an assignment operator that automatically infers the type. In this way:
```go
var foo int
foo = 5
```
is equivalent to:
```go
foo := 5
```
but if you're not careful, it can mess with your scoping:
```go
foo, err := someFunction() // in this case, err is nil
if true {
	bar, err := someOtherFunction() // in this case, err is not nil, but it will
                                        // go out of scope at the end of the block
}
// out here, we're dealing with the original (nil) declaration of err
if err != nil {
	fmt.Printf("we had an error")
}
```
(this prints nothing)

if you find yourself writing something like that, you probably meant this instead:
```go
foo, err := someFunction() // err is nil
if true {
	var bar int
	bar, err = someOtherFunction() //err is not nill
}
if err != nil {
	fmt.Printf("we had an error")
}
```
(this prints we had an error)

---
**Pointers** Pointers are always a tripping hazard for new developers, and Go is no exception. Instead of re-inventing that blog, I recommend you just go read this: https://www.golang-book.com/books/intro/8

---
**The Go compiler is more picky about style than other languages you may have worked with**: I don't have much concrete advice here, but it's something that bugged me when I started. If you're someone who likes your curly brackets ({) on the next line, that's a compile error, and you just have to learn to live with it. Fortunately, there's an automatic formatter called "fmt" that fixes everything for you. On the commandline, run: _go fmt_ and it will fix the formatting in all of your go files.

---
**Same-line statements are possible but they can get confusing for scope reasons**: Go allows you to put multiple statements on the same line separated by semi-colons. I haven't seen this often, but there are a couple of cases where I do. You can use it to check to see if something is in a map:
```go
myMap := make(map[string]string)
myMap["foo"] = "bar"
if value, ok := myMap["baz"]; !ok {
	// the "truthiness" of this if statement is in the last (right-most) statement
	fmt.Printf("baz does not seem to be in myMap\n")
} else {
	fmt.Printf("we found baz, its value is %v\n", value)
}
// be aware that value is not defined in this scope, outside the if/else block
```
And this will check to see if an operation returned an error:
```go
if result, err := someFunction(); err != nil {
	fmt.Printf("had a problem running someFunction. It returned a value of %v and an error, %v\n", result, err.Error())
}
// be aware that because result is declared inside an if-statement, it is not in scope outside the if-block.
```
---
**Postfix Increments**: They're statements, not operators as they are in other languages. fmt.Printf("%v\n", foo++) doesn't work.

---
** Range operator doesn't do exactly what I expected**: It seems to give you a strange, quasi-pointer, not a real reference. Take this example:
```go
package main

import (
	"fmt"
)

func main() {
	mySlice := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	myChan := make(chan *int, 10)

	for _, theInt := range mySlice {
		fmt.Printf("putting a pointer to %v on the chan\n", theInt)
		myChan <- &theInt
	}

	for i := 1; i <= 10; i++ {
		thePtr := <-myChan
		if *thePtr != i {
			fmt.Printf("pulled %v off the chan, expected %v\n", *thePtr, i)
		}
	}
}
```
You might expect it to put each of the members of the mySlice slice on the channel, but it doesn't. The output is:
```
putting a pointer to 1 on the chan
putting a pointer to 2 on the chan
putting a pointer to 3 on the chan
putting a pointer to 4 on the chan
putting a pointer to 5 on the chan
putting a pointer to 6 on the chan
putting a pointer to 7 on the chan
putting a pointer to 8 on the chan
putting a pointer to 9 on the chan
putting a pointer to 10 on the chan
pulled 10 off the chan, expected 1
pulled 10 off the chan, expected 2
pulled 10 off the chan, expected 3
pulled 10 off the chan, expected 4
pulled 10 off the chan, expected 5
pulled 10 off the chan, expected 6
pulled 10 off the chan, expected 7
pulled 10 off the chan, expected 8
pulled 10 off the chan, expected 9
```
So theInt, as defined in that range object, is safe to work with inside the scope of the for statement, but pointers to it seem to change later. So for a use case like this, it's safer simply to iterate:
```go
for i := 0; i < len(mySlice); i++ {
	myChan <- &mySlice[i]
}
``
