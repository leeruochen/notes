# CTF Username Enumeration — Notes

## 1. Generating username wordlists

### Basic numbered pattern (e.g. `staff029`)
```bash
seq -f "user%03g" 0 999 > users.txt
```

### Combined prefixes with 3-digit padding
```bash
for p in admin user staff employee emp manager guest test dev support member client customer; do
  seq -f "${p}%03g" 0 999
done > combined_users.txt
```

### Variations to generate alongside (numbering format isn't always consistent)
```bash
# No padding: user1, user2 ... user999
for p in admin user staff employee; do
  seq -f "${p}%g" 0 999
done > nopad_users.txt

# 2-digit padding: staff01 ... staff99
for p in admin user staff employee; do
  seq -f "${p}%02g" 0 99
done > pad2_users.txt

# Separator variants: staff_029, staff.029
for p in admin user staff employee; do
  seq -f "${p}_%03g" 0 999
  seq -f "${p}.%03g" 0 999
done > sep_users.txt
```

### Merge and dedupe
```bash
cat combined_users.txt nopad_users.txt pad2_users.txt sep_users.txt | sort -u > final_users.txt
wc -l final_users.txt
```

**Tips:**
- Start with the 3-digit padded set (`combined_users.txt`) first, it's the most common CTF convention. Expand only if it doesn't land.
- If you already found one valid username (e.g. `staff029`), check if numbering starts predictably (`staff000`/`staff001`) and whether the range hints at something in scope (founding year, employee count, etc).
- Always check the challenge itself for leaked hints before brute-forcing blind: page source comments, sequential IDs in URLs, email formats in documents, etc.

## 2. Basic Auth fuzzing (username in URL userinfo)

```bash
ffuf -w users.txt -u "http://FUZZ:x@$IP/dir/login.php" -mc 200,401,403 -fs <common_size> -t 50
```

- `-mc all` alone is too loose, pair it with explicit status codes.
- If the page reflects the submitted username back in an error message, response **size will vary per username** and `-fs` becomes unreliable. Use `-fw` (word count) or `-fr` (regex on a fixed error string) instead:
```bash
ffuf -w users.txt -u "http://FUZZ:x@$IP/dir/login.php" -mc 200,401,403 -fr "invalid username" -t 50
```
- Sanity-check request construction first with a known-good username and `-v` before burning the full wordlist:
```bash
ffuf -w <(echo "testuser") -u "http://FUZZ:x@$IP/dir/login.php" -mc all -v
```
- Watch for account lockout. Add `-p <delay>` between requests if the target rate-limits.

## 3. POST form fuzzing (differentiate wrong user vs wrong password)

A wrong username and a wrong password often answer differently, via status code, response size, or wording. Use that difference as the filter.

```bash
ffuf -X POST -d "username=FUZZ&password=x" -u $URL/login \
  -w /usr/share/seclists/Usernames/cirt-default-usernames.txt \
  -mc 401
```

Use `-fs <size>` or `-fc <code>` to hide the common "wrong password" answer instead, when that's the easier signal to filter on.

**What to change for your own test:**
- Swap the wordlist path (`-w`) for your generated numbered list (`final_users.txt` or `combined_users.txt`) instead of the default SecLists file.
- Replace `-u $URL/login` with your actual target URL and set `$URL`/`$IP` as env vars beforehand.
- Confirm the POST field names (`username=`/`password=`) match the actual form field names, check the HTML source or intercept one request in Burp/browser devtools first.
- Pick `-mc` vs `-fc`/`-fs` based on which gives a cleaner signal:
  - If invalid logins return a distinct status code (e.g. 401 for bad user, 200 for bad password) → filter/match on `-mc`/`-fc`.
  - If status codes are identical but response size differs → use `-fs`.
  - If both are identical but wording differs → use `-fr` with a regex on the distinguishing text.
- Add `-t <threads>` and consider `-p <delay>` if you hit rate limiting or lockout.
