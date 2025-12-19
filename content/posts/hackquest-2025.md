---
title: "TCS HackQuest Season 10 Writeup"
date: 2025-12-19
---

Cybersecurity is as much about persistence as it is about technical skill. I recently participated in **TCS HackQuest Season 10** (held on December 13, 2025). The competition was intense, featuring a total of 14 challenges ranging from steganography, reverse engineering, web exploitation, binary analysis, etc.

Despite some technical hurdles with the server in the final hours, I managed to successfully solve **11 out of the 14 questions**. This writeup serves as a detailed breakdown of my approach to these challenges.

---

## Challenge Writeups

### 1. Noise

**Category:** General / Forensic

**Approach:**

* **Extraction:** Downloaded the challenge archive and extracted the contents using `unzip`. I located a specific target file with a `.opt` extension.
* **Strings Analysis:** Since the file appeared to be a binary, I ran the `strings` command to extract all sequences of printable characters.
* **Identification:** I reviewed the multiple matches returned by the search. By filtering for the standard flag format and expected string length, I manually identified and extracted the correct flag.

---

### 2. Hidden Layers

**Category:** Steganography

**Approach:**

* **Initial Discovery:** Extracted the archive to find a PNG file named `image_dF4edEC1A1.png`.
* **Deep Analysis:** Used the `zsteg` tool with the `-a` argument. This performs an exhaustive analysis of all possible Least Significant Bit (LSB) steganography combinations across different color channels.
* **Extraction:** The flag was revealed within the `b1,rgb,lsb,xy` channel output.

---

### 3. Stack Fall

**Category:** Binary Exploitation (Pwn)

**Approach:**

* **Vulnerability Research:** The problem description included the hint "feed it too much." In the context of binary challenges, this is a classic indicator of a **Buffer Overflow** vulnerability.
* **Testing:** I executed standard initial commands to test the application logic. As expected, normal inputs did not yield results.
* **Exploitation:** I provided an excessively long string as input to overflow the allocated buffer. This successfully overwrote the intended memory space and triggered the logic to reveal the flag.

---

### 4. Internal Affairs

**Category:** Web / SSRF

**Approach:**

* **Reconnaissance:** Analyzed the webpage's health check feature and source JavaScript. Attempting to access the internal server using `curl` failed because the application had a filter against "private IPs" like `127.0.0.1`.
* **Filter Bypass:** I bypassed the restriction by using the equivalent IP `0.0.0.0`.
* *Payload:* `curl -X POST http://challenge.tcshackquest.com:19126/fetch -H "Content-Type: application/json" -d '{"target": "http://0.0.0.0:8080", "checkType": "full"}'`


* **Admin Discovery:** The HTML response from the internal request contained a developer comment mentioning a hidden `/admin` endpoint.
* **Endpoint Escalation:** Modified the payload to target `http://0.0.0.0:8080/admin`. This returned the source code of the admin panel.
* **Final Extraction:** Inside the admin source, I found a commented-out API link: `/api/flag_6c968272dd7fb92e3`. Accessing this final endpoint via the same SSRF bypass revealed the flag.

---

### 5. Address Abyss

**Category:** Scripting / Log Analysis

**Approach:**

* **Pattern Recognition:** The challenge involved a massive log file. The goal was to find data hidden within specific IP ranges: `92.7.X.Y` (IPv4) and `2510:a1:...` (IPv6).
* **Data Filtering:** Used extended grep to isolate relevant entries:
`grep -E "^92\.7\.|^2510:a1:" logfile > filtered_logs.txt`
* **Automation:** Developed a Python script to parse the filtered IPs. I used regex to extract:
1. The **Position Index** (from the 3rd octet in IPv4 or 2nd hextet in IPv6).
2. The **Flag Character** (from the final segment of the IP).


* **Reconstruction:** The script converted hex indices to integers, sorted the characters by their position, and joined them to reconstruct the full flag.

---

### 6. Synthetic Stacks

