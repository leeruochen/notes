# Guide to the OSCP Notes ("HolyGrail" Field Manual)

*Complete — all five phases, every cross-cutting utility note, and `sqli.md`. `sqli-draft.md` was an earlier version of the same content and isn't separately mapped.*

---

## 1. What this vault actually is, and what it's for

This is **not a study guide** and it's not trying to teach you what SQL injection or Kerberos *is*. It assumes you already did that learning elsewhere (a course, HackTricks, PayloadsAllTheThings). What it's built for instead is the specific failure mode that sinks people on the actual exam: **you know the concepts, but under time pressure you either fire an exploit before checking it actually applies, or you fool yourself about how much access you actually have.**

Every note in this vault is fighting one of those two failure modes:

1. **"Don't act until the prerequisites are confirmed."** Version numbers alone don't justify running an exploit; a writable file only matters if a privileged process consumes it; a BloodHound edge is not a shell. Notes repeatedly build an explicit prerequisite table or checklist and refuse to let you skip to the payload.
2. **"Don't overclaim what you just gained."** Code-executed ≠ stable shell ≠ privileged identity. Auth-valid ≠ service-authorized ≠ local admin. Local-admin-on-one-host ≠ domain privilege. This exact pattern — name the adjacent states, forbid collapsing them — recurs in Foothold, both AD notes, both privesc checklists, and the BloodHound note's "capability ledger."

So the real content of this vault isn't the commands (you likely know most of them already) — it's the **discipline layered on top of the commands**. That's what's worth internalizing, not memorizing verbatim.

## 2. What you should expect to get out of reading it

Not: the ability to recite every command from memory.

Instead, by the time you've been through this guide and the notes it maps, you should be able to:

- **Locate yourself in the loop instantly.** Given any situation mid-box ("I have a shell," "I found a password," "BloodHound shows an edge"), know immediately which note answers "what do I do next" — without searching.
- **Recognize the same discipline wearing different clothes.** The "confirm prerequisites → act → verify → don't overclaim" shape shows up at every layer (see §5). Once you see it once, you'll recognize it everywhere else in the vault, which makes new notes fast to read.
- **Know which notes are load-bearing vs. reference.** A handful of orchestrator notes (§4) are where you actually make decisions; the rest are leaves you get routed to. You don't need to have the leaves memorized — you need to know the orchestrators well enough to get routed correctly.
- **Catch yourself before the two failure modes above.** That's the actual exam-day payoff — not knowing more exploits, but not burning 45 minutes on a plausible-looking dead end because you skipped a prerequisite check.

---

## 3. The complete phase map

```
1. Active Recon         → scope, host discovery
2. Enumeration          → per-port routing + the web/SMB sub-loops (the biggest phase)
3. Foothold             → turn a confirmed primitive into a stable, understood shell
4. Privilege Escalation → local Linux / Windows escalation
5. Active Directory     → the multi-host domain loop
        ↓
   (new host / credential / identity found → back to step 2)
```

The loop is genuinely cyclic, not just narratively — nearly every phase-ending note explicitly hands you back to enumeration or to the AD loop once you gain something new. This is why the vault doesn't read start-to-finish: it's a graph, entered from whatever state you're actually in.

---

## 4. The orchestrator notes — the ones that actually make decisions

Every phase has one (sometimes several) notes shaped the same way: a numbered sweep or ordered checklist, plus a **finding → interpretation → route** table that sends you to a leaf note. These are the notes worth *knowing well*, not just knowing exist:

| Orchestrator | Phase | Shape |
|---|---|---|
| `00_INDEX.md` | top-level | phase router — the map of maps |
| `00_methodology.md` | 2 (Enum) | full scan → port-number → note lookup table |
| `00_WEB_METHODOLOGY.md` | 2 (Enum, web) | 5 ordered passes (Surface → Features → Components → CVE filter → Timebox), each with its own routing table |
| `00_smb_enum.md` | 2 (Enum, SMB) | fingerprint → route to anonymous/authenticated/vuln-specific leaf |
| `00_FOOTHOLD.md` | 3 | primitive → shell → stabilize → route to privesc, in six steps |
| `00_master_checklist.md` (Linux) | 4 | 12-item cheapest-first checklist |
| `00_master_checklist.md` (Windows) | 4 | numbered sweep + finding→route table |
| `00_AD_METHODOLOGY.md` | 5 | the domain-wide loop, 8 steps, with a capability ledger |
| `00_AD_SERVICES.md` | 5 | AD-specific port index (some files shared with Phase 2's router — see §6) |

Everything else in the vault is a **leaf**: a focused note that one of these orchestrators sends you to for one specific technique or service. You don't navigate to a leaf note cold — you always arrive there from an orchestrator's routing table.

---

## 5. The recurring architecture — same shape, every layer

This is the single most useful thing to notice about the whole vault: **the same structural pattern repeats fractally**, from the top-level Index down to a single privesc technique.

**Pattern:** orchestrator note (numbered steps or passes) → a "finding → interpretation → route" table → leaf notes, each of which repeats: state a **gate/prerequisite** → give the **abuse/command** → require **verification** (never trust the tool's own success message — always independently confirm, usually with `whoami`/`id`) → name the **dead-end/pivot condition** if it doesn't pan out.

You can see this exact shape at every scale:
- Whole engagement: `00_INDEX.md` → five phases
- One phase: `00_WEB_METHODOLOGY.md`'s five passes, or `00_AD_METHODOLOGY.md`'s eight steps
- One service: `00_smb_enum.md` → anonymous/authenticated/attacks
- One technique: any single Exploit Case in the Linux/Windows privesc notes (gate → abuse → verify → dead-end)

Once this clicks, reading a brand-new note in this vault takes a fraction of the time, because you're not learning a new structure each time — you're just filling in the same four slots (gate, abuse, verify, dead-end) with new specifics.

### Two "voices," cutting across phases (not confined to one)

Correcting something I said earlier in this guide: the different writing styles aren't really phase-bound. Looking at the complete vault, there are two voices used throughout, chosen by **note type**, not phase:

- **Orchestrator/decision-tree voice** — `00_FOOTHOLD`, `00_AD_METHODOLOGY` + its sub-notes, `00_WEB_METHODOLOGY`, the Windows privesc leaves, `artifact_triage`, `pivoting_and_tunnelling`, `internal_port_enum`, `public_exploit_research`, `unknown_service_triage`. Structured around prerequisites, verify/fail tables, numbered helpline *facts*.
- **Cheat-sheet/lookup voice** — the port notes (`21_ftp`, `88_kerberos`, `1433_mssql`, etc.), the Linux privesc leaves, and the web feature notes (`injection`, `lfi_rfi`, `file_upload`, `cms`, `auth_bypass`, `default_creds_logins`, `web_apps_panels`, `vhost_fuzzing`), plus the SMB leaves. Built for fast mid-engagement lookup: Goal → Checklist → numbered Commands → one **Helpline block** written as "if you see X, it means Y, do Z."

Both voices carry the same discipline (§1) — they just present it differently depending on whether the note is a *decision to make* or a *technique to look up*.

### Three confidence-tagging systems (different scales, same instinct)

- Phase 4 Windows: `[BT]` battle-tested vs `[T]` taught-not-yet-proven
- Phase 5 Kerberos: `[CORE]` exam-fair vs `[PERSISTENCE]` vs `[REFERENCE]` beyond scope
- Implicit everywhere else: "confirmed / probable / guess" confidence columns in evidence tables (`public_exploit_research`, `unknown_service_triage`, `00_WEB_METHODOLOGY`'s component inventory)

None of these are the same scale, but they're all the same instinct: **mark how proven a claim is before you act on it.**

---

## 6. Phase-by-phase map

### Phase 1 — Active Recon
- **`Active_Recon.md`** — thin, intentionally: an nmap TCP/UDP one-liner and a 3-item checklist. Its whole job is to kick you into Phase 2's port router.

### Phase 2 — Enumeration (the biggest phase, four sub-clusters)

**A. The router:** `00_methodology.md` — full scan, then a port-number lookup table. `unknown_service_triage.md` is its explicit fallback for anything the table doesn't cover (odd ports, `tcpwrapped`, unidentified protocols) — separate *port number* from *protocol identity* before assuming anything.

**B. Individual service/port notes** (one file per port, cheat-sheet voice):
`21_ftp` · `22_ssh` · `25_smtp` · `53_dns` · `88_kerberos` (enum-only half — ticket forgery is Phase 5's `04_kerberos_advanced_paths`) · `110_143_pop3_imap` · `111_2049_nfs` · `161_snmp` · `389_ldap` · `873_rsync` · `1433_mssql` · `3306_mysql` · `3389_rdp` · `5432_postgresql` · `5900_vnc` · `5985_winrm` · `6379_redis` · `27017_mongodb` · `misc_ports.md` (TFTP, Finger, r-services, Oracle TNS, Elasticsearch, Docker API, WHOIS, IPMI — rare-but-real ports bundled together rather than one file each).

> **Cross-phase reuse, confirmed now that both indexes are complete:** `53_dns`, `88_kerberos`, `389_ldap`, `1433_mssql`, `3389_rdp`, and `5985_winrm` are the *literal same files* linked from both `00_methodology.md` (Phase 2) and `00_AD_SERVICES.md` (Phase 5). One note, two entry points — "found this port on an unknown box" vs. "already know I'm in a domain." Most of them end by explicitly handing off to `00_AD_METHODOLOGY.md` once a domain is confirmed, which is the actual mechanism behind the whole vault's cyclic claim.

**C. The web sub-loop** (biggest single cluster, its own orchestrator):
```
00_WEB_METHODOLOGY.md  ─── 5 passes: Surface → Features → Components → CVE-filter → Timebox
        │
        ├─► 00_recon_dirbrute.md   (Pass 1 detail: fingerprinting, dir-brute, cewl, .git leaks)
        ├─► vhost_fuzzing.md       (Pass 1 detail: Host-header vhosts)
        ├─► injection.md           (command injection / SSTI / SSRF / XXE)
        ├─► sqli.md               (SQL injection — the biggest single leaf note in the vault, see below)
        ├─► auth_bypass.md         (non-SQL login bypass: strcmp arrays, magic hashes, JWT)
        ├─► lfi_rfi.md             (local/remote file inclusion, log poisoning)
        ├─► file_upload.md         (upload filter bypasses)
        ├─► cms.md                 (WordPress/Drupal/Joomla — identify, scan, exploit)
        ├─► web_apps_panels.md     (Tomcat/Jenkins/phpMyAdmin/Grafana/Gitea/etc.)
        └─► default_creds_logins.md (defaults → reuse → enumeration → justified brute, in that order)
```
Pass 2's "feature → hypothesis" router table is the key artifact here: it's what turns "this input does X" into "therefore check note Y" — reflected input → `injection`, DB-backed behavior → `sqli`, file-path parameter → `lfi_rfi`, upload feature → `file_upload`, admin editor → `cms`, login panel → `default_creds_logins`. This table is doing for the web phase exactly what `00_AD_METHODOLOGY`'s decision table does for AD.

**`sqli.md` deserves its own paragraph — it's the densest leaf note in the entire vault**, dense enough that it's really a mini-methodology of its own rather than a lookup sheet. Two things make it worth reading slowly rather than skimming like the other web leaves:

- **It opens with a hard exam-scope rule that overrides normal habits:** sqlmap is explicitly *banned on OSCP targets* — manual injection is the plan, sqlmap is labs-only for double-checking your manual work. This is the one place in the vault where a tool most people reach for by default is ruled out entirely, so it's worth having internalized before you're mid-exam and tempted to reach for it out of habit.
- **Its core mental model is "3 layers, only one of which moves":** L1 (delivery — the curl/POST template, fixed per box), L2 (wrapper — which SQL trick smuggles your subquery: UNION / error-based / time-based, picked once per box based on channel × DBMS), L3 (subquery — the actual question, the *only* thing that changes as you walk database → tables → columns → rows). Once L1+L2 are locked for a box, every subsequent step is just swapping L3 — which is the same "confirm once, then only the specifics change" instinct as `01_orientation.md`'s identity sweep or `00_AD_METHODOLOGY`'s domain/DNS/time step, just applied to injection instead of a shell.

It also carries its own version of the vault-wide "verify independently, don't trust the surface signal" rule: a `200==200` HTTP status tells you nothing (diff the response body/length instead), a file write via `INTO OUTFILE`/`COPY TO` doesn't mean a URL exists to reach it, and a Postgres `is_superuser` check must be confirmed before assuming `COPY..TO PROGRAM` RCE is available. Same discipline as everywhere else in the vault, expressed in SQL-specific terms.

The included per-DBMS syntax table (MySQL / MSSQL / PostgreSQL side-by-side for version, concat, paging, RCE primitive, etc.) is the kind of reference you'll come back to repeatedly rather than read once — worth bookmarking mentally rather than memorizing.

**D. The SMB sub-loop** (same orchestrator+leaves shape again):
```
00_smb_enum.md ─── fingerprint, then route by what you have
        ├─► anonymous_enum.md      (null/guest sessions, RID cycling)
        ├─► authenticated_enum.md  (one valid cred → shares, spidering, SAM/LSA if admin)
        └─► smb_attacks.md         (EternalBlue, psexec family, GPP cpassword, SCF capture, NTLM relay)
```

### Phase 3 — Foothold
- **`00_FOOTHOLD.md`** — the densest single note in the vault. Core model: **code executed ≠ stable shell ≠ privileged identity** — three states that must never be conflated, checked independently every time.

### Phase 4 — Privilege Escalation (Linux + Windows, fully parallel structures)
```
Linux: 00_master_checklist → 01_low_hanging_fruit … 12_docker_lxd_groups   (12 leaves, cheapest→riskiest)
Windows: 00_master_checklist → 01_orientation … 08_kernel_exploits, 09_uac (side-branch), 99_windows_stuck_reset
```
Windows half carries the `[BT]`/`[T]` confidence legend (§5). Both halves end the same way almost every exploit case does: `chmod +s /bin/bash; /bin/bash -p` (Linux) or `whoami` in the new process (Windows) — recognize the destination pattern, not just the six ways to reach it.

### Phase 5 — Active Directory (now fully mapped)
```
00_AD_SERVICES.md (port index) → 00_AD_METHODOLOGY.md (the loop)
        ├─► 01_credentials_and_spraying.md   (reuse → spray → never brute; policy check first, always)
        ├─► 02_bloodhound_and_acl_paths.md   (graph edge → dispatcher table by target type → verified action)
        ├─► 03_credential_access.md          (SAM vs LSA vs LSASS vs NTDS vs DCSync — five sources, never conflated)
        └─► 04_kerberos_advanced_paths.md    (ticket reuse/forgery: CORE/PERSISTENCE/REFERENCE tiers)
```
`02_bloodhound_and_acl_paths.md`'s central lesson: the *same* edge name (GenericAll, WriteDacl, AddMember) means something different depending on whether the target is a User, Group, Computer, GPO, OU, or Domain — it is a dispatcher, not a single technique.

### Cross-cutting utility notes (referenced from every phase, owned by none)
- **`99_STUCK_RESET.md`** (general/Linux) and **`99_windows_stuck_reset.md`** (Windows) — rabbit-hole circuit breakers. Both teach the same "confirmed / assumed / hypothesis / cheapest test / stop condition" format used implicitly everywhere else.
- **`artifact_triage.md`** — the one place credential-bearing-file handling lives (identify by magic bytes → classify plaintext/encoded/encrypted/hashed → convert → crack → verify). Explicitly built to stop other notes from duplicating cracking steps.
- **`internal_port_enum.md`** → **`pivoting_and_tunnelling.md`** — what you found (loopback listeners) → how to reach it (forward/SOCKS/routed tunnel).
- **`public_exploit_research.md`** — the generic "I have a product + version" validation engine, reused verbatim in spirit by `00_WEB_METHODOLOGY`'s CVE filter and `08_kernel_exploits.md`.
- **`legacy_stack_buffer_overflow.md`** — the one note in the vault explicitly marked out-of-scope for its own subject. It states plainly that PEN-200 itself considers writing a BOF from scratch beyond the current course, and that the exam-relevant skill is *adapting an existing memory-corruption exploit*, not building one. The note is kept anyway, as a from-scratch method (crash → offset → EIP control → bad chars → return address → shellcode → assemble), but deliberately contains **no fixed offsets, addresses, or target-specific constants** — it's the shape of the process, not a copyable exploit. `00_INDEX.md` back in your first batch already flagged this file as "reference-only," and this confirms exactly why.

---

## 7. If you only remember one sentence per phase

- **Recon:** scan everything, both TCP and UDP, before you conclude a port isn't there.
- **Enumeration:** behavior beats banners — confirm what a feature actually does server-side before picking a vulnerability class.
- **Foothold:** say out loud which of the three states (executed / stable / privileged) you actually have.
- **Privesc:** cheapest and safest first, kernel exploits dead last, verify with a fresh `whoami` every time.
- **AD:** one credential is *valid*, not *admin* — prove access per host, and re-enumerate after every new identity.

---

## 8. Known gaps

With `sqli.md` and `legacy_stack_buffer_overflow.md` in, the "live playbook" layer of the vault is fully mapped. `Resources.md` — a flat list of every external link cited across all the notes (PortSwigger, GTFOBins, HackTricks, Impacket, BloodHound docs, Microsoft/MSRC references, etc.) — partially fills in what a `sources/` folder would likely hold, though it's a link list, not the source material itself.

Still genuinely missing:
- **`learning/`** — the teaching-notes layer `00_INDEX.md` names explicitly. Nothing in what's been shared explains *why* a technique works at a conceptual level; every note here assumes that's already known and gives you decision procedures instead. See the next section for what this means practically.
- **`walkthroughs/`** — raw machine notes. `sqli.md`'s closing section references `walkthroughs/capstone-VM-3-working.md` by name, confirming this folder exists with worked full-box examples, but none were shared.
- The remainder of **`sources/`** beyond what `Resources.md` links to, if any exists beyond that link list.

---

## 9. Does the missing content matter, and is this the right resource to learn from?

Direct answer: **the missing folders matter, and if you're new to CTFs, this vault by itself will teach you less than it looks like it should.**

Here's why. Everything you've shared is written in the voice of someone reviewing material they already learned once — it's optimized for *speed of recall under time pressure*, not for *building the concept from zero*. Take `sqli.md` as the clearest example: it never explains what a database is, why quotes matter in SQL, or why `UNION` requires matching column counts from first principles — it assumes you'd nod along to "type context" and "string context" and gives you the fast-lookup version. The `learning/` folder your friend describes separately is almost certainly where that from-zero explanation actually lives — and that's exactly the part you don't have.

`Resources.md` is a genuinely good sign of where to fill that gap, though — it's not just a list of tool repos, it's mostly primary-source teaching material: PortSwigger's SQLi content (their Web Security Academy, which the cheat sheet is drawn from, is one of the best free from-zero SQLi resources that exists), GTFOBins and HackTricks (both reference-heavy but browsable as teaching material), and official Microsoft docs for the Windows/AD-specific pieces. If your friend used these to *write* the learning notes, working through them yourself would get you a lot of the way to reconstructing what that folder probably contains.

Practically, I'd suggest not trying to learn from this vault first. A more honest sequence:

1. **Build the concepts** somewhere designed to teach them from zero — PortSwigger Web Security Academy (free, excellent for web/SQLi/auth-bypass-type content), TryHackMe's beginner paths or HTB Academy modules (broader OSCP-relevant coverage: Linux/Windows fundamentals, AD basics, privesc concepts), and the actual PEN-200 course material if you have access to it (this vault's own `sources/official/pen200/...` reference in the BOF note suggests it's built directly on top of that course).
2. **Practice on easy boxes** (HTB "Starting Point," TryHackMe beginner rooms) actually typing the commands, not just reading about them.
3. **Then** use this vault the way it's actually built to be used: as the fast-decision layer that stops you from skipping a prerequisite check or overclaiming access once you already roughly know what you're doing.

Used in that order, this is a genuinely well-built resource — the discipline in it (§1, §5) is exactly what separates people who pass OSCP from people who know the material but run out of time or trust a false positive. Used as your *first* exposure to these topics, it'll mostly read as a wall of commands with the reasoning left as an exercise.

---

## 10. ICT2212 (Ethical Hacking module) — where it fits, and the actual route

This section covers a real 13-week module (SIT's ICT2212) running alongside this guide, with the stated goal of an A in the module before moving to OSCP-level study. The Lecture 0 slides shared so far are pure admin/orientation (schedule, assessment weights, ethics/legal background, hacker culture, 1337 speak) — **zero technical content yet**, so they don't teach anything HolyGrail assumes. That's expected; it's week 0.

But the module's own syllabus is worth mapping against HolyGrail now, because it tells you exactly how much conceptual ground this course will and won't cover before you'd move on to OSCP material.

### Syllabus → HolyGrail phase map

| ICT2212 week | Topic | HolyGrail equivalent | Depth expected |
|---|---|---|---|
| 1 | Pentest process & methodology | `00_INDEX.md`'s phase loop, at a conceptual level | Intro |
| 2 | Reconnaissance — footprinting, social engineering | Phase 1 (`Active_Recon.md`) | Intro |
| 3 | Scanning and Enumeration | Phase 2's `00_methodology.md` + port notes | Intro |
| 4 | Sniffing | Not really covered in what you've uploaded — HolyGrail is exam/box-focused, not network-traffic-focused | New ground |
| 5 | System Hacking (gaining access) | Phase 3 (`00_FOOTHOLD.md`) + a taste of Phase 4 | Intro |
| 6 | Side-channel attacks | Not in HolyGrail at all | New ground |
| 8 | Wireless Hacking | Not in HolyGrail at all | New ground |
| 9–10 | Web Hacking | Phase 2's web sub-loop (`00_WEB_METHODOLOGY.md`, `sqli.md`, `injection.md`, etc.) | Intro |
| 11 | Maintaining Access, Covering Tracks | Not really covered in HolyGrail (it's an exam-box vault, not a red-team persistence one) | New ground |
| 12 | Advanced Persistent Threats | Not in HolyGrail | New ground |

Takeaway: **roughly half the module (recon, scanning/enum, gaining access, web hacking) directly primes you for HolyGrail's Phases 1–3 and part of Phase 2's web cluster.** The other half (sniffing, side-channel, wireless, maintaining access/covering tracks, APTs) is genuinely new ground that HolyGrail doesn't touch at all — so this module is not redundant with what you'll eventually study for OSCP, it's complementary. Your friend's "introductory compared to OSCP" framing checks out: same subject areas, shallower depth, plus some topics OSCP doesn't cover.

One more thing worth noticing: the module's own project structure (defend your own hardened VM, then attack an assigned target VM, write it up as R1/R2 reports) is a real hands-on cycle through recon → scan/enum → gain access — which is exactly the loop HolyGrail is built around. Doing this project seriously is worth more than any amount of reading.

### The actual route, updated

Given the concrete timeline (13 weeks, A as the goal, OSCP after), here's the sequencing that makes sense:

**Right now through week 13 — ICT2212 is the priority, not HolyGrail.**
- Follow the lecture + lab material as primary; the module's own recommended resources (*Gray Hat Hacking*, *Hacking with Kali*, CVE/NVD) are what your quizzes and CTF will actually test you on — don't substitute HTB Academy or PortSwigger for the module's own content when the two disagree on emphasis, since grading follows the course, not the industry-standard path.
- For the topics that overlap with HolyGrail (recon, scanning/enum, gaining access, web hacking in weeks 2–3, 5, 9–10), it's fine to peek at the matching HolyGrail phase *after* the lecture as a "here's how this looks once condensed into an exam-speed checklist" — but don't lean on it to learn the concept fresh, per §9.
- Use the free-standing resources from the earlier route (PortSwigger Web Security Academy especially) as supplementary depth for weeks 9–10 web hacking, since that's the area both this module and OSCP weight heavily and PortSwigger's material is genuinely excellent for it.
- Take the team project (defender then attacker) seriously as your first real hands-on practice — it's a smaller-scale, supervised version of exactly what OSCP will later demand solo.

**After the module (or in parallel, time permitting) — the topics ICT2212 doesn't cover, if OSCP is still the target:**
- Linux and Windows privilege escalation in depth (HTB Academy modules) — ICT2212's "System Hacking" week almost certainly won't go as deep as HolyGrail's Phase 4
- Active Directory fundamentals (HTB Academy's AD module) — not in ICT2212's syllabus at all, and it's the single largest phase in HolyGrail
- SQLi and broader web exploitation depth beyond week 9–10's coverage (PortSwigger Academy, if not already done)

**Only then — HolyGrail itself**, per §9's reasoning: once the concepts from both ICT2212 and the supplementary self-study above are solid, HolyGrail becomes the fast-decision layer it's built to be rather than an opaque command list.
