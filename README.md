# DVWA Security Lab Report

**Author:** Meesum Abbas  
**Target Application:** Damn Vulnerable Web Application (DVWA)  

## Environment Setup Description
Pulled `vulnerables/web-dvwa` from docker for `linux/amd-64` platform. Used the command `docker run -d --name dvwa --platform linux/amd64 -p 8080:80 vulnerables/web-dvwa` to create a container with the name `dvwa` and run it for amd64 platform (since working on ARM-architecture Mac).

---

## Executive Summary
This report documents the security testing and vulnerability assessment performed on the DVWA platform. The objective of this assessment is to identify, exploit, and document common web application vulnerabilities.

---


## 1. BruteForce 
* **Vulnerability Type:** Improper Authentication / Lack of Rate Limiting
* **OWASP Category:** A07:2021-Identification and Authentication Failures

### Security Level: Low
#### Attack: 
1. Signed in with the admin password username. 
2. Opened the link of the image and modified the url to get the list of all users. 
3. Used SQL injection to login.
![user sql injection](<evidences/brute/low/Screenshot 2026-03-12 at 10.25.56 AM.png>)
4. Also used cluster bomb attack with the 100 most common passwords to find the passwords for all users. Used Burp Suite Intruder with the payload 
```
GET /vulnerabilities/brute/?username=$username$&password=$password$&Login=Login HTTP/1.1
Host: localhost:8071
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-GB,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Referer: http://localhost:8071/vulnerabilities/brute/
Cookie: PHPSESSID=05qs47qrd3pvvs2gm0cr1he6b1; security=low
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Priority: u=0, i
```
5. Used String Matching to verify which passwords were accepted

![users list](evidences/brute/low/users.png) 
![passwords list](evidences/brute/low/passwords.png)
![matched usernames to passwords](evidences/brute/low/matched.png)

#### Analysis:
- **Why it passed:** No effective brute-force protections (no lockout, no throttling, no CAPTCHA), and weak credential handling allowed automated guessing.
- **How higher levels help:** Adding input hardening, request controls, and tokenized workflows increases effort and reduces direct automation.

---

### Security Level: Medium
#### Attack: 
1. This time sql injection failed.
2. However the same payload (after updating security to medium in cookies) worked in Burp Suite for finding. Payload:
```
GET /vulnerabilities/brute/?username=$username$&password=$password$&Login=Login HTTP/1.1
Host: localhost:8071
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-GB,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Referer: http://localhost:8071/vulnerabilities/brute/
Cookie: PHPSESSID=05qs47qrd3pvvs2gm0cr1he6b1; security=medium
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Priority: u=0, i
```

![medium-bruteforce-run](evidences/brute/medium/bruteforce-pw-mdedium.png)

#### Analysis:
- **Why it passed:** SQL injection was reduced, but password guessing still worked because rate-limiting and account lock protections were still insufficient.
- **How higher levels help:** Per-request tokens and stricter anti-automation controls make repeated forged attempts harder to scale.

---
### Security Level: High
#### Attack: 
1. This time the default brutefroce failed. The paylooad now expects a user_token, generated each time the user visits the login page and is a one time token. Payload
```
GET /vulnerabilities/brute/?username=$username$&password=$password$&Login=Login HTTP/1.1
Host: localhost:8071
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-GB,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Referer: http://localhost:8071/vulnerabilities/brute/
Cookie: PHPSESSID=05qs47qrd3pvvs2gm0cr1he6b1; security=high
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Priority: u=0, i
```
![Brute force](evidences/brute/high/failed.png)
2. However using macros, that fetch the user token from the html each time, we can send the requests.
![making a macro p1](evidences/brute/high/macro-1.png)

![making a macro p2](evidences/brute/high/macro-2.png)

![making a macro p3](evidences/brute/high/macro-3.png)

![making a macro p4](evidences/brute/high/macro-4.png)

![making a macro p5](evidences/brute/high/macro-5.png)
The final Payload: 
![payload](evidences/brute/high/payload.png) 
Success Evidence: 
![success evidence](evidences/brute/high/success.png)

#### Analysis:
- **Why it passed:** Security relied on a token, but Burp macros automated token retrieval and replay, bypassing the intended one-time barrier.
- **How stronger controls help:** Binding tokens to strict request context, adding behavioral rate checks, and MFA/CAPTCHA would further reduce brute-force success.

---

### Note:
I have used Burp Suite for convenience and faster results, though most of this can be achieved by looping over the lists with a curl call. Also, I didn't want to have AI generate the script for me without me actually learning a lot so used this route.


## 2. Command Injection
* **Vulnerability Type:** Command Injection
* **OWASP Category:** A03:2021-Injection

### Security Level: Low
#### Attack:
1. The application takes an IP address and executes a `ping` command.
2. By appending a semicolon `;` or an ampersand `&`, we can execute arbitrary shell commands.
3. Input used: `127.0.0.1 | ls`
4. This allowed us to list the files in the current working directory.

![Command Injection Success](evidences/command-injection/low/result+cmd.png)

