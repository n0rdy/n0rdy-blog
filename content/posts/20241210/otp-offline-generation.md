---
title: "Demystifying OTPs: the logic behind the offline generation of tokens"
image: "/covers/drawings/20241210.jpg"
draft: false
date: 2024-12-10T17:30:00+02:00
tags: ["tech", "web", "security"]
---

Hello there! Another evening, on my way back home, I decided to check the mailbox. I don't mean my email inbox, but the old-school actual box where the postman puts the physical letters. And to my great surprise, I found an envelope there with something inside! While opening it, I spent a few moments hoping that it's the decades delayed letter from Hogwarts. But then I had to get back down to Earth, once I noticed that it's a boring "grown-up" letter from the bank. I skimmed through the text and realized that my "digital-only" bank for cool kids had been acquired by the biggest player on the local market. And as a token of the new beginning, they added this to the envelope:

![image](/images/drawings/20241210-0001.jpg)

Alongside the instructions on how to use it.

If you are like me, and have never come across such a piece of the tech innovation, let me share what I learned from the letter: new owners decided to enforce the security policies from their company, which means that all the user accounts from now on will have the [MFA](https://en.wikipedia.org/wiki/Multi-factor_authentication) enabled (kudos for that, btw). And the device you could see above generates one-time tokens 6-digits long that are used as a second factor while logging in to your bank account. Basically, the same way as apps like Authy, Google Authenticator or 2FAS work, but in a physical shape. 

So, I gave it a try, and the login process went smoothly: the device showed me a 6-digits code, I entered it in my banking app, and this got me in. Hooray! But then something struck me: how does this thing work? There is no way it is connected to the internet somehow, but it manages to generate correct codes that are accepted by my bank server. Hm... Could it have a SIM-card or something similar inside? No way!

Realizing that my life will never be the same, I began wondering about apps I've mentioned above (Authy and friends)? My inner researcher was awakened, so I switched my phone into the airplane mode and, to my big surprise, realized that they work perfectly fine offline: they keep generating codes that are accepted by the servers of the apps. Interesting!

Not sure about you, but I've always taken the one-time token flow for granted and have never actually given it a proper thought (especially due to the fact that these days it is rare that my phone has no internet unless I'm doing some outdoor adventures), so that was the root cause of my surprise. Otherwise, it makes a perfect sense from the security point of view to work this way, as the generation process is purely local, so safe-ish from the external actors. But how does it work?

Well, modern technologies like Google or ChatGPT make it easy to find the answer easily. But this technical problem seemed fun to me, so decided to give it a try and solve it on my own first. 

## Requirements

Let's start with what we have:

- an offline device that generates 6-digits codes
- a server that accepts these codes, validates them and gives a green signal if they are correct

The server validation part hints that the server must be able to generate the same code as the offline device to compare them. Hm..that can be helpful.

My further observations of my new "toy" brought even more discoveries:

- if I turn it off and then off, usually I can see the same code as before
- however, occasionally, it changes

The only logic explanation I could come up with is that these codes have a certain lifetime. I'd like to tell a story of me trying to count the duration of it in "1-2-3-...-N" fashion, but it won't be true: I got a big hint from the apps like Authy and Co, where I saw the 30 seconds TTL. Good find, let's add this to the list of the known facts.

Let's summarize the requirements we have so far:

- predictable (rather than random) code generation in 6 digits format
- the generation logic should be reproducible, which allows getting the same result regardless of the platform
- the code lifetime is 30 seconds, which means that within this timeframe the generation algorithm produces the same value

## The big question

All right, but the main question remains unanswered: how come the offline app can generate the value that will match the one from the other app? What do they have in common? 

If you are into the Lord of the Rings universe, you might remember how Bilbo played a riddle game with Gollum, and got this one to solve:

> This thing all things devours:
> Birds, beasts, trees, flowers;
> Gnaws iron, bites steel;
> Grinds hard stones to meal;
> Slays king, ruins town,
> And beats high mountain down.

Spoiler alert, but Mr. Baggins got lucky and came up with the correct answer accidentally - "Time!". Believe it or not, but this is exactly the answer to our riddle as well: any 2 (or more) apps have access to the same time as long as they have an embedded clock within. The latter is not a problem these days, and the device in question is big enough to fit it. Look around, and the chances are that the time on your hand watch, mobile phone, TV, oven, and clock on the wall is the same. We are into something here, it seems like we have found the base for the OTP (one-time password) computing!

### Challenges

Relying on time has its own set of challenges:

