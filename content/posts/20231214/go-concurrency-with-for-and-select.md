---
title: "Go concurrency simplified. Part 3: Managing channels with for loops and select statements"
image: "/covers/drawings/20231214.png"
draft: false
date: 2023-12-14T23:00:00+02:00
tags: ["go", "concurrency"]
series: "Go concurrency simplified"
---

# Go concurrency simplified. Part 3: Managing channels with `for` loops and `select` statements

Hello there! I feel like I got my covid under control and will be back to daily life soon. In the meantime, I'm sitting at my desk in a nearly empty apartment (I'm moving soon) and wondering whether it's possible to produce an echo if I scream loud enough 🤔 Anyway, I feel like it's the right time to start working on Part 3 of the "Go concurrency simplified" series. Today, we'll move on and explore the ways Go offers us to sync goroutines - it will get us closer to solving the queue situation in the post office we discussed last time. But let's start with a short recap of where we stopped in the previous post (if you missed it, [here is the link](https://n0rdy.foo/posts/20231211/go-waitgroup/)).

## Recap

Last time, we refactored our post office code by replacing the ugly `time.Sleep()` workaround with a `sync.WaitGroup` approach. Here is the entire code for that:

```go
type Customer struct {
	Name string
	Item string
}

func (c *Customer) GiveAway() string {
	item := c.Item
	fmt.Printf("%s gives away %s\n", c.Name, item)
	c.Item = ""
	return item
}

type Worker struct {
	Name string
}

func (w *Worker) StartWorkingDay(deskChan chan string) {
	for {
		item, ok := <-deskChan
		if !ok {
			break
		}
		w.Process(item)
	}

	fmt.Println("the desk is closed - time to go home")
}

func (w *Worker) Process(item string) {
	fmt.Printf("Worker %s received %s\n", w.Name, item)
	fmt.Printf("Worker %s started processing %s...\n", w.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Worker %s processed %s\n\n", w.Name, item)
}

func main() {
	deskChan := make(chan string)
	wg := &sync.WaitGroup{}

	bobWorker := Worker{Name: "Bob"}

	wg.Add(1)
	go func() {
		bobWorker.StartWorkingDay(deskChan)
		wg.Done()
	}()

	zlatan := Customer{Name: "Zlatan", Item: "football"}
	ben := Customer{Name: "Ben", Item: "box"}
	jenny := Customer{Name: "Jenny", Item: "watermelon"}
	eric := Customer{Name: "Eric", Item: "teddy bear"}
	lisa := Customer{Name: "Lisa", Item: "basketball"}

	queue := []Customer{lisa, eric, jenny, ben, zlatan}

	for _, customer := range queue {
		deskChan <- customer.GiveAway()
	}

	close(deskChan)

	wg.Wait()
}
```

If we take a closer look at this code using the experience of the refactoring we did in Part 2, it becomes clear that there is one part that looks hacky - I mean this one:

```go
for {
	item, ok := <-deskChan
	if !ok {
		break
	}
	w.Process(item)
}
```

A quick reminder of what's happening there: the code is supposed to fetch and process items from the channel until it's open. Once the channel is closed, the code should finish. Even though our solution does its job, our engineering experience tells us that it feels kinda weird that the designers of Go implemented channels but made us reinvent the wheel to consume items from them. And that's a good way of thinking! What if I tell you that actually there is a way to use `for` loops with channels? Let's jump into it!

## Managing channels with `for` loops

If we summarize what the code above does, it will be:

- reads items from the channel while it's open
- if the channel is closed, finishes the execution

And that's the exact description of how `for` loop works with the channel. Let me show you how we can rewrite the piece of code above:

```go
for item := range deskChan {
	w.Process(item)
}
```

And that's it - as simple as that. The line `for item := range deskChan` blocks the execution until there is a new item in the channel or until the channel is closed.

Let's apply this change to the `StartWorkingDay` function:

```go
func (w *Worker) StartWorkingDay(deskChan chan string) {
	for item := range deskChan {
		w.Process(item)
	}

	fmt.Println("the desk is closed - time to go home")
}
```

You can find the code samples for the post in this [GitHub repo](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20231214-go-concurrency-with-for-and-select).

And run the code:

```go
Lisa gives away basketball
Eric gives away teddy bear
Worker Bob received basketball
Worker Bob started processing basketball...
Worker Bob processed basketball

Worker Bob received teddy bear
Worker Bob started processing teddy bear...
Jenny gives away watermelon
Worker Bob processed teddy bear

Worker Bob received watermelon
Worker Bob started processing watermelon...
Ben gives away box
Worker Bob processed watermelon

Worker Bob received box
Worker Bob started processing box...
Zlatan gives away football
Worker Bob processed box

Worker Bob received football
Worker Bob started processing football...
Worker Bob processed football

the desk is closed - time to go home

```

Works like a charm!

This straightforward approach opens endless opportunities. Let's imagine that NASA asks us to build a data pipeline that has the following stages:

- Mars rover sends metrics to the International Space Station (ISS)
- ISS enriches the data and forwards it to the NASA data center
- the NASA data center processes the data

### Let's help NASA

This might sound like a challenging task, but since we'll mock the business logic of processing data and focus on the data pipeline part only, you'll see the beauty and the simplicity of Go here:

```go
import (
	"fmt"
	"math/rand"
	"strconv"
)

type MarsRover struct{}

func (mr *MarsRover) GatherMetrics() int {
	fmt.Println("Mars rover: gathering metrics...")
	return rand.Int()
}

type Iss struct{}

func (i *Iss) Enrich(metrics int) string {
	fmt.Printf("ISS: enriching metrics [%d]...\n", metrics)
	return "ISS" + strconv.Itoa(metrics)
}

type NasaDataCenter struct{}

func (ndc *NasaDataCenter) Process(data string) {
	fmt.Printf("Nasa data center: processing data [%s]...\n", data)
}

func main() {
	marsRover := MarsRover{}
	iss := Iss{}
	nasaDataCenter := NasaDataCenter{}

	issChan := make(chan int)
	nasaChan := make(chan string)

	go func() {
		for metrics := range issChan {
			nasaChan <- iss.Enrich(metrics)
		}
	}()

	go func() {
		for data := range nasaChan {
			nasaDataCenter.Process(data)
		}
	}()

	for {
		issChan <- marsRover.GatherMetrics()
	}
}
```

<details> 
  <summary>If you are new to Go and don't know what `rand.Int()` does</summary>
   `math/rand` package offers ways to generate random data. `rand.Int()` returns a random integer value.
  Please, note that if you need a secure random data, use `crypto/rand` package instead.
</details>

<br/>

<details> 
  <summary>If you are new to Go and don't know what `strconv.Itoa(metrics)` does</summary>
   `strconv.Itoa` is function to convert integer value to string. So if you pass `42` there, you'll get `"42"` as a result of it. 
  ITOA stands for "Integer to ASCII".
  This StackOverFlow answer (https://stackoverflow.com/a/2909772) shows that the `itoa` (or `atoi` - so, the opposite action) was mentioned in the manuals in 1971 - that's quite a while ago!
</details>

<br/>

If you run this code, it will be printing data like this:

```text
Mars rover: gathering metrics...
ISS: enriching metrics [7508100157780980863]...
ISS: enriching metrics [8542411963209765260]...
Mars rover: gathering metrics...
Nasa data center: processing data [ISS7508100157780980863]...
Nasa data center: processing data [ISS8542411963209765260]...
ISS: enriching metrics [6666611277333029651]...
Nasa data center: processing data [ISS6666611277333029651]...
```

Please terminate the execution manually by either clicking a stop button in your IDE or pressing `CTRL+C` or `CTRL+D`in your terminal. Otherwise, it will run forever and keep producing logs like I showed above.

As you can see, the solution is super simple and uses concurrency concepts we have learned so far: goroutines, channels, and for-loops. Next time you need to do a task that reminds a sequence of steps that you need to repeat many times, you can reuse the example above and tweak it to your needs. 

So, are we done with the Go concurrency? Have we already learned all we need? I wish! Let's leave the space for now and get back to the post office, as we have a situation there.

### Back to the post office

Our good old friend postman Bob has just received a call from his manager, Triss. She has a new business idea that should increase customer satisfaction - install a phone so the customers can call it when they have questions. This means that Ben will have a new responsibility on top of handling the parcels that clients bring. This is the new setup:

![image](/images/drawings/20231214-0001.png "A new post office setup with a phone")

Hmm...it means that Bob needs to handle 2 tasks instead of 1 from now on. But what is he supposed to do once there are both customers and a phone call? Well, he is free to choose which one to handle - "We have a lot of freedom at work," as Triss likes to say.

But how should we represent the phone with the code? It wouldn't be surprising that you already know the answer - another channel.

```go
func main() {
	phoneChan := make(chan string)

	// listen to the phone calls:
	go func() {
		for call := range phoneChan {
			fmt.Println("Got a call: " + call)
		}
	}()

	// somebody calls the phone:
	phoneChan <- "Yo! What's up?"
	close(phoneChan)
}
```

The output is:

```text
Got a call: Yo! What's up?
```

Let's take another look at the code that we wrote to represent the working day of the postman:

```go
func (w *Worker) StartWorkingDay(deskChan chan string) {
	for item := range deskChan {
		w.Process(item)
	}

	fmt.Println("the desk is closed - time to go home")
}
```

As we can see, the code is perfectly fit to handle the data from 1 channel. But we have just mentioned that from now on, there will be 2 channels instead. Shall we come up with another workaround? It's a good time to introduce a new Go concept for managing multiple channels simultaneously - `select` statement.

## Managing multiple channels with `select` statements

As we see now, the `for` loop is a handy construction when we need to query only 1 channel. And this might cover a lot of cases that you are going to face sooner or later. However, once this limit is reached, it's time to onboard the `select` statement. 

The syntax for this construct is pretty straightforward and (my guess) inspired by the `switch` statement:

```go
select {
case f := <-firstChan:
  handleFirst(f)
case s := <-secondChan:
  handleSecond(s)
case t := <-thirdChan:
  handleThird(t)
}
```

`f := <-firstChan` is exactly the same syntax we used to fetch data from the channel once we first met the concept of channels at all. This means it is possible to do `f, ok := <-firstChan` as well if we'd like to know whether the `firstChan` is opened.

Let's replace our `for` loop with the `select` statement and see how it works:

```go
func (w *Worker) StartWorkingDay(deskChan chan string, phoneChan chan string) {
	select {
	case item := <-deskChan:
		w.Process(item)
	case call := <-phoneChan:
		fmt.Printf("Worker %s received a call: %s\n", w.Name, call)
	}

	fmt.Println("the desk is closed - time to go home")
}
```

We also need to apply changes to the `main` function to setup the phone and add the code to make calls to it with a provided interval:

```go
func main() {
	deskChan := make(chan string)
	phoneChan := make(chan string)
	wg := &sync.WaitGroup{}

	bobWorker := Worker{Name: "Bob"}

	wg.Add(1)
	go func() {
		bobWorker.StartWorkingDay(deskChan, phoneChan)
		wg.Done()
	}()

	zlatan := Customer{Name: "Zlatan", Item: "football"}
	ben := Customer{Name: "Ben", Item: "box"}
	jenny := Customer{Name: "Jenny", Item: "watermelon"}
	eric := Customer{Name: "Eric", Item: "teddy bear"}
	lisa := Customer{Name: "Lisa", Item: "basketball"}

	queue := []Customer{lisa, eric, jenny, ben, zlatan}

	go func() {
		phoneChan <- "Has my package arrived?"
		time.Sleep(1 * time.Second)
		phoneChan <- "What about now?"
	}()

	for _, customer := range queue {
		deskChan <- customer.GiveAway()
	}

	close(deskChan)
	close(phoneChan)

	wg.Wait()
}
```

Let's run the code and see whether it works or not:

```text
Lisa gives away basketball
Worker Bob received a call: Has my package arrived?
the desk is closed - time to go home
fatal error: all goroutines are asleep - deadlock!
goroutine 1 [chan send]:
main.main()
        /n0rdy-blog-code-samples/20231214-go-concurrency-with-for-and-select/04-select-without-loop/select.go:74 +0x2b8

goroutine 19 [chan send]:
main.main.func2()
        /n0rdy-blog-code-samples/20231214-go-concurrency-with-for-and-select/04-select-without-loop/select.go:70 +0x45
created by main.main in goroutine 1
        /n0rdy-blog-code-samples/20231214-go-concurrency-with-for-and-select/04-select-without-loop/select.go:67 +0x25f

```

Wait a minute, that's not the output we were supposed to get! How come there is a deadlock? And why the `the desk is closed - time to go home` message 

Actually, there is a tiny detail that I have entirely forgotten to mention - unlike `for` loops, the `select` statements are not loops but rather 1-time actions, the same as the `switch` operator. That's why this part

```go
	select {
	case item := <-deskChan:
		w.Process(item)
	case call := <-phoneChan:
		fmt.Printf("Worker %s received a call: %s\n", w.Name, call)
	}
```

was executed once, and then the code proceeded to the `fmt.Println("the desk is closed - time to go home")` line, so Bob called it a day and went home. Since the queue of customers was still there, and there was nobody to handle them, we ended up in a deadlock situation. A nasty issue to have!

I think it's clear that we need to make our `select` statement run as a loop somehow. Without playing hide and seek, let me tell you that the Go-idiomatic way of solving this is wrapping it with a `for` loop on top like this:

```go
for {
  select {
	case item := <-deskChan:
		w.Process(item)
	case call := <-phoneChan:
		fmt.Printf("Worker %s received a call: %s\n", w.Name, call)
	}
}
```

However, there is a clear issue: we don't even need to run code to spot that this loop will run indefinitely, and there is no way to stop it. And let me tell you that Go has no dedicated language-level features to solve this. But fear not, we can fix this with what we already know. 

Actually, there are 2 (or maybe even more, but I usually use these 2) idiomatic ways to solve this:

1. Create a boolean flag that will keep the `for` loop running while it is `true`. Once either of the channels is closed, set this flag to `false` and terminate the loop this way.
2. In the `main` function, create a new channel that will be notified once the `select` statement has to finish its execution. Pass this channel to the method with the `select` statement and introduce a new `case` for it. If there is an input to that channel, break the loop. A real-life representation of this approach is like setting a timer/an alarm: once it rings, the postman knows it's time to call it a day.

The 1st approach (the boolean flag one) looks like this:

```go
func (w *Worker) StartWorkingDay(deskChan chan string, phoneChan chan string) {
	keepRunning := true
	for keepRunning {
		select {
		case item, ok := <-deskChan:
			if ok {
				w.Process(item)
			} else {
				keepRunning = false
			}
		case call, ok := <-phoneChan:
			if ok {
				fmt.Printf("Worker %s received a call: %s\n", w.Name, call)
			} else {
				keepRunning = false
			}
		}
	}

	fmt.Println("the desk is closed - time to go home")
}
```

If we run our application now, we'll see that the deadlock issue is gone, and the output is the way it should be:

```text
Lisa gives away basketball
Worker Bob received a call: Has my package arrived?
Worker Bob received basketball
Worker Bob started processing basketball...
Eric gives away teddy bear
Worker Bob processed basketball

Worker Bob received teddy bear
Worker Bob started processing teddy bear...
Jenny gives away watermelon
Worker Bob processed teddy bear

Worker Bob received a call: What about now?
Worker Bob received watermelon
Worker Bob started processing watermelon...
Ben gives away box
Worker Bob processed watermelon

Worker Bob received box
Worker Bob started processing box...
Zlatan gives away football
Worker Bob processed box

Worker Bob received football
Worker Bob started processing football...
Worker Bob processed football

the desk is closed - time to go home
```

The 2nd approach (a new channel) will look a bit different code-wise:

```go
func (w *Worker) StartWorkingDay(deskChan chan string, phoneChan chan string, shutdownChan chan struct{}) {
	for {
		select {
		case item := <-deskChan:
			w.Process(item)
		case call := <-phoneChan:
			fmt.Printf("Worker %s received a call: %s\n", w.Name, call)
		case <-shutdownChan:
			fmt.Println("the desk is closed - time to go home")
			return
		}
	}
}
```

We also need to introduce `shutdownChan` in the `main` function and send a signal to it at some point:

```go
func main() {
	deskChan := make(chan string)
	phoneChan := make(chan string)
	shutdownChan := make(chan struct{})
	wg := &sync.WaitGroup{}

	bobWorker := Worker{Name: "Bob"}

	wg.Add(1)
	go func() {
		bobWorker.StartWorkingDay(deskChan, phoneChan, shutdownChan)
		wg.Done()
	}()

	zlatan := Customer{Name: "Zlatan", Item: "football"}
	ben := Customer{Name: "Ben", Item: "box"}
	jenny := Customer{Name: "Jenny", Item: "watermelon"}
	eric := Customer{Name: "Eric", Item: "teddy bear"}
	lisa := Customer{Name: "Lisa", Item: "basketball"}

	queue := []Customer{lisa, eric, jenny, ben, zlatan}

	go func() {
		phoneChan <- "Has my package arrived?"
		time.Sleep(1 * time.Second)
		phoneChan <- "What about now?"
	}()

	for _, customer := range queue {
		deskChan <- customer.GiveAway()
	}

	shutdownChan <- struct{}{}

	close(deskChan)
	close(phoneChan)
	close(shutdownChan)

	wg.Wait()
}
```

If you find the type of the `shutdownChan` a bit unusual - `struct{}` - this is a Go style of specifying the channel in which data will be ignored. As in our case, we don't care about the data being sent to this channel - all we need is that someone triggered it so we can use it as a signal to shut down the execution. 

If we run this code, we'll see that it works as expected - good job! Triss is happy, and our customers are happy, but what about the postman, Bob? Well, we can't make everyone happy, can we?!

I believe this is a perfect moment to stop. We have learned a lot today, and now you know how to manage channels in a simple yet powerful way. I'm confident that you are more than ready to solve the post office issue finally we started talking about in [Part 1](https://n0rdy.foo/posts/20231207/go-channels-and-goroutines/) of this series. And that's what we are going to do in the next post, as well as turning our post office into a proper data streaming-like pipeline with respect to the working day hours. 

Stay tuned, as you don't want to miss out on the end of this story. See you in Part 4, and in the meantime, have fun! =)