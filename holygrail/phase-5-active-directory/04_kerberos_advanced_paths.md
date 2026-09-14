# Kerberos - Ticket Forgery, Reuse & Delegation (scope-gated)

Enumeration-phase Kerberos (AS-REP roasting, Kerberoasting, user enum) lives in [[88_kerberos]]. This note holds the **ticket-based** paths: reuse a ticket you already have, turn a hash into a ticket, or forge a ticket outright. They are ordered by how much of each is actually core OSCP.

> [!important] Scope legend - read before using anything here
> - **[CORE]** - taught in PEN-200 and fair game on the exam.
> - **[PERSISTENCE]** - covered by the course *as persistence*, **not** a default exam escalation. Use only with clear justification; prefer the direct access you already earned.
> - **[REFERENCE]** - beyond core PEN-200 scope. Kept here for understanding and completeness, but **omitted from the primary AD checklist**. Don't force these on the exam.
>
> No technique below appears without a prerequisite table. If you can't tick the prerequisites, the technique does not apply.

---

## Prerequisites (shared by the forgery paths)

Every ticket path below draws from the same small set of inputs. Obtain them here once, then jump back to the technique. The first item gates *everything*: if DNS or time is wrong, the commands below fail with misleading errors that look like tool bugs.

**DNS and time (always first).** Kerberos needs names to resolve to the DC and clock skew under 5 minutes. Fix both before anything else, drivers: [[88_kerberos]] (clock skew) and [[53_dns]] (DNS to the DC).

### Domain SID
The domain's identifier, recorded as `<domain_sid>`. Silver and golden tickets both need it for `-domain-sid`.

```powershell
# On a domain host (victim), via PowerView. Load PowerView first if needed:  . .\PowerView.ps1
Get-DomainSID          # -> S-1-5-21-...   (this exact string is your -domain-sid value)
```
```bash
# From Kali, with any valid domain cred (no host session required):
impacket-lookupsid <domain_fqdn>/<user>:'<password>'@<dc_fqdn> | grep -i "Domain SID"
```
Confirm you have the `S-1-5-21-...` string, then return to the forging command and paste it into `-domain-sid`.

### Service Principal Name
The exact `service/host` string a silver ticket is scoped to. It must match the target service exactly, both **class** and **hostname**, such as `MSSQL/<target_fqdn>`, `CIFS/<target_fqdn>`, or `HTTP/<target_fqdn>`.

```powershell
# On a domain host (victim), via PowerView:
Get-NetUser -SPN | Select samaccountname,serviceprincipalname
```
```bash
# From Kali (this also requests Kerberoast hashes; here you just want the SPN list):
impacket-GetUserSPNs <domain_fqdn>/<user>:'<password>' -dc-ip <dc_ip>
```
Full Kerberoast / SPN workflow: [[88_kerberos]].

### Service account key
The secret of the account that runs the target service (silver ticket), or `krbtgt` (golden ticket). Where it comes from:

- a credential dump (LSASS, SAM/LSA, NTDS, DCSync): [[03_credential_access]];
- cracking a Kerberoast hash: [[88_kerberos]].

### Password to NTLM and NTLM conversion

#### TLDR

Some Kerberos tools require an NTLM key through `-nthash`. Convert a known plaintext password by calculating MD4 over its UTF-16LE bytes. The result is key material, not proof of service authorization.

#### When you see this

You hold a plaintext service-account password, but the selected and justified ticket workflow accepts `-nthash` rather than a plaintext password.

#### Check first

Confirm the target account and tool input. If a recovered AES key is accepted, use that key rather than deriving NTLM. OpenSSL 3 must expose its legacy provider for MD4.

#### If confirmed

Prompt for the password so it is not placed directly in shell history:

```bash
read -rsp 'Password: ' HG_KRB_PASSWORD
printf '%s' "$HG_KRB_PASSWORD" | iconv -f UTF-8 -t UTF-16LE | openssl dgst -provider legacy -md4
unset HG_KRB_PASSWORD
```

#### Expected result

OpenSSL prints a 32-hex-character NTLM value. A provider or algorithm error means the legacy MD4 provider is unavailable; stop rather than substituting an unverified conversion.

