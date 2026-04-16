# Block City Times Web Challenge Writeup

  

## Overview

  

This challenge looked like a normal news website at first, but the real bug was in how it handled uploaded files and how two internal admin bots interacted with the app.

  

The final flag was:

  

`UMASS{A_mAn_h3s_f@l13N_1N_tH3_r1v3r}`

  

In this writeup, I will explain the logic from the beginning in simple terms:

  

1. What I looked at first

2. What the important code paths were

3. What the vulnerability was

4. How I chained the bugs together

5. How I got the flag

  

## First Recon

  

I started by looking at the site normally.

  

The homepage showed:

  

- several public articles

- a `Submit a Story` feature

- an `Admin Login`

- an `API` link

  

That already suggested a few likely attack surfaces:

  

- file upload

- admin functionality

- hidden API behavior

  

Since source code was provided, I focused on understanding the application flow instead of blindly fuzzing.

  

## Reading the Source

  

The app was mainly a Spring Boot site, but it also had two Node services:

  

- `editorial`

- `report-runner`

  

That was important, because web CTFs often use browser bots to visit attacker-controlled content.

  

### Important files

  

The most useful files were:

  

- `src/main/java/com/example/demo/controller/web/StoryController.java`

- `src/main/java/com/example/demo/controller/admin/ReportController.java`

- `src/main/java/com/example/demo/security/SecurityConfig.java`

- `src/main/resources/application.yml`

- `editorial/server.js`

- `developer/report-api.js`

- `developer/trigger-server.js`

  

## Understanding the Upload Flow

  

The first important logic was in `StoryController.java`.

  

When a user submits a story:

  

- the app accepts `title`, `author`, `description`, and `file`

- it checks the uploaded file's `Content-Type`

- it only allows:

  - `text/plain`

  - `application/pdf`

- it saves the uploaded file in the uploads directory

- then it sends the filename to the internal `editorial` service

  

At first glance, this looks safe enough. But the key mistake is that the code trusts the upload part's MIME type:

  

```java

String contentType = file.getContentType();

if (contentType == null || !outboundProps.getAllowedTypes().contains(contentType)) {

    ...

}

```

  

This means the server is not checking the real file contents. It is only checking what MIME type the client claims the file has.

  

That is the first bug.

  

## Why That Matters

  

Later, uploaded files are served back through:

  

`/files/{filename}`

  

The app uses:

  

```java

String contentType = Files.probeContentType(filePath);

```

  

This means when the file is served, the response type is based on the saved filename or detected file type on disk.

  

So I can do this:

  

- upload a file named `exploit.html`

- tell the server it is `text/plain`

- pass validation

- get it stored as an `.html` file

- later the app serves it as `text/html`

  

That turns a file upload into stored HTML/JavaScript execution.

  

In simple words:

  

The server checked the file one way during upload, but served it a different way later. That mismatch created the vulnerability.

  

## Finding the Bot

  

The next step was to understand what happens after upload.

  

In `editorial/server.js`, I found that after submission, the internal editorial service does this:

  

1. logs into the main app as admin

2. opens the uploaded file URL:

  

`/files/<filename>`

  

That means if I upload malicious HTML, the admin bot will open it while already logged in.

  

This is exactly what I wanted.

  

So the first stage of the exploit became:

  

- upload a malicious HTML file

- wait for the editorial bot to visit it

- execute JavaScript in the context of the main site as admin

  

This is basically stored XSS through file upload.

  

## Looking for What Admin Could Reach

  

Now the question was:

  

What can I do once my JavaScript runs in the admin's browser?

  

I checked the security config.

  

In `SecurityConfig.java`, I saw:

  

- `/admin/**` requires admin

- `/api/config/**` requires admin

- `/files/**` requires admin

- `PUT /api/tags/**` requires admin

- CSRF is ignored for `/api/**`

  

Also, actuator endpoints were protected by admin role, but still reachable to an authenticated admin.

  

That suggested I might be able to use the admin bot's session to call internal admin-only endpoints.

  

## Understanding the Config Logic

  

In `application.yml`, I found:

  

- `app.active-config: prod`

- `app.enforce-production: true`

  

And in `ReportController.java`, I found:

  

- the report feature only works in `dev`

  

Specifically:

  

```java

if (!appProps.getActiveConfig().equals("dev")) {

    return "redirect:/admin?error=reportdevonly";

}

```

  

So at first, the report feature should be disabled.

  

That looked intentional. It meant the report system probably mattered, but I first needed a way to switch the app into `dev`.

  

## Finding the Config-Switching Primitive

  

The app exposed actuator endpoints including:

  

- `/actuator/env`

- `/actuator/refresh`

  

This was visible in `application.yml`.

  

That was a huge clue.

  

In Spring setups like this, if an authenticated admin can write to `/actuator/env` and then call `/actuator/refresh`, they may be able to change runtime properties.

  

So from my uploaded HTML running in the admin's browser, I could try:

  

1. `POST /actuator/env` with:

   `{"name":"app.active-config","value":"dev"}`

2. `POST /actuator/refresh`

  

Because my JavaScript was running same-origin as the site, it could send authenticated requests using the admin's session.

  

This would switch the app from `prod` to `dev` at runtime.

  

## Understanding the Report Bot

  

Then I checked the second Node service.

  

In `developer/report-api.js`, I found the most important logic in the challenge:

  

1. it logs in as admin

2. it sets a cookie:

   `FLAG=<real flag>`

