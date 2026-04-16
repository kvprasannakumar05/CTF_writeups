# Hens and Roosters Writeup

## Challenge

**Name:** Hens and Roosters  
**Category:** Crypto  
**Description:** "Please help me buy more Legos! The store has such aggressive rate limiting I can't even get an ID!"  
**Target:** `http://hensandroosters.crypto.ctf.umasscybersec.org/`  
**Flag:** `UMASS{oil_does_mix_with_oil_but_roosters_dont}`

## Summary

I solved this by combining two application bugs around an otherwise legitimate signature flow:

1. The HAProxy rate limit keyed on the full URL, so I could bypass it by appending a unique query parameter to every request.
2. The backend had a race condition in `/work`, where multiple concurrent requests using the valid signature for `2|uid` could all verify against the same old state and then increment the user's stud count multiple times.

The important point is that I did **not** need to break the UOV signature scheme. I only needed to use the signatures the application itself gave me, then exploit the server logic around them.

## My Initial Read

The challenge was presented as crypto, so my first instinct was to inspect the signature scheme and look for a cryptographic weakness. But the description also explicitly mentioned aggressive rate limiting, which usually means there is an intended web or infra bug somewhere in front of the crypto.

I started by reading the provided files:

- `DOWNLOADABLE_ASSETS/backend/app.py`
- `DOWNLOADABLE_ASSETS/backend/uov.py`
- `DOWNLOADABLE_ASSETS/proxy/haproxy.cfg`
- `DOWNLOADABLE_ASSETS/backend/Dockerfile`

That immediately changed the problem from "break the signature scheme" into "understand how the app uses signatures and where that usage is unsafe."

## Basics

### What the Service Is Trying To Do

The application models a user with a random `uid` and a number of "studs". The intended flow is:

1. Visit `/` to receive a new `uid`.
2. Use `/buy?uid=...` to get a free signature when the user has `0` studs.
3. Send that signature to `/work` to prove eligibility for the next stud.
4. Receive a new signature for the next state.
5. Repeat until enough studs are collected.

The store wants 7 studs before giving the flag.

### What a Digital Signature Does Here

A digital signature lets the server attest to a message. In this challenge, the message format is:

```text
<stud_count>|<uid>
```

So if I had a valid signature for `2|deadbeef...`, the server would accept that as proof that the user `deadbeef...` legitimately reached 2 studs.

### What UOV Is, at a High Level

The backend uses a scheme called UOV, short for **Unbalanced Oil and Vinegar**. This is a multivariate public-key signature scheme. I did not need the full algebra to solve the challenge, but the key takeaway is:

- `sign(msg)` requires the private key.
- `verify(msg, sig)` uses the public key.
- Forging signatures directly is supposed to be hard.

The `uov.py` file confirms that the backend has both a public and private key and uses them normally:

- `sign()` hashes the message and computes a valid signature with the private key.
- `verify()` hashes the message and checks the quadratic equations with the public key.

So from a solver perspective, the crypto itself looked fine enough that I should first exhaust the application logic.

### What Redis Is Doing

The backend stores state in Redis with a 240-second expiration:

- `uid -> studs`
- `payload -> signature` for the free signature case
- `signature -> payload` as a verification cache

That cache matters later, because `/work` remembers whether a signature was already verified for a given payload.

### What a Race Condition Is

A race condition happens when correctness depends on timing between concurrent operations. If two or more requests read shared state before any of them updates it, they may all act on stale assumptions.

In web challenges, a race usually appears when the code does:

1. Read current state.
2. Check whether an action is allowed.
3. Update the state.

If multiple threads do the check before the update becomes visible, they can all pass.

That is exactly what happens here.

## Source Review

### Proxy: The Rate Limiter

From `haproxy.cfg`:

```haproxy
stick-table type string len 2048 size 100k expire 20s store http_req_rate(20s)
http-request track-sc0 url
http-request deny deny_status 429 if { sc_http_req_rate(0) gt 1 }
```

This means HAProxy tracks request rate by **full URL string**. That is a weak choice, because these are all different keys:

- `/`
- `/?x=1`
- `/?x=2`
- `/work?x=a`
- `/work?x=b`

So instead of being limited per client IP or endpoint, I can evade the limiter by adding a fresh query parameter on every request.

That explains the challenge description: the limiter was aggressive, but it was also flawed.

### Backend: The Business Logic

From `app.py`, the endpoints work like this:

#### `/`

```python
uid = os.urandom(8).hex().lower()
r.set(uid, 0, ex=240)
```

This creates a new user with `0` studs.

#### `/buy`

The interesting part is:

```python
payload = str(studs) + '|' + uid
```

Then:

- if `studs == 0`, the server returns a valid signature for `0|uid`
- if `studs < 7`, it tells me to save up
- if `studs >= 7`, it returns the flag

So `/buy` is not just a shop page. It is also the signing oracle for the initial state.

#### `/work`

This is the key function. It:

1. Reads `uid` and `sig`.
2. Reads the user's current `studs`.
3. Builds `payload = "<studs>|<uid>"`.
4. Verifies that `sig` is valid for that payload.
5. If verification succeeds, it does `r.incr(uid)`.
6. If the new stud count is at most 2, it signs the next payload and returns the next signature.

The two intended chained transitions are:

- signature for `0|uid` gives me a signature for `1|uid`
- signature for `1|uid` gives me a signature for `2|uid`

After that, the server says:

```python
if studs > 2:
    return "You're not getting any more free studs!"
```

So at first glance it looks capped at 2 studs. But the code only checks the cap **after** incrementing.

## The Real Vulnerability

The vulnerable logic in `/work` is effectively:

```python
studs = int(r.get(uid))
payload = str(studs) + "|" + uid
verified = verify(sig, payload)

if verified:
    studs = r.incr(uid)
    if studs > 2:
        return "You're not getting any more free studs!"
```

This creates a classic check-then-act race:

1. Several threads read `studs == 2`.
2. They all compute the same payload `2|uid`.
3. They all verify the same valid signature for `2|uid`.
4. They all call `r.incr(uid)`.

Even if only the first increment was logically intended, all of them can succeed because the authorization decision was made using stale state.

That is the entire exploit.

## Why Concurrency Was Plausible

The backend Dockerfile runs:

```text
gunicorn app:app -k gthread --threads 80 -w 1
```

So the service is explicitly configured to process many requests concurrently in one worker using threads. That is exactly the kind of deployment where this race can be exploitable in practice.

If the service were strictly single-threaded, the race would likely fail.

## Why I Did Not Need To Forge Signatures

This is the part that makes the challenge feel "crypto" while still being an application bug.

I only needed three legitimate signatures:

1. `/buy` gives a valid signature for `0|uid`
2. `/work` with that signature returns a valid signature for `1|uid`
3. `/work` again returns a valid signature for `2|uid`

Once I had the valid signature for `2|uid`, I could reuse it in many concurrent requests before the server fully serialized the state transition.

So the crypto provided the capability, but the bug was in how the application consumed that capability under concurrency.

## Exploit Plan

My final plan was:

1. Bypass the limiter by appending a unique query parameter to every request.
2. Request `/` to get a fresh `uid`.
3. Call `/buy?uid=<uid>` to get the signature for `0|uid`.
4. Send that signature to `/work` to get the signature for `1|uid`.
5. Send that signature to `/work` to get the signature for `2|uid`.
6. Send a large batch of concurrent `/work` requests, all using the signature for `2|uid`.
7. Check `/buy?uid=<uid>` again. If the race pushed the count to at least 7, the server returns the flag.

## Why My First Race Attempts Were Weak

At first, I used many Python threads that each launched a separate `curl` process. That worked logically, but it was too slow operationally:

- process creation adds overhead
- connection setup adds jitter
- request start times drift apart

I was seeing partial wins such as 3 or 4 studs, which confirmed the vulnerability but did not reach 7.

So the problem became not "is the bug real?" but "how do I compress the request timing enough to win the race reliably?"

## The Trick That Made It Reliable

I switched to raw sockets and pre-opened all the connections.

Instead of sending each POST request from scratch at the release moment, I:

1. Opened many TCP connections to the server first.
2. Sent the HTTP headers and almost the entire JSON body on each socket.
3. Held back the **final byte** of the request body.
4. Used a thread barrier so every thread released that final byte at almost the same moment.

That meant the backend received many nearly-complete requests that all became valid POST bodies at once. This tightened the race window substantially.

## Why the Script Uses a Fixed IP

In my environment, DNS resolution from Python was unreliable for this host, even though network access itself worked. To avoid that noise, I resolved the host once and then hardcoded:

```python
HOST = "hensandroosters.crypto.ctf.umasscybersec.org"
HOST_IP = "35.236.95.86"
```

The HTTP requests still include:

```python
Host: hensandroosters.crypto.ctf.umasscybersec.org
```

So this is just an execution-environment workaround, not part of the challenge logic.

## Script Walkthrough

My final exploit script is `solve.py`.

### Imports and Constants

At the top:

```python
HOST = "hensandroosters.crypto.ctf.umasscybersec.org"
HOST_IP = "35.236.95.86"
PORT = 80
```

I also define regexes to parse:

- the generated `uid`
- the 508-hex-character signatures
- the final flag
- the current stud count from `/buy`

The `508` comes directly from the backend:

```python
sig_len = 508
```

### `uniq()`

```python
def uniq() -> str:
    return uuid.uuid4().hex
```

This is how I bypass the HAProxy limiter. Every request gets a different `?x=<random>` value, so the proxy sees each URL as a new stick-table key.

### `http_request()`

```python
def http_request(method: str, path: str, body: str | None = None, timeout: int = 15) -> str:
    conn = http.client.HTTPConnection(HOST_IP, PORT, timeout=timeout)
    ...
```

This is a simple helper for ordinary GET and POST requests before the race. I connect to the IP directly, but I set the `Host` header to the original domain.

This function is used for:

- getting a new UID
- fetching the free signature
- walking the normal `0 -> 1 -> 2` signature chain
- probing whether I won