#### What did I gain?

`credential or NTLM hash` for the same identity. No TGT, service ticket, authorization, or shell exists yet.

#### Next

Use the key only in the selected ticket workflow, then classify and verify its output with the ticket outcome table below.

### ccache and KRB5CCNAME
Impacket writes the ticket to a `.ccache` file and reads it back from the `KRB5CCNAME` environment variable.

```bash
export KRB5CCNAME=<name>.ccache   # point Kerberos tools at the forged/obtained ticket
klist                             # confirm the ticket: right principal, right service, not expired
```
One extra gotcha for forged tickets: the **SPN's hostname must resolve** to the host you are hitting (add it to `/etc/hosts` if needed), or Kerberos auth fails on a name mismatch even though the ticket itself is valid.

### Ticket outcome table: NTLM, TGT, TGS, service ticket, ccache

| Material                 | What it proves                                                | Scope                                              | Next verification                                                     |
| ------------------------ | ------------------------------------------------------------- | -------------------------------------------------- | --------------------------------------------------------------------- |
| **NTLM hash**            | reusable authentication material for compatible paths         | identity plus protocol constraints                 | one targeted authentication test or overpass workflow                 |
| **TGT**                  | Kerberos identity material issued or forged for one principal | can request permitted service tickets until expiry | `klist`, then request and use only the needed service ticket          |
| **TGS / service ticket** | a ticket for one SPN and service                              | exact service, host naming, principal, and expiry  | use that service and verify authorization                             |
| **ccache**               | a file container for one or more tickets                      | only the tickets actually present                  | set `KRB5CCNAME`, run `klist`, then perform the scoped service action |

---

## [CORE] Overpass-the-hash (hash → TGT)

You have an NTLM hash and the target rejects NTLM, or you specifically need Kerberos. Overpass-the-hash trades the hash for a real Kerberos TGT issued by the KDC, so from then on you authenticate as that user over Kerberos.

| Prerequisite | Value                                                           |
| ------------ | --------------------------------------------------------------- |
| Have         | a user's NTLM hash (source: [[03_credential_access]])           |
| Need         | reachable DC; DNS + time correct → [[88_kerberos]] · [[53_dns]] |
| Result       | a usable TGT for that user, written to a `.ccache`              |

```bash
impacket-getTGT <domain_fqdn>/<user> -hashes :<ntlm_hash>        # -> <user>.ccache
export KRB5CCNAME=<user>.ccache
nxc smb <ip> -k --use-kcache                              # then use Kerberos auth
```

**Verify:** `klist` shows a TGT for the right user, and a `-k` action succeeds.
**Fail:** `KRB_AP_ERR_SKEW` = clock off (fix time); name/lookup errors = DNS to the DC is wrong; `KDC_ERR_PREAUTH_FAILED` = the hash is wrong.

---

## [CORE] Pass-the-ticket (reuse a ccache/ticket)

You already hold a ticket, either a `.ccache` you forged or obtained, or one exported from a host's memory, and you simply present it. No password and no hash: the ticket *is* the credential, valid until it expires.

| Prerequisite | Value                                                                                                                                              |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Have         | a valid `.ccache` (Linux) or exported ticket (Windows), from overpass-the-hash, a silver/golden forge, or a memory dump ([[03_credential_access]]) |
| Need         | ticket not expired; DNS + time correct → [[88_kerberos]] · [[53_dns]]; for a TGS, the service still matches                                        |
| Result       | authenticate as that principal to the ticket's scope (TGT = broad, one TGS = one service)                                                          |

```bash
export KRB5CCNAME=/path/ticket.ccache
klist                                   # confirm principal + validity
impacket-psexec -k -no-pass <domain_fqdn>/<user>@<target_fqdn>
```

**Verify:** the `-k -no-pass` action gives service access as the ticket's user.
**Fail:** expired ticket (re-obtain it); wrong SPN/host for a TGS; `KRB_AP_ERR_SKEW` = time off.

---

## [CORE] Silver ticket (forge a TGS for one service)