- timezones - which one to use?
- clocks tend to get out of sync, and it's [a big challenge](https://en.wikipedia.org/wiki/Clock_synchronization) in the distributed systems

Let's address them one by one:

- timezone: the simplest solution here is to make sure that all the devices rely on the same zone, and UTC can be a good location-agnostic candidate
- as for clocks getting out of sync: actually, we might not even need to solve it but rather accept as something inevitable, as long as the drift is within the second or two, which might be tolerable considering the 30 seconds TTL. The hardware producer of the device should be able to predict when such drift will be achieved, so then the device will use it as its expiration date, and the bank will simply replace them with the new ones, or will have a way to connect them to the network to calibrate the clock. At least, that's my train of thoughts here.

## Implementation

Ok, this is settled, so let's try to implement the very first version of our algorithm using time as the basis. Since we are interested in a 6 digits result, it sounds like a smart choice to rely on a timestamp rather than the human-readable date. Let's start from there:

```go
// gets current timestamp:
current := time.Now().Unix()
fmt.Println("Current timestamp: ", current)
```

According to the Go docs, the `.Unix()` returns

> the number of seconds elapsed since January 1, 1970 UTC.

This is what printed to the terminal:

```plain
Current timestamp:  1733691162
```

That's a good start, but if we rerun that code, the timestamp value will change, while we'd like to keep it stable for 30 seconds. Well, piece of cake, let's divide it by 30 and use that value as a base:

```go
// gets current timestamp
current := time.Now().Unix()
fmt.Println("Current timestamp: ", current)

// gets a number that is stable within 30 seconds
base := current / 30
fmt.Println("Base: ", base)
```

Let's run it:

```plain
Current timestamp:  1733691545
Base:  57789718
```

And again:

```plain
Current timestamp:  1733691552
Base:  57789718
```

The base value remains the same. Let's wait for a bit, and run it again:

```plain
Current timestamp:  1733691571
Base:  57789719
```

The base value has changed, as the 30 seconds window has passed - good job!

If the "divide by 30" logic doesn't make sense, let me explain it with a simple example: 

- imagine that our timestamp returns 1
- if we divide 1 by 30, the result will be 0, as when we are using a strictly typed programming language, dividing integer by integer returns another integer, which isn't concerned about the floating-point part
- this means that for the next 30 seconds we'll be getting `0` while timestamp is between 0 and 29
- once timestamp reaches the value of 30, the result of division is 1, until 60 (where it becomes 2), and so on

I hope it makes more sense now.

However, not all the requirements are satisfied yet, as we need a 6 digits result, while current base has 8 digits as of today, but at some point of time in the future it might reach 9 digits point, and so on. Well, let's use another simple math trick: let divide the base by 1 000 000, and get the remainder, which will always have exactly 6 digits, as the reminder can be any number from 0 to 999 999, but never larger:

```go
// gets current timestamp:
current := time.Now().Unix()
fmt.Println("Current timestamp: ", current)

// gets a number that is stable within 30 seconds:
base := current / 30
fmt.Println("Base: ", base)

// makes sure it has only 6 digits:
code := base % 1_000_000

// adds leading zeros if necessary:
formattedCode := fmt.Sprintf("%06d", code)
fmt.Println("Code: ", formattedCode)
```

The `fmt.Sprintf("%06d", code)` part appends leading zeros in case if our code value has less than 6 digits. For example, `1234` will be converted into `001234`.
The entire code for this post can be found [here](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20241210-otp-offline-generation).

Let's run this code:

```plain
Current timestamp:  1733692423
Base:  57789747
Code:  789747
```

All right, we get our 6-digits code, hooray! But something doesn't feel right here, does it? If I give you this code, and you will run it at the same time as I do, you'll get the same code, as I. This doesn't make it a secure one-time password, right? Here comes a new requirement:

- the result should be different for different users

Of course, some collisions are inevitable, if we have beyond 1 million users, as this is the max possible unique values per 6 digits. But these are rare and technically unavoidable collisions, not the algorithm design flaw like we have now.

I don't think any clever mathematical tricks will help us here on their own: if we need separate results per user, we need a user-specific state to make it happen. As engineers and, at the same time, users of many services, we know that to grant access to their APIs, services rely on private keys, which are unique per user. Let's introduce a private key for our use case as well to distinguish between the users. 

### Private key

A simple logic for generating private keys as integers between 1 000 000 and 999 999 999:

```go
var pkDb = make(map[int]bool)

func main() {
	prForUser := nextPrivateKey()
	fmt.Println("Private key for user: ", prForUser)
}

func nextPrivateKey() int {
	r := randomPrivateKey()
	for pkDb[r] {
		r = randomPrivateKey()
	}
	pkDb[r] = true
	return r
}

// generates random number from 1 000 000 to 999 999 999
func randomPrivateKey() int {
	return rand.Intn(999_999_999-1_000_000) + 1_000_000
}
```

We are using `pkDb` map as a way to prevent duplicates among private keys, and if the duplicate has been detected, we run the generation logic once more until we get a unique result.

Let's run this code to get the private key sample:

```plain
Private key for user:  115537846
```

Let's make a use of this private key within our code generation logic to make sure we are getting different results per private key. Since our private key is of integer type, the simplest thing we can do is to add it to the base value, and keep the remaining algorithm as it is:

```go
func generateCode(pk int64) string {
	// gets current timestamp:
	current := time.Now().Unix()
	fmt.Println("Current timestamp: ", current)

	// gets a number that is stable within 30 seconds:
	base := current / 30 + pk
	fmt.Println("Base: ", base)

	// makes sure it has only 6 digits:
	code := base % 1_000_000

	// adds leading zeros if necessary:
	return fmt.Sprintf("%06d", code)
}
```

Let's make sure it produces different results for different private keys:

```go
var pkForUser1 int64 = 115537846
var pkForUser2 int64 = 715488689

codeForUser1 := generateCode(pkForUser1)
fmt.Println("Code for user 1: ", codeForUser1)

fmt.Println()

codeForUser2 := generateCode(pkForUser2)
fmt.Println("Code for user 2: ", codeForUser2)
```

The outcome looks as we wanted and expected:

```plain
Current timestamp:  1733695429
Base:  173327693
Code for user 1:  327693

Current timestamp:  1733695429
Base:  773278536
Code for user 2:  278536
```

Works like charm! This means that the private key should be injected into the device that generates codes before it is sent to the users like me: that should not be a problem at all for the banks.

Are we done now? Well, only if we are satisfied with the artificial scenario we used. If you have ever enabled the MFA for any services / websites you have account on, you might have noticed that the web resource asks you to scan the QR code with your second-factor app of choice (Authy, Google Authenticator, 2FAS, etc.) that will input the secret code into your app and start generating 6-digits code from that moment on. Alternatively, it is possible to enter the code manually.

I'm bringing this up to mention that it is possible to take a peek at the format of the real private keys used in the industry. They are usually 16-32 characters long Base32-encoded strings that look like this: 

```plain
JBSWY3DPEB2GQZLSMUQQ
```

As you can see, this is quite different from the integer private keys we used, and the current implementation of our algorithm won't work if we are to switch to this format. How can we adjust our logic?

### Private key as a string

Let's start with a simple approach: our code won't compile, because of this line:

```go
base := current/30 + pk
```

as `pk` is of type `string` from now on. So why don't we convert it into integer? While there are way more elegant and performant ways to do that, here is the simplest thing I came up with:

```go
// converts the string pk to a number
func hash(pk string) int64 {
	var num int64 = 0
	for _, char := range pk {
		num = 31*num + int64(char)
	}
	return num
}
```

This is heavily inspired by the Java `hashCode()` implementation for the `String` data type, which makes it good-enough for our scenario.

Here is the adjusted logic:

```go
func main() {
	var pkForUser1 int64 = 115537846
	var pkForUser2 int64 = 715488689

	codeForUser1 := generateCode(pkForUser1)
	fmt.Println("Code for user 1: ", codeForUser1)

	fmt.Println()

	codeForUser2 := generateCode(pkForUser2)
	fmt.Println("Code for user 2: ", codeForUser2)
}

func generateCode(pk int64) string {
	// gets current timestamp:
	current := time.Now().Unix()
	fmt.Println("Current timestamp: ", current)

	// gets a number that is stable within 30 seconds:
	base := current / 30 + pk
	fmt.Println("Base: ", base)

	// makes sure it has only 6 digits:
	code := base % 1_000_000

	// adds leading zeros if necessary:
	return fmt.Sprintf("%06d", code)
}
```

Here is the terminal output:

```plain
Current timestamp:  1733771953
Base:  9056767456302232090
Code:  232090
```

Nice, 6-digits code is there, good job. Let's wait to get to the next time window and run it again:

```plain
Current timestamp:  1733771973
Base:  9056767456302232091
Code:  232091
```