### `get_uid()`

```python
def get_uid() -> str:
    text = http_request("GET", f"/?x={uniq()}")
```

This requests the homepage with a unique query parameter and extracts the fresh user ID.

### `get_free_sig(uid)`

```python
def get_free_sig(uid: str) -> str:
    text = http_request("GET", f"/buy?uid={uid}&x={uniq()}")
```

When the user has 0 studs, `/buy` returns a valid signature for `0|uid`.

### `work(uid, sig)`

```python
def work(uid: str, sig: str, timeout: int = 15) -> str:
    body = json.dumps({"uid": uid, "sig": sig})
    return http_request("POST", f"/work?x={uniq()}", body=body, timeout=timeout)
```

This is the normal helper for `/work`. I use it for the first two legitimate steps of the chain.

### `get_next_sig(uid, sig)`

```python
def get_next_sig(uid: str, sig: str) -> str:
    text = work(uid, sig)
```

This calls `/work` and extracts the next signature from the response. I use it twice:

- `sig0 -> sig1`
- `sig1 -> sig2`

After that, `sig2` is the signature for `2|uid`, which is the one I race with.

### `race(uid, sig, racers)`

This is the core of the exploit.

I first create the JSON body:

```python
body = json.dumps({"uid": uid, "sig": sig}).encode()
body_prefix = body[:-1]
body_last = body[-1:]
```

Then for each racer I:

1. generate a unique `/work?x=...` path
2. create a TCP connection
3. disable Nagle with `TCP_NODELAY`
4. send the entire HTTP request except for the final byte of the body

The important part is:

```python
sock.sendall(header + body_prefix)
```

At that point, every connection is almost finished.

Then I create worker threads and synchronize them with:

```python
barrier = threading.Barrier(racers + 1)
```

When the barrier releases, each thread sends:

```python
sock.sendall(body_last)
```

That final byte completes the request body on each socket, so the backend gets a burst of valid POST requests almost simultaneously.

Each thread then reads the HTTP response and returns only the body text.

### `probe(uid)`

```python
def probe(uid: str) -> str:
    return http_request("GET", f"/buy?uid={uid}&x={uniq()}")
```

After the race, I call `/buy` again:

- if the stud count is still below 7, it says something like `Only 4 studs?`
- if the count reached 7 or more, it returns the flag

### `attempt(index, racers)`

This function performs one full solve attempt:

1. get a UID
2. get the `0|uid` signature
3. derive `1|uid`
4. derive `2|uid`
5. launch the race
6. probe the account

It also prints debugging info like:

- how many raced requests got the `You're not getting any more free studs!` response
- how many failed as `No free studs for faked keys!`
- what the final `/buy` response was

Those numbers are useful because they tell me whether I am close or completely missing the race.

### `main()`

```python
racers = int(sys.argv[1]) if len(sys.argv) > 1 else 40
max_attempts = int(sys.argv[2]) if len(sys.argv) > 2 else 50
```

This lets me tune:

- how many concurrent race sockets to open
- how many fresh users to try before giving up

The successful run used a socket-based race and returned the flag immediately.

## End-to-End Attack Flow

Here is the whole attack in compact form:

```text
GET  /?x=random              -> create uid with 0 studs
GET  /buy?uid=...&x=random   -> receive sig(0|uid)
POST /work?x=random          -> submit sig(0|uid), receive sig(1|uid)
POST /work?x=random          -> submit sig(1|uid), receive sig(2|uid)
RACE many POST /work?x=random with sig(2|uid)
GET  /buy?uid=...&x=random   -> if studs >= 7, receive flag
```

## Why the Race Works Even Though Later Requests "Fail"

One subtle point is that the application returns:

```text
You're not getting any more free studs!
```

for requests whose `r.incr(uid)` result is greater than 2.

That message sounds like rejection, but it arrives **after the increment already happened**. So those requests are still useful to me. In fact, once I am racing from `2|uid`, I want lots of requests to increment the counter above 2.

The response is "no more free studs," but the side effect is still "stud count increased."

That is the bug.

## Lessons From This Challenge

### 1. Crypto Challenges Can Still Be Application Security Problems

The service used a real signature scheme, but the exploit path was in state handling, caching, and concurrency.

### 2. Rate Limiters Need the Right Key

Limiting on full URL is weak when the attacker controls the query string. A better key would usually be client IP, session, or a normalized route.

### 3. Valid Authorization Tokens Can Still Be Misused

I never forged a signature. I reused a legitimate signature in a timing window where the server incorrectly treated stale state as current state.

### 4. Synchronization Quality Matters in Race Exploits

A conceptual race is not automatically exploitable. The difference between "sometimes reaches 4 studs" and "actually gets the flag" was the delivery method:

- naive threaded requests were too sloppy
- pre-opened sockets with a held final byte were tight enough

## Final Result

The final flag was:

```text
UMASS{oil_does_mix_with_oil_but_roosters_dont}
```

## Files

- Exploit script: `solve.py`
- This writeup: `writeup.md`
