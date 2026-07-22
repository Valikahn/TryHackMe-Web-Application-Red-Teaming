# Padelify Challenge

![Banner](./../IMAGES/padelify_img.png?raw=true)

**Pathway:** *Web Application Red Teaming* | **Section:** *Bypassing WAF* | **Challenge:** *[Padelify](https://tryhackme.com/room/padelify)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **23 July 2026**.
>
> **Spoiler warning:** This write-up documents the successful exploitation chain, although credentials, session identifiers, exact challenge-specific payloads and flag values are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, session identifiers, sensitive configuration values, exact payloads or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on laboratories covering penetration testing, networking, web application security, privilege escalation and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Padelify is a web application red-teaming challenge focused on bypassing a web application firewall and chaining multiple web vulnerabilities to gain privileged access.

The fictional application manages player registrations and match approvals for a padel championship. The objective was to obtain access first as a moderator and then as an administrator.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and hostname resolution.
2. Enumerating the exposed TCP services.
3. Discovering accessible web directories, PHP endpoints, logs and configuration paths.
4. Identifying that the PHP session cookie lacked the `HttpOnly` security attribute.
5. Exploiting a stored cross-site scripting weakness to execute JavaScript in a moderator's browser.
6. Exfiltrating the moderator's session identifier to an attacker-controlled HTTP listener.
7. Replaying the stolen session while using a browser-like `User-Agent` to avoid a WAF block.
8. Accessing the moderator dashboard and obtaining the first flag.
9. Identifying a local file inclusion weakness in the live match functionality.
10. URL-encoding a sensitive file path to bypass WAF pattern matching.
11. Reading an application configuration file containing an administrative secret.
12. Authenticating as the administrator and obtaining the final flag.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: padelify.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> padelify.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts padelify.thm
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
> A crowded hosts file eventually becomes DNS spaghetti - technically operational, but needlessly painful to untangle.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old Padelify entry can be removed with:

```bash
sudo sed -i '/padelify\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `ping` for confirming basic network reachability.
- `rustscan` and `nmap` for TCP port, service and script enumeration.
- `dirsearch` and `gobuster` for web content discovery.
- Firefox for interacting with the web application and submitting the stored XSS payload.
- Python 3's built-in HTTP server for receiving the moderator's browser callback.
- cURL for replaying session cookies, testing WAF behaviour, exploiting local file inclusion and authenticating to the application.
- `grep` and `tee` for saving responses and extracting relevant evidence.
- Standard Linux utilities for managing local evidence and reviewing output.

Click [HERE](https://github.com/Valikahn/TryHackMe-Web-Application-Red-Teaming#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A rapid scan was performed first:

```bash
rustscan -a padelify.thm --ulimit 5000 -- -sC -sV -Pn
```

A full TCP scan was then used to verify that no additional services were exposed:

```bash
nmap -T4 -n -sC -sV -Pn -p- padelify.thm
```

The principal findings were:

```text
22/tcp open ssh
80/tcp open http
```

Service detection identified:

```text
SSH: OpenSSH on Ubuntu
HTTP: Apache HTTP Server on Ubuntu
Application title: Padelify - Tournament Registration
```

Nmap also reported that the `PHPSESSID` cookie did not include the `HttpOnly` attribute. This was an important security finding because client-side JavaScript could potentially access the session identifier.

### Web Content Discovery

Dirsearch was used against the web root:

```bash
dirsearch -u http://padelify.thm/
```

Gobuster was initially used to enumerate directories and files exposed by the web application:

```bash
gobuster dir \
  -u http://padelify.thm/ \
  -w /usr/share/wordlists/dirb/big.txt
```

The first attempt failed because the server returned an HTTP 403 Forbidden response for requests generated by Gobuster, including requests for paths that did not exist:

```text
Error: the server returns a status code that matches the provided options for non existing URLs:
http://padelify.thm/<random-path> => 403
```

Gobuster normally creates a random test path before enumeration begins. It uses the response to determine what a normal non-existent resource looks like. In this case, the application or web application firewall returned the same 403 status code for the random path that it used for blocked requests, preventing Gobuster from reliably distinguishing genuine findings from rejected requests.

The scan was repeated with a browser-like User-Agent header:

```bash
gobuster dir \
  -u http://padelify.thm/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -H "User-Agent: Mozilla/1.0"
```

The -H option adds a custom HTTP request header. Here, it replaced Gobuster's recognisable default User-Agent with:

```text
Mozilla/1.0
```

A User-Agent identifies the client software making an HTTP request. Browsers normally send values that begin with Mozilla/5.0, while automated tools such as Gobuster usually identify themselves by name.

The application or WAF appeared to treat Gobuster's default request profile as automated or suspicious traffic and returned 403 Forbidden. Supplying a browser-like value made the requests resemble ordinary web-browser traffic closely enough for enumeration to continue.

This did not authenticate the request or exploit the application directly. It changed only the client identification header and demonstrated that the filtering behaviour relied partly on a request characteristic that could be easily modified.

The scan identified several relevant locations:

```text
/config/
/css/
/dashboard.php
/footer.php
/header.php
/javascript/
/js/
/login.php
/logout.php
/logs/
/logs/error.log
/register.php
/status.php
```

It also confirmed that access to sensitive Apache files and the server status endpoint was forbidden:

```text
/.htaccess
/.htpasswd
/server-status
```

The most important findings were the exposed /config/ and /logs/ directories, as these later provided information about the application's structure and supported the wider attack chain.

> [!NOTE]
> A User-Agent header should not be treated as a security control. It is fully controlled by the client and can be changed by browsers, scripts and command-line tools. Blocking requests solely because they identify themselves as Gobuster may slow down basic scanning, but it does not prevent an attacker from repeating the request with a different header.

The `dashboard.php` endpoint redirected unauthenticated visitors to the login page. Directory listing was enabled beneath both `/config/` and `/logs/`, and the exposed error log provided additional insight into the application's file structure.

### Application Behaviour

The application allowed users to submit registration information. Submitted content was later reviewed by a privileged user, creating a possible stored-input attack surface.

A status page was also available:

```text
http://padelify.thm/status.php
```

This page could restart the room's services when a clean application state was required.

## Exploits

### Stored Cross-Site Scripting

A stored cross-site scripting weakness was identified in user-controlled registration content. The submitted value was later rendered in a moderator's browser without adequate output encoding.

A callback listener was started on the Kali VM:

```bash
python3 -m http.server 8080
```

The browser payload used the attacker's VPN address and attempted to send the accessible session cookie to the listener. The exact challenge payload is intentionally redacted:

```html
<REDACTED>
```

After the privileged user reviewed the submitted content, the Python server received HTTP requests from the target:

```text
<TARGET_IP> - - [<REDACTED>] "GET /?c=PHPSESSID=<REDACTED> HTTP/1.1" 200 -
```

This confirmed that:

- The stored payload executed in the moderator's browser.
- JavaScript could read the session cookie.
- The moderator's session identifier was successfully exfiltrated.
- The missing `HttpOnly` attribute materially increased the impact of the XSS vulnerability.

### Moderator Session Replay

The captured moderator session was replayed against the dashboard.

An initial cURL request returned a WAF-generated response:

```html
<title>⚠️ 403 Forbidden</title>
```

This showed that the WAF was evaluating more than the supplied cookie. The default cURL request profile was blocked even though the session itself was valid.

The request was repeated with a normal browser-style `User-Agent`:

```bash
curl -sS \
  -A 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0' \
  -b 'PHPSESSID=<REDACTED>' \
  http://padelify.thm/dashboard.php \
  | tee moderator-dashboard.html \
  | grep -oE 'THM\{[^}]+\}'
