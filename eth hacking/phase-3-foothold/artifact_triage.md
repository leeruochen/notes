# Artifact Triage (files, archives, keys, databases, secrets)

You found a file that might hold credentials: an archive, a database, a key, a browser store, a document, a mail file, or an app config. This note is the general workflow for *any* such artifact, so you don't need a per-product recipe. It replaces the long cracking sections other notes used to duplicate.

## Goal
Turn an unfamiliar credential-bearing file into usable identities/secrets, or a confident "nothing here", by identifying its format, extracting content safely, and only cracking when cracking is actually the right move.

## Open this when
- FTP/SMB/anon share yielded files ([[21_ftp]] · [[00_smb_enum]])
- Post-foothold file hunting surfaced configs/keys/archives ([[08_interesting_files]] Linux · [[05_stored_credentials]] Windows)

---

# Step 1: Copy, record provenance, hash
Work on a **copy**, never the original. Note where it came from and its hash (integrity + report evidence).
```bash
mkdir -p ~/loot && cp <artifact> ~/loot/ && cd ~/loot   # create the working dir BEFORE copying
sha256sum <artifact>            # provenance + integrity
```

# Step 2: Identify by content, not extension
Extensions lie. Ask the bytes what it is.
```bash
file <artifact>                 # magic-based type
strings -n 8 <artifact> | less  # readable text, hints, embedded creds
xxd <artifact> | head           # magic bytes when file is unsure
```
- Recognisable magic (PK = zip/office, `-----BEGIN` = PEM key, `SQLite format 3` = sqlite, `7z`/`Rar!`, etc.) tells you the real format → pick the right tool in Step 4/5.
- A `.txt`/`.dat`/`.bak` that's actually an archive or DB is common, so trust `file`.

# Step 3: Archives, list before extracting and preserve paths
```bash
unzip -l <a>.zip ; 7z l <a>.7z ; tar tzvf <a>.tgz     # LIST first (see structure/paths)
7z x <a>.7z -o<dir>                                    # extract preserving paths
```
- Directory/file **names** are intelligence (usernames, hostnames, app names), so record them before opening contents.
- Password-protected archive → that's a *hash to crack* (Step 6), not a wall.

# Step 4: Inspect metadata & structured formats
```bash
exiftool <doc/image>            # author, user, software, paths in metadata
cat *.conf *.ini *.xml *.json *.yml 2>/dev/null   # app configs: creds, hosts, connection strings
```
Different structured formats hide secrets differently: config files (plaintext/encoded), databases (hashed user tables, connection strings), office docs (embedded creds/macros/metadata), mail files (bodies, headers, attachments), browser stores (saved logins, often OS-protected).

### SQLite: `.db`, `.sqlite`, `.sqlite3`
Identify the copied/hashed artifact first. Extensions lie.

```bash
file <artifact>
```

If SQLite, inspect the schema before choosing a query:

```bash
sqlite3 <artifact> '.tables'
sqlite3 <artifact> '.schema'
sqlite3 <artifact> '.schema <table>'
sqlite3 <artifact> "PRAGMA table_info('<table>');"
sqlite3 -header -column <artifact> 'SELECT * FROM "<table>" LIMIT 50;'
```

Look for users, passwords/hashes, tokens, notes, hosts/URLs, connection strings, and filesystem paths. Do not assume table or column names. Keep fields from the same row/record together.

Need broader inspection?

```bash
sqlite3 <artifact> '.dump' | less
```