#### Analysis:
- **Why it passed:** User input reached shell execution directly with no strong sanitization or allowlist validation.
- **How higher levels help:** Character filtering and stricter command controls reduce direct command chaining opportunities.

---

### Security Level: Medium
#### Attack:
1. In this level, some special characters like `&&` and `;` are blacklisted by the application's source code.
2. However, other operators like `|` (pipe) or `&` (background) are still allowed or the blacklist might be incomplete.
3. By inspecting the source code, we can identify which characters are filtered and find alternatives.
4. Input used: `127.0.0.1 | ls`

![Source code showing blacklist](evidences/command-injection/medium/source-blacklisted.png)
![Command Injection Success Medium](evidences/command-injection/medium/result+cmd.png)

#### Analysis:
- **Why it passed:** Blacklist filtering was incomplete, so alternative operators still bypassed restrictions.
- **How higher levels help:** More restrictive filtering can block more payloads, but robust allowlisting and avoiding shell calls are stronger fixes.

---

### Security Level: High
#### Attack:
1. The blacklist is more extensive in this level, attempting to filter out most command separators.
2. A common mistake in blacklisting is including spaces or not accounting for all variations of an operator.
3. In this case, the blacklist for `| ` (pipe followed by a space) was used instead of just `|`.
4. By using the operator without a space or using a different bypass, we can still execute commands.
5. Input used: `127.0.0.1 |ls`

![Source code showing advanced blacklist](evidences/command-injection/high/source-blacklisted.png)
![Command Injection Success High](evidences/command-injection/high/result+cmd.png)

#### Analysis:
- **Why it passed:** Defensive logic still depended on blacklist patterns, and small parsing gaps (like spacing assumptions) enabled bypass.
- **How stronger controls help:** Replacing shell execution with safe APIs and strict allowlisted inputs would remove this class of bypass.

### Note:
We can use tools liuke BURP Suite or a simple python script, that bruteforces through all the possible characters and some combinations and check the result, which will allow us to find the vulnerability when the source code is protected.

---

## 3. Cross-Site Request Forgery (CSRF)
* **Vulnerability Type:** Cross-Site Request Forgery (CSRF)
* **OWASP Category:** A01:2021-Broken Access Control

### Security Level: Low
#### Attack:
1. The application allows changing the user's password via a simple GET request without any CSRF tokens.
2. An attacker can craft a malicious URL and trick a logged-in user into clicking it.
3. Example payload and request:
![CSRF Request](evidences/csrf/low/request.png)
4. The password is successfully changed without the user's explicit consent: 
![CSRF Result](evidences/csrf/low/result.png)

#### Analysis:
- **Why it passed:** No CSRF token or origin validation was enforced, so a crafted URL could trigger state-changing actions.
- **How higher levels help:** Referer/origin checks and token validation add request legitimacy checks before accepting password changes.

---

### Security Level: Medium
#### Attack:
1. The application introduces a check on the `HTTP Referer` header to ensure the request originates from the same server.
2. We begin with a legitimate request to observe the standard behavior and how security is enforced.
![actual-request + security-increase](evidences/csrf/medium/actual-request%20+%20security-increase.png)
3. Using Burp Suite, we use the same referer token with a password change request to modify it.
![intercepted+forged-request](evidences/csrf/medium/intercepted+forged-request.png)
4. We verify that the password has been successfully changed through the intercepted request.
![password-changed-verification](evidences/csrf/medium/password-changed-verification.png)
5. We also observe that simply using an incorrect original password (without the referral bypass) would normally lead to failure.
![actual-password-incorrect](evidences/csrf/medium/actual-password-incorrect.png)
6. Finally, the forged request is fully accepted by the application.
![forged-pw-accepted](evidences/csrf/medium/forged-pw-accepted.png)

#### Analysis:
- **Why it passed:** Security depended mainly on referer-based checks, which can be replayed or manipulated during interception.
- **How higher levels help:** Per-request CSRF tokens tied to session state are harder to forge than header-only validation.

---

### Security Level: High
#### Attack:
1. The application now uses a `user_token` (CSRF token) that is unique per session/request, making simple forgery difficult.
2. We start by observing a legitimate password change request to understand the token structure.
![initial-password-change-request](evidences/csrf/high/initial-password-change-request.png)
3. Since the token is required, we must fetch a fresh one. This usually requires an secondary vulnerability like XSS to read the token from the page.
![fetching-a-new-user-token](evidences/csrf/high/fetching-a-new-user-token.png)
4. Once we have a valid token, we can craft an attack link that includes this specific `user_token`.
![attack-link-with-user-token](evidences/csrf/high/attack-link-with-user-token.png)
5. When the victim clicks the link containing the valid token, the forgery is successful.
![successful-forgery](evidences/csrf/high/successful-forgery.png)

#### Analysis:
- **Why it passed:** Token protection was present, but token theft/reuse (via same-origin weakness such as XSS) allowed a valid forged request.
- **How stronger controls help:** Using the current password (encrypted in transit) to verify the user before changing it.



