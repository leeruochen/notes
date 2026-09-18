
# Nmap Scans

## Description
What the service is, what to do on contact, and the key things to look for.

## One Liner Scan
```bash
#tcp scan 
nmap -sVC -p- -v -T4 -sT --open $ip 

#udp scan
sudo nmap -sU -p 1-1024 -v $ip 

#autorecon 
sudo autorecon $ip 
```


## Checklist
- [ ] Version + banner grabbed
- [ ] Port numbers
- [ ] Watch for: {edge case}