Hm...it works, but the code is, basically the increment of the previous value, which is not good, as this way OTPs are predictable, and having its value and knowing what time it belongs to, it is very simple to start generating the same values without the need of knowing the private key. Here is the simple pseudocode for this hack:

```plain
now = currentTimestamp()
diff = now - oldOtpTimestamp
numOfOtpsIssuedSinceThen = diff / 30
currentOtp = keepWithinSixDigits(oldOtp + numOfOtpsIssuedSinceThen)
```

where `keepWithinSixDigits` will make sure that after the `999 999` the next value is `000 000` and so on to keep the value within the 6-digits limit possibilities.

As you can see, it is a serious security flaw. Why does it happen? If we take a look at the `base` calculation logic, we'll see that it relies on 2 factors:

- current timestamp divided by 30
- hash of the private key

The hash produces the same value for the same key, so its value is constant. As for the `current / 30` , it has the same value for 30 seconds, but once the window has passed, the next value will be the increment of the previous one. Then `base % 1_000_000` behaves the way we see. Our previous implementation (with private keys as integers) had the same vulnerability, but we didn't notice that - lack of testing to blame.

We need to transform `current / 30` into something to make the change of its value more noticeable. 

### Distributed OTP values

There are multiple ways to achieve that, and some cool math tricks exist out there, but for the educational purposes let's prioritize the readability of the solution we'll go with: let's extract `current / 30` into a separate variable `base` and include it into the hash computation logic:

```go
...
base := current / 30
...

// converts base and pk into a number
func hash(base int64, pk string) int64 {
	num := base
	for _, char := range pk {
		num = 31*num + int64(char)
	}
	fmt.Println("Hash: ", num)
	return num
}
```

This way, even though the base will change with 1 every 30 seconds, after being used within the `hash()` function logic, the weight of the diff will increase due to the series of performed multiplications.

Here is the updated code example:

```go
func main() {
	pk := "JBSWY3DPEB2GQZLSMUQQ"

	codeForUser1 := generateCode(pk)
	fmt.Println("Code: ", codeForUser1)
}

func generateCode(pk string) string {
	// gets current timestamp:
	current := time.Now().Unix()
	fmt.Println("Current timestamp: ", current)

	// gets a number that is stable within 30 seconds:
	base := current / 30

	// combines the base with the private key to get a number unique to the user:
	baseWithPk := hash(base, pk)
	fmt.Println("Base: ", baseWithPk)

	// makes sure it has only 6 digits:
	code := baseWithPk % 1_000_000

	// adds leading zeros if necessary:
	return fmt.Sprintf("%06d", code)
}

// converts base and pk into a number
func hash(base int64, pk string) int64 {
	num := base
	for _, char := range pk {
		num = 31*num + int64(char)
	}
	fmt.Println("Hash: ", num)
	return num
}
```

Let's run it:

```plain
Current timestamp:  1733773490
Hash:  -1073062424175310899
Base:  -1073062424175310899
Code:  -310899
```

Boom! How come did we get the minus value here? Well, it seems like we ran out of the `int64` ranges, so it capped the values to the minus and started over - my Java fellows are familiar with this from the `hashCode()` behavior.  The fix is simple: let's take the absolute value from the result, so then the minus sign is ignored:

```go
// combines the base with the private key to get a number unique to the user:
baseWithPk := hash(base, pk)

// makes sure it is positive:
absBaseWithPk := int64(math.Abs(float64(baseWithPk)))
```

Here is entire code sample with the fix:

```go
func main() {
	pk := "JBSWY3DPEB2GQZLSMUQQ"

	codeForUser1 := generateCode(pk)
	fmt.Println("Code: ", codeForUser1)
}

func generateCode(pk string) string {
	// gets current timestamp:
	current := time.Now().Unix()
	fmt.Println("Current timestamp: ", current)

	// gets a number that is stable within 30 seconds:
	base := current / 30

	// combines the base with the private key to get a number unique to the user:
	baseWithPk := hash(base, pk)

	// makes sure it is positive:
	absBaseWithPk := int64(math.Abs(float64(baseWithPk)))
	fmt.Println("Base: ", absBaseWithPk)

	// makes sure it has only 6 digits:
	code := absBaseWithPk % 1_000_000

	// adds leading zeros if necessary:
	return fmt.Sprintf("%06d", code)
}

// converts base and pk into a number
func hash(base int64, pk string) int64 {
	num := base
	for _, char := range pk {
		num = 31*num + int64(char)
	}
	fmt.Println("Hash: ", num)
	return num
}
```