3. it visits a URL based on a user-controlled endpoint

4. it reads:

   `document.body.innerText`

5. it returns that text in JSON

  

This meant the report runner was a second bot, and this one carries the flag in a cookie.

  

So the real problem became:

  

How do I make the report bot visit attacker-controlled HTML?

  

## The Filter in the Report Feature

  

In `ReportController.java`, the endpoint parameter is checked like this:

  

```java

if (!endpoint.startsWith("/api/")) {

    return "redirect:/admin?error=reportbadendpoint";

}

```

  

So the app tries to restrict the report bot to only visit API paths.

  

But this check is too weak.

  

It only checks the string prefix.

  

That means a path like:

  

`/api/../files/exploit.html`

  

still starts with `/api/`, so it passes validation.

  

But browsers normalize the path before requesting it.

  

So:

  

`/api/../files/exploit.html`

  

becomes:

  

`/files/exploit.html`

  

That is a path traversal / normalization bypass on the endpoint filter.

  

## Full Exploit Chain

  

At that point, the full chain was clear.

  

### Step 1: Upload a malicious HTML file

  

I created an HTML payload and uploaded it as:

  

- filename: `exploit.html`

- declared MIME type: `text/plain`

  

This passes the upload check, but the file is later served as HTML.

  

### Step 2: Let the editorial bot open it

  

The editorial service logs in as admin and visits my uploaded file.

  

So my JavaScript runs with admin privileges on the app's origin.

  

### Step 3: Use admin access to switch the app into dev mode

  

My script sends:

  

```http

POST /actuator/env

{"name":"app.active-config","value":"dev"}

```

  

and then:

  

```http

POST /actuator/refresh

```

  

Now the app behaves as if it is in development mode.

  

That unlocks the report feature.

  

### Step 4: Trigger the report bot with a bypassed endpoint

  

My script then submits a report request to:

  

`/admin/report`

  

with endpoint:

  

`/api/../files/<my_uploaded_filename>`

  

That passes the `startsWith("/api/")` check, but resolves to my uploaded HTML file.

  

### Step 5: The report bot visits my HTML while holding the flag cookie

  

The report bot:

  

- logs in as admin

- sets cookie `FLAG=UMASS{...}`

- visits my file

  

Now my same uploaded HTML runs again, but this time in the report bot's browser context, where `document.cookie` contains the flag.

  

### Step 6: Make the report bot reveal the flag

  

The report bot extracts:

  

`document.body.innerText`

  

So in the payload, when it detects the `FLAG` cookie, it simply writes the cookie into the page body.

  

That makes the report system capture the flag and include it in the returned report data.

  

### Step 7: Exfiltrate the flag somewhere public

  

I wanted a simple way to retrieve the flag without needing an external webhook.

  

So my payload took the flag from the report response and stored it into a public API location.

  

I used:

  

`PUT /api/tags/article/1`

  

Because admin is allowed to update article tags, and the article tag API is publicly readable.

  

Then I could fetch:

  

`/api/tags/article/1`

  

and see the flag there.

  

That gave me the flag cleanly.

  

## Why the Attack Worked

  

This challenge was really a chain of small issues:

  

1. The upload validation trusted user-controlled MIME type

2. Uploaded files were later served as active HTML

3. An internal admin bot opened those uploaded files

4. Admin browser access could modify runtime config through actuator endpoints

5. The report feature trusted a weak string check for endpoint restriction

6. The report bot carried the flag in a readable cookie

7. The report bot reflected page body text back to the app

  

Each issue alone might not be enough. Together, they led to full compromise.

  

## My Payload Logic

  

My exploit HTML worked in two stages.

  

### First stage: editorial bot context

  

If the page did not have a `FLAG` cookie yet, the script:

  

- switched the app to `dev`

- refreshed config

- requested `/admin`

- extracted the CSRF token

- submitted `/admin/report` with endpoint `/api/../files/<same file>`

  

### Second stage: report bot context

  

If the page did have a `FLAG` cookie, the script:

  

- wrote `document.cookie` into the page body

  

The report bot then captured that text.

  

Finally, the first stage script parsed the returned report HTML, extracted the flag, and saved it into article tags.

  

## Exploit Idea in Simple Words

  

The easiest way to describe the attack is:

  

- I uploaded a fake text file that was actually HTML

- the admin bot opened it

- my JavaScript ran as admin

- I used that to enable the hidden dev report feature

- I tricked the second bot into visiting the same malicious HTML

- that second bot had the flag in a cookie

- my HTML made the bot print its cookie

- the app showed me the result

  

## Example Upload Strategy

  

The important trick was to upload the file with:

  

- `.html` filename

- fake MIME type `text/plain`

  

For example, using multipart upload, the file part can be sent as:

  

```bash

file=@exploit.html;type=text/plain;filename=exploit.html

```

  

That is enough to bypass the upload filter.

  

## Lessons From This Challenge

  

This challenge is a good example of why browser-bot workflows are dangerous when combined with uploads.

  

Main lessons:

  

- Never trust client-provided MIME type for upload validation

- Never serve untrusted uploads from the main application origin

- Admin bots should never browse attacker content on the same origin

- Do not expose dangerous actuator endpoints in production

- Prefix checks like `startsWith("/api/")` are not a real path security boundary

- Secrets like flags should not be placed in readable JavaScript cookies

  

## Final Flag

  

`UMASS{A_mAn_h3s_f@l13N_1N_tH3_r1v3r}`

--- 