When you hold a service account's key, you can forge that service's ticket (a TGS) yourself and present it straight to the service, never contacting the KDC. The forged ticket claims whatever identity you choose, so you appear as any user (typically an admin) *to that one service only*. This is the go-to when you cracked or dumped a service account that has no interactive rights of its own.

| Prerequisite | Value                                                                                                               |
| ------------ | ------------------------------------------------------------------------------------------------------------------- |
| Have | - the service account's **NTLM or AES key** → [[#Service account key\|how to get it]]<br>- the **domain SID** → [[#Domain SID\|how to get it]]<br>- the **exact SPN** of the target service → [[#Service Principal Name\|how to get it]] |
| Need         | the target service accepts the ticket (PAC-validation patching can reject a forged non-existent user); time correct |
| Result       | access to *that one service* on the SPN's host, as any user you name                                                |
| Scope        | one service only, not the whole domain (that would be a golden ticket)                                              |

```bash
impacket-ticketer -nthash <service_ntlm_hash> -domain-sid <domain_sid> -domain <domain_fqdn> -spn <service>/<target_fqdn> <user>
export KRB5CCNAME=<user>.ccache
# then use the ticket against that service, e.g. for an MSSQL SPN:
impacket-mssqlclient -k <host>
```

**Verify:** you can actually *use* the targeted service as the impersonated user (e.g. `impacket-mssqlclient -k <host>` connects and `SELECT SYSTEM_USER` shows the impersonated identity). The forge command "succeeding" only means a file was written, not that the service accepted it.
**Fail:** wrong SPN **class** or **hostname** (must match the service exactly) = `KDC_ERR_*` or access denied; forged non-existent user on a PAC-patched DC = rejected, so use a real account name; the SPN's hostname doesn't resolve = name error (fix `/etc/hosts`).

---

## [PERSISTENCE] Golden ticket (forge TGTs from krbtgt)

Forging with the `krbtgt` key lets you mint TGTs for anyone in that domain. You can only do this after compromising that domain's `krbtgt`, so the normal use is persistence, not a default route to Domain Admin. The narrow within-forest child-to-parent exception is documented below as `[REFERENCE]`.

