## Goal
Walk the MIB with a community string (usually `public`) to dump users, processes (incl command-line passwords!), software, and ports. SNMP (UDP 161) is a goldmine and easy to miss.

**Checklist**
- [ ] scanned UDP? (SNMP hides from TCP scans)
- [ ] community string found? (public, private, manager, community)
- [ ] full walk done?
- [ ] process command-lines checked for passwords?
- [ ] MIBs installed for readable output?

# Commands

### Step 1: Find the community string (UDP!)
```bash
sudo nmap -sU -p161 --script snmp-brute $IP
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt $IP
# common: public, private, manager, community
```

### Step 2: Full walk + formatted
```bash
snmpwalk -v 2c -c public $IP           # everything (messy but thorough)
snmpbulkwalk -v 2c -c public $IP .     # faster for big walks (note the final dot)
snmp-check -c public $IP               # nicely formatted
```

### Step 3: Targeted OIDs (Windows)
```bash
snmpwalk -v 2c -c public $IP 1.3.6.1.2.1.25.4.2.1.2    # running processes
snmpwalk -v 2c -c public $IP 1.3.6.1.2.1.25.4.2.1.5    # process CMD LINES (passwords in argv!)
snmpwalk -v 2c -c public $IP 1.3.6.1.4.1.77.1.2.25     # user accounts
snmpwalk -v 2c -c public $IP 1.3.6.1.2.1.25.6.3.1.2    # installed software
snmpwalk -v 2c -c public $IP 1.3.6.1.2.1.6.13.1.3      # TCP listening ports
snmpwalk -v 2c -c public $IP 1.3.6.1.2.1.2.2.1.2       # network interfaces
snmpwalk -v 2c -c public $IP 1.3.6.1.2.1.1             # system info
```

### Step 4: Install MIBs for readable names
```bash
sudo apt install snmp-mibs-downloader
sudo sed -i 's/^mibs :/#mibs :/' /etc/snmp/snmp.conf
snmpwalk -v 2c -c public $IP hrSWRunName          # process names
snmpwalk -v 2c -c public $IP hrSWRunParameters    # process args (creds!)
```

### Step 5: NET-SNMP extend (info disclosure vs RCE — don't conflate)
NET-SNMP can be configured to run local commands and expose their output over SNMP ("extend"). In a full walk, look for the extend subtree / these names:
```bash
snmpwalk -v 2c -c <community> $IP 1.3.6.1.4.1.8072.1.3.2   # nsExtendObjects subtree
# readable names (MIBs installed): NET-SNMP-EXTEND-MIB::
#   nsExtendCommand   = the binary/script an extend entry runs
#   nsExtendArgs      = its arguments
#   nsExtendOutputFull/nsExtendOutput1Line = the command's OUTPUT (already executed on the host)
```
**Decision gate — two very different findings:**
1. **Read-only community + existing extend entries** = *information disclosure*. You can read what admins configured to run and its output (may leak internal data, paths, results) — but you **cannot add** your own commands. This is a clue, not RCE.
2. **Confirmed read-write community** = you may create a *new* extend entry that runs an arbitrary command = RCE.

**Prove write access first, then (and only then) use `snmpset`:**
```bash
# is the community actually read-write? test a benign, reversible write before anything else.
snmpset -v 2c -c <community> $IP <writable_oid> <type> <value>   # succeeds -> RW; "notWritable"/timeout -> RO
# only after RW is proven: define + trigger an extend command (understand each OID's TYPE: 's' string, 'i' int)
```
> [!warning] Read ≠ write ≠ RCE
> Seeing extend objects in a walk means the feature is *configured*, not that *you* can write to it. A read community lets you read output; it does not let you add commands. Don't fire `snmpset` blind — confirm RW, and know the object type you're setting.

Primary reference: [NET-SNMP extend documentation](https://net-snmp.sourceforge.io/docs/man/snmpd.conf.html) (the `extend`/`nsExtendObjects` behaviour).

### Helpline
```bash
# COMMUNITY STRING = access level: "public" works = READ (walk everything). "private" works = READ-WRITE
#   (rare -> possible RCE via NET-SNMP extend / snmpset).
# THE HIGH-VALUE OIDs (what each returns):
#   hrSWRunParameters 1.3.6.1.2.1.25.4.2.1.5 = process COMMAND-LINE ARGS -> passwords leak here
#     (mysqld -p<pass>, scripts with hardcoded creds) -> reuse everywhere. THIS is the SNMP jackpot.
#   hrSWRunName ..4.2.1.2 = process names ; 77.1.2.25 = Windows users ; 25.6.3.1.2 = installed software ;
#     6.13.1.3 = TCP listening ports (internal services to port-forward later).
# WHAT TO DO: user list -> Kerberos/spray. internal web path revealed -> hit it directly (HTB Pit).
#   IPSec PSK leaked -> VPN into the internal net (HTB Conceal).
# ALWAYS scan UDP (sudo nmap -sU -p161) — SNMP is invisible to TCP scans, the most-missed OSCP port.
# install MIBs (snmp-mibs-downloader; comment "mibs :") so OIDs show readable names.
# PROCESS LIST = TWO wins, not one:
#   (1) argv passwords (above).  (2) process NAMES of background/helper software with NO
#       listening port (mail milters, AV, backup agents) -> searchsploit each name.
#       This is how you find a vulnerable service the TCP scan never shows.
#       (PG ClamAV: SNMP named clamav-milter -> CVE-2007-4560 -> root; invisible on nmap.)
```
