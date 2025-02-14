---
title: "What Okta Bcrypt incident can teach us about designing better APIs"
image: "/covers/drawings/20250122.webp"
draft: false
date: 2025-01-22T18:00:00+01:00
tags: ["api", "security", "opinion", "bcrypt"]
---

Hello there! If you follow tech news, you might have heard about the [Okta security incident](https://trust.okta.com/security-advisories/okta-ad-ldap-delegated-authentication-username/) that was reported on 1st of November. The TLDR of the incident was this: 

> The Bcrypt algorithm was used to generate the cache key where we hash a  combined string of userId + username + password. Under a specific set of conditions, listed below, this could allow users to authenticate by  providing the username with the stored cache key of a previous  successful authentication.

This means that if the user had a username above 52 chars, any password would suffice to log in. Also, if the username is, let's say, 50 chars long, it means that the bad actor needs to guess only 3 first chars to get in, which is quite a trivial task for the computers these days. Too bad, isn't it? 

On the other hand, such long usernames are not very usual, which I agree with. However, some companies like using the entire name of the employee as the email address. So, let's say, Albus Percival Wulfric Brian Dumbledore, a headmaster of Hogwarts, should be concerned, as `albus.percival.wulfric.brian.dumbledore@hogwarts.school` is 55 chars. Ooops!

![image](/images/drawings/20250122-0001.webp)

This was possible due to the nature of Bcrypt hashing algorithm that has a maximum supported input length of 72 characters (read more [here](https://en.wikipedia.org/wiki/Bcrypt#Maximum_password_length)), so in Okta case the characters above the limit were ignored while computing the hash, and therefore, not used in the comparison operation. We can reverse engineer that:

- `72 - 53 = 19` - user id with separators if any
- this way, the password will be outside the 72 chars limit, and, therefore, ignored by the Bcrypt algorithm

However, there was one thing that made me wonder: if there is a known limit of the algorithm, why is it not enforced by the crypto libraries as a form of input validation? A simple `if input length > 72 -> return error` will do the trick. I assumed that they might have used some custom library for Bcrypt implementation and simply forgotten about the input validation, which can happen. So, I decided to check how other programming languages behave.

## Go and Bcrypt

Let's start with Go, and implement the Okta incident-like case with the help of the official `golang.org/x/crypto/bcrypt` library:

```go
package main

import (
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"golang.org/x/crypto/bcrypt"
)

func main() {
	// 18 + 55 + 1 = 74, so above 72 characters' limit of BCrypt
	userId := randomString(18)
	username := randomString(55)
	password := "super-duper-secure-password"

	combinedString := fmt.Sprintf("%s:%s:%s", userId, username, password)

	combinedHash, err := bcrypt.GenerateFromPassword([]byte(combinedString), bcrypt.DefaultCost)
	if err != nil {
		panic(err)
	}

	// let's try to break it
	wrongPassword := "wrong-password"
	wrongCombinedString := fmt.Sprintf("%s:%s:%s", userId, username, wrongPassword)

	err = bcrypt.CompareHashAndPassword(combinedHash, []byte(wrongCombinedString))
	if err != nil {
		fmt.Println("Password is incorrect")
	} else {
		fmt.Println("Password is correct")
	}
}

func randomString(length int) string {
	bytes := make([]byte, length)
	_, err := rand.Read(bytes)
	if err != nil {
		panic(err)
	}
	return base64.URLEncoding.EncodeToString(bytes)[:length]
}
```

All the code samples can be found [here](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20250122-bcrypt-api)

What this code does is:

- generates 18-chars long userId
- generates 55-chars long username
- concatenates them with each other and a dummy password `super-duper-secure-password` with the use of `:` as a separator
- computes Bcrypt hash from the concatenated string
- then concatenates the same userId and username with a different password `wrong-password`
- uses bcrypt API to compare whether the 2nd concatenated string matches the hash of the 1st one

Let's run the code and see the result:

```plain
panic: bcrypt: password length exceeds 72 bytes

goroutine 1 [running]:
main.main()
	/n0rdy-blog-code-samples/20250121-bcrypt-api/01-bcrypt-in-go/main.go:20 +0x2d1
```

Good job, Go! If we check the source code of the `bcrypt.GenerateFromPassword(...)` function, we'll see this piece of code at the very beginning:

```go
if len(password) > 72 {
	return nil, ErrPasswordTooLong
}
```

Perfect! At this point, I became even more suspicious about the tool Okta used, as it seemed like the industry figured that out based on this example. Spoiler alert: it's not that simple. 

Let's proceed with Java.

*Btw, if you like my blog and don’t want to miss out on new posts, consider subscribing to my newsletter [here](https://mail.n0rdy.foo/subscription/form). You'll receive an email once I publish a new post.*

## Java and Bcrypt

Java doesn't support Bcrypt from its core API, but my simple Google search showed that Spring Security library has implemented it. For those who are not into Java ecosystem, Spring is the most used and battle-tested frameworks out there, that has libraries for almost anything: Web, DBs, Cloud, Security, AI, etc. Pretty powerful tool, that I've used a lot in the past, and still sometimes use for my side projects.

### Spring Security

So, I added the latest version of Spring Security to the project and reproduced the same scenario, as in Go example above:

```java
import org.apache.commons.lang3.RandomStringUtils;
import org.springframework.security.crypto.bcrypt.BCrypt;

public class BcriptSpringSecurity {
    public static void main(String[] args) {
        // 18 + 55 + 1 = 74, so above 72 characters' limit of BCrypt
        var userId = RandomStringUtils.randomAlphanumeric(18);
        var username = RandomStringUtils.randomAlphanumeric(55);
        var password = "super-duper-secure-password";

        var combinedString = String.format("%s:%s:%s", userId, username, password);

        var combinedHash = BCrypt.hashpw(combinedString, BCrypt.gensalt());

        // let's try to break it
        var wrongPassword = "wrong-password";
        var wrongCombinedString = String.format("%s:%s:%s", userId, username, wrongPassword);

        if (BCrypt.checkpw(wrongCombinedString, combinedHash)) {
            System.out.println("Password is correct");
        } else {
            System.out.println("Password is incorrect");
        }
    }
}
```

I ran the code, and to my great surprise, saw this outcome:

```plain
Password is correct
```

I took a peak at the implementation code, and was disappointed: even though there are a bunch of checks on salt:

```
if (saltLength < 28) {
	throw new IllegalArgumentException("Invalid salt");
}
...
if (salt.charAt(0) != '$' || salt.charAt(1) != '2') {
	throw new IllegalArgumentException("Invalid salt version");
}
...
minor = salt.charAt(2);
if ((minor != 'a' && minor != 'x' && minor != 'y' && minor != 'b') || salt.charAt(3) != '$') {
	throw new IllegalArgumentException("Invalid salt revision");
}
...

```

I didn't see any validation of the input that will be hashed. Hm...

I decided to check other Google results, and the next Java library in the list was `bcrypt` from Patrick Favre ([link to GitHub repo](https://github.com/patrickfav/bcrypt)) with 513 starts and the last release version 0.10.2 (so, not stable) from 12th of February 2023 (almost 2 years old). This suggested that I'd not use it in production, but why not to run our tests.

### Bcrypt from Patrick Favre

```java
import at.favre.lib.crypto.bcrypt.BCrypt;
import org.apache.commons.lang3.RandomStringUtils;

public class BcryptAtFavre {

    public static void main(String[] args) {
        // 18 + 1 + 55 = 74, so above 72 characters' limit of BCrypt
        var userId = RandomStringUtils.randomAlphanumeric(18);
        var username = RandomStringUtils.randomAlphanumeric(55);
        var password = "super-duper-secure-password";

        var combinedString = String.format("%s:%s:%s", userId, username, password);

        var combinedHash = BCrypt.withDefaults().hashToString(12, combinedString.toCharArray());

        // let's try to break it
        var wrongPassword = "wrong-password";
        var wrongCombinedString = String.format("%s:%s:%s", userId, username, wrongPassword);

        var result = BCrypt.verifyer().verify(combinedHash.toCharArray(), wrongCombinedString);
        if (result.verified) {
            System.out.println("Password is correct");
        } else {
            System.out.println("Password is incorrect");
        }
    }
}
```

Let's run it:

```plain
Exception in thread "main" java.lang.IllegalArgumentException: password must not be longer than 72 bytes plus null terminator encoded in utf-8, was 102
	at at.favre.lib.crypto.bcrypt.LongPasswordStrategy$StrictMaxPasswordLengthStrategy.innerDerive(LongPasswordStrategy.java:50)
	at at.favre.lib.crypto.bcrypt.LongPasswordStrategy$BaseLongPasswordStrategy.derive(LongPasswordStrategy.java:34)
	at at.favre.lib.crypto.bcrypt.BCrypt$Hasher.hashRaw(BCrypt.java:303)
	at at.favre.lib.crypto.bcrypt.BCrypt$Hasher.hash(BCrypt.java:267)
	at at.favre.lib.crypto.bcrypt.BCrypt$Hasher.hash(BCrypt.java:229)
	at at.favre.lib.crypto.bcrypt.BCrypt$Hasher.hashToString(BCrypt.java:205)
	at BcryptAtFavre.main(BcryptAtFavre.java:14)
```

Nice, good job, Patrick, you saved the day for Java!

After checking the source code, I found this piece:

```java
@Override
public byte[] derive(byte[] rawPassword) {
    if (rawPassword.length >= maxLength) {
        return innerDerive(rawPassword);
    }
    return rawPassword;
}
```

and the strict strategy that threw the exception we've seen:

```java
final class StrictMaxPasswordLengthStrategy extends BaseLongPasswordStrategy {
    StrictMaxPasswordLengthStrategy(int maxLength) {
        super(maxLength);
    }

    @Override
    public byte[] innerDerive(byte[] rawPassword) {
        throw new IllegalArgumentException("password must not be longer than " + maxLength + " bytes plus null terminator encoded in utf-8, was " + rawPassword.length);
    }
}
```

We can see that this strict strategy is used as a part of the default configs:

```java
public static Hasher withDefaults() {
    return new Hasher(Version.VERSION_2A, new SecureRandom(), LongPasswordStrategies.strict(Version.VERSION_2A));
}
```

Cool!

Let's switch to JavaScript.

## JavaScript and Bcrypt

Here I used the [bcryptjs](https://www.npmjs.com/package/bcryptjs) which has over 2 million weekly downloads based on the NPM stats.

``` javascript
const bcrypt = require('bcryptjs')

function randomString (length) {
  const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
  let result = ''
  for (let i = length; i > 0; --i) {
    result += chars[Math.floor(Math.random() * chars.length)]
  }
  return result
}

function runTest () {
  // 18 + 55 + 1 = 74, so above 72 characters' limit of BCrypt
  const userId = randomString(18)
  const username = randomString(55)
  const password = 'super-duper-secure-password'

  const combinedString = `${userId}:${username}:${password}`

  const combinedHash = bcrypt.hashSync(combinedString)

  // let's try to break it
  const wrongPassword = 'wrong-password'
  const wrongCombinedString = `${userId}:${username}:${wrongPassword}`

  if (bcrypt.compareSync(wrongCombinedString, combinedHash)) {
    console.log('Password is correct')
  } else {
    console.log('Password is wrong')
  }
}

runTest()
```

The output is:

```plain
Password is correct
```

Not great. The source code reveals that similar to Spring Security, the library validates the salt

```javascript
if (salt.charAt(0) !== '$' || salt.charAt(1) !== '2') {
     err = Error("Invalid salt version: "+salt.substring(0,2));
     if (callback) {
         nextTick(callback.bind(this, err));
         return;
     }
     else
         throw err;
}
...
```

but not the input length.

Let's try if Python can do any better.

## Python and Bcrypt

Using [bcrypt](https://github.com/pyca/bcrypt) library with 1.3k starts and the latest release in November.

```python
import random
import string

import bcrypt

def random_string(length):
    return ''.join(random.choice(string.ascii_letters) for i in range(length))

if __name__ == '__main__':
    # 18 + 55 + 1 = 74, so above 72 characters' limit of BCrypt
    user_id = random_string(18)
    username = random_string(55)
    password = "super-duper-secure-password"

    combined_string = "{0}:{1}:{2}".format(user_id, username, password)

    combined_hash = bcrypt.hashpw(combined_string.encode('utf-8'), bcrypt.gensalt())

    # let's try to break it
    wrong_password = "wrong-password"
    wrong_combined_string = "{0}:{1}:{2}".format(user_id, username, wrong_password)

    if bcrypt.checkpw(wrong_combined_string.encode('utf-8'), combined_hash):
        print("Password is correct")
    else:
        print("Password is incorrect")
```

The result is same as we observed for most of our test subjects:

```plain
Password is correct
```

All right, but what about some newer and more safety-oriented language - let's try Rust.

## Rust and Bcrypt

Here I need to be honest: since I'm not a Rust expert at all, I used a help of a Claude AI to write this code. So, if you see any issues there, please, let me know in the comments section, so I can fix that.

As a library, I used [rust-bcrypt](https://github.com/Keats/rust-bcrypt) based on my AI friend advice.

```rust
use rand::RngCore;
use base64::{Engine as _, engine::general_purpose::URL_SAFE};
use std::error::Error;

fn random_string(length: usize) -> String {
    let mut bytes = vec![0u8; length];
    rand::thread_rng().fill_bytes(&mut bytes);
    URL_SAFE.encode(&bytes)[..length].to_string()
}

fn main() -> Result<(), Box<dyn Error>> {
    // 18 + 55 + 1 = 74, so above 72 characters' limit of BCrypt
    let user_id = random_string(18);
    let username = random_string(55);
    let password = "super-duper-secure-password";

    let combined_string = format!("{}:{}:{}", user_id, username, password);
    let combined_hash = bcrypt::hash(combined_string.as_bytes(), bcrypt::DEFAULT_COST)?;

    // let's try to break it
    let wrong_password = "wrong-password";
    let wrong_combined_string = format!("{}:{}:{}", user_id, username, wrong_password);

    match bcrypt::verify(wrong_combined_string.as_bytes(), &combined_hash) {
        Ok(true) => println!("Password is correct"),
        Ok(false) => println!("Password is incorrect"),
        Err(e) => println!("{}", e),
    }

    Ok(())
}
```

The output is:

```plain
Password is correct
```

I can see the validation of the cost:

```rust
if !(MIN_COST..=MAX_COST).contains(&cost) {
    return Err(BcryptError::CostNotAllowed(cost));
}
```

but not of the input. And here is the place where the explicit truncation of 72 chars happens (the comment is from the library source code):

```rust
// We only consider the first 72 chars; truncate if necessary.
// `bcrypt` below will panic if len > 72
let truncated = if vec.len() > 72 {
    if err_on_truncation {
        return Err(BcryptError::Truncation(vec.len()));
    }
    &vec[..72]
} else {
    &vec
};

let output = bcrypt::bcrypt(cost, salt, truncated);
```

## Why?

That was my first question after seeing that the majority of the tools follow the pattern that leads to the vulnerability. [Wikipedia article about Bcrypt](https://en.wikipedia.org/wiki/Bcrypt) gave a hint:

> Many implementations of bcrypt truncate the password to the first 72 bytes, following the OpenBSD implementation

Interesting! Let's check the OpenBSD implementation of this algorithm, and [here is the link](https://github.com/openbsd/src/blob/master/lib/libc/crypt/bcrypt.c) to it. The first point of interest lies here:

```c
/* strlen() returns a size_t, but the function calls
 * below result in implicit casts to a narrower integer
 * type, so cap key_len at the actual maximum supported
 * length here to avoid integer wraparound */
key_len = strlen(key);
if (key_len > 72)
	 key_len = 72;
key_len++;
```

And from that moment on, `key_len` is used as a limit to iterate over the input string within, for example:

```c
u_int32_t
Blowfish_stream2word(const u_int8_t *data, u_int16_t databytes,
    u_int16_t *current)
{
	u_int8_t i;
	u_int16_t j;
	u_int32_t temp;

	temp = 0x00000000;
	j = *current;

	for (i = 0; i < 4; i++, j++) {
		if (j >= databytes)
			j = 0;
		temp = (temp << 8) | data[j];
	}

	*current = j;
	return temp;
}
```

Where `key_length` is passed as a `databytes` parameter. So this piece of code:

```c
if (j >= databytes)
	j = 0;
```

will make sure that no chars over the limit (72) will end up being processed.

Git blame shows that the `if (key_len > 72)` line is 11 years old

![image](/images/screenshots/20250122-0001.webp)

while the `if (j >= databytes) j = 0;` is 28 years old (what were you busy with in 1997, ah?) 

![image](/images/screenshots/20250122-0002.webp)

So, it's been a while since the API has been reiterated.

## Some thoughts on that

### Disclaimer

Let me start with a short disclaimer: I have a huge respect for people who spend their free time and mental capacity on maintaining open-source projects. That's a large amount of work, that is not paid, and, unfortunately, quite often not appreciated by the users of the tools. That's why they have all the legal and ethical rights to build the project the way they see them. My opinions below are not targeted towards anyone in particular.

My initial goal was to create issues for each of the mentioned library, but I noticed that this behavior has been already reported to each of them:

- ~~https://github.com/spring-projects/spring-security/issues/15725~~ (the previously mentioned issue has been removed for some reasons, so here is an alternative one: https://github.com/spring-projects/spring-security/issues/2354)
- https://github.com/dcodeIO/bcrypt.js/issues/102
- https://github.com/pyca/bcrypt/issues/691
- https://github.com/Keats/rust-bcrypt/issues/87

Check the discussions and their outcomes by following those links.

### Thoughts and lessons

As a guy who spent a few years of my career on building tools and solutions to be used by other software engineers, I understand the frustration: you invested your time and effort into writing a clear documentation and guides, but a certain number of your users don't bother checking it at all, and just use the tool the way they think it should be used. However, that's the reality that I had to accept and started thinking about how can I make my tools handle those use cases. Here are a few principles I came up with in that process.

#### Don't let the people use your API incorrectly

In my opinion, from the API perspective, the approach when the tool silently cuts the part of the input and processes the remaining one only, it is an extremely poor design choice. What makes things worse is the fact that Bcrypt is used in the domain of security and sensitive data, and, as we can see, most of the tools mentioned above, use `password` as the name of the input parameter of the hashing method. **The good design should explicitly reject the invalid input** with the error / exception / any other mechanism the platform uses. So, basically, exactly what Go and Patrick's Java library did. This way, incidents like Okta one would be impossible by design (btw, I'm not shifting the blame away from Okta, considering the domain they operate in).

It is ok, though, to offer the non-default unsafe option, that will let the users pass longer input that will be truncated if the user explicitly asks for that. A prefix/suffix like `unsafe`, `truncated`, etc. can be a good addition to the names of the method that expose these options.

#### Be predictable

If we take a step back from the Bcrypt case, imagine other examples, if such a pattern becomes common in the industry:

- We created a new user account on HBO to watch a new season of Rick and Morty, and there is a warning that the max size of the password should not exceed 18 chars. However, the password generator of your password manager tool uses 25 chars as a default length of the produced password. So, the password manager inserts that password while creating an account, but the server cuts the last 7 chars, hashes the rest, and saves the hash to the DB. How easy would it be for us to be able to log in to HBO next time and watch a new episode?
- The tech lead of the new project configured a linter tool, and set the max line length as 100 chars. While performing a check, linter removes the chars above the defined limit, and informs that the check has passed. How useful would it be?

A good API design should remember that when it comes to tech, nobody likes surprises.

#### No ego

While following a few online discussions about the Bcrypt Okta incident, I noticed something else: while the majority of comments agreed that we should design APIs like these better, there were a few folks that took a very defensive stance and exposed their ego: "Read a paper before using anything!", "APIs are only correcting the input after the stupid users!", etc. Based on my experience, ego is a big enemy of engineering. And I wouldn't be surprised if you have a story or two in that regard as well. So, yeah, let's not bring our egos to our APIs.

#### Be helpful

Don't get me wrong, I do understand the gist that the users should have some basic knowledge before using any tool. But let's get back to the reality: how many different tools, programming languages, databases, protocols, frameworks, libraries, algorithms, data structures, clouds, AI models, etc. does a software engineer use per week these days? I tried to count for my use case, but stopped after the number had reached 30. Is it possible to know all of them deep? To know all the edge cases and limits? For some of them and to some degree is a reasonable ask, as well as having an expertise in 1 or 2, but definitely not all. The hard truth is that on average, the industry today requires the wide spectrum of knowledge over the deep one (check any job opening to verify that claim). Therefore, while designing the tools, why not to help our fellow colleagues? For example, if our tool accepts only positive numbers, let's add `if num < 1 -> return error`  to our solution, and make the life simpler for somebody out there. 

Especially, if the tool might be used in the security-sensitive context, where humans are usually the weak point in the thread modelling. The good API can help there.

#### Be brave

It's not so often that the API we design is something completely new to the world. Most likely, there are other solutions like ours out there. And the chances are that they've been already doing certain things the particular way. However, that doesn't mean that we need to follow the same path. Kudos to the Go team and Patrick's Java library for being brave to do things the different way as the industry does in the Bcrypt example. Let's learn from them.

#### Reiterate

Regardless of the original design choices and intentions, it's never too late to reiterate on some of them if we see a need or have discovered new information. That's, actually, a place where a lot of us fail due to different reasons, with some of them listed above.

## Instead of a conclusion

The Okta incident exposed large security issues out there. Our test showed, even 3 months after the incident, the industry is still vulnerable to the same outcome, so the chances are that more to come. However, we, as software engineers, can learn from that, and apply these lessons while designing APIs to make them predictable and easier to use.

I hope that was useful, and triggered some thoughts. Thanks a lot for reading my post, and see you in the following ones, there are plenty of topics to discuss. Have fun! =)