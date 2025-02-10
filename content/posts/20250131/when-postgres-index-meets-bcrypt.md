---
title: "When Postgres index meets Bcrypt"
image: "/covers/pics/20250131.webp"
draft: false
date: 2025-01-31T08:30:00+02:00
tags: ["postgres", "security", "bcrypt", "performance", "optimization", "debug"]
---
Hello there! In the [previous post “What Okta Bcrypt incident can teach us about designing better APIs”](https://n0rdy.foo/posts/20250121/okta-bcrypt-lessons-for-better-apis/), we discussed the 72-chars limit of the input value of the Bcrypt hashing algorithm that caused quite a big security incident in the industry. That reminded me about another example of Bcrypt misuse that I, personally, came across a few years ago while investigating a quite nasty performance issue with one of the services. Let's jump right into it!

A product manager of one of the neighboring teams approached me and asked if I could help them with the performance degradation that had been experienced lately with their newest feature, when the users were prompted to enter their SSN (social security number), and they'd get a dashboard with the personalized data about them. The oversimplified architecture looked like this:

![image](/images/drawings/20250131-0001.webp)

As you can see, the flow consists of 3 steps:

1. The user enters their SSN in the UI, and the UI sends it to the Internal service (actually, SSN was fetched securely via the national electronic identification system, but that's not important for the story)
2. Internal service checks whether the data for the user with such SSN is already present in the DB (Postgres): if yes, that data is returned to the UI
3. Otherwise, the request is made to the Third-party API to fetch the desired data, then it's being processed, saved into the DB, and returned to the user

The nature of the user data didn't require real-time updates, so the same data was stored and used for the month, after which it was considered to be expired, and was deleted by the cronjob. A pretty straightforward system.

The issue (the team was experiencing) was the fact that it took 10+ seconds to follow the 3-steps flow we discussed above, which led to a terrible UX and a high bounce rate. That made a good sense, as the average attention span of the users nowadays is way less than 10 seconds. But why was it so slow?

The bug issue was reproducible in the production setup, the logs/metrics were not so useful with the clues for the cause. So, I cloned the project code to my laptop and launched a Postgres instance via Docker Compose. Additionally, I started [mitmproxy](https://mitmproxy.org/) to be able to intercept and inspect HTTP requests on my machine, and created a template of the request to the Internal service API with my own SSN in Postman. My debugging setup was ready, so it was time to run and test the app.

## Running it locally

To my great surprise, after sending a request from Postman, I got a response in the matter of tens of milliseconds. mitmproxy showed that there was indeed the request to the Third-party API, but it had a minimal latency.

![image](/images/screenshots/20250131-0001.webp)

Hm…interesting. I tried repeating the same request again, and again got a pretty quick answer, but this time without any calls to the third-party API. That made a perfect sense, as my user info was present in Postgres. I tried sending the same requests a few more times, but didn't have any luck with reproducing the issue.

Despite the failure, that gave me a good clue: it seemed like I could exclude the Third-party API from the list of suspects, or at least for now. Of course, it's important to mention that the local setup is different from the production one, so there is always a chance that something might be slowing down the egress traffic. However, for simplicity of the investigation, I decided to make sure that nothing in the app caused the issue before checking possible external factors, like cloud proxies, security filters, etc.

I realized that it's time to take a look at the code. If you'd like to follow along, [here](https://github.com/n0rdy/n0rdy-blog-code-samples/tree/main/20250128-postgres-seq-scan-despite-indexing) are the code samples that reproduce the setup we are discussing, check the `README.md` file for how to run instructions.

*Btw, if you like my blog and don’t want to miss out on new posts, consider subscribing to my newsletter [here](https://mail.n0rdy.foo/subscription/form). You'll receive an email once I publish a new post.*

## Browsing the code

You might know that there is a saying “it's always DNS”, which applies when some service is down due to no obvious reasons. From my experience, when there is a performance issue with a relatively simple app, “it's often database”. So, even though, I started navigating the source code from the top to bottom (API -> DB), I didn't expect to see anything suspicious within the layers above the one that runs DB queries. And, indeed, the interesting bits of code were 2 functions of the `Repo` struct responsible for inserting a new user into the DB and selecting a particular user from the DB by SSN. Here how they looked:

```go
func (r *Repo) SelectUserBySsn(ssn string) (*common.User, error) {
	now := time.Now()

	var user common.User
	err := r.dbPool.QueryRow(
		context.Background(),
		"SELECT id, user_info FROM users WHERE ssn_hash = crypt($1, ssn_hash)",
		ssn,
	).Scan(&user.Id, &user.UserInfo)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, nil
		}
		fmt.Println(err)
		return nil, err
	}

	fmt.Println("Time taken to get user:", time.Since(now))
	return &user, nil
}

func (r *Repo) InsertUser(user common.NewUserEntity) error {
	_, err := r.dbPool.Exec(
		context.Background(),
		"INSERT INTO users (id, ssn_hash, user_info) VALUES ($1, crypt($2, gen_salt('bf')), $3)",
		user.Id, user.Ssn, user.UserInfo,
	)
	if err != nil {
		fmt.Println(err)
		return err
	}
	return nil
}
```

The `crypt($1, ssn_hash)` and `crypt($2, gen_salt('bf')` parts immediately caught my eyes: “aha, so we are not storing plain SSNs in the DB, but hashed versions”. Interesting discovery! This didn't give me any hints on the exact algorithm that was used for hashing, but a quick Google search by the `gen_salt('bf')` part helped me to discover that it's a Bcrypt one. And indeed, after I did `SELECT * FROM users` in my local setup, I noticed that the `ssn_hash` column had this value `$2a$06$r4UXr0F80tz57DGzCQBf6uIpQealvM3Rb6E/TvsyFkJJaiLXt9J8W`, and the `$2a$` prefix was indeed a Bcrypt one. All right, we are into something! The chances are that some of you, who are experts in cryptography and/or databases (or, Postgres, at least), already can see the reason for the poor performance of the app (kudos to you!), but that was not the case for me, so I had to keep digging.

I realized that DB was my best lead at the moment, so I jumped away from the source code to the Postgres console in my IntelliJ IDEA, and ran this query:  `SELECT * FROM users WHERE ssn_hash = crypt('0852285111', ssn_hash)` (`0852285111` is my fake SSN in this example). I got the result in 9 milliseconds, which was quite fast, and didn't hint any issues with the performance.

But then I connected to the production DB and ran the same query. It took 15 seconds to get the similar response. Wow! What's more interesting, was the fact that fetching all the existing users via `SELECT * FROM users` took only 6 milliseconds. Puzzling, isn't it?

Despite being a bit puzzled, that definitely felt like good progress, as I had a strong clue that DB was the bottleneck here. And it became clear that my failures to reproduce that locally was due to the fact that my local table had only 1 row, while the production one had 5000 users as of that moment, and kept growing hour by hour. I could have kept debugging the issue within the production Postgres instance, as it was reproducible there, but doing that is not a good practice due to many reasons, like degrading performance even more, or running some unsafe queries that might do some harm to the data there. So, at first, I wanted to take a snapshot of the production DB and use it locally, but realized that it was not much better idea as well: taking a snapshot might be a costly operation for Postgres. Plus, there were privacy implications of me copying the sensitive data of the customers to my local machine. That's why I went with the third option of inserting a lot of fake data into my local DB.

## Fake it until you make it

The fact that our table was so simple and consisted of 3 columns only, made the task very simple. So, I ended up writing this fake data generation code:

```go
func GenRandomUsers(num int) []common.NewUserEntity {
	users := make([]common.NewUserEntity, num)
	for i := 0; i < num; i++ {
		users[i] = GenRandomUser()
	}
	return users
}

func GenRandomUser() common.NewUserEntity {
	ssn := genRandomSsn()
	fmt.Println("Generated SSN:", ssn)
	return common.NewUserEntity{
		Id:       uuid.New().String(),
		Ssn:      ssn,
		UserInfo: GenRandomUserInfo(),
	}
}

func GenRandomUserInfo() string {
	return fmt.Sprintf("Some info %s %s", strconv.FormatInt(time.Now().UnixMilli(), 10), uuid.New().String())
}

func genRandomSsn() string {
	ssn := ""
	for i := 0; i < 10; i++ {
		ssn += strconv.Itoa(rand.IntN(10))
	}
	return ssn
}
```

And made a use of it within the `fillDb` function that I triggered on the app start up:

```go
func fillDb(repo *db.Repo) {
	users, err := repo.SelectUsers()
	if err != nil {
		panic(err)
	}
	if len(users) > 0 {
		fmt.Println("users already exist, skipping db fill")
		return
	}

	fmt.Println("generating random users...")
	newUsers := utils.GenRandomUsers(5000)

	fmt.Println("inserting random users...")
	err = repo.InsertUsers(newUsers)
	if err != nil {
		panic(err)
	}
	fmt.Println("inserted random users")
}
```

Once I ran the app, it took more than 15 seconds to finish inserting the users, which I found weird, as the `InsertUsers` code made only 1 call to DB:

```go
func (r *Repo) InsertUsers(users []common.NewUserEntity) error {
	now := time.Now()

	query := "INSERT INTO users (id, ssn_hash, user_info) VALUES "
	var args []interface{}
	for i, user := range users {
		query += fmt.Sprintf("($%d, crypt($%d, gen_salt('bf')), $%d),", i*3+1, i*3+2, i*3+3)
		args = append(args, user.Id, user.Ssn, user.UserInfo)
	}
	query = query[:len(query)-1] // remove the trailing comma

	_, err := r.dbPool.Exec(
		context.Background(),
		query,
		args...,
	)
	if err != nil {
		fmt.Println(err)
		return err
	}
	fmt.Println("Time taken to insert users:", time.Since(now))
	return nil
}
```

It became clearer and clearer to me that the `crypt($%d, gen_salt('bf'))` part was to blame, but I didn't understand why it was so just yet. Anyway, the intermediate goal was achieved, as at that point of time, I could reproduce the performance issue in my local setup as well, the `SELECT * FROM users WHERE ssn_hash = crypt('0852285111', ssn_hash)` query became as slow as within the production DB, even though my SSN wasn't among the existing DB records.

I decided to take a pause for a moment and check the facts:

- the reasons of the slow performance were queries to the DB
- the performance degraded with the growth of the number of the records within the DB table
- something is fishy with the `crypt()` function for both INSERT and SELECT queries

The second fact was a good hint, so I decided to check what Postgres query planner could tell me about the plan for executing the query:

```sql
EXPLAIN
SELECT *
FROM users
WHERE ssn_hash = crypt('0852285111', ssn_hash);
```

The keyword `EXPLAIN` in this case triggers Postgres to build the plan for the query, but not run it. The results were like this:

```plain
Seq Scan on users  (cost=0.00..182.00 rows=25 width=138)
"  Filter: (ssn_hash = crypt('0852285111'::text, ssn_hash))"
```

Aha, `Seq Scan on users` gives us a hint that Postgres is about to perform a sequential scan, which means that it will go through every single row of the table to find the matching one - `O(n)` if we'd like to use the algorithmic complexity here. This could very much explain the correlation between the increase in the number of rows and performance degradation.

Let's add `ANALYSE` keyword to actually execute the query and get more metrics:

```sql
EXPLAIN ANALYSE
SELECT *
FROM users
WHERE ssn_hash = crypt('0852285111', ssn_hash);
```

The output:

```plain
Seq Scan on users  (cost=0.00..182.00 rows=25 width=138) (actual time=15037.772..15037.774 rows=1 loops=1)
"  Filter: (ssn_hash = crypt('0852285111'::text, ssn_hash))"
  Rows Removed by Filter: 5000
Planning Time: 0.048 ms
Execution Time: 15037.788 ms
```

Which proves what we had just seen above.

So, I thought that I solved the riddle: the `ssh_hash` column lacked the index. I was almost proud of myself, but that was a very brief moment, as after browsing both the DB table structure and the schema definition queries, I realized that there is actually an index there:

```sql
CREATE INDEX users_ssn_hashed_idx ON users (ssn_hash);
```

But why did Postgres keep ignoring it and performing sequential scans nevertheless? And despite that fact, why it took Postgres 15 seconds to scan 5000 rows? And why select all the rows were very quick (5 ms)? The only difference between the two was the use of the `crypto()` function within the `WHERE` statement. That narrowed down the problem. So, I realized that it was the time to get back to basics and do some research on the nature of the Bcrypt algorithm.

![image](/images/drawings/20250131-0002.webp)

## Bcrypt

Here is what [Wikipedia](https://en.wikipedia.org/wiki/Bcrypt) specifies about Bcrypt:

> The input to the bcrypt function is the password string (up to 72  bytes), a numeric cost, and a 16-byte (128-bit) salt value.  The salt is typically a random value.  The bcrypt function uses these inputs to  compute a 24-byte (192-bit) hash.  The final output of the bcrypt  function is a string of the form:
>
> ```plain
> $2<a/b/x/y>$[cost]$[22 character salt][31 character hash]
> ```
>
> For example, with input password `abc123xyz`, cost `12`, and a random salt, the output of bcrypt is the string
>
> ```plain
> $2a$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
> \__/\/ \____________________/\_____________________________/
> Alg Cost      Salt                        Hash
> ```

If we distill it a bit, we can see that the input of the Bcrypt function is:

- password
- numeric cost
- salt (typically random)

If we take a look at the crypt function within the insert user statement, we'll see this `crypt($2, gen_salt('bf')`, where SSN is passed as the `$2`. So, we do have a password (SSN) and salt (`gen_salt('bf'))` The numeric cost is missing, but let's take a look at the output of the `gen_salt('bf')` function. You might need to install the `pgcrypt` extension by `CREATE EXTENSION IF NOT EXISTS pgcrypto;` if you don't have it to be able to follow along.

```sql
SELECT gen_salt('bf');
```

The output is `$2a$06$Qkp8j0vDoCuqNCgyOuIRxO`. If we compare with the example from the Wikipedia above, we can see that this salt includes the cost within (`06`).

Let's run the `SELECT gen_salt('bf');` again. We got `$2a$06$enmGwTijs9cvlsIv9dcQQe`, which is different from the previous result. As Wikipedia warned us

> The salt is typically a random value.

And what happens if we are passing random salt to the crypto function? Right, the result will be random as well. Let's double-check:

```sql
SELECT crypt('0852285111', gen_salt('bf'));
```

The first time I ran it, I got this outcome: `$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce`. The second ran got a different result (as we anticipated): `$2a$06$nbwOka/0ve5RvM71f35jEOMGA5qHaa0H1I3RtkYq/sNIXW.Z90KAm`. If we keep going, we'll keep getting different results. And this is cool, as Bcrypt was created to be used for passwords hashing, which means that even if different users have the same password, the hashed version for them will be different - pretty neat!

But if results are random, how can we check the hash in the DB corresponds to the plain text record we get as the user input? Well, as you can guess, we need to pass the same salt as the input. But where do we get the salt? Wikipedia has already given us an answer: the salt is the part of the hashed output. Let's try it out by copying the first 29 chars of the output we got above: `$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce` -> `$2a$06$DQqtgCtvY9GkT3uHpShehu` and check if that helps:

```sql
SELECT '$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce' = crypt('0852285111', '$2a$06$DQqtgCtvY9GkT3uHpShehu');
```

The result is `true`. Perfect! But wait a minute: do we need to extract the salt and friends manually each time? Let's take a look at the query we used to select the user by SSN from DB to look for some hints:

```sql
SELECT id, user_info FROM users WHERE ssn_hash = crypt($1, ssn_hash)
```

where `$1` is the SSN value. This is a bit different as instead of only salt, we are passing the entire hash value into the crypt function this time. Let's try it out:

```sql
SELECT '$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce' = crypt('0852285111', '$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce');
```

Still `true`. How come? The answer is simple: since salt (with the prefix) has a defined length of 29, Postgres retrieves it for us, and drops the rest of it.

The beauty of Postgres (which is one of my favorite tools out there) is the fact that it's open-source. So, let's take a peak and its source code to make sure that our assumptions are correct.

### Bcrypt in Postgres source code

Here is [the mirror](https://github.com/postgres/postgres) of the official PostgreSQL GIT repository. Let's find the `pgcrypto` file which corresponds to the extension we installed to be able to run crypto functions. [Here](https://github.com/postgres/postgres/blob/master/contrib/pgcrypto/pgcrypto.c) is it is. This piece of code is triggered when we do `crypt(...)` via SQL:

```c
/* SQL function: pg_crypt(psw:text, salt:text) returns text */
PG_FUNCTION_INFO_V1(pg_crypt);

Datum
pg_crypt(PG_FUNCTION_ARGS)
{
	text	   *arg0 = PG_GETARG_TEXT_PP(0);
	text	   *arg1 = PG_GETARG_TEXT_PP(1);
	char	   *buf0,
			   *buf1,
			   *cres,
			   *resbuf;
	text	   *res;

	buf0 = text_to_cstring(arg0);
	buf1 = text_to_cstring(arg1);

	resbuf = palloc0(PX_MAX_CRYPT);

	cres = px_crypt(buf0, buf1, resbuf, PX_MAX_CRYPT);

	pfree(buf0);
	pfree(buf1);

	if (cres == NULL)
		ereport(ERROR,
				(errcode(ERRCODE_EXTERNAL_ROUTINE_INVOCATION_EXCEPTION),
				 errmsg("crypt(3) returned NULL")));

	res = cstring_to_text(cres);

	pfree(resbuf);

	PG_FREE_IF_COPY(arg0, 0);
	PG_FREE_IF_COPY(arg1, 1);

	PG_RETURN_TEXT_P(res);
}
```

If we jump into `pg_crypt` in this line `cres = px_crypt(buf0, buf1, resbuf, PX_MAX_CRYPT);`, we'll end up inside the `px-crypt.c` file [here](https://github.com/postgres/postgres/blob/master/contrib/pgcrypto/px-crypt.c#L90):

```c
char *
px_crypt(const char *psw, const char *salt, char *buf, unsigned len)
{
	const struct px_crypt_algo *c;

	CheckBuiltinCryptoMode();

	for (c = px_crypt_list; c->id; c++)
	{
		if (!c->id_len)
			break;
		if (strncmp(salt, c->id, c->id_len) == 0)
			break;
	}

	if (c->crypt == NULL)
		return NULL;

	return c->crypt(psw, salt, buf, len);
}
```

As we see, this code navigates through the `px_crypt_list` while trying to find a match for the provided `salt` (in our case, it is the entire hashed string) by comparing the `id_len` number of first characters with something called `id`. [Here is the doc](https://en.cppreference.com/w/c/string/byte/strncmp) for the `strncmp` function if you would like to learn more.

Let's see how the `px_crypt_list` looks like, as it might explain to us the nature of the `id_len` and `id` params.

```c
struct px_crypt_algo
{
	char	   *id;
	unsigned	id_len;
	char	   *(*crypt) (const char *psw, const char *salt,
						  char *buf, unsigned len);
};

static const struct px_crypt_algo
			px_crypt_list[] = {
	{"$2a$", 4, run_crypt_bf},
	{"$2x$", 4, run_crypt_bf},
	{"$2$", 3, NULL},			/* N/A */
	{"$1$", 3, run_crypt_md5},
	{"_", 1, run_crypt_des},
	{"", 0, run_crypt_des},
	{NULL, 0, NULL}
};
```

Aha, so the `id` is the salt prefix that Wikipedia called `alg`  (`$2a$` in our case), and the `id_len` is the length of that prefix. That makes a lot of sense now. And the `run_crypt_bf` is the actual function that will be invoked in our case. Nice, let's take a look at it [here](https://github.com/postgres/postgres/blob/master/contrib/pgcrypto/px-crypt.c#L61):

```c
static char *
run_crypt_bf(const char *psw, const char *salt,
			 char *buf, unsigned len)
{
	char	   *res;

	res = _crypt_blowfish_rn(psw, salt, buf, len);
	return res;
}
```

And let's follow along by jumping into the `_crypt_blowfish_rn` which leads us to [a new file](https://github.com/postgres/postgres/blob/master/contrib/pgcrypto/crypt-blowfish.c#L582) - `crypt-blowfish.c`. I will not post the entire body of that function here, as it is around 200 lines long. However, this is an interesting part for us:

```c
/*
 * Blowfish salt value must be formatted as follows: "$2a$" or "$2x$", a
 * two digit cost parameter, "$", and 22 digits from the alphabet
 * "./0-9A-Za-z". -- from the PHP crypt docs. Apparently we enforce a few
 * more restrictions on the count in the salt as well.
 */
if (strlen(setting) < 29)
	ereport(ERROR,
		(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
		errmsg("invalid salt")));
```

where `settings` is the `salt`. We can see a number 29 here, which rings a bell, doesn't it? That's precisely the length of the salt prefix we used in one of the examples above. Also, please, note that this validation only checks if the input length is not less than 29, but it doesn't care whether it's above the limit. That gives us a hint that the code below should handle it somehow:

```c
if (setting[0] != '$' ||
		setting[1] != '2' ||
		(setting[2] != 'a' && setting[2] != 'x') ||
		setting[3] != '$' ||
		setting[4] < '0' || setting[4] > '3' ||
		setting[5] < '0' || setting[5] > '9' ||
		(setting[4] == '3' && setting[5] > '1') ||
		setting[6] != '$')
	{
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
				 errmsg("invalid salt")));
	}

	count = (BF_word) 1 << ((setting[4] - '0') * 10 + (setting[5] - '0'));
	if (count < 16 || BF_decode(data.binary.salt, &setting[7], 16))
	{
		px_memset(data.binary.salt, 0, sizeof(data.binary.salt));
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
				 errmsg("invalid salt")));
	}
```

Take a look at this piece: `BF_decode(data.binary.salt, &setting[7], 16)`. Do you remember the example from Wikipedia:

> ```plain
> $2a$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
> \__/\/ \____________________/\_____________________________/
> Alg Cost      Salt                        Hash
> ```

?

As we can see, the actual salt begins at the 8th char (that's why `setting[7]`), and the `&setting[7]` means that all the elements starting from 8th are sliced here. The `BF_decode` function looks like this:

```c
static int
BF_decode(BF_word *dst, const char *src, int size)
{
	unsigned char *dptr = (unsigned char *) dst;
	unsigned char *end = dptr + size;
	const unsigned char *sptr = (const unsigned char *) src;
	unsigned int tmp,
				c1,
				c2,
				c3,
				c4;

	do
	{
		BF_safe_atoi64(c1, *sptr++);
		BF_safe_atoi64(c2, *sptr++);
		*dptr++ = (c1 << 2) | ((c2 & 0x30) >> 4);
		if (dptr >= end)
			break;

		BF_safe_atoi64(c3, *sptr++);
		*dptr++ = ((c2 & 0x0F) << 4) | ((c3 & 0x3C) >> 2);
		if (dptr >= end)
			break;

		BF_safe_atoi64(c4, *sptr++);
		*dptr++ = ((c3 & 0x03) << 6) | c4;
	} while (dptr < end);

	return 0;
}
```

While there is a lot of bit manipulation logic here, which I won't dare to explain, but what's important here is that `dptr < end` will make sure that the result will be 16 bytes long. But the salt should be 22?

In Bcrypt, while the salt is indeed encoded as 22 characters in base64 format, it actually decodes to 16 bytes of raw data. When encoding binary data to base64:

- every 3 bytes of binary data becomes 4 characters in base64
- this is because each Base64 character represents 6 bits (2^6 = 64 possibilities)
- so 3 bytes (24 bits) of binary data maps to 4 base64 chars (4 * 6 = 24 bits)

For the Bcrypt salt:

- 16 bytes of raw data = 128 bits
- when encoded in base64: ⌈(16 * 8) / 6⌉ = ⌈128/6⌉ = 22 characters
- the ceiling function is used because we need to round up to the next complete base64 character

This way, we can see that it is safe to pass the entire hashed string as the salt input of the Postgres `crypt()` function. All right, that was fun, but let's get back to our Bcrypt discussion.

### What do we know now?

Before we summarize what we know as of now, let me bring you one more quote about Bcrypt from the Wikipedia article:

> There are then a number of rounds in which the standard Blowfish keying  algorithm is applied, using alternatively the salt and the password as  the key, each round starting with the subkey state from the previous  round.  In theory, this is no stronger than the standard Blowfish key  schedule, but the number of rekeying rounds is configurable; this  **process can therefore be made arbitrarily slow**, which helps deter  brute-force attacks upon the hash or salt.

So, the Bcrypt algorithm is relatively slow by design, as the protection from the brute-force attacks.

Let's create a list of the known facts about the Bcrypt algorithm:

- requires a salt to generate the hash
- ideally, the salt should be random
- random salt leads to the different results for the same input to hash
- to compare the existing hashed string with the plain one, the original salt should be passed as the input of the crypto function
- Postgres allows passing the hashed string as a salt, and takes care of extracting the salt part of it to perform hashing

With all of this information, let's finally try to solve the riddle of the poor performance of the query to fetch the user by SSN.

## Postgres index and Bcrypt

So, why does Postgres keep performing sequential scans despite the index for that column? Let's take another look at the query we are trying to perform:

```sql
SELECT *
FROM users
WHERE ssn_hash = crypt('0852285111', ssn_hash);
```

As we already know, to recompute the hash, the `crypto()` function has to receive the original salt as the input. But, as you can see, the only known thing when we run this query is the plain SSN value. This means that the only possible option that Postgres has is to do the following:

- iterate over each existing row in the `users` table
- fetch `ssn_hash` value
- pass the plain SSN and fetched `ssn_hash` value (as a salt) to the `crypt()` function and compute the hash
- compare the result with the fetched `ssn_hash` value
- if matches, we found what we are looking for, otherwise, we need to proceed to the next item, and so on

Since the algorithm described need to iterate over all the existing items, it makes sense while Postgres doesn't rely on indexes at all. If we modify the query to compare the `ssn_hash` with the plain string instead of the `crypt()` function like this:

```sql
EXPLAIN ANALYSE
SELECT *
FROM users
WHERE ssn_hash = '$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce';
```

we can see that this time Postgres will use the index:

```plain
Index Scan using users_ssn_hashed_idx on users  (cost=0.28..8.30 rows=1 width=138) (actual time=0.015..0.015 rows=0 loops=1)
  Index Cond: (ssn_hash = '$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce'::text)
Planning Time: 0.448 ms
Execution Time: 0.033 ms
```

Also, remember that we mentioned that Bcrypt algorithm is relatively slow? But what if we multiply slow by 5000 (number of table records we have)? That's the actual cause of the performance degradation that we have been investigating all this time!

We can simply verify it by running:

```sql
EXPLAIN ANALYSE
SELECT *
FROM users;
```

The output:

```plain
Seq Scan on users  (cost=0.00..157.00 rows=5000 width=138) (actual time=0.009..0.441 rows=5001 loops=1)
Planning Time: 0.094 ms
Execution Time: 0.641 ms
```

As you can see, even though the query iterated over all the rows, it didn't need to perform Bcrypt hashing at all, the execution time is extremely low.

Ok, so we know now that Bcrypt is the one to blame for the poor performance. But does it mean that Bcrypt is just a bad algorithm performance-wise?

## Bcrypt: where to use and where not to use

Similar to any other tool, Bcrypt has its use case: hashing passwords. This means that the expected use case for it is different from the one we have above, as we shouldn't search by it. Let me demonstrate the scenario where Bcrypt can be used:

```sql
CREATE TABLE user_credentials
(
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    login         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL
);

CREATE INDEX user_credentials_login_idx ON user_credentials (login);
```

And let's insert a few items:

```go
func main() {
	dbUrl := "postgres://adminU:adminP@localhost:5432/n0rdyblog"
	dbPool, err := pgxpool.New(context.Background(), dbUrl)
	if err != nil {
		panic(err)
	}
	defer dbPool.Close()

	numOfUsers := 5000

	for i := 0; i < numOfUsers; i++ {
		login := randomString(10)
		password := randomString(20)

		dbPool.Exec(context.Background(), "INSERT INTO user_credentials(login, password_hash) VALUES($1, crypt($2, gen_salt('bf')))", login, password)
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

The way we are supposed to run queries now is this:

- when the user tries to log in, they are passing us their credentials: login and password
- here is the query we run to verify the provided credentials are correct:

```sql
SELECT *
FROM user_credentials
WHERE login = 'ChCpOSGOCx'
  AND password_hash = crypt('ygEUnWIFIBoJ7ZQQqQ3j', password_hash);
```

Even though the `user_credentials` table has 5000 items, this query is very fast, let's see how fast:

```sql
EXPLAIN ANALYSE
SELECT *
FROM user_credentials
WHERE login = 'ChCpOSGOCx'
  AND password_hash = crypt('ygEUnWIFIBoJ7ZQQqQ3j', password_hash);
```

The results:

```plain
Index Scan using user_credentials_login_idx on user_credentials  (cost=0.28..8.31 rows=1 width=88) (actual time=4.502..4.513 rows=1 loops=1)
  Index Cond: (login = 'ChCpOSGOCx'::text)
"  Filter: (password_hash = crypt('ygEUnWIFIBoJ7ZQQqQ3j'::text, password_hash))"
Planning Time: 0.067 ms
Execution Time: 4.542 ms
```

The key difference between this scenario and the one that we had with SSNs, is that here Postgres is able to use indexing due to the `WHERE login = 'ChCpOSGOCx'` part, and then it computes Bcrypt hash only for the row returned by the index. Since, our `login` column is unique, it means that Bcrypt computation will happen only 1 time, so `O(1)` instead of `O(n)`.

That's how Bcrypt is supposed to be used to benefit from its security without paying too high-performance costs.

Nice, but how can we resolve the original problem then?

## Let's fix the performance

Of course, there is an obvious technical fix by switching to another faster algorithm with the predictable outcome like SHA-256, and the performance issue will be solved. But there are 2 points worth discussing:

- since the input of the SHA-256 (or any other deterministic algorithm) will be SSN, which has a known format of 10 digits, it means that the hash will be easy to crack by the brute-forcing attack, as 10^10 (10 billion) possible options of the SSN are something that is easy to generate for the modern computers

Here is the simple brute-force code to generate all the possible SSN hash values:

```go
const (
	ssnLength = 10
)

func main() {
	start := time.Now()

	// initializes the buffer with ASCII zeros
	current := make([]byte, ssnLength)

	// generates all possible combinations
	generateAllPossibleSSNs(current, 0)

	duration := time.Since(start)
	fmt.Printf("Completed in %v\n", duration)
}

// generateAllPossibleSSNs generates all possible SSNs and hashes them
func generateAllPossibleSSNs(current []byte, position int) {
	if position == ssnLength {
		sha256.Sum256(current)
		return
	}

	// converts 0-9 directly to ASCII bytes ('0' = 48 in ASCII)
	for i := byte(48); i < byte(58); i++ {
		current[position] = i
		generateAllPossibleSSNs(current, position+1)
	}
}
```

It took 30 minutes to run it on my MacBook Pro 2019, which is definitely an outdated machine as of 2025. If we had another use case, when we, for example, need to store hashes of API keys that are of UUID v4 type, so random, SHA-256 could be a good choice for such a scenario.

This brings us to another point:

- why do we hash SSNs in the first place?

Whenever I mentor other software engineers, I always keep saying that writing code is the easy part of our job, and we should validate each problem and requirement first rather than jumping into the coding mode right away. Same here: what do we gain from hashing the SSNs? Is it a privacy requirement or just an assumption by someone that “SSNs are sensitive, so we shouldn't store the plain values for them”? Is there a chance that we already store them in another table / database in our system in a plain way? Or do we have an ID for the user to store instead of SSN? If not, what are the privacy and legal requirements for the hashed values? As we see, without the salt, SSNs are easy to brute-force, but random salt leads to poor performance.

In some cases, it can appear that, actually, the solution as simple as not hashing the data, if it is not considered too sensitive. In other cases, we might need to have one common salt for all the hashes, which we will store by using Gandalf's rule of thumb:

![image](/images/pics/20250131-0001.webp)

rather than keeping open/within the hash as in our previous example.

The common salt fixes the performance issue with the  Bcrypt column, as the query with a known salt will use indexing:

```sql
EXPLAIN ANALYSE
SELECT *
FROM users
WHERE ssn_hash = crypt('0852285111', '$2a$06$DQqtgCtvY9GkT3uHpShehu');
```

The result is:

```plain
Index Scan using users_ssn_hashed_idx on users  (cost=0.28..8.30 rows=1 width=138) (actual time=0.038..0.038 rows=0 loops=1)
  Index Cond: (ssn_hash = '$2a$06$DQqtgCtvY9GkT3uHpShehuxb3eHD50H6XNxzEyoxZkuDjEt87/6Ce'::text)
Planning Time: 3.462 ms
Execution Time: 0.118 ms
```

As you might remember, we started with 15 seconds per query, and now we are down to 3 milliseconds, which is 5000 times faster — an exciting achievement.

However, as we discussed above, Bcrypt keeps the salt within its hashing result, so this way our common salt is exposed, which breaks the Gandalf's rule of thumb. So, despite the performance bump, **Bcrypt is not an appropriate algorithm for this particular scenario**.

Something faster like SHA-256, SHA-3, BLAKE2/3 are better choices if a common salt is given. If the common salt is not an option, or for some reason there is a need to use exactly Bcrypt, another alternative could be to move the hashing logic to the application, so hashes are passed to the DB as strings. But I'd still go with the first suggestion here. Of course, this needs benchmarking, consulting your security team and checking Postgres support for your particular case before deciding.

And that's it, the team and the users are happy, the system is healthy again. We saved the day, and learned a thing or two about the Bcrypt algorithm.

That was quite a ride, so thanks a lot for reading my post. I truly hope it was useful, and I'll see you in my next ones, as more to come soon. Have fun! =)

**Edit (8th of February 2025):** The earlier version of the post said that Bcrypt is an appropriate choice with the common salt, which is a wrong claim to make, and a big blunder on my side, apologies for that. The issue was discovered and mentioned in [the Reddit discussion](https://www.reddit.com/r/programming/comments/1iie34g/comment/mbbzwfl/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button), so all credit goes to `u/KrakenOfLakeZurich`

**Edit (10th of February 2025)** The earlier version mentioned that SSN was provided via UI form, which is an oversimplification for the sake of the story. Added an explicit elaboration about that.