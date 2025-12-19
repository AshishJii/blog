---
title: "TOTP Explained: How a Stopwatch and Math Replaced Your Password"
date: 2025-12-19
---

Ever wondered about that six-digit code your authenticator app displays? The one that changes every 30 seconds? That's **TOTP (Time-based One-Time Password)**, and it's one of the most elegant pieces of security technology we use every day.

## The Problem with Passwords

Traditional username-password authentication is inherently vulnerable. Even with complex passwords containing letters, numerals, and symbols, a single data breach from an inadequately secured server or a malicious observer looking over your shoulder can expose your credentials and grant unlimited access to your account.

## TOTP: A Clock-Based Security System

TOTP works on a simple principle: both you (the authenticator) and the authentication server share a secret key, and use the current time to independently generate matching codes.

### Cryptographic Implementation

The system relies on a cryptographic calculation:

- Take the timestamp in UNIX format and divide it by 30 so that it remains the same for 30 seconds. Use `floor()` to remove decimals.
  ```js
  time_step = floor(unix_time / 30)
  ```

- The `time_step` and `secret_key` are fed into HMAC-SHA1.
  ```js
  hmac_result = HMAC-SHA1(secret_key, time_step)
  ```

> HMAC-SHA1 stands for Hash-based Message Authentication Code using Secure Hash Algorithm. The SHA1 variant creates a 160-bit output (20 bytes).

- *Dynamic truncation* is performed to extract a 4-byte segment from this 20-byte hash. It involves:

    - Take the last byte of the hash (offset byte):
    ```js
    hmac_result[19]     // 0-based indexing
    ```
    - Find the lowest 4 bits of the offset byte:
    ```js
    offset = hmac_result[19] & 0x0f     // (0x0f = 00001111)
    ```
    - Extract 4 consecutive bytes starting from that offset and read them as a big-endian integer:
    ```js
    four_bytes = hmac_result[offset:offset+4]
    raw_code = int.from_bytes(four_bytes, 'big')
    ```
    - A 32-bit integer is generated. Mask the MSB (most significant bit) since different languages interpret it as signed/unsigned differently. This ensures uniformity by using only 31 bits:
    ```js
    masked_code = raw_code & 0x7fffffff
    ```
    - Apply `% 1000000` to get six decimal digits.
    ```js
    code = masked_code % 1000000
    ```

This is the final code.

### How It Actually Works

- **Step 1:**  
 When you enable two-factor authentication, the server generates a secret key (along with other metadata) and saves it in its database. The server then presents this key to you. You can either directly copy it to an authenticator app or scan the QR code using the app to save it locally.

- **Step 2:**  
 When you need to login, the authenticator app combines that secret key with the current timestamp (rounded to a 30-second window to keep it constant) using the above cryptographic function to generate a 6-digit code.

- **Step 3:**  
 When you enter this code, the server independently calculates what the code should be and verifies if they match. If they do, it logs you in.

### What Makes TOTP Secure

1. **Time-based validity:** Each code expires after 30 seconds, making any intercepted code useless.

2. **Removes Human Weakness:** Humans often save their passwords in plaintext, which is easily exploited by attackers.

3. **Secret Key Safety:** After the initial transfer, the secret key is never transmitted again, so even if a man-in-the-middle intercepts the packets, they cannot use the 6-digit codes for replay attacks.

4. **Completely Offline:** The authenticator app never needs to communicate with the server directly, so it can run completely offline.

## Conclusion

TOTP transformed authentication from **"something you know"** (passwords) to **"something you know AND something you have"** (password plus device). Those six digits represent cryptographic proof of identity, valid only at that current moment.

As cyber threats evolve, TOTP represents an important milestone in authentication technology. It's a fascinating application of cryptographic methods at the consumer level. This foundation enabled the development of even more advanced systems:

- *WebAuthn*: Leveraging device biometrics such as facial recognition or fingerprint scanning
- *Passkeys*: Cryptographic credentials stored directly on user devices
- *Hardware Security Keys*: Dedicated physical authentication devices like YubiKeys

If you haven't enabled TOTP on your critical accounts yet, consider implementing it today. It's a straightforward step that significantly strengthens your security posture.
