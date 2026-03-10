
<img width="567" height="595" alt="Screenshot 2026-03-08 131833" src="https://github.com/user-attachments/assets/a645be85-b5cd-4f90-8d4b-fd93294d4abe" />

### **1. Phase I: Initial Reconnaissance & Vulnerability Discovery**

My first step was exploring the API via Swagger UI (`/docs`) and the frontend JavaScript (`app.js`). I identified several key endpoints:

- `GET /leaderboard`: Showed user popularity.
    
- `POST /vote_for`: The primary action for users.
    
- `GET /flag`: A hint-dropping endpoint.
    

**The Vulnerability (Broken Error Handling):**

While testing the `/vote_for` endpoint, I discovered that attempting to vote for the same person twice triggered an error. Crucially, the error message leaked a `recent_voters` array containing the usernames of everyone who had voted for that target.

---

### **2. Phase II: Mapping the Graph (The "8 Suspects")**

Using the leak, I wrote a Python script to iterate through the entire user base (103 users). Your first hypothesis for **"Target V"** was "Target 5"—a user who had cast exactly **5 outbound votes**.

- **The Findings:** My script identified 8 users who fit this criteria: `oscar`, `devon.`, `rileyio`, `oliversys`, `nina.`, `bob`, `eve_`, and `lina_ops`.
    
- **The Mistake:** I initially submitted flags based on these users in random or alphabetical order. They were all rejected. This was a "Rabbit Hole"—the number 5 was a hint for a sequence, not necessarily the total vote count.
    

---

### **3. Phase III: The Botnet & The Red Herrings**

I noticed a hint in the `/flag` endpoint: `"not enough votes need 5"`.

- **The Thought Process:** I hypothesized that your _own_ account needed a popularity score of 5. You built a **Bash botnet** that registered 5 fake accounts to vote for your master account.
    
- **The Result:** Even with 5 votes, the `/flag` endpoint remained static, and the `/hidden_flag` endpoint was a Rickroll. This was a classic CTF "troll" designed to waste my time and rate-limit my IP.
    

---

### **4. Phase IV: Deep Forensic Analysis (The "Correct" Path)**

I went back to the documentation and pulled the **OpenAPI Specification**. This was the turning point.

- **The Discovery:** The spec revealed that the `/flag` endpoint accepted an **array** of strings via a query parameter named `votes`.
    
- **Target V Identity:** I identified **Victor** (the NATO word for 'V') as the true target. While he had 25 votes, most were "noise" from other players.
    
- **The Breakthrough:** I wrote a script to dump Victor's votes **chronologically**. I noticed a perfect 5-minute interval between 5 specific votes cast just before the CTF went live:
    
    1. `15:22` -> **emilysys**
        
    2. `15:27` -> **devon.**
        
    3. `15:32` -> **judy**
        
    4. `15:37` -> **dave**
        
    5. `15:42` -> **alice**
        

---

### **5. The Final Solve**

Using the sequence identified in the forensic logs, I constructed the final API request:

`GET /flag?votes=emilysys&votes=devon.&votes=judy&votes=dave&votes=alice`

The server validated the **exact chronological sequence** and returned the real flag.

---

### **Mistakes & Lessons Learned**

|**Mistake**|**Lesson**|
|---|---|
|**Trusting the UI**|The frontend leaderboard only showed 100 users, hiding the real "Target V" (Victor) and the admin. Always trust the raw API data over the UI.|
|**Ignoring the Spec**|I spent hours guessing flag formats before finding that the OpenAPI spec explicitly defined the `votes` query parameter. Always read the docs first!|
|**Over-Automation**|Initial scripts ignored the _time_ of the votes. In shared-instance CTFs, timestamps are the only way to separate "Author Intent" from "Player Noise."|