- live database -> collect matching `<artifact>-wal` / `<artifact>-shm` with provenance;
- not SQLite, encrypted, locked, or inconsistent -> return to Step 2 and verify the copy/sidecars instead of guessing;
- exact identity/secret pair -> [[01_credentials_and_spraying#Paired users/passwords or users/hashes: one-to-one validation, `--no-bruteforce`, Cartesian warning|paired validation]];
- unlabelled value -> candidate only, never a Cartesian spray.

### PCAP / PCAPNG: `.pcap`, `.pcapng`, Wireshark, TShark, Follow TCP Stream

Identify and inventory the copied capture first:

```bash
file <capture>
capinfos <capture>
tshark -r <capture> -q -z endpoints,ip
tshark -r <capture> -q -z conv,tcp
```

Start from observed protocols and conversations, then filter. Do not guess that traffic is HTTP.

```bash
tshark -r <capture> -Y '<display_filter>' -T fields -e frame.number -e ip.src -e ip.dst -e tcp.stream
tshark -r <capture> -q -z follow,tcp,ascii,<stream_number>
tshark -r <capture> -Y 'http.request' -V
```

Wireshark equivalent: apply a display filter, select a relevant packet, then **Follow -> TCP Stream**.

- preserve each request/response or stream with its hosts, user, cookie, token, password, hash, or challenge-response;
- exact pairs route to [[01_credentials_and_spraying#Paired users/passwords or users/hashes: one-to-one validation, `--no-bruteforce`, Cartesian warning|paired validation]]; an unlabelled value remains a candidate, not permission to spray;
- TLS usually hides application data unless matching session keys are available;
- exported files return to Step 1 for provenance, hashing, and identification;
- empty filter -> wrong protocol/filter or no matching packets; handshake-only -> no captured application data; incomplete/corrupt stream -> missing segments, truncation, or capture loss; unreadable payload -> encryption, compression, encoding, or a non-text protocol. Re-copy a corrupt capture and verify its hash before further analysis.

# Step 5: Classify the secret *before* touching a cracker
The single most important decision: what *kind* of protection is this?

| Class | Looks like | Right move |
|---|---|---|
| **Plaintext** | `password=Secret1` | just use it |
| **Encoded** | base64/hex/URL (`cGFzcw==`) | **decode**, this is reversible, not cracking |
| **Obfuscated** | ROT/xor/custom, source often nearby | reverse the transform (read the code) |
| **Reversibly encrypted** | a known scheme with a recoverable/known key (GPP, some app "encryption") | **decrypt** with the documented key/tool, not cracking |
| **Encrypted (unknown key)** | ciphertext, key = a password | crack the *password* if a hash form exists (Step 6) |
| **Hashed** | `$2y$…`, `5f4dcc3b…`, NTLM | crack the hash (Step 6) |

> [!warning] Don't crack what you can decode
> Base64/hex are **encodings**, not encryption: `echo '<val>' | base64 -d`. GPP `cpassword` and many app secrets use a **published key**; decrypt them, don't feed them to hashcat. Cracking a value that's merely encoded/reversibly protected wastes your timebox.

# Step 6: Only now, convert → INSPECT → pick cracker → crack → verify
When (and only when) you have a genuine hash / password-protected artifact:

**1. Convert with the right `*2john` (discovery, not memorisation):**
```bash
ls /usr/share/john/*2john* ; locate 2john        # zip2john, ssh2john, keepass2john, office2john, pdf2john, ...
<format>2john <artifact> > hash.txt              # e.g. zip2john secret.zip > hash.txt
```

**2. INSPECT the converter output before choosing a cracker; don't pipe it blind:**
```bash
cat hash.txt            # what does the record actually look like?
```
`*2john` emits **John-format** records, which usually carry a leading label: `secret.zip:$zip2$*...` (a **filename** prefix) or `user:$sshng$...`. That label is fine for John and **breaks Hashcat**. Two separate, source-documented paths:

- **John path (accepts the label as-is):**
  ```bash
  john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt   # John autodetects the format
  john --show hash.txt                                         # recover cracked results
  ```
  John jumbo docs list every supported format/converter; treat them as the authority for John.

- **Hashcat path (wants the hash TOKEN only):** Hashcat expects just the hash, **not** the `filename:`/`user:` prefix and not extra John fields. Do **not** run a generic extractor over the record: `grep -oE '\$[^:]+'` / `cut` truncate any hash that legitimately contains `:` (NetNTLMv2, sshng, and others). Instead:
  ```bash
  hashcat --example-hashes | less                 # or https://hashcat.net/wiki/doku.php?id=example_hashes
  ```
  - Identify the mode's **exact** expected format from `--example-hashes` / the wiki *first*.
  - Preserve the record verbatim; remove a leading `filename:`/`user:` label **only if** that mode's example omits it, and keep every **internal** delimiter that belongs to the hash.
  - Put the exact token in its **own file** (avoids shell-quoting damage to `$...$`); quote a literal token only if you must pass it inline:
  ```bash
  hashcat -m <mode> hash_token.txt /usr/share/wordlists/rockyou.txt   # token in a file = safest
  ```

> [!warning] Never edit the hash payload; only strip a label the format requires
> Remove a `filename:`/`user:` prefix or an extra John field **only when the target cracker's documented example format doesn't include it**. Do not alter the `$...$` bytes themselves. If unsure, use the **John path** (it takes the record verbatim).

**Failure interpretation (Hashcat rejecting the line):**
- `Separator unmatched` / `Token length exception` → the line still has a label/extra fields, or the `-m` doesn't match the token's structure → re-inspect against `--example-hashes` and strip only the label.
- `No hashes loaded` → wrong mode, or the token isn't what that mode expects → switch `-m`, or fall back to the John path.

**Verify the recovered secret for real (unchanged):**
```bash
7z t -p'<cracked>' <a>.7z          # opens? then it's correct
ssh -i <key> <user>@<host>         # one controlled auth attempt, not a spray
```

### KeePass/KeePassXC: after cracking
Reuse the Step 6 `keepass2john`/cracking workflow. After recovering the master password, open the copied KDBX and inspect labelled entries:

```bash
keepassxc-cli open <database>.kdbx
# Inside the interactive prompt:
ls
show -s '<entry>'
```

- key file/hardware challenge may still be required;
- preserve title, URL/host, username, password, notes, and custom attributes as one record;
- avoid bulk plaintext export; protect and remove any export when finished;
- validate exact pairs via [[01_credentials_and_spraying#Paired users/passwords or users/hashes: one-to-one validation, `--no-bruteforce`, Cartesian warning|paired validation]];
- unlabelled password -> candidate only, not permission to spray.

# Step 7: Feed the loop
- **Usernames:** extract separately (filenames, doc authors, DB user tables, mail headers) → users list.
- **Secrets:** every recovered credential → the credential/reuse matrix ([[99_STUCK_RESET]] reuse matrix · [[01_credentials_and_spraying]] in a domain), tested on plausible services only.

# Common false positives / stop conditions
- A "hash" that's actually base64 of plaintext → decode, don't crack.
- A key file with no matching service reachable → record it, move on.
- Cracking a strong hash with no wordlist hint and no time budget → stop; look for the plaintext elsewhere first.

> [!important] Helpline
> Copy + hash → identify by magic → list archives → classify the secret (plaintext/encoded/encrypted/hashed) **before** cracking. Decode what's reversible. Convert with the right `*2john`, then **inspect the record**: John takes the `name:hash` line as-is; Hashcat wants the `$...$` token only (strip the label, never the hash bytes) and its `-m` must match `--example-hashes`. Verify by *opening the artifact*. Usernames and secrets both feed the matrix.
