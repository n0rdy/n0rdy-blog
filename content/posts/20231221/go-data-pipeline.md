---
title: "Go concurrency simplified. Part 4: Post office as a data pipeline"
image: "/covers/drawings/20231221.webp"
draft: false
date: 2023-12-21T23:20:00+01:00
tags: ["go", "concurrency", "pipeline"]
series: "Go concurrency simplified"
---

Hello there! The main part of my moving to a new place adventures seems to be behind. Since I'm still waiting for a furniture delivery, I'm writing this post while lying on the floor using my foam camping mat as a sofa. It's not the most ideal setup, but it works. Anyway, I feel like today is the right time to start working on Part 4 of the ["Go concurrency simplified" series](https://n0rdy.foo/series/go-concurrency-simplified). So far, we have learned and explored the key Go concurrency concepts, such as goroutines, channels, and the ways to synchronize and manage them. And today, we are going to combine all of our knowledge to make this world a better place to live (not really) or at least to help the post office (and our good old friend postman Bob) by making the queue handling process more effective. In the meantime, let's briefly recap where we stopped in Part 3 (if you missed it, [here is the link](https://n0rdy.foo/posts/20231214/go-concurrency-with-for-and-select/)). 

## Recap

The queue situation is still alarming, to say the least. On top of that, the management decided to assign a new responsibility to a postman in addition to handling the queue - answering the phone calls from the customers. Here is what it looks like:

![image](/images/drawings/20231214-0001.webp "A new post office setup with a phone")

This is the code that backs this setup:

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

func (w *Worker) Process(item string) {
	fmt.Printf("Worker %s received %s\n", w.Name, item)
	fmt.Printf("Worker %s started processing %s...\n", w.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Worker %s processed %s\n\n", w.Name, item)
}

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

This is a decent solution for the tutorial, but I'd not deploy it to the actual production server. Why? Let me ask you first: How much time would it take for this program to complete? Feel free to ignore the phone calls for now. The approximate answer is at least 5 seconds. But why 5? Well, we have 5 customers in the queue, and we have this piece of code that runs for each customer:

```go
func (w *Worker) Process(item string) {
	fmt.Printf("Worker %s received %s\n", w.Name, item)
	fmt.Printf("Worker %s started processing %s...\n", w.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Worker %s processed %s\n\n", w.Name, item)
}
```

Did you notice the `time.Sleep(1 * time.Second)` part? This is our bottleneck, as each customer will need to wait for at least this amount of time. Let's add a simple timer to see whether our estimate is correct:

```go
func main() {
	start := time.Now()

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

	end := time.Now()
  fmt.Println("\n=====================================")
	fmt.Printf("Execution time in milliseconds: %v\n", end.Sub(start).Milliseconds())
}
```

You can find the code samples for the post in this [GitHub repo](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20231221-go-data-pipeline).

The idea is pretty straightforward here: 

- the first line of the `main` function determines the start time: `start := time.Now()`
- once the business logic is completed, we determine the end time: `end := time.Now()`
- we subtract one from another, convert the result to milliseconds, and print it to see the total execution time

Here is what I can see on my machine:

```text
=====================================
Execution time in milliseconds: 5002
```

So, our estimate of 5 seconds was entirely accurate. But is 1 second per customer a realistic number? If you have ever been to the post office, you know that it takes way more than 1 second to send the parcel: depending on where you live, it can take 1 to 10+ minutes to fix that. That's why if we apply the 10 seconds timing (still too little, but tolerable to wait while running the program) to our code like this:

```go
func (w *Worker) Process(item string) {
	fmt.Printf("Worker %s received %s\n", w.Name, item)
	fmt.Printf("Worker %s started processing %s...\n", w.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Minute)

	fmt.Printf("Worker %s processed %s\n\n", w.Name, item)
}
```

it will change the total execution time from 5 seconds to 50 seconds. But what if we have more than 5 customers in the queue? Since Christmas is coming, here where I live the queues are pretty long these days, especially right after working day hours.

I hope it's clear now that we need to come up with something even better to improve the efficiency of the post office business and make the customers happier.

## Back to the post office

It's clear to us that the main problem here is that it takes a lot of time for the postman to process a parcel, and the customers have to wait in the meantime. Luckily, the same is clear to Triss - a post office manager. She understands that the postman Bob needs help. And since this post office branch is very important, it is possible to reallocate some workers from the other places here, so  Oda, Robert, and Martha will join. However, instead of opening 3 new desks to serve customers, Triss came up with something more innovative: the idea is to split the tasks in a way that Bob will still handle the customers, but instead of performing all the parcel clearance routines, from now on he is going to check the ID of the customers, take the parcel from them and pass it forward. The customer can leave right away, so the next in the queue can proceed. In the meantime, either of the 3 new employees takes the parcel and performs the rest of the actions with it.

If you have ever taken the algorithms classes, you might have heard of the ["divide and conquer" approach](https://en.wikipedia.org/wiki/Divide-and-conquer_algorithm) to solving problems, where the large task is divided into smaller subtasks that are easier to solve. Triss studied management at the university, so she didn't have the algorithms in her curriculum - nevertheless, that's exactly the approach she took here.

Let's see what the new setup will look like:

![image](/images/drawings/20231221-0001.webp "A new post office setup with 3 new employees")

Once Bob receives the parcel from the customer, he passes it forward right away:

![image](/images/drawings/20231221-0002.webp "A new post office setup with 3 new employees: step 1")

One of the new colleagues picks it up right away. At the same time, the customer can leave, so the queue is moving:

![image](/images/drawings/20231221-0003.webp "A new post office setup with 3 new employees: step 2")

This is quite an efficient process, as customers won't need to wait for the entire parcel clearance process to finish. Let's think about how we can code this.

### A new way of working as a code

Until now, we had only one type of worker responsible for everything post-office-related. That's why we have 1 struct that defines this type of employee:

```go
type Worker struct {
	Name string
}
```

From now on, we have 2 types of workers:

- the one who stands at the desk and handles the customers: Bob
- the ones who are in the back and handle the parcels once Bob passes them - our 3 new colleagues: Oda, Robert, and Martha

Let's rename the existing `Worker` struct to `DeskWorker` and agree that it represents the 1st type. Also, let's change the `Process` function a bit to specify the activities that Bob will do:

```go
func (dw *DeskWorker) Process(item string) {
	fmt.Printf("Desk worker %s received %s\n", dw.Name, item)
	fmt.Printf("Desk worker %s started checking ID of the customer with the %s item...\n", dw.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Desk worker %s finished checking ID of the customer with the %s item\n", dw.Name, item)

	fmt.Printf("Desk worker %s started passing %s to the back office...\n", dw.Name, item)
	dw.BackOfficeDeskChan <- item
	fmt.Printf("Desk worker %s passed %s to the back office\n\n", dw.Name, item)
}
```

We have yet to define the logic of passing the parcel to the back office, but we'll get back to this soon.

We need a new struct for the 2nd type -  `BackOfficeWorker` :

```go
type BackOfficeWorker struct {
	Name string
}
```

Similar to the existing struct, `BackOfficeWorker` needs 2 functions: `StartWorkingDay` and `Process`: 

```go
func (bow *BackOfficeWorker) StartWorkingDay(backOfficeDeskChan chan string, shutdownChan chan struct{}) {
	for {
		select {
		case item := <-backOfficeDeskChan:
			bow.Process(item)
		case <-shutdownChan:
			fmt.Printf("the back office is closed - time to go home, %s\n", bow.Name)
			return
		}
	}
}

func (bow *BackOfficeWorker) Process(item string) {
	fmt.Printf("Back office worker %s received %s\n", bow.Name, item)
	fmt.Printf("Back office worker %s started processing %s...\n", bow.Name, item)

	// to simulate long processing
	time.Sleep(10 * time.Second)

	fmt.Printf("Back office worker %s finished processing %s\n", bow.Name, item)
}
```

Unlike `DeskWorker`, these should not worry about the phone calls (poor Bob). Also, for the simulation purposes, there is a difference in processing time: 

- 1 second for the desk worker
- 10 seconds for the desk office workers

These numbers are unrealistic, but the scale might be good enough for us.

Let's define 3 new workers in the `main` function next to the Bob declaration:

```go
bobWorker := DeskWorker{Name: "Bob"}
odaWorker := BackOfficeWorker{Name: "Oda"}
robertWorker := BackOfficeWorker{Name: "Robert"}
marthaWorker := BackOfficeWorker{Name: "Martha"}
```

Since we have 2 types of employees now instead of 1, let's have 2 shutdown channels for each:

```go
deskShutdownChan := make(chan struct{})
backOfficeDeskShutdownChan := make(chan struct{})
```

We need to make sure we send a shutdown signal 3 times to the `backOfficeDeskShutdownChan`:

```go
deskShutdownChan <- struct{}{}
// 3 stands for the number of back office workers
for i := 0; i < 3; i++ {
	backOfficeDeskShutdownChan <- struct{}{}
}
```

The channel for the back office is missing as well:

```go
deskChan := make(chan string)
backOfficeDeskChan := make(chan string)
```

Let's make sure to start the working day for every back office employee the same way as for the desk worker:

```go
wg.Add(1)
go func() {
	bobWorker.StartWorkingDay(deskChan, phoneChan, deskShutdownChan)
	wg.Done()
}()

wg.Add(1)
go func() {
	odaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
	wg.Done()
}()

wg.Add(1)
go func() {
	robertWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
	wg.Done()
}()

wg.Add(1)
go func() {
	marthaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
	wg.Done()
}()
```

We can refactor this code by introducing a private function to start the employees' working day, but I'll keep it as it is for readability purposes - feel free to change the code on your own, as it's a good exercise!

The only missing part is when Bob passes the parcels to the back office. Let's extend the `DeskWorker` struct with a new field for the back office channel:

```go
type DeskWorker struct {
	Name               string
	BackOfficeDeskChan chan string
}
```

We can use this channel now to pass the parcel into it within the `Process` function:

```go
func (dw *DeskWorker) Process(item string) {
	fmt.Printf("Desk worker %s received %s\n", dw.Name, item)
	fmt.Printf("Desk worker %s started checking ID of the customer with the %s item...\n", dw.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Desk worker %s finished checking ID of the customer with the %s item\n", dw.Name, item)
	
  fmt.Printf("Desk worker %s started passing %s to the back office...\n", dw.Name, item)
	dw.BackOfficeDeskChan <- item
	fmt.Printf("Desk worker %s passed %s to the back office\n\n", dw.Name, item)
}
```

Please, don't forget to initialize this field in the place where we declare the `bobWorker` variable:

```go
bobWorker := DeskWorker{Name: "Bob", BackOfficeDeskChan: backOfficeDeskChan}
```

The code after all of these changes looks like this:

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

type DeskWorker struct {
	Name               string
	BackOfficeDeskChan chan string
}

func (dw *DeskWorker) StartWorkingDay(deskChan chan string, phoneChan chan string, shutdownChan chan struct{}) {
	for {
		select {
		case item := <-deskChan:
			dw.Process(item)
		case call := <-phoneChan:
			fmt.Printf("Desk worker %s received a call: %s\n", dw.Name, call)
		case <-shutdownChan:
			fmt.Println("the desk is closed - time to go home")
			return
		}
	}
}

func (dw *DeskWorker) Process(item string) {
	fmt.Printf("Desk worker %s received %s\n", dw.Name, item)
	fmt.Printf("Desk worker %s started checking ID of the customer with the %s item...\n", dw.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Desk worker %s finished checking ID of the customer with the %s item\n", dw.Name, item)

	fmt.Printf("Desk worker %s started passing %s to the back office...\n", dw.Name, item)
	dw.BackOfficeDeskChan <- item
	fmt.Printf("Desk worker %s passed %s to the back office\n\n", dw.Name, item)
}

type BackOfficeWorker struct {
	Name string
}

func (bow *BackOfficeWorker) StartWorkingDay(backOfficeDeskChan chan string, shutdownChan chan struct{}) {
	for {
		select {
		case item := <-backOfficeDeskChan:
			bow.Process(item)
		case <-shutdownChan:
			fmt.Printf("the back office is closed - time to go home, %s\n", bow.Name)
			return
		}
	}
}

func (bow *BackOfficeWorker) Process(item string) {
	fmt.Printf("Back office worker %s received %s\n", bow.Name, item)
	fmt.Printf("Back office worker %s started processing %s...\n", bow.Name, item)

	// to simulate long processing
	time.Sleep(10 * time.Second)

	fmt.Printf("Back office worker %s finished processing %s\n", bow.Name, item)
}

func main() {
	start := time.Now()

	deskChan := make(chan string)
	backOfficeDeskChan := make(chan string)
	phoneChan := make(chan string)
	deskShutdownChan := make(chan struct{})
	backOfficeDeskShutdownChan := make(chan struct{})
	wg := &sync.WaitGroup{}

	bobWorker := DeskWorker{Name: "Bob", BackOfficeDeskChan: backOfficeDeskChan}
	odaWorker := BackOfficeWorker{Name: "Oda"}
	robertWorker := BackOfficeWorker{Name: "Robert"}
	marthaWorker := BackOfficeWorker{Name: "Martha"}

	wg.Add(1)
	go func() {
		bobWorker.StartWorkingDay(deskChan, phoneChan, deskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		odaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		robertWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		marthaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
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

	deskShutdownChan <- struct{}{}
	// 3 stands for the number of back office workers
	for i := 0; i < 3; i++ {
		backOfficeDeskShutdownChan <- struct{}{}
	}

	close(phoneChan)
	close(deskChan)
	close(backOfficeDeskChan)
	close(deskShutdownChan)

	wg.Wait()

	end := time.Now()
	fmt.Printf("Execution time in milliseconds: %v\n", end.Sub(start).Milliseconds())
}
```

Let's run the code and see whether it works:

```text
Lisa gives away basketball
Desk worker Bob received a call: Has my package arrived?
Desk worker Bob received basketball
Desk worker Bob started checking ID of the customer with the basketball item...
Eric gives away teddy bear
Desk worker Bob finished checking ID of the customer with the basketball item
Desk worker Bob started passing basketball to the back office...
Desk worker Bob passed basketball to the back office

Back office worker Oda received basketball
Back office worker Oda started processing basketball...
Desk worker Bob received teddy bear
Desk worker Bob started checking ID of the customer with the teddy bear item...
Jenny gives away watermelon
Desk worker Bob finished checking ID of the customer with the teddy bear item
Desk worker Bob started passing teddy bear to the back office...
Desk worker Bob passed teddy bear to the back office

Desk worker Bob received watermelon
Desk worker Bob started checking ID of the customer with the watermelon item...
Ben gives away box
Back office worker Martha received teddy bear
Back office worker Martha started processing teddy bear...
Desk worker Bob finished checking ID of the customer with the watermelon item
Desk worker Bob started passing watermelon to the back office...
Desk worker Bob passed watermelon to the back office

Desk worker Bob received box
Desk worker Bob started checking ID of the customer with the box item...
Zlatan gives away football
Back office worker Robert received watermelon
Back office worker Robert started processing watermelon...
Desk worker Bob finished checking ID of the customer with the box item
Desk worker Bob started passing box to the back office...
Back office worker Oda finished processing basketball
Back office worker Oda received box
Back office worker Oda started processing box...
Desk worker Bob passed box to the back office

Desk worker Bob received football
Desk worker Bob started checking ID of the customer with the football item...
Back office worker Martha finished processing teddy bear
Desk worker Bob finished checking ID of the customer with the football item
Desk worker Bob started passing football to the back office...
Desk worker Bob passed football to the back office

the desk is closed - time to go home
Back office worker Martha received football
Back office worker Martha started processing football...
Back office worker Robert finished processing watermelon
the back office is closed - time to go home, Robert
Back office worker Oda finished processing box
the back office is closed - time to go home, Oda
Back office worker Martha finished processing football
the back office is closed - time to go home, Martha

=====================================
Execution time in milliseconds: 22003
```

The application works as expected, which is nice. On top of that, we can see quite a dramatic increase in the performance from 50 to 22 seconds. Exciting! But can we do better than that?

### Better than that

Triss is impressed with such results. But she is still able to see the room for improvement. She noticed that after serving 3 customers and picking up the 4th parcel, Bob hung for quite some time, as there were no free back-office workers to pass the parcels to. If you ran the code above, you might have noticed the same: 

- the `Desk worker Bob started passing box to the back office...` message was printed
- the execution paused
- in +-10 seconds, the flow resumed

Well, nothing unexpected has happened, as there are only 3 back-office employees for 5 customers. This makes Triss think: "OK, if I push hard enough during the next meeting with the executives, I'll make them allocate 2 more employees to this branch". And since she is an extremely good negotiator, Carl and Luna joined the post office crew in no time. 

![image](/images/drawings/20231221-0004.webp "A new post office setup with 5 new employees")

Let's change our code to represent this:

```go
func main() {
	start := time.Now()

	deskChan := make(chan string)
	backOfficeDeskChan := make(chan string)
	phoneChan := make(chan string)
	deskShutdownChan := make(chan struct{})
	backOfficeDeskShutdownChan := make(chan struct{})
	wg := &sync.WaitGroup{}

	bobWorker := DeskWorker{Name: "Bob", BackOfficeDeskChan: backOfficeDeskChan}
	odaWorker := BackOfficeWorker{Name: "Oda"}
	robertWorker := BackOfficeWorker{Name: "Robert"}
	marthaWorker := BackOfficeWorker{Name: "Martha"}
	carlWorker := BackOfficeWorker{Name: "Carl"}
	lunaWorker := BackOfficeWorker{Name: "Luna"}

	wg.Add(1)
	go func() {
		bobWorker.StartWorkingDay(deskChan, phoneChan, deskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		odaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		robertWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		marthaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		carlWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		lunaWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
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

	deskShutdownChan <- struct{}{}
	// 5 stands for the number of back office workers
	for i := 0; i < 5; i++ {
		backOfficeDeskShutdownChan <- struct{}{}
	}

	close(phoneChan)
	close(deskChan)
	close(backOfficeDeskChan)
	close(deskShutdownChan)

	wg.Wait()

	end := time.Now()
	fmt.Println("\n=====================================")
	fmt.Printf("Execution time in milliseconds: %v\n", end.Sub(start).Milliseconds())
}
```

Let's run it:

```text
Lisa gives away basketball
Eric gives away teddy bear
Desk worker Bob received basketball
Desk worker Bob started checking ID of the customer with the basketball item...
Desk worker Bob finished checking ID of the customer with the basketball item
Desk worker Bob started passing basketball to the back office...
Desk worker Bob passed basketball to the back office

Desk worker Bob received teddy bear
Desk worker Bob started checking ID of the customer with the teddy bear item...
Jenny gives away watermelon
Back office worker Robert received basketball
Back office worker Robert started processing basketball...
Desk worker Bob finished checking ID of the customer with the teddy bear item
Desk worker Bob started passing teddy bear to the back office...
Desk worker Bob passed teddy bear to the back office

Desk worker Bob received a call: Has my package arrived?
Desk worker Bob received watermelon
Desk worker Bob started checking ID of the customer with the watermelon item...
Back office worker Oda received teddy bear
Ben gives away box
Back office worker Oda started processing teddy bear...
Desk worker Bob finished checking ID of the customer with the watermelon item
Desk worker Bob started passing watermelon to the back office...
Desk worker Bob passed watermelon to the back office

Desk worker Bob received box
Desk worker Bob started checking ID of the customer with the box item...
Zlatan gives away football
Back office worker Carl received watermelon
Back office worker Carl started processing watermelon...
Desk worker Bob finished checking ID of the customer with the box item
Desk worker Bob started passing box to the back office...
Back office worker Martha received box
Back office worker Martha started processing box...
Desk worker Bob passed box to the back office

Desk worker Bob received a call: What about now?
Desk worker Bob received football
Desk worker Bob started checking ID of the customer with the football item...
Desk worker Bob finished checking ID of the customer with the football item
Desk worker Bob started passing football to the back office...
Desk worker Bob passed football to the back office

the desk is closed - time to go home
Back office worker Luna received football
Back office worker Luna started processing football...
Back office worker Robert finished processing basketball
the back office is closed - time to go home, Robert
Back office worker Oda finished processing teddy bear
the back office is closed - time to go home, Oda
Back office worker Carl finished processing watermelon
the back office is closed - time to go home, Carl
Back office worker Martha finished processing box
the back office is closed - time to go home, Martha
Back office worker Luna finished processing football
the back office is closed - time to go home, Luna

=====================================
Execution time in milliseconds: 15003

```

15 seconds - that's impressive, isn't it? And I'll tell you what: we can't make it faster with current constraints. Let's see why:

- Bob spends 1 second on each parcel before passing it further -> 5 seconds
- each back office worker spends 10 seconds on parcel -> 10 seconds
- total: 5 + 10 = 15 seconds

If 50 -> 15 seconds of progress doesn't sound like a significant boost for you, please remember that we are using artificially simplified time numbers. However, if we use more realistic ones (let's say, 5 minutes per customer), we'll see:

- the original code (from the Recap section): 25 minutes
- the latest code (with the 1 minute for `DeskWorker` and 4 minutes for `BackOfficeWorker` setup): 9 minutes

That's quite a difference, especially from the customer side. For a queue of 10 people, it will be even more significant if we have 10 workers: 50 vs. 19 minutes.

Does it mean we have just solved the post office issue we've discussed since [Part 1](https://n0rdy.foo/posts/20231207/go-channels-and-goroutines/) of this series? Yes, for this setup. But there is one more thing that we can improve here to make the code more realistic to the real post office.

## A realistic post office as a data pipeline

While it's fair to be satisfied with the work we have done so far, there is 1 hardcoded thing left that prevents us from being 100% happy: currently, we have hardcoded 5 customers into a list:

```go
zlatan := Customer{Name: "Zlatan", Item: "football"}
ben := Customer{Name: "Ben", Item: "box"}
jenny := Customer{Name: "Jenny", Item: "watermelon"}
eric := Customer{Name: "Eric", Item: "teddy bear"}
lisa := Customer{Name: "Lisa", Item: "basketball"}

queue := []Customer{lisa, eric, jenny, ben, zlatan}
```

But it's not realistic, as in real life the customers come and go during working hours rather than the queue being static. What should we use instead of the list to make things more dynamic? Hmm... I'm sure you know the answer - a channel! It's pretty easy to imagine the post office as a channel:

- the channel is initiated when the entry door to the post office has been opened - this means that the working day has begun
- once the customer enters the building through the entry door - it means that the new item has been written to the channel
- once the working day is over, the entry door is closed - the channel is closed too

Wait a minute! With the new approach, the number of customers is uncertain, so how many back office workers do we need? That's an excellent question! And from this moment, it becomes harder to explain this by using a real-life example. Luckily, we are not limited by the reality here, that's why:

- if you are into Sci-Fi, imagine that our back-office worker is a robot that can immediately clone itself once there is a new item to process in a way that the copy does all the clearance routine and disappears once it finishes its work. Basically, all the "original" robot has to do is create a clone and pass the parcel to them
- if you are more into fantasy (like me), imagine that our back-office worker is a wizard, and once there is a new item to process, they cast a spell so the clearance routine for this particular parcel is completed on its own
- or if you are more into Xmas mood these days, imagine that our back-office worker is Santa, who has an unlimited number of elves that are willing to help and take a parcel each

Regardless of the example, the idea I'm describing is the situation when there is a way to have an unlimited number of back-office workers spawned once there is a new parcel and released once it's processed. It's hard to imagine this in real life, as it's nearly impossible to have an unlimited number of employees due to such constraints as money and space. Unlike real life, it's different in software engineering: we can easily spawn a massive amount of goroutines even on a local machine because [goroutines are cheap](https://blog.nindalf.com/posts/how-goroutines-work/). And if we have a super powerful machine, their maximum number can be almost indefinite.

Let's use this knowledge to modify our code. As a big fan of fantasy, I'll go with the wizard scenario.

![image](/images/drawings/20231221-0005.webp "A new post office setup with a wizard")

We need to replace the existing back-office worker with a wizard one. Let's start by renaming `BackOfficeWorker` to `WizardBackOfficeWorker`:

```go
type WizardBackOfficeWorker struct {
	Name string
}
```

Within the `StartWorkingDay` function the `BackOfficeWorker` (the muggle one) has this logic for handling items:

```go
func (bow *BackOfficeWorker) StartWorkingDay(backOfficeDeskChan chan string, shutdownChan chan struct{}) {
	for {
		select {
		case item := <-backOfficeDeskChan:
			bow.Process(item)
		case <-shutdownChan:
			fmt.Printf("the back office is closed - time to go home, %s\n", bow.Name)
			return
		}
	}
}
```

This means that the workers processed the items themselves. As discussed above, the wizard won't do this but instead cast a spell (create a new worker) to delegate the processing process to it. On a code level, it means that we'll have to create a new goroutine for each item and let it do the job:

```go
func (bow *WizardBackOfficeWorker) StartWorkingDay(backOfficeDeskChan chan string, shutdownChan chan struct{}) {
	wg := &sync.WaitGroup{}

	for {
		select {
		case item := <-backOfficeDeskChan:
			fmt.Printf("Wizard %s received %s\n", bow.Name, item)

			wg.Add(1)
			go func(item string) {
				defer wg.Done()

				fmt.Printf("Wizard %s casted a spell to process %s\n", bow.Name, item)
				bow.Process(item)
			}(item)
		case <-shutdownChan:
			fmt.Printf("the back office is closed - time to go home, %s\n", bow.Name)
			wg.Wait()
			return
		}
	}
}
```

As you see, we had to introduce an instance of the `sync.WaitGroup` here to ensure that once the shutdown signal is received, the function won't quit before all the goroutines (spawned by it) are completed. That's why we do `wg.Wait()` before the `return` part. If you are not familiar with the concept of syncing, please refer to [Part 2](https://n0rdy.foo/posts/20231211/go-waitgroup/) of this series - I dedicated the entire post to this.

The powerful part here is that once the new goroutine is launched by the `go func(item string) {` code, the wizard is immediately available for the next parcel - no need to be blocked for the 10 seconds (processing time) like in the previous implementation.

There is one part that might be a bit confusing to you if you are new to Go:

```go
go func(item string) {
	defer wg.Done()

	fmt.Printf("Wizard %s casted a spell to process %s\n", bow.Name, item)
	bow.Process(item)
}(item)
```

In [Part 2](https://n0rdy.foo/posts/20231211/go-waitgroup/), we touched on a similar concept that looked like this:

```go
func() {
	// do something
}()
```

We said that what happens here is that the `func(){}` part declares a new anonymous function while the `()` part invokes it right away. 

What we have here is similar, but this time, our anonymous function accepts a parameter `item` of a type `string`:

```go
func(item string) {
	// do something
}
```

And to invoke it, we have to pass a value for this parameter - basically, the same as for the regular function, that's why we have the `(item)` part:

```go
func(item string) {
	// do something
}(item)
```

`item` is the variable defined above in the code: `case item := <-backOfficeDeskChan:`

But ok, let's get back to the original topic, as I believe you are more than equipped to understand the rest of the code there. Let's modify the `Process` function of the `WizardBackOfficeWorker` to specify that it is a cast who does the processing, not the wizard themselves:

```go
func (bow *WizardBackOfficeWorker) Process(item string) {
	fmt.Printf("Wizard %s's spell started processing %s...\n", bow.Name, item)

	// to simulate long processing
	time.Sleep(10 * time.Second)

	fmt.Printf("Wizard %s's spell finished processing %s\n", bow.Name, item)
}
```

As you can see, the `time.Sleep(10 * time.Second)` part still remains, as it takes the same time to do the magic parcel handling as the regular one.

The next step should be to replace the `slice` with a `channel` to define the customers' queue. But how should we get the customers to put in that channel? Another good question, you are on fire today! We need a function to generate random customers and, ideally, with random waiting to make it more realistic, as customers come to the post office at a different frequency. For that, I asked ChatGPT to generate a list of 50 random names and 50 random Xmas presents that fit into the postal package. I won't provide these lists here to save space, but feel free to check them out on a [GitHub repo](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20231221-go-data-pipeline/04-data-pipeline/generator.go). Once we have it, the rest of the generator code is trivial:

```go
var (
  names = {
    // a list of 50 names generated by ChatGPT
  }
  
  items = {
    // a list of 50 items generated by ChatGPT
  }
)

func generateCustomerWithRandomWait() Customer {
	// to simulate a random customer arrival while the post office is open
	time.Sleep(time.Duration(rand.Intn(5)) * time.Second)
	return randomCustomer()
}

func randomCustomer() Customer {
	return Customer{
		Name: randomName(),
		Item: randomItem(),
	}
}

func randomName() string {
	return names[rand.Intn(len(names))]
}

func randomItem() string {
	return items[rand.Intn(len(items))]
}
```

Now, we have everything to start using a channel for the queue representation. 

```go
// a new channel to send a signal to once it's time to close the post office
postOfficeShutdownChan := make(chan struct{})

// ...............
// some other code
// ...............

queueChan := make(chan Customer)
go func() {
	for {
		select {
		case <-postOfficeShutdownChan:
			fmt.Println("the post office is closed - time to go home")
			close(queueChan)
			return
		default:
			// to simulate a random customer arrival while the post office is open
			customer := generateCustomerWithRandomWait()
			fmt.Printf("%s enters the post office\n", customer.Name)
			queueChan <- customer
		}
	}
}()

for customer := range queueChan {
	deskChan <- customer.GiveAway()
}
```

As you can see, inside the `go func()`, the code will generate a new customer and put it into the `queueChan` within the `default` section until there is a signal to shutdown the post office. If the shutdown signal is received, the `queueChan` will be closed immediately. A for loop below consumes items from the `queueChan` while it is open. We don't have the code for sending the signal to the `postOfficeShutdownChan`, so let's add it:

```go
postOfficeShutdownChan := make(chan struct{})

// to simulate a long working day
time.AfterFunc(5*time.Minute, func() {
	postOfficeShutdownChan <- struct{}{}
})
```

`time.AfterFunc` is a new concept for us, but it's pretty straightforward: it accepts a time duration and a function and invokes this function after the provided duration has passed. In our case, it will send the shutdown signal to the `postOfficeShutdownChan` in 5 minutes. This way, we are going to simulate a working day - feel free to change it to 8 hours if you have such a lot of time to wait for the program completion =)

Here is the entire code after our changes:

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

type DeskWorker struct {
	Name               string
	BackOfficeDeskChan chan string
}

func (dw *DeskWorker) StartWorkingDay(deskChan chan string, phoneChan chan string, shutdownChan chan struct{}) {
	for {
		select {
		case item := <-deskChan:
			dw.Process(item)
		case call := <-phoneChan:
			fmt.Printf("Desk worker %s received a call: %s\n", dw.Name, call)
		case <-shutdownChan:
			fmt.Println("the desk is closed - time to go home")
			return
		}
	}
}

func (dw *DeskWorker) Process(item string) {
	fmt.Printf("Desk worker %s received %s\n", dw.Name, item)
	fmt.Printf("Desk worker %s started checking ID of the customer with the %s item...\n", dw.Name, item)

	// to simulate long processing
	time.Sleep(1 * time.Second)

	fmt.Printf("Desk worker %s finished checking ID of the customer with the %s item\n", dw.Name, item)

	fmt.Printf("Desk worker %s started passing %s to the back office...\n", dw.Name, item)
	dw.BackOfficeDeskChan <- item
	fmt.Printf("Desk worker %s passed %s to the back office\n", dw.Name, item)
}

type WizardBackOfficeWorker struct {
	Name string
}

func (bow *WizardBackOfficeWorker) StartWorkingDay(backOfficeDeskChan chan string, shutdownChan chan struct{}) {
	wg := &sync.WaitGroup{}

	for {
		select {
		case item := <-backOfficeDeskChan:
			fmt.Printf("Wizard %s received %s\n", bow.Name, item)

			wg.Add(1)
			go func(item string) {
				defer wg.Done()

				fmt.Printf("Wizard %s casted a spell to process %s\n", bow.Name, item)
				bow.Process(item)
			}(item)
		case <-shutdownChan:
			fmt.Printf("the back office is closed - time to go home, %s\n", bow.Name)
			wg.Wait()
			return
		}
	}
}

func (bow *WizardBackOfficeWorker) Process(item string) {
	fmt.Printf("Wizard %s's spell started processing %s...\n", bow.Name, item)

	// to simulate long processing
	time.Sleep(10 * time.Second)

	fmt.Printf("Wizard %s's spell finished processing %s\n", bow.Name, item)
}

func main() {
	start := time.Now()

	deskChan := make(chan string)
	backOfficeDeskChan := make(chan string)
	phoneChan := make(chan string)
	deskShutdownChan := make(chan struct{})
	backOfficeDeskShutdownChan := make(chan struct{})
	postOfficeShutdownChan := make(chan struct{})

	// to simulate a long working day
	time.AfterFunc(5*time.Minute, func() {
		postOfficeShutdownChan <- struct{}{}
	})

	wg := &sync.WaitGroup{}

	bobWorker := DeskWorker{Name: "Bob", BackOfficeDeskChan: backOfficeDeskChan}
	wizardBackOfficeWorker := WizardBackOfficeWorker{Name: "Radagast"}

	wg.Add(1)
	go func() {
		bobWorker.StartWorkingDay(deskChan, phoneChan, deskShutdownChan)
		wg.Done()
	}()

	wg.Add(1)
	go func() {
		wizardBackOfficeWorker.StartWorkingDay(backOfficeDeskChan, backOfficeDeskShutdownChan)
		wg.Done()
	}()

	go func() {
		time.Sleep(5 * time.Second)
		phoneChan <- "Has my package arrived?"
		time.Sleep(1 * time.Second)
		phoneChan <- "What about now?"
	}()

	queueChan := make(chan Customer)
	go func() {
		for {
			select {
			case <-postOfficeShutdownChan:
				fmt.Println("the post office is closed - time to go home")
				close(queueChan)
				return
			default:
				// to simulate a random customer arrival while the post office is open
				customer := generateCustomerWithRandomWait()
				fmt.Printf("%s enters the post office\n", customer.Name)
				queueChan <- customer
			}
		}
	}()

	for customer := range queueChan {
		deskChan <- customer.GiveAway()
	}

	deskShutdownChan <- struct{}{}
	backOfficeDeskShutdownChan <- struct{}{}

	wg.Wait()

	close(phoneChan)
	close(deskChan)
	close(backOfficeDeskChan)
	close(deskShutdownChan)
	close(backOfficeDeskShutdownChan)
	close(postOfficeShutdownChan)

	end := time.Now()
	fmt.Println("\n=====================================")
	fmt.Printf("Execution time in milliseconds: %v\n", end.Sub(start).Milliseconds())
}
```

Let's run this code. It will take 5 minutes to complete, so please feel free to pour a cup of tea or coffee in the meantime. You should be able to see a lot of activity printed into the terminal that ends with something like this:

```text
............................................................................................
the post office is closed - time to go home
Anika gives away handwritten holiday card
Desk worker Bob received handwritten holiday card
Desk worker Bob started checking ID of the customer with the handwritten holiday card item...
Wizard Radagast's spell finished processing miniature snowman kit
Desk worker Bob finished checking ID of the customer with the handwritten holiday card item
Desk worker Bob started passing handwritten holiday card to the back office...
Desk worker Bob passed handwritten holiday card to the back office
the desk is closed - time to go home
Wizard Radagast received handwritten holiday card
the back office is closed - time to go home, Radagast
Wizard Radagast casted a spell to process handwritten holiday card
Wizard Radagast's spell started processing handwritten holiday card...
Wizard Radagast's spell finished processing teddy bear
Wizard Radagast's spell finished processing football
Wizard Radagast's spell finished processing pocket-sized board games
Wizard Radagast's spell finished processing festive cocktail mixers
Wizard Radagast's spell finished processing wine sampler
Wizard Radagast's spell finished processing handmade jewelry
Wizard Radagast's spell finished processing handwritten holiday card

=====================================
Execution time in milliseconds: 311080

```

This is a perfect post office implementation:

- we have a start of the working day
- we have a dynamic customers that come and go
- we have a fast track to serve the customers
- we have an end of the working day that gracefully shuts down the execution by allowing already started work to finish

And this is exactly how the data pipelines look like out there: if we drop the technical part, what is left at the end of the day is the data flow pipeline, where we can see a data transformation happening on each step with a clear [producer-consumer pattern](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem):

![image](/images/drawings/20231221-0006.webp "A post office as a data pipeline")

Good job, my friend! Triss is happy, so should we be, as we can officially claim that we have just solved the inefficient post office queue handling issue we've been discussing since [Part 1](https://n0rdy.foo/posts/20231207/go-channels-and-goroutines/). 

## Where to go from here

Before wrapping the ["Go concurrency simplified" series](https://n0rdy.foo/series/go-concurrency-simplified) up, let me give you some hints on where you can go from here:

- try to play with a code we implemented and change it: for example, introduce a new type of worker - a delivery worker that is responsible for picking up the parcels processed by the back office and delivering them to the destination; then try adding a new desk, so there are 2 desk workers to handle the customers, etc.
- implement your own data pipeline: for example, you can create a search engine that will receive a path to a directory to scan, then read all the text files from this directory & subdirectories and search for some word you choose within those files - it should be doable with the knowledge you've learned so far; feel free to share links your GitHub repos with the program you've implemented in the comments section
- take a look at the concurrent code written by other devs out there: for example, feel free to check the internals of my library [Pippin](https://github.com/n0rdy/pippin), but I bet there are many better projects out there to learn from - Google/Bing/DuckDuckGo/Kagi and ChatGPT can help to find the right one
- check the [Go documentation](https://go.dev/doc/), especially the sections about the concurrency
- check the [Go documentation](https://pkg.go.dev/sync@go1.21.5) for the `sync` package to see what else Go has to offer for the synchronization routines
- check the [Go documentation](https://pkg.go.dev/golang.org/x/sync) for the `x/sync` package to see the beta version of the features and data structures that Go devs are experimenting with - one day, they might (or might not) be a part of the language core
- watch [a great talk by Rob Pike](https://www.youtube.com/watch?v=f6kdp27TYZs) - one of the Go creators on the "Go Concurrency Patterns" topic
- if you understood the "A realistic post office as a data pipeline" chapter of this post, you have a good foundation to learn about the [event-driven architecture](https://aws.amazon.com/event-driven-architecture/) as it uses the similar principles that we applied here
- also, this knowledge applies to learning more about data engineering, as this field of software engineering relies heavily on the event-driven approach via tools like [Spark](https://spark.apache.org/), [Flink](https://flink.apache.org/), [Kafka](https://kafka.apache.org/), etc.

## Farewell

It is time to wrap up the ["Go concurrency simplified" series](https://n0rdy.foo/series/go-concurrency-simplified). It was quite a journey, and I'm more than happy that we've made it through together. I hope you've learned a lot and are ready to write a concurrent program on your own using the knowledge from this series.

Even though it's the end of this series, there will be more posts about different concepts of software engineering and tech, including (but not limited to) different parts of the Go concurrency concepts. If you don't want to miss out on my future publications, consider subscribing to [my newsletter](https://mail.n0rdy.foo/subscription/form) to stay tuned and get my new posts delivered right to your email inbox. 

Also, you can find me on [X/Twitter](https://twitter.com/_n0rdy_), [Mastodon,](https://me.dm/@n0rdy) and [Bluesky](https://bsky.app/profile/n0rdy.bsky.social).

I wish you a Merry Christmas and a Happy New Year for those who celebrate - careful with the amount of "Last Christmas" and "All I want for Christmas" songs you listen to =)

Have fun! =)