```

The request succeeded and returned the moderator flag:

```text
THM{....}
```

This demonstrated that the WAF's decision could be influenced by superficial request characteristics such as the `User-Agent`, rather than relying solely on robust session and authorisation controls.

### Local File Inclusion

After moderator access was established, the live match functionality exposed a page parameter:

```text
/live.php?page=<REDACTED>
```

The parameter controlled content included by the PHP application. Testing showed that it could be used to read a local application file.

A direct request for the sensitive path was detected by the WAF. The same path was then URL-encoded so that the application decoded it after the WAF's initial inspection.

The exact encoded challenge path is intentionally redacted:

```bash
curl -sS \
  -A 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0' \
  -b 'PHPSESSID=<REDACTED>' \
  'http://padelify.thm/live.php?page=<REDACTED>'
```

The response included the contents of an application configuration file:

```ini
version = "<REDACTED>"
enable_live_feed = true
enable_signup = true
env = "<REDACTED>"
site_name = "Padelify Tournament Portal"
db_path = "<REDACTED>"
admin_info = "<REDACTED>"
```

The `admin_info` value provided the secret required for the administrative account.

### WAF Bypass Through URL Encoding

The WAF appeared to match a blocked file path before normalisation or decoding was completed. Encoding the path altered the request representation while preserving its meaning to the PHP application.

This created a parsing discrepancy:

1. The client sent an encoded value.
2. The WAF did not recognise the blocked plaintext pattern.
3. The web server or application decoded the parameter.
4. The vulnerable include operation processed the resulting local path.
5. The sensitive configuration file was returned.

This is a common WAF bypass class. Security controls and applications must interpret canonicalised input consistently, otherwise attackers can exploit differences in decoding order.

### Administrator Authentication

The recovered secret was submitted to the login endpoint with the administrative username:

```bash
curl -sS \
  -A 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0' \
  -c admin-cookies.txt \
  -d 'username=admin&password=<REDACTED>' \
  http://padelify.thm/login.php \
  -D -
