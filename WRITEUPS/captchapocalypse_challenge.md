# CAPTCHApocalypse Challenge

![Banner](./../IMAGES/captchapocalypse_img.png?raw=true)

**Pathway:** *Web Application Red Teaming* | **Section:** *Custom Tooling* | **Challenge:** *[CAPTCHApocalypse](https://tryhackme.com/room/captchapocalypse)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **21 July 2026**.
>
> **Spoiler warning:** This write-up documents the attack chain and automation method, although credentials, exact challenge-specific values and the flag are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, sensitive values, exact challenge answers or other information that would directly give away the challenge.
> - `THM{....}` represents the redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
>
> Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

CAPTCHApocalypse is a web application challenge focused on custom tooling, browser automation, CAPTCHA handling and client-side cryptography.

The objective was to identify the administrator's password by testing only the first 100 entries from `rockyou.txt`, successfully authenticate to the dashboard and recover the room flag.

The successful attack chain involved:

1. Confirming VPN routing, the `tun0` address and hostname resolution.
2. Adding the room hostname to `/etc/hosts`.
3. Enumerating the target's exposed TCP services.
4. Reviewing the login page, JavaScript and available PHP endpoints.
5. Identifying a CAPTCHA-protected administrator login.
6. Discovering that the browser encrypted login data before sending it to the server.
7. Building a reduced password list containing the first 100 RockYou entries.
8. Using Selenium to automate Chromium and preserve browser session state.
9. Using Tesseract OCR to read each newly generated CAPTCHA.
10. Submitting one password candidate per valid CAPTCHA.
11. Detecting the successful redirect to the authenticated dashboard.
12. Recovering the redacted TryHackMe flag.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: captcha.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> captcha.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts captcha.thm
```

VPN routing and the tunnel address were also verified:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected route used the TryHackMe VPN tunnel and the assigned VPN source address:

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
sudo sed -i '/captcha\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `nmap` and `rustscan` for TCP port, service and script enumeration.
- `curl` for downloading and reviewing the web application and JavaScript.
- `gobuster` for web content discovery.
- Firefox for manually validating the login and authenticated dashboard.
- Chromium and ChromeDriver for browser automation.
- Selenium for controlling the browser and interacting with the login form.
- Tesseract OCR and `pytesseract` for reading CAPTCHA images.
- Pillow for processing CAPTCHA screenshots.
- Python 3 for orchestrating the automated login workflow.
- `head`, `cp`, `sed`, `cat` and other standard Linux utilities for preparing and reviewing files.
- Git for obtaining the reference automation tool.

Click [HERE](https://github.com/Valikahn/TryHackMe-Jr-Penetration-Tester#tools-commonly-used) to return to the repository README.

The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Hostname and VPN Validation

Before scanning, the room hostname, VPN route and attacker tunnel address were confirmed:

```bash
getent hosts captcha.thm
ip route get <TARGET_IP>
ip -br address show tun0
```

This established that:

- `captcha.thm` resolved to `<TARGET_IP>`.
- Traffic to the target left through `tun0`.
- The Kali VM used `<TUN0_IP>` as its VPN source address.

This step avoided wasting time troubleshooting application behaviour caused by incorrect routing or hostname resolution.

### Port and Service Discovery

A full TCP scan was performed:

```bash
nmap --min-rate=1000 -vv captcha.thm -p-
```

A second scan confirmed the results:

```bash
nmap -T4 -p- captcha.thm
```

RustScan was then used with Nmap service and default-script detection:

```bash
rustscan -a captcha.thm --ulimit 5000 -- -sC -sV -Pn
```

The exposed services were:

```text
22/tcp open  ssh
80/tcp open  http
```

Service detection identified:

```text
SSH: OpenSSH on Ubuntu
HTTP: Apache httpd on Ubuntu
```

The HTTP title was `Login`, confirming that the primary challenge surface was the web application.

### Web Application Review

The login page was retrieved with cURL:

```bash
curl -s http://captcha.thm/
```

The HTML revealed:

- A username field.
- A password field.
- A hidden CSRF token.
- A CAPTCHA image loaded from `captcha.php`.
- A CAPTCHA input field.
- A JavaScript-controlled login button.
- Client-side scripts for cryptographic operations.

The page referenced the following files:

```text
/script.js
/server.php
/captcha.php
/dashboard.php
/logout.php
```

### Content Discovery

Gobuster was used to identify accessible files and directories:

```bash
gobuster dir \
  -u http://captcha.thm \
  -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt \
  -x php
```

Relevant results included:

```text
/index.php
/server.php
/captcha.php
/dashboard.php
/logout.php
/js/
/css/
/view/
```

Direct access to `dashboard.php` redirected unauthenticated users back to `index.php`, confirming that the flag was protected by a valid authenticated session.

### Client-Side JavaScript Analysis

The login JavaScript was downloaded and reviewed:

```bash
curl -s http://captcha.thm/script.js
```

The script disclosed the full browser-side login workflow.

The form values were collected from the page:

```text
action
csrf_token
username
password
captcha_input
```

They were converted into a URL-encoded string, encrypted with an embedded RSA public key and submitted to `server.php` as JSON:

```javascript
const encrypted = encryptData(requestData);

const response = await fetch("server.php", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ data: encrypted })
});
```

The response was also encrypted. The browser decrypted it using an embedded client private key and checked for application messages indicating success, failure or an invalid CAPTCHA.

The exact key material is intentionally omitted:

```text
Server public key: <REDACTED>
Client private key: <REDACTED>
```

This analysis established that a simple plaintext POST request or conventional password brute-force command would not accurately reproduce the application's expected workflow.

## Exploits

### Preparing the Required Password List

The room explicitly limited testing to the first 100 lines of `rockyou.txt`.

A reduced wordlist was created:

```bash
head -n 100 /usr/share/wordlists/rockyou.txt > /tmp/VK/rockyou-100.txt
```

The file contained only the permitted candidates and avoided unnecessary testing.

### Installing Browser Automation and OCR Dependencies

Chromium and ChromeDriver versions were checked:

```bash
chromium --version
chromedriver --version
```

Their major versions matched, reducing the risk of Selenium compatibility issues.

Tesseract OCR was installed:

```bash
apt install -y tesseract-ocr
```

The required Python packages were installed:

```bash
apt install -y python3-selenium python3-pil python3-pytesseract
```

Additional modules required by the automation script were installed with:

```bash
python3 -m pip install --break-system-packages \
  selenium-stealth \
  fake-useragent
