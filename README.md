# 🧬 JWT Authentication Bypass via `kid` Header Path Traversal — A Sneaky Admin Takeover

> *"Sometimes all it takes is a null byte and some path traversal to crack an admin panel wide open."*
> — Aditya Bhatt

### 🚨 TL;DR

In this write-up, we exploit a poorly implemented JWT validation mechanism that uses the `kid` (Key ID) parameter to fetch the secret key from the server's file system. By abusing path traversal and manipulating the JWT's `kid` value, we redirect the key lookup to `/dev/null`, and sign our token using a null byte — effectively bypassing authentication and impersonating the administrator. 🛡️

---

## 🧠 Quick Background — What is `kid` in JWT?

In JSON Web Tokens (JWTs), the `kid` (Key ID) header is used by the server to identify which cryptographic key to use when verifying the token's signature. Ideally, this should be a controlled identifier matched to internal secrets.

But what if the server is **fetching the key file directly from its filesystem** using this value? That’s when path traversal (`../../../`) becomes your best friend.

---

## 🛠️ Lab Setup — PortSwigger's Playground

Lab: [JWT authentication bypass via kid header path traversal](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal)
Account: `wiener:peter`
Goal: Access `/admin` as the `administrator` user, then delete `carlos`.

---

## 🔬 Exploit Walkthrough (PoC Style)

This is a precise **step-by-step** proof-of-concept that maps to how I performed the lab. No theory here — just raw hacking:

**Background: Install JWT Editor Extension in Burp Suite**

![446720600-709c7242-cd05-40e0-93d5-c7a5edbef181](https://github.com/user-attachments/assets/7e0a4cd6-67b4-490f-9fa2-e138f135901a) <br/>

**1.** Access the Lab: [Lab Link](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal) and log in using `wiener:peter`.

![1](https://github.com/user-attachments/assets/f5dfee84-0b46-4f82-bbc7-315330d90954) <br/>

**2.** Try accessing `/admin`. We receive:

```
Admin interface only available if logged in as an administrator.
```

![2](https://github.com/user-attachments/assets/075ed783-3672-4d96-b9d1-cf999c071fed) <br/>

**3.** Intercept the `/admin` request and **send it to Repeater**.

![3](https://github.com/user-attachments/assets/1dcd7ebd-8082-4a46-9d35-715d4f4c51eb) <br/>

**4.** In Burp’s **JWT Editor tab**, go to **Keys > New Symmetric Key**. Generate a random key, then **manually set the `k` value to `AA==`** (Base64 for null byte).

![4](https://github.com/user-attachments/assets/1e79f39e-4289-4a82-bb9d-2bb260488a4c) <br/>

**5.** Switch back to the JWT tab in Repeater. Select the JWT payload.

![5](https://github.com/user-attachments/assets/ad431a22-e3b1-4aa5-82f2-a3405c24ec8b) <br/>

**6.** Modify:

* `sub` claim to `"administrator"`
* `kid` header to the path traversal string: `../../../../../../../dev/null`

![6](https://github.com/user-attachments/assets/df09a52e-72e3-4302-b140-7cc85c44c10f) <br/>

**7.** Click **Sign**, select the null-byte key, and tick “**Don’t modify the header**”.

![7](https://github.com/user-attachments/assets/4176b4ca-8c26-4500-a680-7cdb03103d7a) <br/>

**8.** **Send** the request — **Boom! You're in. Admin access achieved.**

![8](https://github.com/user-attachments/assets/21ba8ada-a46c-40fb-94a8-4a3416a80754) <br/>

**9.** In the response, locate the link: `/admin/delete?username=carlos`, change the URL, and **Send**.

![9](https://github.com/user-attachments/assets/c9c3da02-7560-4f7e-9370-74269664d253) <br/>

**10.** Right-click the request → **Show in Browser**. Copy and paste to confirm that **Carlos is deleted** and **the lab is solved**. 🏁

![10](https://github.com/user-attachments/assets/bb4545be-6d73-4afb-9e15-8720a2033bcc) <br/>

---

## 🧩 Behind the Scenes — Why It Works

Here’s the juicy part: The server is trying to be "dynamic" by retrieving the key file based on the `kid` header. When we send `../../../../../../../dev/null`, the server uses this file to verify the signature. But since `/dev/null` returns an **empty file**, we cleverly match this with a JWT signed using a **null byte key (`AA==`)**.

It's basically a server saying:

> "Oh, the key is nothing? Fine. I trust that." 🙃

---

## 🐞 Bug Bounty Impact

This is **critical severity** in a real-world app:

* Complete auth bypass
* Full admin control
* No brute force or social engineering
* Fully automatable
* Stealthy (logs may not show an error)

If you ever see a JWT with a `kid` header — and especially if it's vulnerable to path traversal — **dig deeper**. This technique is a **goldmine** for bug bounty hunters and red teamers.

---

## 🧪 Payload Summary

```json
// JWT Header
{
  "alg": "HS256",
  "kid": "../../../../../../../dev/null"
}

// JWT Payload
{
  "sub": "administrator"
}

// Signing key (Base64 null byte)
AA==
```

---

## 🧠 Lessons Learned

* Always **validate and sanitize** values like `kid`, especially if they influence file paths.
* Never let users influence where your app loads its crypto keys from.
* Security through obscurity or dynamic file loading is **not** a secure practice.
* `/dev/null` is a hacker's playground when it comes to weak implementations.

---

## 🎉 Final Words

This was a fun and elegant lab that demonstrates how dangerous JWT misconfigurations can be. From an authentication bypass to full admin takeover — all with some clever traversal and a null byte.

🧑‍💻 Keep hacking, keep learning — and remember...

> 🔓 **Happy Hacking!**
> *— Aditya Bhatt* 🐾

---
