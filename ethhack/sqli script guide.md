# sqli_guide.py — guided SQLi solver

One script, one entry point, stdlib only (no `pip install` needed). You paste a
URL; it finds the injection point, builds a TRUE/FALSE oracle, fingerprints the
DBMS, and extracts the target value — printing a plain-English recipe (and an
equivalent `sqlmap` command) at the end so you can reproduce it by hand.

```
python3 sqli_guide.py <url> [options]
```

## The model

Every SQLi solve is the same four phases. Only phase 3 really differs target
to target — the script automates discovering it instead of you hand-coding it:

| phase | what it does |
|---|---|
| 1. LOCATE | find where input lands: an existing `?param`, or a swept query/cookie/header, or a POST body you point it at |
| 2. CONTEXT | probe quote/paren context and obvious filtering (`'`, `=`, `<`, `>`) |
| 3. ORACLE | auto-calibrate a working `TRUE`/`FALSE` boolean condition — trying template × primitive combos, then filter-evasion tiers, then time-based, then error-based |
| 4. EXTRACT | reflected page → UNION; otherwise blind binary search (`BETWEEN`, filter-safe) |

## Quick start

```bash
# the normal case: URL already has the injectable param
python3 sqli_guide.py "https://host/catalog?item=1"

# no params on the URL -> sweeps query/cookie/header for you
python3 sqli_guide.py "https://host/account"

# POST endpoint (no URL param to show it) -> tell it where to inject
python3 sqli_guide.py "https://host/cart" --place json --param id

# signed/hashed param (h = md5(id)) -> detected automatically from the URL
python3 sqli_guide.py "https://host/orders?id=1&h=c4ca4238a0b923820dcc509a6f75849b"
```

Quote the URL in bash so `&` and `?` aren't eaten by the shell.

## Finding the injection point before you run it

The script still needs *a* URL or a `--place`/`--param` pair to start from.
How to find that from the page:

- **GET with a visible param** (e.g. "View" on a catalog item): click it, copy
  the resulting URL straight into the command.
- **A button that doesn't change the URL** (e.g. "Add to cart"): it's a POST.
  Open DevTools → Network, click the button, find the request, right-click it
  → *Copy as fetch (Node.js)* or *Copy as cURL* to see the body and find the
  parameter name. Or just view-source / Ctrl+U if the form field name is in
  the HTML. Use `--place form` or `--place json` + `--param <name>` with it.
- **No form, no visible param** (e.g. "No member is registered under that id"
  with nothing to click): the input is probably a cookie. Just give the script
  the bare URL — phase 1 sweeps common cookie/header names automatically.
- **A search box**: usually a plain `?q=` GET param, same as the first case.

## Options

| flag | meaning |
|---|---|
| `--place query\|cookie\|header\|form\|json` | force the injection location (needed for POST bodies, which a URL can't express) |
| `--param NAME` | force the injection parameter name |
| `--data '{"id":"1"}'` or `'k=v&k2=v2'` | seed other static params alongside `--place`/`--param` |
| `--yes` | non-interactive: accept every suggested default without prompting |
| `--delay 2.0` | seconds between requests (use if the server rate-limits you) |
| `--target "(SELECT ... )"` | what to steal — a scalar subquery. Default: `(SELECT flag FROM flag LIMIT 1)` |
| `--list` | enumerate tables, let you pick a table/column (and optionally dump all rows), instead of auto-extracting `flag.flag` |
| `--insecure` | skip TLS certificate verification — needed for self-signed certs or bare-IP `https://` targets |

## Common test-day situations

**"The flag isn't in a table called `flag`."**
Point `--target` at wherever it actually lives:
```bash
python3 sqli_guide.py "https://host/x?id=1" --target "(SELECT secret FROM vault LIMIT 1)"
```

**"I don't know the table/column name."**
Run with `--list` instead of extracting — it enumerates tables, then that
table's columns, and lets you pick (or dump every row of a column, not just
the first):
```bash
python3 sqli_guide.py "https://host/x?id=1" --list
```

**"The flag format changed" (e.g. `SIT{...}` → `ehCTF{...}`).**
Nothing to change. Extraction always dumps the raw column value character by
character regardless of format — only the pretty `>>> FLAG:` highlighter used
in the UNION/error-based paths needs to recognize the shape, and it already
matches any `NAME{...}` token (letters/digits/`_`/`-`, any case, any name), so
`ehCTF{...}`, `SIT{...}`, `testflag{...}` etc. all highlight with no edit.

**"The DBMS isn't SQLite."**
Nothing to do — phase 3b fingerprints SQLite/MySQL/PostgreSQL/MSSQL/Oracle
automatically and switches the string functions (`SUBSTR`/`ASCII`/`LENGTH`/
concat operator) to match.

**"The site strips `=`, `<`, `>` from my input."**
Nothing to do — phase 3 already prefers `BETWEEN`, which is keyword-only and
survives that filter, and reports when it had to.

**"The site strips the literal words `union`/`select`/`--`."**
Nothing to do — if the plain boolean oracle calibrates fine (its conditions
rarely contain those words) but the later `SELECT`-based extraction would
break, the script notices and automatically starts splitting keywords with an
inline comment (`SEL/**/ECT`) before extracting, table-listing, or
fingerprinting. You'll see a line like:
```
extraction needs SELECT; current evasion insufficient -- escalating keyword-split
OK: SELECT now survives (spacer='/**/', kwsplit=True)
```

**"The only visible difference between true/false is a result count going to zero"**
(e.g. a page shows `<strong>0</strong>` only when the condition is false).
Nothing to do — phase 3's discriminator looks for an exact marker unique to
either side, not just a "true-only" marker, so a false-only marker (sqlmap's
`--not-string` case) is found the same way.

**"It's `https://` on a self-signed cert / bare IP."**
Add `--insecure`.

**"The param is inside a `LIKE '%...%'` clause."**
Nothing to do — the template matrix already includes `%'`-prefixed
breakouts (`%' AND {cond}#`, `%' OR {cond}#`, etc.) alongside the plain
`'`/`"`/`)` ones, tried automatically.

## What it prints at the end

Every successful run ends with a plain-text recipe: the DBMS, the injection
template, the exact TRUE/FALSE oracle, any filter evasion that was needed, and
an equivalent `sqlmap` command — useful for write-ups or for an exam where
LLMs aren't allowed but you still want the steps on hand.

## Known limits

- UNION detection assumes the reflected marker survives one of `||`,
  `concat(...)`, or `+` — covers SQLite/MySQL/Postgres/Oracle/MSSQL, but an
  unusual concat syntax could still slip through to the (still-correct) blind
  path.
- Time-based and error-based oracles are implemented but only exercised in
  logic/testing, not against a live time-blind or error-only target yet.
- Second-order (stored) injection, stacked queries, and auth/session-gated
  endpoints aren't handled — those need manual setup first.