**Category:** Forensics / Stego

**Approach:**

* **File Identification:** Although the file had a `.png` extension, the `file` command identified it as 7-zip archive data.
* **Cracking:** Renamed it to `.7z`. The archive was password-protected. I used `7z2john` to generate a hash and then used **John the Ripper** to brute-force it. The password was cracked as `superman`.
* **Decoding:** Extraction yielded `hq.txt`, which contained a long Base64 string. The header `iVBORw0KGgo...` indicated it was actually a PNG image.
* **Final Step:** Decoded the string back to an image: `base64 -d hq.txt > image.png`. The resulting image was a QR code, which I scanned to get the flag.

---

### 7. Know Meh Better

**Category:** Reverse Engineering

**Approach:**

* **Unpacking:** The challenge provided an encrypted text file and a `know_meh_better.exe`. I used `pyinstxtractor` to unpack the PyInstaller archive.
* **Decompilation:** Used `uncompyle6` to decompile the extracted `.pyc` bytecode into readable Python source code.
* **Logic Analysis:** The source revealed the flag was:
1. Base64 encoded.
2. XOR-encrypted using the docstring of the Python `len` function (`len.__doc__`) as the key.
3. Converted to a hex string.


* **Solver Script:** Wrote a Python script to reverse these steps: hex → bytes → XOR decrypt with `len.__doc__` → Base64 decode.

---

### 8. Refresh Ritual

**Category:** Web Automation

**Approach:**

* **Observation:** The webpage had a dynamic password hidden in the "Hint" attribute of an HTML input. However, there was a strict 3-second timeout for submission, making manual entry impossible.
* **Automation:** Developed a Bash script using `curl` to:
1. Fetch the page and save the session cookie (`-c cookies.txt`).
2. Use `sed` to parse the dynamic password from the HTML.
3. Immediately send a POST request with the password and the stored cookie (`-b cookies.txt`).


* **Result:** The script successfully beat the timer and retrieved the flag from the server response.

---

### 9. Unfair Flip

**Category:** Web / Logic Exploitation

**Approach:**

* **Code Review:** Inspected the client-side JavaScript to find a `generateProof` function. This function calculated a checksum based on the results of a coin flip game.
* **Logic Forgery:** Instead of playing the game, I reimplemented the hashing logic in Python. I calculated the valid proof integer (4071233025) required for a "winning" state of three Heads (`["H", "H", "H"]`).
* **Exploitation:** Sent a `curl` POST request to the `/get_flag` endpoint with the forged JSON payload: `{"coins": ["H", "H", "H"], "proof": 4071233025}`. This bypassed the game logic and returned the flag.

---

### 10. Fast and Rebound

**Category:** Web / DNS Rebinding

**Approach:**

* **Identification:** The "NeonPix" application had a `/fetch_image` endpoint that blocked `localhost` and `127.0.0.1`.
* **Bypass:** I used **DNS Rebinding** via the domain `localtest.me`. This domain passes string filters but resolves to `127.0.0.1` on the backend.
* **Execution:** Wrote a Python script to send a POST request targeting the internal service: `{"url": "http://localtest.me:8080/flag"}`. The backend fetched the data from itself, revealing the flag.

---

### 11. Catch Me if You Bot

**Category:** Web Recon

**Approach:**

* **Reconnaissance:** Inspected `robots.txt` and discovered a required User-Agent: `HQBOT`.
* **Discovery:** Masquerading as `HQBOT`, I fetched `sitemap.xml`, which revealed a hidden developer page (`devl0per-*.html`).
* **Race Condition:** Accessing the developer page triggered a meta-refresh redirect. I had to immediately follow the time-sensitive URL to `/dev-website/` to capture the flag before the link expired.

---

## Conclusion

TCS HackQuest Season 10 was a fantastic learning experience. While the technical glitches toward the end were a bit frustrating, solving 11 challenges provided a great opportunity to apply various tools and scripting techniques.
