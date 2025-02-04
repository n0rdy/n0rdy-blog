---
title: "Understanding CORS"
image: "/covers/drawings/20240103.webp"
draft: false
date: 2024-01-03T17:00:00+01:00
tags: ["web", "tutorial", "beginners", "eli5"]
---
Hello there! Happy New Year! I hope you had an opportunity to get some rest during the winter holidays and maybe even made a snowman or two =)

Several days ago, I had a dialog with a friend of mine (let's call her Eowyn) who has recently started her path in software engineering:

**Eowyn:** Hey, buddy! I'm building a web project that has the frontend and backend parts. Whenever I click a button on the UI (that triggers a DELETE request to the server), my browser blows with these errors:
![image](/images/screenshots/20240103-0001.webp)
I tried calling the same endpoint with the `curl` command, and it works. I did the same via Postman, and it also works. It makes no sense! What the heck?

**Me:** Hehe, congrats, this is a historical moment in your career - the day you discovered CORS! =)

To her credit, she didn't want my help in fixing the issue, she wanted to understand it. That's why I spent the next half an hour explaining the topic of CORS. And since that was not the first time I had to do that, I realized that it is an excellent opportunity to write a post on this subject and, next time, share the link to it instead of explaining it once again.

**A disclaimer**: this post is called `Understanding CORS,` and that's exactly the goal we are going to pursue today. The target audience is the folks who know nothing or very little about the CORS and would like to learn what's happening on a high level behind the errors like above. We won't dive deep and won't cover all the use-/edge-cases of this topic - if you are looking for more advanced reading, here is [an excellent longread from Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

## Real-life example

If you are following my blog, you might have noticed that I like to explain technical aspects via real-life scenarios/examples. I believe this is the shortest and easiest path to understanding. We'll do the same today.

Let me introduce you to Geralt, a witcher, one of the best in his craft. Due to his skills and due to the number of dangerous beasts out there, Geralt's services are in high demand. People contact him so often that he has to hire Bob, a secretary, to manage the calls he receives. Thus, once somebody calls Geralt's office, it is Bob who manages the call. It usually works like this:

- once somebody calls Geralt's office, Bob picks up the phone, gathers the info about the call (who, why, and where they are calling from), and asks them to wait

![image](/images/drawings/20240103-0001.webp)

- Bob calls Geralt, shares the info about the caller, and asks whether Geralt is willing to talk to them

![image](/images/drawings/20240103-0002.webp)

As we can see, Geralt replied that if, during the next 2 hours, people were calling from the place called Ingenheim about hunting, Bob should connect them with him right away.

- Bob gets back to the caller and connects them with Geralt

![image](/images/drawings/20240103-0003.webp)

- once there is another call within the next 2 hours coming from a different place than Ingenheim, Bob rejects the request to connect them with Geralt:

![image](/images/drawings/20240103-0004.webp)

However, if there is someone (like the witcher's buddy Yarpen) who knows Geralt's direct phone number, they can still call him regardless of the place and reason for making a call:

![image](/images/drawings/20240103-0005.webp)

I think this should make perfect sense. But how come is it related to the CORS?

## From the real life to software engineering

If we try to map the real-life example we used to the software engineering concepts, it will look like this:

- the person who calls the office - frontend application
- Bob - browser
- Geralt - backend application

And the step-by-step sequence of events is the following:

- once the frontend app tries to send a request to the backend API, a browser makes a so-called pre-flight request to the backend API and asks about the allowed options: who is allowed to call the API and what type of requests can be sent 
- the API sends a response with such options and (optionally) includes the duration for which the browser should cache those settings and rely on them instead of making another pre-flight request
- if the frontend app and the request it tries to make are within the allow list, the browser lets it through
- otherwise, the request is rejected with the error we saw at the very beginning of this post

However, this mechanism is easy to bypass by skipping the browser and sending a request directly (like via `curl`, `Postman,` or any other HTTP client) - that's exactly what Yarpen did above by calling Geralt's direct phone number instead of the office's one.

Shall we assume that CORS is quite a poor security mechanism, as we can easily bypass it? The answer is "it depends". If we'd like to secure our API in a way that only the allowed services can call it, CORS is not a good idea as a single-on solution, as it doesn't apply to server-to-server communications. CORS' primary use case is CSRF (Cross-Site Request Forgery) attack prevention. Let's discuss what it is.

### CSRF

Imagine that today is a payday, so you have just logged in to your online bank account to check the balance, and you can see that the money has arrived - nice! Glad and happy you have opened the social network feed and started scrolling it. And, all of a sudden, there is another "bang": someone has posted a link with the description that there is a big sale on your favorite pickles - that's your lucky day, isn't it? You followed the link, but there are no pickles behind it, just a blank page. "That's unfair! How could someone make such jokes?" Was your thought, after which you closed that page.

Suppose you have monitored your network traffic while visiting the "pickles scam" page. In that case, you'd notice that even though the page was blank, it actually contained a small JavaScript code that made a request to the API of your bank and requested a money transfer to the unknown (for you) account. You've been logged in to your online bank account, so the bank is unaware that you made this transfer without even knowing. But if the bank has CORS configs enabled, the browser will verify whether this "pickles scam" domain is allowed to call the bank's API, and since it's not allowed, the request will be rejected. Even though you are left without pickles, your money is safe.

I asked ChatGPT to provide me with more examples of CSRF attacks, and here is what I got:

**Example 1: Changing Email Address**

1. ***Scenario***: Alice is logged into her email account on `emailservice.com`.
2. ***Attack***: She then visits a malicious website, `malicioussite.com`, which contains a hidden form that is automatically submitted by JavaScript. This form is crafted to send a POST request to `emailservice.com` to change her email settings (like her recovery email address).
3. ***Result***: If `emailservice.com` doesn't have proper CSRF protections, it might process this request as if Alice intentionally submitted it, leading to her recovery email being changed without her knowledge.

**Example 2: Social Media Post**

1. ***Scenario***: Bob is logged into a social media platform.
2. ***Attack***: He clicks on a link that leads him to a malicious site. This site contains a script that makes a request to the social media platform to post a message or send a message to all his contacts.
3. ***Result***: If the social media platform doesn't verify the authenticity of the request, it could result in spam or malicious messages being sent from Bob's account.

**Example 3: Changing Password**

1. ***Scenario***: Dana is logged into a forum.
2. ***Attack***: She receives an email with a link to an interesting article. Clicking the link takes her to a website that secretly contains a form that sends a request to the forum to change her password.
3. ***Result***: Without CSRF protection, Dana’s password could be changed without her consent, potentially locking her out of her account.

As you can see, all of this could have been avoided if the server had CORS configured. How can we do that, though? Let's finally see some code!

## Some code

As we have already determined, there are 3 types of actors here:

- browser
- frontend
- backend

In this example, we'll use Brave, a Chromium-based browser, for UI simplicity purposes. Let's jump into the code, then.

### Backend

We are going to build a tiny books API that has 3 endpoints:

- get all books
- add a new book
- delete all books

It's a dummy application, and that's why we'll store all the data in the memory. Here is the entire Go code for our backend:

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"github.com/go-chi/chi/v5"
	"net/http"
)

var books = []string{"The Lord of the Rings", "The Hobbit", "The Silmarillion"}

type Book struct {
	Title string `json:"title"`
}

func main() {
	err := runServer()
	if err != nil {
		if errors.Is(err, http.ErrServerClosed) {
			fmt.Println("server shutdown")
		} else {
			fmt.Println("server failed", err)
		}
	}
}

func runServer() error {
	httpRouter := chi.NewRouter()

	httpRouter.Route("/api/v1", func(r chi.Router) {
		r.Get("/books", getAllBooks)
		r.Post("/books", addBook)
		r.Delete("/books", deleteAllBooks)
	})

	server := &http.Server{Addr: "localhost:8888", Handler: httpRouter}
	return server.ListenAndServe()
}

func getAllBooks(w http.ResponseWriter, req *http.Request) {
	respBody, err := json.Marshal(books)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write(respBody)
}

func addBook(w http.ResponseWriter, req *http.Request) {
	var book Book
	err := json.NewDecoder(req.Body).Decode(&book)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	books = append(books, book.Title)

	w.WriteHeader(http.StatusCreated)
}

func deleteAllBooks(w http.ResponseWriter, req *http.Request) {
	books = []string{}

	w.WriteHeader(http.StatusNoContent)
}
```

You can find it in [this GitHub repo](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20240103-cors).

As you can see, I used an external dependency, `github.com/go-chi/chi/v5` , for the API. It is completely optional to achieve the same with pure Go, but I did that to improve the readability of the code. Other than that, the code is pretty simple: it reads, writes, or deletes the data from/into the slice of books and sends a successful response.

Let's run it: the server will be running as `http://localhost:8888`

It's time to define a frontend now:

### Frontend

Here we need a simple HTML page with JS scripts to make requests to the backend API, and a tiny Go server to serve the page. Here is the HTML code:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Books</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet"
          integrity="sha384-T3c6CoIi6uLrA9TneNEoa7RxnatzjcDSCmG1MXxSR1GAsXEV/Dwwykc2MPK8M2HN" crossorigin="anonymous">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"
            integrity="sha384-C6RzsynM9kWDrMNeT87bh95OGNyZPhcTNXj1NW7RuBCsyN/o0jlpcV8Qyq46cDfL"
            crossorigin="anonymous"></script>
</head>
<body>
<div class="container p-3">
    <button type="button" class="btn btn-primary" id="getBooks">Get books</button>
    <button type="button" class="btn btn-danger" id="deleteAllBooks">Delete all books</button>
    <br>
    <br>

    <form>
        <div class="mb-3">
            <label for="inputBookTitle" class="form-label">Book title</label>
            <input type="text" class="form-control" id="inputBookTitle" aria-describedby="emailHelp">
        </div>
        <button type="submit" class="btn btn-primary">Add</button>
    </form>
</div>

<script>
  function getBooks () {
    fetch('http://localhost:8888/api/v1/books')
      .then(response => response.json())
      .then(data => {
        const booksList = document.querySelector('.books-list')
        if (booksList) {
          booksList.remove()
        }

        const ul = document.createElement('ul')
        ul.classList.add('books-list')
        data.forEach(book => {
          const li = document.createElement('li')
          li.innerText = book
          ul.appendChild(li)
        })
        document.body.appendChild(ul)
      })
  }

  function deleteAllBooks () {
    fetch('http://localhost:8888/api/v1/books', {
      method: 'DELETE'
    })
      .then(response => {
        if (response.status === 204) {
          getBooks()
        } else {
          const div = document.createElement('div')
          div.innerText = 'Something went wrong'
          document.body.appendChild(div)
        }
      })
  }

  const getBooksButton = document.getElementById('getBooks')
  const deleteAllBooksButton = document.getElementById('deleteAllBooks')
  const input = document.querySelector('input')
  const form = document.querySelector('form')

  getBooksButton.addEventListener('click', () => getBooks())
  deleteAllBooksButton.addEventListener('click', () => deleteAllBooks())

  form.addEventListener('submit', (event) => {
    event.preventDefault()

    const title = input.value

    fetch('http://localhost:8888/api/v1/books', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ title })
    })
      .then(response => {
        if (response.status === 201) {
          input.value = ''
          getBooks()
        } else {
          const div = document.createElement('div')
          div.innerText = 'Something wend wrong'
          document.body.appendChild(div)
        }
      })
  })
