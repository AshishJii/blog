---
title: "CTF Write-up: Nitro"
date: 2025-12-16
---

**Category:** Scripting / Automation  
**Description:** A race-against-time challenge where the user must process data and submit a response faster than humanly possible.

## 1\. Challenge Description

The challenge provides a landing page with a clear directive: **"Manual attempts miss the window—only code will do."**

The workflow requires the following steps:

1.  **GET** a random string from `/task`.
2.  **Reverse** the string.
3.  **Base64 encode** the reversed string.
4.  **Format** the result into a specific template: `CSK__{{payload}}__2025`.
5.  **POST** the final string to `/submit`.

Any attempt to perform these steps manually results in a generic "Too slow\!" error, confirming that the server enforces a strict timeout (likely under 1 second).

## 2\. Reconnaissance

Manually visiting `/task` returns an HTML page containing the target string inside a paragraph tag:

```html
<p>Here is the input string: [RANDOM_STRING]</p>
```

This means our script cannot just grab the raw response body; it must parse the HTML (or use Regex) to extract only the random string.

**The Transformation Logic:**

1.  **Input:** `AXV7kzeY3jeu`
2.  **Reverse:** `uej3Yezk7VXA`
3.  **Base64:** `dWVqM1llems3VlhB`
4.  **Final Payload:** `CSK__dWVqM1llems3VlhB__2025`

## 3\. The Solution (Automation Script)

We can solve this using a compact JavaScript snippet.

To inject the payload, we can use the browser's Developer Tools. This script must be executed in the Console (F12) *while on the challenge page* to ensure the request originates from the same domain, avoiding CORS/Same-Origin errors.

```javascript
(async () => {
    const taskEndpoint = '/task';
    const submitEndpoint = '/submit';

    try {
        // Step 1: Fetch the task and extract the random string
        const html = await (await fetch(taskEndpoint)).text();
        const inStr = html.match(/<p>Here is the input string: (.*?)<\/p>/)[1];
        
        // Step 2: Process the string (Reverse -> Base64 -> Format)
        const processed = btoa(inStr.split('').reverse().join(''));
        const payload = `CSK__${processed}__2025`;

        // Step 3: Submit immediately
        const response = await fetch(submitEndpoint, {
            method: 'POST',
            headers: { 'Content-Type': 'text/plain' },
            body: payload
        });

        // Step 4: Get Flag
        console.log("Server Response:", await response.text());

    } catch (e) {
        console.error("Automation failed:", e);
    }
})();
```

## 4\. Execution & Result

Running the script in the console instantly performs the fetch-transform-post cycle within the allowed time window.

**Output:**

```text
Server Response: Nice automation! Here is your flag: ClOuDsEk_ReSeArCH_tEaM_CTF_2025{ab03730caf95ef90a440629bf12228d4}
```

**Flag:** `ClOuDsEk_ReSeArCH_tEaM_CTF_2025{ab03730caf95ef90a440629bf12228d4}`
