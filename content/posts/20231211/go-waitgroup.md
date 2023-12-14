---
title: "Go concurrency simplified. Part 2: Syncing goroutines with sync.WaitGroup"
draft: false
date: 2023-12-11T18:00:00+02:00
tags: ["go", "concurrency", "waitgroup"]
series: "Go concurrency simplified"
---

# Go concurrency simplified. Part 2: Syncing goroutines with `sync.WaitGroup`

Hello there! Despite the beautiful snowy weather outside, I'm at home these days with covid, so I can dedicate some additional time to blogging. 

Last time, we discussed the very basic concepts of Go concurrency: goroutines and channels. If you missed that post, please check it out [here](https://n0rdy.foo/posts/20231207/go-channels-and-goroutines/), it has some cool drawings =) Today, we'll move on and explore the ways Go offers us to sync goroutines - it will help us get rid of some hacky workarounds we have used so far.

## Recap

Have you already forgotten about it? My silly drawing will help you to remember:

![image](/images/drawings/20231207-0001.png "A queue")

Also, while discussing channels, we realized that there are the following associations with the Go language here:

- a post office desk -> Go channel
- a basketball -> a value written to the channel
- a customer -> a code that writes to the channel
- a postman -> a code that reads from the channel

Here is the summary of that with another beautiful drawing authored by me:

![image](/images/drawings/20231207-0003.png "A post office like a Go channel")

We seem to have a good theoretical foundation of the goroutines and channels. But our post office code still relies on the sequential approach with a for-loop and synchronous processing within. Here is the recap of how it looks:

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

func (w *Worker) Process(item string) {
	fmt.Printf("Worker %s received %s\n", w.Name, item)
	fmt.Printf("Worker %s started processing %s...\n", w.Name, item)

  // UPDATED: switched from 1 minute to 1 second to reduce the execution time
	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Worker %s processed %s\n\n", w.Name, item)
}

func main() {
	bobWorker := Worker{Name: "Bob"}

	zlatan := Customer{Name: "Zlatan", Item: "football"}
	ben := Customer{Name: "Ben", Item: "box"}
	jenny := Customer{Name: "Jenny", Item: "watermelon"}
	eric := Customer{Name: "Eric", Item: "teddy bear"}
	lisa := Customer{Name: "Lisa", Item: "basketball"}

	queue := []Customer{lisa, eric, jenny, ben, zlatan}

	for _, customer := range queue {
		item := customer.GiveAway()
		bobWorker.Process(item)
	}
}
```

This and other code examples are available in this [GitHub repo](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20231211-go-waitgroup).

Running this code gives the following output:

```text
Lisa gives away basketball
Worker Bob received basketball
Worker Bob started processing basketball...
Worker Bob processed basketball

Eric gives away teddy bear
Worker Bob received teddy bear
Worker Bob started processing teddy bear...
Worker Bob processed teddy bear

Jenny gives away watermelon
Worker Bob received watermelon
Worker Bob started processing watermelon...
Worker Bob processed watermelon

Ben gives away box
Worker Bob received box
Worker Bob started processing box...
Worker Bob processed box

Zlatan gives away football
Worker Bob received football
Worker Bob started processing football...
Worker Bob processed football
```

Nice! We are in a good position to start improving our code.

## Concurrency time

That was supposed to sound like ["Adventure Time"](https://en.wikipedia.org/wiki/Adventure_Time), if you were wondering =)

As we can see, the sequential parcels' processing algorithm is pretty straightforward:

- customer gives away an item which is persisted into the variable: `item := customer.GiveAway()`
- this item is passed to the postman for processing: `bobWorker.Process(item)`

As a first step to the concurrent approach, let's introduce a few changes in between those two: 

- customer will put the item into the channel instead of a variable
- postman will get the item from the channel rather than directly from the variable

The only missing part is the channel, but we know how to create it from the previous article in this series. Here is what the modified `main` method looks like:

```go
func main() {
	deskChan := make(chan string, 1)
	
	bobWorker := Worker{Name: "Bob"}

	zlatan := Customer{Name: "Zlatan", Item: "football"}
	ben := Customer{Name: "Ben", Item: "box"}
	jenny := Customer{Name: "Jenny", Item: "watermelon"}
	eric := Customer{Name: "Eric", Item: "teddy bear"}
	lisa := Customer{Name: "Lisa", Item: "basketball"}

	queue := []Customer{lisa, eric, jenny, ben, zlatan}

	for _, customer := range queue {
		deskChan <- customer.GiveAway()

		item := <-deskChan
		bobWorker.Process(item)
	}
	
	close(deskChan)
}
```

The same result as before is printed to the terminal. 

One interesting moment to notice is `make(chan string, 1)`. As you see, we explicitly specified the capacity for the channel as 1. This is an important moment I skipped in Part 1 of this series for simplicity's sake, but it's the right time to discuss this now. 

### Go channels capacity

In the Go language, the channel capacity shows the number of elements you can send to the channel without blocking the send operation. Basically, this means the following:

- if the capacity is 0, the send operation will block the current goroutine until the other goroutine reads from this channel
- if the capacity is 1, it is possible to send 1 element into the channel without blocking the goroutine
- if the capacity is n, it is possible to send n elements into the channel without blocking the goroutine

The default capacity is 0, meaning that `make(chan int)` creates a channel with a capacity of 0.

Let's try to show this in action with a piece of code to make it even more clear. Let's start with the 0 capacity channel:

``` go
func main() {
	zeroCapacityChan := make(chan string)
	
	zeroCapacityChan <- "a"
	
	received := <-zeroCapacityChan
	fmt.Println(received)
}
```

Before running this code, let's try to predict what the result will be. Let's read the above statement about the 0-capacity channel again: ` the send operation will block the current goroutine until the other goroutine reads from this channel` . Let's see what we have here: all the code runs in 1 (main) goroutine, which means that the `zeroCapacityChan <- "a"` will block it. So the only goroutine we have will be blocked forever. As we discussed in the previous post, this is a deadlock situation, and kudos to the Go compiler developers that make it detect situations like this one. 

Let's run the code to see whether we were right or not:

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /n0rdy-blog-code-samples/20231210-go-concurrency-with-for-and-select/03-channels-capacity/capacity.go:8 +0x36

```

Good job! Let's change the code by providing capacity as 1 to fix the error we've experienced:

```go
func main() {
	oneCapacityChan := make(chan string, 1)

	oneCapacityChan <- "a"

	received := <-oneCapacityChan
	fmt.Println(received)
}
```

And run it:

```text
a
```

Works like a charm! I hope it makes sense now why we did `deskChan := make(chan string, 1)` in the post office example above. Let's get back to it.

### Back to the post office

So, now our code uses a channel, so can we call it concurrent and add "Go concurrency" to our CV? Well, not really, it is still a sequential code that runs in one goroutine. However, since we are almost experts in goroutines now, let's tweak our code the way that:

- we'll remove the postman code from the door loop and create a new function that will constantly be listening to the desk channel and receive items from it once available - we'll run this function into a separate goroutine
- inside a for loop, the customer will the way it does now: `deskChan <- customer.GiveAway()`
- since we are introducing a new goroutine, we can create a channel with the default capacity, as we are safe from the deadlock from now on

While steps 2 and 3 are pretty straightforward, let's focus on the 1st one. How come and why should the postman listen constantly to the service desk? Since we have already allowed ourselves to apply a lot of simplifications to the post office way of working, let's agree that the only responsibility of the postman there is to stand at the desk the whole day and serve the customers, if any. If there are no customers, the working day looks like this:

![image](/images/drawings/20231211-0001.png "A working day with no customers")

And we have already seen the drawings of the working day with the queue of clients. 

Since we have established the responsibilities for the postman, let's add a new function `StartWorkingDay` to the `Worker` struct that receives a desk channel and constantly listens to it:

```go
func (w *Worker) StartWorkingDay(deskChan chan string) {
	for {
		item := <-deskChan
		w.Process(item)
	}
  fmt.Println("the desk is closed - time to go home")
}
```

Since the Go language doesn't have a concept of while loops, `for {}` is a direct equivalent of the `while (true) {}` from other programming languages.

One problem with this code, though: this for loop will never finish, and `the desk is closed - time to go home` part will never be printed. This means that our postman's working day will last forever 😱

Let's try to help the poor guy, as I bet he definitely has something else to do outside the working hours. But how can we do that? Hmm...what do we know about the closed channels so far? In the previous part of the series, we mentioned this:

> if the channel is closed, it is impossible to write into it (`panic: send on closed channel`), but the reader can [get the existing value from the channel](https://go.dev/ref/spec#Receive_operator). If the reader keeps reading from the channel, it will receive the default values (e.g. `0` for `int`)

`If the reader keeps reading from the channel, it will receive the default values` is something that can be pretty useful for our use case. Empty string (`""`) is a default value for the `string` data type. What if we add a simple check there: if the received item is an empty string, can we assume that the desk channel is closed? This is what it's going to look like:

```go
func (w *Worker) StartWorkingDay(deskChan chan string) {
	for {
		item := <-deskChan
		if item == "" {
			break
		}
		
		w.Process(item)
	}
	fmt.Println("the desk is closed - time to go home")
}
```

That will definitely help, so our postman won't need to work endlessly anymore. But let's stop and think for a moment: can we think of any edge cases here? For example, how are we supposed to distinguish the empty string sent by the closed channel from the empty string sent by the customer? Once any customer wants to send an empty parcel, our postman will treat it as a signal to call it a day - I don't think this will be that good for the post office business. We need a more comprehensive solution here.

There is one trick that I'd like to show you that can help us here. When I introduced channels to you in my previous post, I mentioned that this is the way to read a value from the channel:

```go
value := <- ch
```

While this is true, there is one tiny addition we can make to this code if we'd like to know whether the channel was opened or closed the moment we read from it:

```go
value, ok := <- ch
```

`ok` is a boolean value that is `true` if the channel is still open or `false` if it has already been closed. This will become very handy soon enough.

Let's use that knowledge to replace our error-prone workaround:

```go
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
```

Time to adjust the `main` method and see whether this new approach works - we'll run the `StartWorkingDay` as a new goroutine:

```go
func main() {
	deskChan := make(chan string)

	bobWorker := Worker{Name: "Bob"}
	go bobWorker.StartWorkingDay(deskChan)

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
}
```

If we run the code, we'll see the following output:

```text
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
```

That doesn't look right at all, as our application had exited before Bob managed to finish processing the football. And the `the desk is closed - time to go home` message has never been printed. For those who followed along with the previous post, it shouldn't be a big surprise, as we had exactly the same situation there when the `main` goroutine finished before the child one, and the execution was interrupted. Here is how it looked and how we fixed it back then:

```go
func main() {
	go printNTimes("Hello there.", 5)
	printNTimes("General Kenobi. You are a bold one.", 5)

	// to prevent the program from exiting before goroutine finishes
	time.Sleep(1 * time.Second)
}
```

If you have a feeling deep within that the `time.Sleep(1 * time.Second)` way of fixing this issue is somewhat a dirty hack - good intuition, my friend! But let's use it anyway to make sure it solves the issue, but increase the waiting to 2 seconds, so it is enough:

```go
func main() {
	deskChan := make(chan string)

	bobWorker := Worker{Name: "Bob"}
	go bobWorker.StartWorkingDay(deskChan)

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

	// to make sure that the worker has finished processing the last item
	time.Sleep(2 * time.Second)
}
```

The result looks as expected now:

```text
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

This is satisfying indeed, but not 100% for sure. As we mentioned above, the `time.Sleep(2 * time.Second)` part is a temporary workaround, not a proper solution. Can we improve that part? I bet we can! Before jumping into a solution mode, let's take a moment to think about why we need that workaround in the first place at all.

As you remember, the problem we solved by introducing this sleeping interval was the fact that the main goroutine finishes before the worker one, so the postman doesn't get enough time to complete his work. Like in real life: even if the working day ends at 17:00, it doesn't mean we should turn the lights on in the entire building immediately. This means that if there was a way to know whether the postman has done his work, we would keep the application running as long as needed. The most straightforward approach I can think of here is introducing a `boolean` flag that will be `true` if the postman work is finished and `false` otherwise. Let's do that.

```go
type Worker struct {
	Name           string
	isDoneForToday bool
}
```

<details> 
  <summary>If you are new to Go and don't know the difference between the lowercase and uppercase fields' namings</summary>
   If the name is uppercase - it means that the field is public and can be accessible from the outside of the current package, while the lowercase stands for private visibility. 
  By the way, the same approach applies to the functions' namings.
</details>
<br/>

So far, so good! Let's make sure to set it to `false` once we call the `StartWorkingDay` and change it to `true`. Here is how it looks now:

```go
func (w *Worker) StartWorkingDay(deskChan chan string) {
	w.isDoneForToday = false
	
	for {
		item, ok := <-deskChan
		if !ok {
			break
		}
		w.Process(item)
	}
	
	fmt.Println("the desk is closed - time to go home")
	w.isDoneForToday = true
}
```

With this in place, we need a way to use the `isDoneForToday` field to let the postman finish his job. Sounds like a room for a new function for the `Worker` struct:

```go
func (w *Worker) WaitToFinish() {
	for !w.isDoneForToday {
	}

	fmt.Printf("Worker %s has finished work for today\n", w.Name)
}
```

As you can see, we have a for loop with the empty body that keeps running until the worker is done for today. We can safely do that, as another goroutine runs the `StartWorkingDay` function, which will change the `isDoneForToday` to `true` once the work has been completed - we have just implemented this part. This change will trigger the `for` loop to be over, so then the execution will be unblocked. 

Time to replace the `time.Sleep` workaround with the call to our new function in a `main` function:

```go
func main() {
	deskChan := make(chan string)

	bobWorker := Worker{Name: "Bob"}
	go bobWorker.StartWorkingDay(deskChan)

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

	bobWorker.WaitToFinish()
}
```

Running this code returns the same result as before - good job. However, it still feels a bit hacky, and if your intuition tells you that we might have just reinvented the wheel - that's a good intuition (or an experience)! The `for !w.isDoneForToday {}` part is definitely a workaround to make the code wait for a signal to proceed. 

If you are ever in a similar need, before starting to google third-party libraries that might have solved this, a good rule of thumb is to check what Go has to offer as a part of the language. If your problem lies within the concurrency domain, start checking [the `sync` package](https://pkg.go.dev/sync). And while there are several canonical ways to solve our problem, we'll focus on one exact solution for educational purposes - relying on `sync.WaitGroup`.

## `sync.WaitGroup` - one tool to sync them all

So, what kind of creature is `sync.WaitGroup`? Here is what the [official Go docs](https://pkg.go.dev/sync#WaitGroup) say about it:

> A WaitGroup waits for a collection of goroutines to finish. The main goroutine calls Add to set the number of goroutines to wait for. Then each of the goroutines runs and calls Done when finished. At the same time, Wait can be used to block until all goroutines have finished.
> A WaitGroup must not be copied after first use.

If we split it into the step-by-step instructions, it will look like this:

- create a new wait group `wg` in the main goroutine
- before starting each new goroutine, do `wg.Add(1)` to let the wait group know about the number of running goroutines in a group
- pass a **pointer** to the wait group to each goroutine - I've highlighted the word "pointer" as this is a crucial moment, as otherwise, we'll pass the copy, which goes against the rule `A WaitGroup must not be copied after first use.` from the docs - it will simply not work the way it should
- in the main goroutine call `wg.Wait()` in the place where you'd like to wait for the child goroutines to finish their tasks
- in each of the child goroutines call `wg.Done()` once the work is done - it will reduce the counter of the running goroutines within the wait group

Basically, `wg.Wait()` will keep the main goroutine in the waiting stage until the counter is 0. That's why we should be careful while using `sync.WaitGroup`, as we might hang the program forever if we forgot to call `wg.Done()` for each `wg.Add(1)`.

We'll use the `printNTimes` example from the previous article to try this with a code. Let's recap what it looks like:

```go
func main() {
	go printNTimes("Hello there.", 5)
	printNTimes("General Kenobi. You are a bold one.", 5)

	// to prevent the program from exiting before goroutine finishes
	time.Sleep(1 * time.Second)
}

func printNTimes(s string, n int) {
	for i := 0; i < n; i++ {
		fmt.Println(s)
	}
}
```

The aim here is to get rid of the `time.Sleep(1 * time.Second)` part by replacing it with the `sync.WaitGroup` following the instructions above. We'll need a new function, printNTimesAsync, that will accept the pointer to the ``sync.WaitGroup`` on top of the other params. Also, we'll create an instance of the `sync.WaitGroup` in the `main` function and do the required actions with it:

```go
func main() {
	wg := &sync.WaitGroup{}

	wg.Add(1)
	go printNTimesAsync("Hello there.", 5, wg)
	printNTimes("General Kenobi. You are a bold one.", 5)

	// to prevent the program from exiting before goroutine finishes
	wg.Wait()
}

func printNTimes(s string, n int) {
	for i := 0; i < n; i++ {
		fmt.Println(s)
	}
}

func printNTimesAsync(s string, n int, wg *sync.WaitGroup) {
	for i := 0; i < n; i++ {
		fmt.Println(s)
	}
	wg.Done()
}
```

If we run it, it works the way it should:

```text
General Kenobi. You are a bold one.
Hello there.
Hello there.
Hello there.
Hello there.
Hello there.
General Kenobi. You are a bold one.
General Kenobi. You are a bold one.
General Kenobi. You are a bold one.
General Kenobi. You are a bold one.
```

Let's make this example a bit more complex and say that we'd like to repeat the printing part

```go
go printNTimesAsync("Hello there.", 5, wg)
printNTimes("General Kenobi. You are a bold one.", 5)
```

10 times to show the same in scale.

```go
func main() {
	wg := &sync.WaitGroup{}

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go printNTimesAsync("Hello there.", 5, wg)
		printNTimes("General Kenobi. You are a bold one.", 5)
	}

	// to prevent the program from exiting before goroutine finishes
	wg.Wait()
}

func printNTimes(s string, n int) {
	for i := 0; i < n; i++ {
		fmt.Println(s)
	}
}

func printNTimesAsync(s string, n int, wg *sync.WaitGroup) {
	for i := 0; i < n; i++ {
		fmt.Println(s)
	}
	wg.Done()
}
```

I will not show the result of running this code here, as it will take too many lines of text, but you can try it yourself and see that it works well. Which makes sense, of course! 

As of now, you should be pretty familiar with the concept of `sync.WaitGroup`, so let's apply it to our post office example. 

These are the steps we are going to take here:

- remove `isDoneForToday` field from the `Worker` struct and its usage across the codebase
- remove `WaitToFinish` function
- create an instance of `sync.WaitGroup` in the `main` function and use it to control the execution of the worker goroutine
- use `wg.Wait()` instead of the removed `WaitToFinish` function at the end of the `main` function.

This is what the updated `main` function looks like:

```go
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

<details> 
  <summary>If you are new to Go and don't understand the `go func() {...}()` construction</summary>
   The `func() {...}` part declares the anonymous function - in this context, the way to invoke multiple lines of code as a part of the same goroutine. 
  The `()` part calls this function immediately.
  We could have achieved a similar result by extracting the body of this function into a separate function and calling it here. But since we don't need that function for our business logic, I used this approach.
</details>
<br/>

Running this code gives the same result as the previous version - nice work! 

I believe this is a good place to stop for today. We learned a pretty important concept of Go today - `sync.WaitGroup` - that you are going to use a lot once you start writing concurrent code on a daily basis. And we have managed to eliminate some hacks in our code - always a good achievement.

In Part 3 of the series, we'll talk about the ways to manage channels by using `for` and `select` statements, which will get us closer to the solution to the queue situation in the post office. That's why stay tuned, and in the meantime, have fun! =)