Let's run it:

```plain
Current timestamp:  1733774391
Hash:  3581577654419246315
Base:  3581577654419246080
Code:  246080
```

Let's run it again to make sure that the OTP values are distributed now:

```plain
Current timestamp:  1733774402
Hash:  4351623792829383276
Base:  4351623792829383168
Code:  383168
```

Nice, finally a decent solution!

Actually, that was the moment when I stopped my manual implementation process, as I had my share of fun and learned something new. However, it's neither the best solution nor the one I'd go live with. Among other things, it has a big flaw: as you can see, our logic always deals with big numbers due to the hashing logic and timestamp values, which means that it's highly unlikely for us to be able to generate results that start with 1 or more zeros: e.g., `012345` , `001234`, etc., even though they are completely valid. Due to that, we are 100 000 possible values short, which is 10% from the possible number of outcomes of the algorithm - the chances of collisions are higher this way. Not cool!

### Where to go from here

I won't go deep into the implementation that is used in the real applications, but for those who are curious, I'll share two RFCs worth taking a look at:

- [HOTP: An HMAC-Based One-Time Password Algorithm](https://datatracker.ietf.org/doc/html/rfc4226)
- [TOTP: Time-Based One-Time Password Algorithm](https://datatracker.ietf.org/doc/html/rfc6238)

And here is the pseudocode implementation that will work the intended way based on the RFCs above:

```plain
function generateTOTP(secretKey):
    current = time.now().unix()
    counter = floor(currentTime / 30)
    hmac = HMAC-SHA1(secretKey, counter)
    offset = last_byte(hmac) & 0x0F
    truncatedHash = hmac[offset:offset+4]  // Extract 4 bytes
    binary = truncatedHash & 0x7FFFFFFF    // Ensure positive number
    otp = binary % 1_000_000               // Reduce to desired length
    return otp
```

As you can see, we got very close to that, but the original algorithm uses more advanced hashing (HMAC-SHA1 in this example), and performs some bitwise operations to normalise the output.

## Security considerations: reuse rather than build yourself

However, there is one more thing I'd like to cover before we call it a day: security. I'd strongly encourage you against implementing the logic of generating OTPs on your own, as there are plenty of libraries out there that have it done for us. The room for error is huge, and it's a short distance to the vulnerability that will be discovered and exploited by the bad actors out there.

Even if you get the generation logic right and will cover it with tests, there are other things that can go wrong. For example, how much do you think it will take to brute-force the 6-digit code? Let's experiment:

```go
func main() {
	pk := "JBSWY3DPEB2GQZLSMUQQ"

	code := generateCode(pk)
	fmt.Println("Code: ", code)

	// brute forces the OTP and measures the time it takes to find it in milliseconds:
	start := time.Now()
	for i := 0; i < 1_000_000; i++ {
		if fmt.Sprintf("%06d", i) == strconv.FormatInt(code, 10) {
			fmt.Println("Found: ", i)
			finish := time.Now()
			fmt.Println("Time: ", finish.Sub(start).Milliseconds(), "ms")
			break
		}
	}
}

func generateCode(pk string) int64 {
	// gets current timestamp:
	current := time.Now().Unix()

	// gets a number that is stable within 30 seconds:
	base := current/30 + hash(pk)

	// makes sure it has only 6 digits:
	return base % 1_000_000
}

// converts the string pk to a number
func hash(pk string) int64 {
	var num int64
	for _, c := range pk {
		num += int64(c)
	}
	return num
}
```

Let's run this code:

```plain
Code:  661504
Found:  661504
Time:  75 ms
```

And once more:

```plain
Code:  524928
Found:  524928
Time:  67 ms
```

As you can see, it takes around 70 ms to guess the code via simple brute-forcing for-loop. That's 400+ times faster than the lifetime of the OTP! The server of the app/website using OTP mechanism needs to prevent that by, for example, not accepting new codes for the next 5 or 10 seconds after 3 failed attempts. This way the attacker gets only 18 or 9 attempts correspondingly within the 30 seconds window, which is not enough for the pool of 1 million possible values.

And there are other things like this that are easy to overlook. So, let me repeat: don't build this from scratch, but rely on the existing solutions.

Anyway, I hope you learned something new today, and the OTP logic won't be a mystery for you from this point on. Also, if at some point in life, you'll need to make your offline device generate some values using a reproducible algorithm, you have a good idea where to start.

Thanks for the time you spent reading this post, and have fun! =)