# Little Rat | CVE‑2025‑61927 CTF Challenge Suite

This repository presents an **explanation** of **proof‑of‑concept vulnerable application** and the corresponding attack chain used in three forensics/OSINT CTF tasks (**Little Rat 1, 2, 3**). The application demonstrates a sandbox escape vulnerability in `happy-dom` and how it can be combined with legitimate services (GitHub, Cloudflare Tunnel, filebin.net) to achieve credential dumping and data exfiltration – all while appearing as a normal “blogspot” SSR app.

**Disclaimer**  
This code was developed exclusively for the FarEastCTF 2025 competition. The vulnerability is based on the official CVE‑2025‑61927 PoC and was created after the patch was released. To avoid the distribution of weaponized exploit code, this repository contains only the **design, CTF tasks, and forensics artifacts overviews** – not the full source of the exploit payloads.

---

## The Vulnerability – CVE‑2025‑61927 (Overview)

The `happy-dom` library (versions prior to the patch) allows JavaScript inside a server‑rendered `<script>` tag to escape its sandbox via prototype chain manipulation. This gives the attacker full access to the Node.js `child_process` module, enabling arbitrary command execution on the host.

All payloads in this challenge are delivered via this mechanism.  
*(For technical details of the CVE, please refer to the official advisory: [GHSA‑37j7‑fg3j‑429f](https://github.com/capricorn86/happy-dom/security/advisories/GHSA-37j7-fg3j-429f). The actual exploit code is not disclosed here.)*

---

## Assumptions

- Windows security of the 2019 year and earlier (before Tamper Protection was introduced).
- The user PC runs both client and server locally (the attack affects the user’s own machine).
- The attacker fully controls the domain used for license URL updates (via GitHub commit messages).
- Only legitimate third‑party services are used for C2 and exfiltration (GitHub, Cloudflare, filebin.net).
- Installation is voluntary – the user grants admin rights because they believe it is a normal setup.
- Admin rights are used only during installation; after reboot the attacker operates with user rights.
- The LSASS dump is copied to a user‑accessible file during the admin phase (bypassing ACL restrictions).
- The vulnerable app remains running (the victim uses the blog regularly).

---

## What it does

The application is a simple **Server‑Side Rendered (SSR) blog** that uses `happy-dom` to render HTML on the Node.js backend. Because of a sandbox‑escape vulnerability, an attacker can inject JavaScript that executes **inside the Node.js process** – leading to remote command execution, registry manipulation, and file exfiltration.

The attack chain is disguised as a legitimate software installation:

1. The victim runs an installer with admin rights (required for Node.js system-wide installation).
2. The installer disables Windows Defender real‑time monitoring, dumps LSASS memory, and stores the dump as a user‑accessible file.
3. Node.js and the vulnerable app are installed; the machine reboots.
4. The victim launches the “blogspot” app, which periodically fetches a C2 URL from a public GitHub commit message.
5. The C2 sends JavaScript payloads that exploit vulnarability, executing commands on the victim’s PC and exfiltrating data via logging or `filebin.net`.

From the attacker’s perspective, only legitimate services are used – no custom infrastructure is required.

---

## CTF Tasks & Flags

The challenge suite is divided into three forensics/OSINT tasks, each revealing a flag.

### Little Rat 1 – Registry License Theft
- **Description (CTF)**: *“Half of the journalists lost their license keys – find the original key.”*
- **Attack step**: The payload reads `HKCU\Environment\Flag`, then overwrites it with a dummy value.
- **Flag**: `FECTF{Le7s_aSSuMe_THIs_1s_4_1icEn5e_K3y}` (Base64‑encoded in the registry event log).

### Little Rat 2 – Exfiltrated File Name
- **Description (CTF)**: *“Computers started lagging – find what file was stolen.”*
- **Attack step**: Among many decoy commands, `curl` uploads the LSASS dump copy to `filebin.net`.
- **Flag**: `FECTF{YouDidn'tSeeMe.meow}` (the name of the uploaded file).

### Little Rat 3 – Proving the Data Exfiltration
- **Description (CTF)**: *“Analyse the network traffic and prove data exfiltration.”*
- **Attack step**: The `filebin.net` upload returns an ID; the wrapped archive contains a stack trace with the flag.
- **Flag**: `FECTF{yBbl_K0H7ENH3P_CJI0M4J1cR}` (found inside `wrapped.txt` after fixing the ZIP header).

For a detailed walkthrough of each task, see the individual `WriteUp.md` files.

---

## How the Attack Chain Works (Overview)

1. **Installation (admin rights)**  
   - The installer disables Real‑time Monitoring.  
   - Dumps LSASS to `C:\Windows\Temp\lsass_dump.dmp`.
   - Copies raw bytes to a user‑readable file (e.g., `YouDidn'tSeeMe.meow`).
   - Installs Node.js and the vulnerable app.
   - Reboots (cleanup via `RunOnce`).

2. **Post‑reboot (user rights)**
   - Victim starts the `blogspot` app.
   - The app fetches the C2 URL from a GitHub commit message.
   - Periodically asks the C2 for payloads (POST `/C2`).

3. **Payload execution**
   - The C2 returns a `<script>` containing the sandbox‑escape exploit.
   - The SSR function injects it into the HTML → escape happens.
   - **Task 1 payload**: reads and replaces registry `Flag`, returns stolen value.
   - **Task 2 payload**: executes many benign commands + a target `curl` upload of the LSASS copy to `filebin.net`.
   - The upload ID is sent back to the C2.

4. **Exfiltration**
   - The LSASS dump is available at `https://filebin.net/<id>`.
   - After removing garbage bytes and fixing the ZIP header, the stack trace reveals the final flag.

---

## MITRE ATT&CK Techniques

| Technique ID | Technique Name | Tool Used | Attacker’s Abuse |
|--------------|----------------|-----------|------------------|
| T1059.003 | Command and Scripting Interpreter | `cmd.exe`, `powershell.exe` | Execute encoded PowerShell commands |
| T1562.001 | Impair Defenses: Disable Tools | `Set-MpPreference` | Disable Windows Defender real‑time monitoring |
| T1003.001 | OS Credential Dumping: LSASS | `rundll32.exe comsvcs.dll, MiniDump` | Dump LSASS memory (admin phase) |
| T1027 | Obfuscated Files or Information | Base64 encoding | Hide malicious commands from logs |
| T1102 | Web Service | GitHub API, Cloudflare Tunnel, filebin.net | C2 discovery, C2 hosting, data exfiltration |
| T1190 | Exploit Public‑Facing Application | `happy-dom` | Sandbox escape to run Node.js code |
| T1055 | Process Injection | Prototype chain manipulation | Escape sandbox and access `child_process` |
| T1041 | Exfiltration Over C2 Channel | POST to `/C2` | Send stolen license and filebin.net ID |

---

## Afterword

This project was created as a self‑contained, realistic scenario for CTF challenge to study the reverse side of forensic analysis of multi‑stage attacks that blend legitimate services, sandbox escapes, and post‑exploitation techniques. The write‑ups are archived here **without including the full code and exploit payloads**, in order to avoid the distribution of weaponized code.