| Prerequisite | Value |
| --- | --- |
| Have | - the **krbtgt** hash (via DCSync/NTDS → [[03_credential_access]])<br>- the **domain SID** → [[#Domain SID\|how to get it]] |
| Need | an existing account name (post-2022 PAC validation); time correct |
| Result | arbitrary TGTs = domain-wide access, as persistence |

```bash
impacket-ticketer -nthash <krbtgt_ntlm_hash> -domain-sid <domain_sid> -domain <domain_fqdn> <existing_user>
```
Use only when persistence is explicitly justified; otherwise just reuse the DA credentials you already have.

**Verify:** `klist` shows a TGT, and a domain action (e.g. `nxc smb <DC> -k --use-kcache`) works as the chosen user.
**Fail:** wrong SID or krbtgt hash = `KDC_ERR_*`; a forged non-existent user can be rejected by PAC validation, so use a real account name.

---

## [REFERENCE] Within-forest child-to-parent ExtraSid / ExtraSids / SIDHistory

Search aliases: forest trust, child domain, parent domain, extra-sid, ExtraSids, SIDHistory, Enterprise Admins, RID 519.

A forged child-domain TGT can carry a parent-domain group SID in the PAC `ExtraSids` field. In the specific same-forest child-to-parent case, this may make the child principal authorize as an Enterprise Admin in the forest root. This is post-child-domain compromise and outside core PEN-200.

A trust edge is structural, not automatically actionable. Confirm direction, type, transitivity, quarantine, selective authentication, and SID filtering. External, cross-forest, quarantined, or unclear boundaries stop at the Microsoft reference below.

**Prerequisites**
- child and parent are in the same AD forest, confirmed from trust evidence;
- child-domain `krbtgt` AES256 key is controlled;
- child domain SID and parent domain SID are known and kept distinct;
- an existing child-domain username is selected;
- DNS/time and the parent DC FQDN work;
- SID filtering/trust behavior for this exact relationship has been confirmed.

`<parent_domain_sid>-519` is the forest-root Enterprise Admins SID. A bare domain SID identifies a domain, not a principal or group. Do not apply this workflow to external or cross-forest trusts.

```batch
nltest /domain_trusts /v
```

Confirm the parent/child direction, forest relationship, and flags before forging. Then, from Kali:

```bash
impacket-ticketer -aesKey <child_krbtgt_aes256> \
  -domain <child_domain_fqdn> \
  -domain-sid <child_domain_sid> \
  -extra-sid <parent_domain_sid>-519 \
  <existing_child_user>
export KRB5CCNAME=<existing_child_user>.ccache
klist
```

**Verify:** `ticketer` writing a ccache proves only file creation. `klist` must show the intended child principal and valid TGT. Then test one parent service by FQDN:

```bash
nxc smb <parent_dc_fqdn> -k --use-kcache
```

Kerberos authentication proves ticket acceptance; `Pwn3d!` or a separately verified action proves authorization. KDC/SPN errors point to DNS, time, FQDN, or realm problems. Ticket rejection or access denied points to the child key, SIDs, trust direction/type, filtering, PAC validation, or insufficient authorization.

**Capability:** At most parent-domain service authorization carried by a forged child principal. No plaintext password is recovered.

**Cleanup:**

```bash
unset KRB5CCNAME
rm -f <existing_child_user>.ccache
```

**Next:** Use [[#[CORE] Pass-the-ticket (reuse a ccache/ticket)|the existing ccache/pass-the-ticket route]]. If the one service test succeeds, re-enumerate only the authorised parent scope. Otherwise stop and return to the trust evidence. See [Microsoft MS-PAC SID filtering rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/55fc19f2-55ba-4251-8a6a-103dd7c66280) and [Impacket `ticketer.py`](https://github.com/fortra/impacket/blob/master/examples/ticketer.py).

---

## [REFERENCE] Delegation discovery: unconstrained, constrained, and RBCD

Delegation is outside the primary PEN-200/OSCP checklist. Keep it searchable and prerequisite-driven, but do not force it into an exam path.

| Type | Recognition | Minimum prerequisite | Candidate result |
| --- | --- | --- | --- |
| **Unconstrained** | `TRUSTED_FOR_DELEGATION` | administrator/SYSTEM control of the delegation source plus incoming authentication | another principal's TGT |
| **Constrained** | `msDS-AllowedToDelegateTo` | usable control of the delegating account | service ticket to an allowed SPN |
| **Resource-based, RBCD** | `msDS-AllowedToActOnBehalfOfOtherIdentity` or a writable target attribute | controlled service/computer principal plus write access to the target attribute | service ticket to one target SPN |

```bash
impacket-findDelegation <domain_fqdn>/<user>:'<password>' -dc-ip <dc_ip>
nxc ldap <dc_ip> -d <domain_fqdn> -u <user> -p '<password>' --trusted-for-delegation
```

`--trusted-for-delegation` checks only unconstrained delegation. An empty RBCD attribute means it is not configured, not that no write opportunity exists. The write right must still be confirmed separately.

### [REFERENCE] RBCD and msDS-AllowedToActOnBehalfOfOtherIdentity

#### TLDR

RBCD configures which controlled service or computer principal may impersonate a user to a service on the target computer. It does not reveal a password, grant a human user domain privilege, or automatically provide host execution.

#### When you see this

BloodHound shows `AllowedToAct`, `AddAllowedToAct`, `WriteAccountRestrictions`, or computer-target `GenericAll`/`GenericWrite`, or LDAP shows the RBCD attribute.

#### Check first

Require all of these: a computer target; confirmed write access to the exact attribute; a controlled computer/service principal with a usable key, or confirmed permission and nonzero MachineAccountQuota to create one; correct DNS/time; exact target FQDN and SPN; an impersonated identity authorized to that service; recorded baseline; cleanup plan.

#### If confirmed

This remains a reference route. Use the current [SpecterOps AllowedToAct documentation](https://bloodhound.specterops.io/resources/edges/allowed-to-act) and [Impacket `rbcd.py`, `getST.py`, and `addcomputer.py`](https://github.com/fortra/impacket/tree/master/examples). The safe progression is:

```text
read baseline attribute
-> write only the controlled principal when justified
-> read back and verify
-> request one ticket for the exact SPN required
-> verify principal, SPN, and expiry with klist
-> test authorization through that same service
-> remove only the added principal and verify the baseline
```

Do not flush a pre-existing RBCD configuration. If a computer was created only for the lab path, remove it only after the attribute cleanup is verified.

#### Expected result

The immediate result is a TGS/service ticket for one SPN while impersonating one user. DNS, time, SPN, forwardability, attribute-write, or authorization failures stop the path. Host execution is a separate result.

#### What did I gain?

`Kerberos service ticket and exact SPN scope`. Add `service authorization` or `host execution and execution identity` only after the service action succeeds.

#### Next

Use the exact ticket scope, re-enumerate as the resulting principal where possible, then restore the recorded attribute state. If any prerequisite is unproven, record the edge and return to [[02_bloodhound_and_acl_paths]].

### [REFERENCE] CoerceToTGT

#### TLDR

The source is configured for unconstrained delegation. With administrator/SYSTEM control of that source, a separate target may be coerced to authenticate to it so the source receives the target computer's TGT. A domain controller machine TGT may authorize replication. This is not RBCD, a host-hop arrow, a human password, or automatically a human Domain Administrator TGT.

#### When you see this

BloodHound shows `CoerceToTGT` from a user or computer configured for unconstrained delegation to a domain.

#### Check first

Prove administrator/SYSTEM control of the delegation source. Then identify a separate coercible target, confirm reachability and one applicable coercion primitive, confirm the target is not protected from delegation, and state what the resulting machine TGT would authorize.

#### If confirmed

Use the current [official SpecterOps CoerceToTGT workflow](https://bloodhound.specterops.io/resources/edges/coerce-to-tgt). Keep the decision sequence explicit:

```text
confirm unconstrained-delegation source
-> prove administrator/SYSTEM control on that source
-> choose a separate target whose machine TGT advances the objective
-> confirm reachability and coercion prerequisites
-> monitor incoming tickets on the source
-> trigger one verified coercion method
-> confirm the expected target machine TGT arrived
-> convert or inject it for the chosen platform
-> verify principal and expiry with klist
-> perform only an action authorized by that machine TGT
-> re-enumerate and remove temporary ticket material
```

#### Expected result

The expected target machine TGT appears and `klist` confirms its principal and expiry. Stop if source control is missing, source and target are conflated, no coercion primitive applies, the expected TGT never appears, the principal is protected, or the captured TGT lacks the required downstream capability.

#### What did I gain?

`Kerberos TGT` for the captured machine principal. For a replication objective, only a successful [[03_credential_access#DCSync (replication rights, no DC shell)|DCSync]] test adds domain-level credential access.

#### Next

Verify only the intended downstream capability, re-enumerate, and clean up temporary ticket material. If the machine TGT does not authorize the objective, stop and return to the graph.

Primary semantics: [SpecterOps BloodHound edge documentation](https://bloodhound.specterops.io/resources/edges/coerce-to-tgt), [Microsoft Kerberos constrained delegation overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-constrained-delegation-overview), and [Impacket examples](https://github.com/fortra/impacket/tree/master/examples).

---

## Verification & failure interpretation (all paths)
```bash
klist                          # is the ticket present, for the right principal, not expired?
KRB5CCNAME=t.ccache nxc smb <host> -k --use-kcache   # does Kerberos auth actually work?
```
- Works → proceed as that identity; re-enumerate ([[00_AD_METHODOLOGY]] Step 6).
- `KRB_AP_ERR_SKEW` → fix time. Name errors → fix DNS to the DC. `KDC_ERR_*` → wrong key/SPN/account.

> [!important] Helpline
> DNS + time first, always. **Silver/overpass/PtT are core; golden is persistence (you must already own krbtgt); delegation is reference-only.** Every path needs its prerequisite row ticked and a `klist` plus service-access verification. Look up current forgery syntax from primary docs, don't memorise chains.