```

A successful login produced:

```text
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=<REDACTED>; path=/
Location: dashboard.php
```

The saved administrative session was then used to access the dashboard:

```bash
curl -sS \
  -A 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0' \
  -b admin-cookies.txt \
  http://padelify.thm/dashboard.php \
  | tee admin-dashboard.html \
  | grep -oE 'THM\{[^}]+\}'
```

The final flag was returned:

```text
THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The allocated target and `tun0` addresses were confirmed. The local hostname `padelify.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that requests reached the application through its expected hostname.

### 2. Service Discovery
RustScan and Nmap identified SSH and Apache HTTP services. The HTTP script results also showed that the PHP session cookie lacked the `HttpOnly` attribute.

### 3. Web Enumeration
Dirsearch and Gobuster identified application endpoints, configuration and log directories, authentication pages, the dashboard and the service reset page.

### 4. Stored Input Identification
The public registration workflow accepted user-controlled content that was later reviewed by a privileged moderator. This provided a stored-input route into the moderator's browser.

### 5. Stored XSS Execution
A stored JavaScript payload was submitted through the application. When the moderator viewed the entry, the payload executed in the privileged browser session.

### 6. Session Exfiltration
Because the session cookie was accessible to JavaScript, the payload sent the moderator's `PHPSESSID` to an HTTP server running on `<TUN0_IP>`.

### 7. WAF-Aware Session Replay
Replaying the session with cURL's default request profile produced `403 Forbidden`. Adding a realistic Firefox `User-Agent` allowed the same valid moderator session to reach the dashboard.

### 8. Moderator Access
The stolen session provided moderator-level access and revealed the first challenge flag:

```text
THM{....}
```

### 9. Local File Inclusion Discovery
The moderator-accessible live match page accepted a `page` parameter that was used by the server to include local content.

### 10. Encoded Path WAF Bypass
The sensitive file path was URL-encoded. The WAF failed to recognise the encoded representation, while the PHP application decoded and included the requested file.

### 11. Configuration Disclosure
The included configuration file disclosed an administrative secret. Exact values are omitted from this public write-up.

### 12. Administrator Login
The recovered secret was used to authenticate as `admin`. A new administrative `PHPSESSID` was issued and stored locally.

### 13. Final Objective
The administrative session provided access to the dashboard and the final challenge flag:

```text
THM{....}
```

## Key Lessons

Padelify demonstrated several important web application red-teaming and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting higher-level application behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM for TryHackMe rooms.
- Hostname-based applications may behave incorrectly when accessed only by IP address.
- Directory listing and exposed logs can disclose valuable application structure.
- A missing `HttpOnly` attribute allows JavaScript to read a session cookie and increases the impact of XSS.
- Stored XSS is especially dangerous when privileged staff review attacker-controlled content.
- Session identifiers must be treated as credentials because possession may be enough to impersonate a user.
- A WAF is not a substitute for secure application design.
- Request characteristics such as `User-Agent` should not be relied upon as a meaningful security boundary.
- Input must be canonicalised before security rules are applied.
- URL encoding can expose inconsistencies between WAF parsing and application parsing.
- User-controlled values must never be passed directly into server-side file inclusion functions.
- Configuration files should not contain reusable administrative secrets.
- Authentication success should be confirmed using status codes, redirects and issued cookies.
- Public write-ups should explain the method without publishing active credentials, exact flags or unnecessary challenge giveaways.

The most important lesson was that each weakness amplified the next. Stored XSS alone provided a moderator session, but moderator access exposed an LFI route. The LFI then disclosed an administrative secret, and the WAF could be bypassed at multiple stages because its interpretation of requests differed from the application's.

## Remediation Notes

### Cross-Site Scripting Prevention

- Apply context-appropriate output encoding to all untrusted content.
- Use a mature templating system with automatic escaping enabled.
- Validate and sanitise submitted registration data on the server.
- Avoid rendering user-controlled HTML unless there is a genuine business requirement.
- Use a strict Content Security Policy to restrict script execution and outbound connections.
- Review privileged moderation interfaces as high-risk XSS targets.
- Test stored content in every location where it may later be displayed.

### Session Security

- Set the `HttpOnly` attribute on all authentication cookies.
- Set the `Secure` attribute when the application is served over HTTPS.
- Apply an appropriate `SameSite` policy.
- Regenerate the session identifier after successful authentication and privilege changes.
- Invalidate sessions after logout, expiry and suspected compromise.
- Bind sensitive actions to server-side authorisation checks rather than trusting possession of a cookie alone.
- Monitor for session reuse from unusual clients or network locations.

### Local File Inclusion Prevention

- Never pass user-controlled values directly to `include`, `require` or similar filesystem operations.
- Replace file paths with a strict allow-list of approved page identifiers.
- Map safe identifiers to fixed server-side templates.
- Reject directory traversal, absolute paths, stream wrappers and encoded path separators.
- Store sensitive configuration files outside the web root.
- Run the web service with the minimum filesystem permissions required.
- Avoid returning raw include errors or filesystem paths to users.

### WAF and Input Normalisation

- Canonicalise and fully decode request data before applying security rules.
- Ensure the WAF and application use the same interpretation of paths and parameters.
- Detect repeated, mixed and non-standard encoding.
- Apply controls to the decoded value as well as the original representation.
- Do not treat `User-Agent` values as proof that a request came from a legitimate browser.
- Use the WAF as defence in depth rather than the primary protection against application flaws.
- Log WAF bypass attempts and compare them with application-level request logs.
- Regularly test controls using encoded and obfuscated payload variants.

### Secrets and Configuration Management

- Remove plaintext administrative secrets from application configuration files.
- Store secrets in a dedicated secrets-management system.
- Limit file ownership and permissions to the service account that requires access.
- Rotate any secret exposed through the application or included in a public write-up.
- Avoid using shared or reusable credentials between application roles.
- Keep configuration files outside directories that may be included or served by the web application.
- Ensure production and staging secrets are different.

### Logging and Directory Exposure

- Disable directory listing unless it is explicitly required.
- Prevent public access to application logs.
- Store logs outside the web root.
- Avoid recording credentials, session identifiers or sensitive request values.
- Apply log rotation and restrictive permissions.
- Alert on repeated access to configuration files, encoded traversal strings and unexpected include targets.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale room entries after completing each challenge.
- Maintain a separate working directory for scan output, cookies, HTML responses and evidence.
- Record the target address, local hostname and `tun0` address before beginning.
- Redact session identifiers, credentials, exact payloads and flags before publishing.
- Stop temporary HTTP listeners and remove saved cookie files after completing the room.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