```

### Obtaining and Reviewing the Automation Tool

The reference tool was cloned into the working directory:

```bash
cd /tmp/VK/

git clone https://github.com/aRedOpsEagle/CAPTCHApocalypse-TryHackMe-Tool.git
```

The script and requirements were reviewed before execution:

```bash
sed -n '1,240p' \
  /tmp/VK/CAPTCHApocalypse-TryHackMe-Tool/script.py

cat \
  /tmp/VK/CAPTCHApocalypse-TryHackMe-Tool/requirements.txt
```

Reviewing automation code before running it is important, even in a lab. It confirms what files are accessed, what target is contacted and whether the script performs any unexpected actions.

### Configuring the Target and Wordlist

The placeholder target URL was replaced with the room hostname:

```bash
sed -i \
  "s|http://TARGET_MACHINE_IP|http://captcha.thm|" \
  /tmp/VK/CAPTCHApocalypse-TryHackMe-Tool/script.py
```

The bundled wordlist was replaced with the locally generated first 100 RockYou entries:

```bash
cp /tmp/VK/rockyou-100.txt \
  /tmp/VK/CAPTCHApocalypse-TryHackMe-Tool/newrockyou.txt
```

This ensured that the tool respected the challenge instruction exactly.

### CAPTCHA Automation Workflow

The Python tool launched Chromium in headless mode and visited the login page for each password candidate.

For every attempt, it performed the following sequence:

1. Loaded a fresh login page and session.
2. Located the CAPTCHA image.
3. Saved the CAPTCHA element as a screenshot.
4. Passed the image to Tesseract OCR.
5. Restricted OCR output to uppercase letters and digits.
6. Entered `admin` as the username.
7. Entered the current password candidate.
8. Entered the OCR result into the CAPTCHA field.
9. Clicked the application's login button.
10. Allowed the website's own JavaScript to encrypt and submit the request.
11. Checked whether the browser reached `dashboard.php`.
12. Retried a password when the CAPTCHA appeared to have been misread.
13. Moved to the next password only after a confirmed authentication failure.

The key OCR configuration was:

```python
config="--psm 7 -c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
```

This improved accuracy by telling Tesseract to expect a single line containing only the characters used by the CAPTCHA.

### Running the Tool

The automation was launched from the repository directory:

```bash
cd /tmp/VK/CAPTCHApocalypse-TryHackMe-Tool
python3 script.py
```

The tool reported each candidate as one of three states:

```text
FAILED
MISREAD
SUCCESS
```

`FAILED` indicated that the CAPTCHA was accepted but the password was incorrect.

`MISREAD` indicated that the server did not return a normal login failure, so the CAPTCHA was retried.

`SUCCESS` indicated that the browser had been redirected to the authenticated dashboard.

The exact successful password is intentionally redacted:

```text
Password: <REDACTED>
Status: SUCCESS
```

### Dashboard Access and Flag

Successful authentication redirected the browser to:

```text
http://captcha.thm/dashboard.php
```

The dashboard welcomed the authenticated administrator and displayed the challenge flag.

The result is intentionally redacted:

```text
THM{....}
```

The successful login was also verified manually in Firefox using the recovered administrator credential.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation

The target route and `tun0` address were validated. The room hostname was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the application was accessed using the hostname it expected.

### 2. Service Discovery

Nmap and RustScan identified SSH and Apache HTTP services. The web application on port 80 was selected as the principal attack surface.

### 3. Web Enumeration

The login page exposed a CSRF token, CAPTCHA and JavaScript-driven submission process. Gobuster identified supporting PHP endpoints and the protected dashboard.

### 4. JavaScript Review

Reviewing `script.js` showed that the browser assembled all login fields, encrypted the request with RSA and submitted it as JSON to `server.php`.

### 5. Authentication Workflow Analysis

The server's encrypted response was decrypted in the browser using an embedded private key. Response messages distinguished a bad password from an invalid CAPTCHA.

### 6. Password Scope Preparation

Only the first 100 RockYou entries were extracted into a dedicated file, meeting the room's stated constraint.

### 7. Browser Automation Setup

Chromium, ChromeDriver, Selenium, Pillow and Tesseract were prepared. Matching browser and driver versions helped prevent automation failures.

### 8. CAPTCHA Recognition

Each CAPTCHA was captured directly from the page and passed to Tesseract using a restricted character set.

### 9. Automated Login Attempts

Selenium populated the username, current password candidate and OCR result, then clicked the application's own login button. This allowed the page's legitimate cryptographic JavaScript to handle the request.

### 10. Error-Aware Retry Logic

The tool treated a normal login failure differently from a CAPTCHA misread. Invalid CAPTCHA attempts were retried without incorrectly discarding the current password candidate.

### 11. Successful Authentication

One candidate from the permitted 100-entry list authenticated successfully as the administrator:

```text
Username: admin
Password: <REDACTED>
```

### 12. Final Objective

The authenticated dashboard displayed the room flag:

```text
THM{....}
```

## Key Lessons

CAPTCHApocalypse demonstrated several useful web application and custom-tooling lessons:

- Confirm VPN routing and hostname resolution before troubleshooting the application.
- Hostname-based TryHackMe rooms may not behave correctly without the required `/etc/hosts` entry.
- Keep `/etc/hosts` clear and remove stale mappings after completing each room.
- Read the application's client-side JavaScript before attempting to reproduce complex requests.
- Client-side encryption does not prevent automation when the browser is given everything required to perform the operation.
- Embedding sensitive cryptographic material in downloadable JavaScript provides no meaningful secrecy.
- A traditional HTTP brute-force tool is not always the right choice.
- Browser automation can reproduce JavaScript, cookies, CSRF handling, redirects and dynamic page state more reliably.
- CAPTCHA failures must be distinguished from password failures to avoid skipping valid candidates.
- OCR accuracy improves when the expected format and character set are constrained.
- Matching Chromium and ChromeDriver versions prevents avoidable Selenium errors.
- Challenge constraints should be followed exactly; in this room, only the first 100 RockYou entries were permitted.
- Third-party tools and scripts should be reviewed before execution.
- A successful automated result should be validated manually where practical.
- Public write-ups should explain the method without publishing exact passwords or live flags.

The central lesson was that the CAPTCHA and encryption were not separate barriers. They were parts of one browser-driven workflow. Automating the genuine browser process avoided the need to manually rebuild every cryptographic detail while still demonstrating how the application operated.

## Remediation Notes

### Authentication and Rate Limiting

- Enforce server-side rate limiting per account, session, source address and device fingerprint.
- Apply progressive delays after repeated failed logins.
- Temporarily lock accounts after a defined number of failures.
- Monitor for repeated authentication attempts across changing CAPTCHA sessions.
- Alert on automated or unusually rapid login behaviour.
- Require multi-factor authentication for administrator accounts.
- Prevent weak or commonly used passwords through password screening.
- Store passwords using a modern adaptive password hash such as Argon2id or bcrypt.

### CAPTCHA Design

- Do not rely on CAPTCHA as the primary defence against credential attacks.
- Use a modern risk-based anti-automation platform where justified.
- Bind each CAPTCHA challenge to the user session and intended action.
- Expire CAPTCHA challenges quickly.
- Reject replayed CAPTCHA tokens.
- Generate challenges using cryptographically secure randomness.
- Consider accessibility and provide an alternative verification method.
- Combine CAPTCHA with rate limiting, account protections and behavioural detection.

### Client-Side Cryptography

- Do not place private keys or other secrets in client-side JavaScript.
- Treat all browser-delivered code and values as public information.
- Use TLS to protect credentials and session data in transit.
- Perform authentication and security-critical validation exclusively on the server.
- Avoid custom cryptographic protocols where standard, reviewed mechanisms already exist.
- Rotate any key material that has been exposed to clients.
- Document the purpose and threat model of any application-layer encryption.

### Session and CSRF Security

- Regenerate the session identifier after successful authentication.
- Set cookies with `Secure`, `HttpOnly` and appropriate `SameSite` attributes.
- Bind CSRF tokens to the user session and validate them server-side.
- Expire authentication sessions after inactivity.
- Invalidate sessions during logout and password changes.
- Protect the dashboard with server-side authorisation on every request.

### Error Handling

- Avoid returning responses that allow automation to distinguish between an incorrect password and other authentication states unless operationally necessary.
- Use consistent external error messages.
- Record detailed diagnostic information only in protected server-side logs.
- Ensure CAPTCHA and login failures do not disclose implementation details.
- Monitor repeated CAPTCHA failures followed by password failures.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale challenge entries after each room.
- Maintain a separate working directory for scripts, wordlists, screenshots and scan output.
- Record the target address, hostname and `tun0` address at the beginning of each lab.
- Review downloaded automation scripts before running them.
- Preserve only evidence needed for learning and remove unnecessary sensitive artefacts afterwards.

## Disclaimer

This write-up is intended solely for education, training and documentation of an authorised TryHackMe lab. All tools, commands and automation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, automate authentication against or access a system without clear and explicit authorisation from its owner.

---

[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**

⭐ If this project is useful, consider starring it on GitHub.