</script>
</body>
</html>
```

And a Go server:

```go
package main

import (
	"errors"
	"fmt"
	"github.com/go-chi/chi/v5"
	"net/http"
)

func main() {
	err := runServer()
	if err != nil {
		if errors.Is(err, http.ErrServerClosed) {
			fmt.Println("client server shutdown")
		} else {
			fmt.Println("client server failed", err)
		}
	}
}

func runServer() error {
	httpRouter := chi.NewRouter()

	httpRouter.Get("/", serveIndex)

	server := &http.Server{Addr: "localhost:3333", Handler: httpRouter}
	return server.ListenAndServe()
}

func serveIndex(w http.ResponseWriter, req *http.Request) {
	http.ServeFile(w, req, "20240103-cors/01-no-cors/client/index.html")
}
```

If we run the Go code, it will serve the HTML page to http://localhost:3333` 

Based on my drawings, you might have already noticed that I have an exceptional talent in design that's the UI is another masterpiece of mine:

![image](/images/screenshots/20240103-0002.webp)

Let's test what we have now.

### Testing time

Navigate to http://localhost:3333/ in your browser, and open a DevTools there: it's `Option+Command+I` on MacOS or `View -> Developer -> Developer Tools` . Ideally, we should be able to see the `Network` tab and the `Console` like this:

![image](/images/screenshots/20240103-0003.webp)

Let's try to add a new book - "Harry Potter", and click "Add". Boom!

![image](/images/screenshots/20240103-0004.webp)

The error looks somewhat familiar, doesn't it? It's not exactly the same as at the beginning of this post, but it's very similar. Let's try to understand what it actually means.

You might have noticed that our backend code doesn't mention CORS at all. It is indeed true, we haven't implemented any CORS configs as of now. But that doesn't matter for the browser: it tried to make a preflight request anyway

![image](/images/screenshots/20240103-0005.webp)

If we click on it, it will expand some details, and we can see that the browser tried to make an OPTIONS request to the same path as the add book endpoint, and received a `405 Method Not Allowed` response, which makes sense as we haven't defined the OPTIONS endpoint in our backend.

![image](/images/screenshots/20240103-0006.webp)

If we get back to our real-life example for a moment, what happened here is the following:

- somebody calls Geralt's office, so Bob picks up the phone, gathers the info about the call (who, why, and where they are calling from), asks them to wait, and calls Geralt to double-check
- but "Houston, we have a problem": Geralt's phone is off, so there's no way Bob can get Geralt's preferences for today

Will Bob connect the caller with Geralt somehow? Or will he agree on the services required without talking to the witcher? Of course, no, Bob has to reject the customer for now. The same happens in our case: since the browser has no idea about the backend API CORS settings, it simply refuses to make any requests there - safety first!

