## Goal
Fingerprint the stack, detect a WAF, scrape a custom wordlist, and brute directories/files with the RIGHT extensions. Web is the #1 OSCP entry vector — enumerate it hard before concluding there's nothing.

**Checklist**
- [ ] tech stack + WAF fingerprinted?
- [ ] source, comments, robots.txt, sitemap, /.git checked?
- [ ] custom wordlist scraped from the site (cewl)?
- [ ] dir brute with extensions matching the stack?
- [ ] second pass (bigger list + backups)? recursed? vhosts fuzzed?

# Commands

### Step 1: Fingerprint + WAF
```bash
whatweb http://$IP
wafw00f http://$IP                                          # detect a WAF (changes your tooling)
curl -sIk http://$IP/ | grep -iE "server|x-powered|set-cookie|location"
curl -s  http://$IP/ | grep -iE "generator|powered|version|framework"
nmap --script http-enum,http-headers,http-methods -p80,443,8080,8443 $IP
# by hand: view-source, /robots.txt, /sitemap.xml, HTML comments, linked JS files
```

### Step 2: Exposed .git / source leak
```bash
curl -s http://$IP/.git/HEAD                     # "ref: refs/heads/.." = repo exposed
git-dumper http://$IP/.git/ ./src                # dump whole repo -> read source + creds
# no git-dumper? GitTools (gitdumper.sh then extractor.sh)
```

### Step 3: Scrape a custom wordlist (cewl)
```bash
cewl http://$IP -m 5 -w cewl_words.txt                     # site words -> password brute
cewl http://$IP -e --email_file emails.txt                 # harvest emails / usernames
cewl http://$IP -d 3 -m 6 --with-numbers -w cewl_big.txt   # deeper crawl
# use it:  hydra ... -P cewl_words.txt   |   feroxbuster ... -w cewl_words.txt

http://$ip/joomla/administrator/ -d 4 -m 5 -w $PWD/cewl2.txt #example used 
```

### Step 4: Directory brute (feroxbuster — recurses by default)
```bash
feroxbuster -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,txt,bak,zip,xml,json,conf,asp,aspx,jsp -t 50
feroxbuster -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/big.txt -x php,txt,bak -t 50  # 2nd pass
```

### Step 5: Alternates + wildcard handling
```bash
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,txt,bak -t 50 -b 404,403
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -u http://$IP/FUZZ -e .php,.html,.bak,.txt -fc 404
ffuf -w <list> -u http://$IP/FUZZ -fs <default_size>       # 200-for-everything? filter by size
nikto -h http://$IP
```

### Helpline
```bash
# WHAT TO LOOK FOR:
#   /.git/HEAD returns "ref: refs/..." = full SOURCE dump available (git-dumper) -> creds in code/history.
#   robots.txt Disallow lines = paths the admin hid -> visit every one.
#   backup files (.bak .old .zip .swp ~) = leaked SOURCE -> grep for passwords/keys/DB strings.
#   a WAF (wafw00f) -> slow down, avoid loud sqlmap, URL-encode payloads.
# EXTENSIONS matter more than the tool — match the stack: php | asp,aspx | jsp | py.
#   feroxbuster/gobuster need -x ; dirsearch needs -f to append extensions to seclists lists.
# EMPTY result = wrong wordlist/ext, NOT "nothing here" -> escalate common -> medium -> big.
# CEWL builds a target-specific wordlist from the site's own words — beats rockyou on custom passwords.
# find hidden PARAMETERS: ffuf -u "http://$IP/page?FUZZ=1" -w .../burp-parameter-names.txt -fs <size>
```

