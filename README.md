# Web Application Red Teaming Writeups ![PathwayCompleted](https://img.shields.io/badge/pathway-in_progress-orange?style=for-the-badge)
![Banner](./IMAGES/webappredteaming_img.png?raw=true)
![License](https://img.shields.io/badge/License-CC_BY_4.0-green) ![Writeups](https://img.shields.io/badge/Completed_Writeup-6/9-blue) [![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-FFDD00?logo=buy-me-a-coffee&logoColor=black)](https://www.buymeacoffee.com/v4l1k4hn) ![GitHub User's stars](https://img.shields.io/github/stars/valikahn?style=flat&logo=github) ![Discord](https://img.shields.io/discord/521382216299839518?style=flat&logo=discord&color=purple) [![TryHackMe_Profile](https://img.shields.io/badge/TryHackMe-Certificates-black)](https://tryhackme.com/p/V4L1K4HN?tab=certificates)

This repository contains my personal [TryHackMe](https://tryhackme.com/) writeups, study notes and walkthroughs from the [TryHackMe Web Application Red Teaming](https://tryhackme.com/path/outline/webappredteaming) learning path.

The purpose of this repository is to document my methodology, commands, observations, mistakes, exploitation chains and lessons learned while working through advanced web application red teaming rooms. These notes are intended for revision, portfolio building, technical reference and continuous improvement.

## About This Repository

Each writeup is based on work completed within an authorised TryHackMe training environment. The rooms and systems covered are intentionally designed for cyber security education, practical exploitation and controlled experimentation.

Where relevant, the writeups may include IP addresses assigned during a lab session. Testing is performed using the TryHackMe AttackBox or a personal Kali Linux virtual machine connected to the TryHackMe network through OpenVPN.

**PLEASE NOTE:** This repository focuses on the Web Application Red Teaming path. The writeups may involve advanced exploitation concepts, including cryptographic weaknesses, custom tooling, vulnerability chaining, web application firewall bypass techniques and attacks against LLM-enabled applications.

The format of each entry will vary according to the room, objective and level of complexity. Some rooms may focus on a single technique, while others may require several weaknesses to be chained together into a full exploitation path.

> [!IMPORTANT]
> This repository will **NEVER** contain material taken from TryHackMe professional certification examinations or other restricted assessments. It will **NOT*** provide any flags, passwords, cracked credentials or confidential material that will slow the rate of learning.
> 
> If you are looking for any assistance, answers, guides, specific walkthroughs you will **NOT** find any of those here, help/assistance is limited, but available via the [TryHackMe Discord](https://discord.com/invite/tryhackme).

## What a Writeup May Include

Depending on the room, module and learning objective, a writeup may contain:

- Room and module overview;
- Learning objectives;
- Environment and tooling notes;
- Target and attacker IP details where relevant;
- Enumeration and reconnaissance steps;
- Web application mapping;
- Request and response analysis;
- Vulnerability identification and validation;
- Cryptographic weakness analysis;
- Custom tooling or automation logic;
- Exploitation chain development;
- WAF bypass methodology;
- LLM attack surface analysis;
- Initial access methodology where applicable;
- Shell handling or session management where applicable;
- Commands and selected sanitised output;
- Mistakes, troubleshooting and alternative approaches;
- Key findings and lessons learned;
- Defensive recommendations or remediation notes; and
- References and supporting documentation.

Writeups are intended to explain the methodology and decision-making process rather than provide a simple answer list.

## Learning Path

Learning Path: [Web Application Red Teaming](https://tryhackme.com/path/outline/webappredteaming)

This path focuses on taking web application pentesting beyond vulnerability discovery and into practical exploitation. It covers complex vulnerability chains and shows how individual weaknesses can be combined to demonstrate meaningful impact in a controlled red team-style environment.

The path currently includes:

| Section | Focus Area |
| --- | --- |
| Cryptographic Failures | Understanding and exploiting weaknesses in cryptographic implementation. |
| Custom Tooling | Building or adapting tools to automate exploitation workflows. |
| Chaining Vulnerabilities | Combining multiple issues to achieve a larger objective. |
| Bypassing WAF | Understanding web application firewall behaviour and bypass techniques. |
| Attacking LLMs | Exploring prompt injection, insecure output handling, privacy risks and model-related abuse cases. |

The learning path is designed to strengthen practical exploitation methodology, technical reasoning and the ability to show the real impact of discovered vulnerabilities.

## Published Writeups

| No. | Section | Challenge | Difficulty | Status |
| --- | --- | --- | --- | --- |
| 1 | Custom Tooling | [CAPTCHApocalypse](https://tryhackme.com/room/captchapocalypse) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/captchapocalypse_challenge.md) |
| 2 | Chaining Vulnerabilities | [Extract](https://tryhackme.com/room/extract) | ![Hard](https://img.shields.io/badge/Hard-red) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/extract_challenge.md) |
| 3 | Chaining Vulnerabilities | [Voyage](https://tryhackme.com/room/voyage) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/voyage_challenge.md) |
| 4 | Chaining Vulnerabilities | [Sequence](https://tryhackme.com/room/sequence) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/sequence_challenge.md) |
| 5 | Bypassing WAF | [Padelify](https://tryhackme.com/room/padelify) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/padelify_challenge.md) |
| 6 | Bypassing WAF | [Farewell](https://tryhackme.com/room/farewell) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/farewell_challenge.md) |
| 7 | Attacking LLMs | [Juicy](https://tryhackme.com/room/juicy) | ![Medium](https://img.shields.io/badge/Medium-orange) | ![In Progress](https://img.shields.io/badge/In%20Progress-yellow) |
| 8 | Attacking LLMs | [BankGPT](https://tryhackme.com/room/bankgpt) | ![Easy](https://img.shields.io/badge/Easy-brightgreen) | ![Planned](https://img.shields.io/badge/Planned-lightgrey) |
| 9 | Attacking LLMs | [HealthGPT](https://tryhackme.com/room/healthgpt) | ![Easy](https://img.shields.io/badge/Easy-brightgreen) | ![Planned](https://img.shields.io/badge/Planned-lightgrey) |

### Status Key

| Status | Meaning |
| --- | --- |
| ![Planned](https://img.shields.io/badge/Planned-lightgrey) | The room has been selected but documentation has not started. |
| ![In Progress](https://img.shields.io/badge/In%20Progress-yellow) | The room is currently being completed and documented. |
| ![Complete](https://img.shields.io/badge/Complete-brightgreen) | The writeup has been published. |
| ![Archived](https://img.shields.io/badge/Archived-red) | The room or writeup is no longer actively maintained. |
| ![Issue_Reported](https://img.shields.io/badge/Issue-Reported-orange) | A lab or room issue has been encountered and reported. |

## Tools Commonly Used

The tools used will vary by room, objective, target behaviour and attack stage. The following list is representative rather than exhaustive and includes web application testing, internal pivoting, container analysis and host-level exploitation.

| Reconnaissance and Enumeration | Web and Application Testing | Exploitation and Access | Custom Tooling and Automation |
|---|---|---|---|
| [Nmap](https://nmap.org/) | [Burp Suite](https://www.kali.org/tools/burpsuite/) | [Metasploit Framework](https://www.kali.org/tools/metasploit-framework/) | [Python](https://www.python.org/) |
| [RustScan](https://github.com/RustScan/RustScan) | [OWASP ZAP](https://www.kali.org/tools/zaproxy/) | [Netcat](https://www.kali.org/tools/netcat-traditional/) | [Requests](https://requests.readthedocs.io/) |
| [Gobuster](https://www.kali.org/tools/gobuster/) | [ffuf](https://www.kali.org/tools/ffuf/) | [Hydra](https://www.kali.org/tools/hydra/) | [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/) |
| [Feroxbuster](https://www.kali.org/tools/feroxbuster/) | [SQLmap](https://www.kali.org/tools/sqlmap/) | [NetExec](https://www.netexec.wiki/) | [Playwright](https://playwright.dev/) |
| [WhatWeb](https://www.kali.org/tools/whatweb/) | [Postman](https://www.postman.com/) | [OpenSSH](https://www.openssh.com/) | [Selenium](https://www.selenium.dev/) |
| [Nikto](https://www.kali.org/tools/nikto/) | Browser Developer Tools | [socat](https://www.kali.org/tools/socat/) | Bash scripting |
|  | [cURL](https://curl.se/) |  | [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) |
|  |  |  | [Pillow](https://python-pillow.org/) |
|  |  |  | [ChromeDriver](https://developer.chrome.com/docs/chromedriver/) |
|  |  |  | [Chromium](https://www.chromium.org/) |
|  |  |  | [pytesseract](https://github.com/madmaze/pytesseract) |

| Cryptography and Encoding | WAF and Filtering Analysis | LLM Security Testing | Supporting References |
| --- | --- | --- | --- |
| [OpenSSL](https://www.openssl.org/) | Burp Suite Repeater | Manual prompt testing | [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) |
| [Hashcat](https://www.kali.org/tools/hashcat/) | Burp Suite Intruder | Input and output handling review | [OWASP Top 10](https://owasp.org/www-project-top-ten/) |
| [John the Ripper](https://www.kali.org/tools/john/) | Encoding and case manipulation | Data leakage testing | [OWASP API Security Top 10](https://owasp.org/API-Security/) |
| [CyberChef](https://gchq.github.io/CyberChef/) | Payload mutation | Indirect prompt injection review | [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) |
| [hashpump](https://github.com/mheistermann/HashPump-partialhash) | Header and parameter tampering | Model behaviour analysis | [PortSwigger Web Security Academy](https://portswigger.net/web-security) |
|  |  |  | [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) |
|  |  |  | [GTFOBins](https://gtfobins.github.io/) |

| CMS and Service Testing | Pivoting and Tunnelling | Container and Linux Analysis | Build and Kernel Tooling |
| --- | --- | --- | --- |
| [JoomScan](https://github.com/OWASP/joomscan) | [OpenSSH](https://www.openssh.com/) | `capsh` | [GCC](https://gcc.gnu.org/) |
| Joomla API testing | SSH local port forwarding | Linux capabilities | [GNU Make](https://www.gnu.org/software/make/) |
| [NetExec](https://www.netexec.wiki/) | [Netcat](https://www.kali.org/tools/netcat-traditional/) | Docker network enumeration | Linux kernel headers |
| [SearchSploit](https://www.kali.org/tools/exploitdb/) | [socat](https://www.kali.org/tools/socat/) | `find` | `insmod` |
| Credential reuse testing | Internal service discovery | `uname` | Loadable kernel modules |
| Browser Developer Tools | [cURL](https://curl.se/) | Container boundary validation | `call_usermodehelper()` |

## Methodology

My general workflow is:

1. Review the room objectives and identify any prerequisite knowledge.
2. Prepare the lab environment and record the relevant connection details.
3. Confirm the target scope before running active testing.
4. Map the application, endpoints, parameters, authentication flow and observable controls.
5. Capture baseline requests and responses before making changes.
6. Identify possible weaknesses and validate them carefully.
7. Build a repeatable exploitation path rather than relying on one-off behaviour.
8. Chain vulnerabilities only within the authorised lab scope.
9. Record commands, observations, failed approaches and important output.
10. Redact flags, answers, credentials, hashes, tokens and other restricted material.
11. Explain why each successful technique worked rather than listing commands without context.
12. Record lessons learned and link the activity to wider cyber security practice.
13. Add defensive recommendations or remediation notes where appropriate.
14. Review the finished writeup for accuracy, clarity and compliance with TryHackMe rules.

The emphasis is on repeatable methodology, clear reasoning and controlled exploitation. Finding the bug is useful; proving the impact safely is where the learning lands properly.

## Content and Spoiler Policy

These notes may contain spoilers, command output, vulnerability details and complete attack or investigation paths. Anyone actively completing a room should attempt it independently before reading the associated writeup.

This repository will not intentionally publish:

- TryHackMe flags or answer strings;
- passwords or cracked credentials;
- reusable session tokens;
- private keys or sensitive certificates;
- certification examination content;
- copied room instructions or substantial portions of TryHackMe material;
- exploit material aimed at real-world unauthorised targets; or
- material that TryHackMe or a room author has asked learners not to share.

> [!NOTE]
> Some values in this writeup have been intentionally redacted to protect the integrity of the challenge and prevent unintended spoilers. Placeholders such as `<TARGET_IP>` and `<TUN0_IP>` are used where IP addresses were required to explain the methodology without exposing environment-specific details. Other sensitive or challenge-revealing information has been replaced with `<REDACTED>`. Any flags have also been redacted, either as `THM{...}` for TryHackMe-style flags or as `<REDACTED>` where the flag format differs.
>
> These redactions allow the process to be explained clearly while ensuring the final answer, challenge secrets, and key identifying details are not disclosed.

If restricted or sensitive information is included accidentally, please report it through the repository's GitHub [Discussions](https://github.com/Valikahn/TryHackMe-Web-Application-Red-Teaming/discussions) area so it can be reviewed and removed.

## Ethical Use Disclaimer

These writeups are for educational purposes only and are based on authorised TryHackMe lab environments.

All tools, commands, techniques, and methodologies referenced in these writeups were used within controlled training environments where permission was provided by the owner and/or operator of the lab platform. The systems discussed are intentionally vulnerable machines designed for cybersecurity learning, practice, and assessment.

Do not use these techniques, tools, or methods against systems, networks, applications, or services that you do not own or do not have explicit written permission to test. Unauthorised access, scanning, exploitation, or disruption of systems is illegal and unethical.

The tools and methods listed in this repository are examples of approaches used during specific rooms or learning exercises. They are not the only possible solutions, and other tools, techniques, or workflows may be used depending on the target environment, room design, and individual methodology.

## Accuracy and Maintenance

TryHackMe periodically updates, replaces or retires rooms and learning paths. Links, room sequences and path content may therefore change after a writeup is published.

Each writeup should be treated as a record of the room as it appeared on the date documented. Where a material change is identified, the relevant page may be updated or marked as archived.

If you notice a broken link, outdated instruction, formatting problem, technical error or any other noticeable issue within a writeup, please report it through the repository's **Issues** tab. When raising an issue, include the name of the affected writeup, a brief description of the problem and, where possible, the relevant section or line.

Constructive corrections are welcome and help keep the repository accurate, useful and maintainable.

## Connect With Me

Thanks for checking out my TryHackMe writeups. These notes form part of my ongoing cybersecurity learning journey, where I document rooms, techniques, tools, mistakes and lessons learned while working through different challenges.

You can view my TryHackMe profile here:

[TryHackMe Profile - V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)

I am also active within cybersecurity learning communities, including Discord, where I discuss labs, tools, methodologies and general security topics with other learners and practitioners.

Feel free to follow my progress, compare approaches or get in touch if you are working through similar rooms.

Walkthrough requests are always welcome, although publication will depend on my availability and whether sharing the content complies with the platform's rules.

Created by V4L1K4HN as part of my cybersecurity learning journey through TryHackMe.

## License

Unless otherwise stated, the original written content in this repository is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/legalcode.en).

Copyright © 2026 V4L1K4HN.

You may share and adapt the licensed material for any purpose, including commercially, provided that:

- appropriate credit is given to V4L1K4HN;
- a link to the license is provided; and
- any changes made to the original material are clearly indicated.

This license applies only to original material created by the repository author. TryHackMe content, branding, room materials, third-party software, trademarks, externally sourced material, and any other third-party intellectual property remain subject to their respective owners' terms and licenses.

See the [LICENSE](./LICENSE) file for the complete legal terms.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