Let's fix that!

### Fixing time

The frontend app remains the same, but as for the backend, we need to make a few changes:

- introduce a new function to enable CORS:

```go
func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3333")
	
  // specifies which methods are allowed to access this API
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE")
	
  // specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type")
	
  // specifies for how long the browser can cache the results of a preflight request (in seconds)
	w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

As you can see, we have introduced 4 CORS settings:

- domain from which it is allowed to call our API - `http://localhost:3333`
- HTTP methods that are allowed to be used with our API -  `GET, POST, DELETE`
- request headers that are allowed to be passed to our API - `Accept, Content-Type`
- time for which the browser can remember and cache these settings - 2 hours in seconds

A short disclaimer: there are more CORS-related headers, but these are enough for understanding. I'll share the links to dive deeper at the end of this post.

- introduce an OPTIONS endpoint alongside the existing ones and a function to handle it:

```go
...
	httpRouter.Route("/api/v1", func(r chi.Router) {
		r.Options("/books", corsOptions)
		r.Get("/books", getAllBooks)
		r.Post("/books", addBook)
		r.Delete("/books", deleteAllBooks)
	})
...

func corsOptions(w http.ResponseWriter, req *http.Request) {
	enableCors(w)
	w.WriteHeader(http.StatusOK)
}
```

- add `enableCors` invocation to the existing functions of other endpoints, for example:

```go
func getAllBooks(w http.ResponseWriter, req *http.Request) {
	respBody, err := json.Marshal(books)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	enableCors(w)
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write(respBody)
}
```

The final code looks like this:

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"github.com/go-chi/chi/v5"
	"net/http"
	"strconv"
)

var books = []string{"The Lord of the Rings", "The Hobbit", "The Silmarillion"}

type Book struct {
	Title string `json:"title"`
}

func main() {
	err := runServer()
	if err != nil {
		if errors.Is(err, http.ErrServerClosed) {
			fmt.Println("server shutdown")
		} else {
			fmt.Println("server failed", err)
		}
	}
}

func runServer() error {
	httpRouter := chi.NewRouter()

	httpRouter.Route("/api/v1", func(r chi.Router) {
		r.Options("/books", corsOptions)
		r.Get("/books", getAllBooks)
		r.Post("/books", addBook)
		r.Delete("/books", deleteAllBooks)
	})

	server := &http.Server{Addr: "localhost:8888", Handler: httpRouter}
	return server.ListenAndServe()
}

func corsOptions(w http.ResponseWriter, req *http.Request) {
	enableCors(w)
	w.WriteHeader(http.StatusOK)
}

func getAllBooks(w http.ResponseWriter, req *http.Request) {
	respBody, err := json.Marshal(books)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	enableCors(w)
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write(respBody)
}

func addBook(w http.ResponseWriter, req *http.Request) {
	var book Book
	err := json.NewDecoder(req.Body).Decode(&book)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	books = append(books, book.Title)

	enableCors(w)
	w.WriteHeader(http.StatusCreated)
}

func deleteAllBooks(w http.ResponseWriter, req *http.Request) {
	books = []string{}

	enableCors(w)
	w.WriteHeader(http.StatusNoContent)
}

func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3333")

	// specifies which methods are allowed to access this API (GET is allowed by default)
	w.Header().Set("Access-Control-Allow-Methods", "POST, DELETE")

	// specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

	// specifies for how long the browser can cache the results of a preflight request (in seconds)
	w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

Let's run this code to see whether it works. It should - if you experience any issues, try to restart both frontend and backend apps, and even access http://localhost:3333 via incognito mode, as there might be some issues due to browser caching. 

If you click all the buttons a few times by trying to add some books and get/delete all of them, you'll see that it works as expected. Even more, if we take a look at the `Network` tab, we'll see that there is only 1 preflight request in total:

![image](/images/screenshots/20240103-0007.webp)

Of course, we are not surprised about that, as we have just configured the `Access-Control-Max-Age` header for those purposes.

I believe you should already have a good understanding of CORS. But let's take a final step and try to mess with CORS configs to see how it breaks the flow again.

### Breaking (purposely) time

We'll do that in a one-by-one fashion by breaking (and reverting) each of the CORS configs. Please remember to restart both frontend and backend apps once we apply a change, as otherwise, we won't see any effect. Let's gooooo!

#### Access-Control-Allow-Origin

Our frontend app runs on `http://localhost:3333`, and that's exactly the value we have within the `Access-Control-Allow-Origin` header. Let's change it to something else:

```go
func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "http://example.com")

	// specifies which methods are allowed to access this API
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE")

	// specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type")

	// specifies for how long the browser can cache the results of a preflight request (in seconds)
	w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

Let's restart the apps and see what will happen if we try to use them.

![image](/images/screenshots/20240103-0008.webp)

Well, the outcome is as expected: only `http://example.com` is allowed to call the API. What's even more interesting is the fact that if we put the localhost value there but with a different port, we'll still get an error:

![image](/images/screenshots/20240103-0009.webp)

The same applies for `http` vs `https` - CORS is very strict:

![image](/images/screenshots/20240103-0010.webp)

There is a possibility, though, of allowing anyone to call your API - by using `*` as a value for the `Access-Control-Allow-Origin` header:

```go
func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "*")

	// specifies which methods are allowed to access this API
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE")

	// specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type")

	// specifies for how long the browser can cache the results of a preflight request (in seconds)
	w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

Even if we try to run our frontend app on different ports, it will pass the CORS step successfully. Use this only if you know what you are doing!

Ok, let's change the `Access-Control-Allow-Origin` to the original one and jump to the next one.

#### Access-Control-Allow-Methods

Before started playing with this header, let me share an important gotcha with you: GET and POST methods are allowed by default regardless the settings. It means that `w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE")` has the same effect as `w.Header().Set("Access-Control-Allow-Methods", "DELETE")`

That's why there is no need to delete them and wonder why the application still works. However, let's try to get rid of the `DELETE` one:

```go
func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3333")

	// specifies which methods are allowed to access this API
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST")

	// specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type")

	// specifies for how long the browser can cache the results of a preflight request (in seconds)
	w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

Once we restart both apps, we'll see that there are no issues with getting the books or adding a new one, but trying to delete them fires an error.

![image](/images/screenshots/20240103-0011.webp)

If the flow works for you with no issues, it is the browser caching ([the 2nd hardest thing](https://martinfowler.com/bliki/TwoHardThings.html) in the computer science) who spoils the fun. To fix that, try either:

- ticking `Disable cache` box under the `Network` tab
- opening a new incognito window

As we can see, the errors clearly states the `Method DELETE is not allowed by Access-Control-Allow-Methods in preflight response` - we knew that already, didn't we?

Let's revert the values and proceed to the next header.

#### Access-Control-Allow-Headers

We don't explicitly use `Accept` header in our code, that's why let's keep it there. But we do use the `Content-Type` one when we make a POST request. You know what to do with it =)

```go
func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3333")

	// specifies which methods are allowed to access this API
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE")

	// specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Accept")

	// specifies for how long the browser can cache the results of a preflight request (in seconds)
	w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

If we click all the buttons we have, we'll see that getting and deleting books work, but adding a new one fails as expected:

![image](/images/screenshots/20240103-0012.webp)

`Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response.` makes sense.

Production applications use way more headers, that's why review them carefully in order to configure CORS properly.

Time to revert the changes and proceed to the last header in out list.

#### Access-Control-Max-Age

If not set, the default value is `0` which means that browser shouldn't cache the prefligh request data at all. Let's comment the header out and see what happens next:

```go
func enableCors(w http.ResponseWriter) {
	// specifies which domains are allowed to access this API
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3333")

	// specifies which methods are allowed to access this API
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE")

	// specifies which headers are allowed to access this API
	w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type")

	// specifies for how long the browser can cache the results of a preflight request (in seconds)
	//w.Header().Set("Access-Control-Max-Age", strconv.Itoa(60*60*2))
}
```

Observe the `Network` tab after restarting the app:

![image](/images/screenshots/20240103-0013.webp)

No cache policy forces browser to make a preflight request each time there is a POST or DELETE call (there are no preflight requests for GET calls - it's a rule). It's acceptable while doing testing, but it leads to the undesired load to your servers if there is no `Access-Control-Max-Age` header provided, that's why the rule of thumb is to have it. There is no ideal value for it though, it depends on your situation and requirements.

And that's basically all I wanted to show you today. I'm sure you have a good understanding about CORS now, and can dive deeper on your own to learn even more on that topic.

## Where to go from here

As I mentioned on the disclaimer section, there is [an excellent longread from Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) on that topic - I know that now you are well-equipped for it. 

If you are more into the RFC types of read, [here is the one that covers CORS as well](https://fetch.spec.whatwg.org/#http-cors-protocol) - the link leads exactly to the CORS section, but feel free to read all of it, if you have time and inspiration. 

Anyway, it's getting late in my time zone, so I need to call it a day and get some sleep. I hope you learn some new today and get a good grasp on what CORS is and how it works under the hood. See you in the next posts!

Have fun =)