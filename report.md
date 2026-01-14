# Cowrie Honeypot Report (SSH Threat Telemetry)

**Author:** Jakov  
**Honeypot:** Cowrie (SSH) on VPS  
**Observation window:** 2026-01-01 to 2026-01-14 (timestamps in UTC)  
**Data sources:** `cowrie.json*` and `cowrie.log*`

---

## Executive summary

A Cowrie SSH honeypot was deployed on a public VPS to observe real-world SSH scanning and credential attacks. Over the observation window, the honeypot recorded **44,786 inbound SSH connection attempts** originating from **1,002 unique source IP addresses**. Attack behavior was dominated by automation: repeated password spraying, rapid disconnects, and standardized host-fingerprinting commands after login.

> Important note: Cowrie is a **decoy**. “Successful logins” in this report indicate attackers authenticated to the **honeypot**, not to the host OS.

---

## Environment and setup (high level)

- Public VPS running Linux (Ubuntu)
- Cowrie configured to listen on **2222**
- Real SSH administrative access was moved to a separate high, non-standard port to isolate it from the honeypot.
- Port redirection to ensure unsolicited SSH traffic is captured by Cowrie

---

## Key totals

- **Total events:** 273,823
- **SSH connections:** 44,786
- **Unique attacker IPs:** 1,002
- **Login attempts (failed + success):** 43,217
- **Failed logins:** 32,035
- **Successful honeypot logins:** 11,182  *(decoy only)*
- **Captured command inputs:** 21,930

---

## Daily activity

| Date (UTC) | Connections | Unique IPs | Failed logins | Successful logins |
|---|---:|---:|---:|---:|
| 2026-01-01 | 395 | 17 | 363 | 13 |
| 2026-01-02 | 2615 | 84 | 1743 | 766 |
| 2026-01-03 | 1138 | 77 | 800 | 240 |
| 2026-01-04 | 2451 | 84 | 1982 | 355 |
| 2026-01-05 | 3033 | 102 | 2045 | 675 |
| 2026-01-06 | 3683 | 96 | 3161 | 438 |
| 2026-01-07 | 3976 | 108 | 3496 | 401 |
| 2026-01-08 | 1756 | 111 | 1269 | 385 |
| 2026-01-09 | 1939 | 91 | 1696 | 220 |
| 2026-01-10 | 1594 | 93 | 1113 | 333 |
| 2026-01-11 | 9095 | 116 | 8469 | 499 |
| 2026-01-12 | 9841 | 102 | 3115 | 6463 |
| 2026-01-13 | 968 | 83 | 834 | 112 |
| 2026-01-14 | 2302 | 80 | 1949 | 282 |


Notable spikes occurred on **2026-01-11** and **2026-01-12** (UTC), driven by high-volume automated credential attempts.

---

## Credential attack analysis

### Top usernames (failed attempts)

| Username | Failed attempts |
|---|---:|
| admin | 2300 |
| user | 1367 |
| oracle | 1232 |
| postgres | 1115 |
| ubuntu | 981 |
| test | 964 |
| root | 702 |
| guest | 671 |
| mysql | 655 |
| debian | 630 |

### Top passwords (failed attempts)

| Password | Failed attempts |
|---|---:|
| 123456 | 3453 |
| password | 1656 |
| 123 | 1614 |
| 12345 | 1440 |
| 1234 | 1194 |
| 1 | 928 |
| qwerty | 809 |
| 123456789 | 632 |
| 12345678 | 611 |
| 654321 | 540 |

### Most common successful credential pairs (honeypot)

| Credential (user, pass) | Count |
|---|---:|
| ('root', 'password') | 236 |
| ('root', 'admin') | 199 |
| ('root', 'qwerty') | 179 |
| ('root', '12345') | 178 |
| ('root', '123456789') | 172 |
| ('root', '1234') | 140 |
| ('root', '12345678') | 137 |
| ('root', '123') | 126 |
| ('root', 'toor') | 122 |
| ('root', '111111') | 118 |

**Interpretation:** attackers heavily target common default accounts (e.g., `admin`, `ubuntu`, `oracle`, `postgres`) and the most common leaked passwords (`123456`, `password`, `qwerty`, etc.). The concentration of “success” on `root` suggests automated tools repeatedly trying weak root credentials.

---

## Attacker tooling (SSH client fingerprints)

Cowrie captures client “version strings” which often reveal the library/tool used.

| SSH client version string | Count |
|---|---:|
| SSH-2.0-Go | 39552 |
| SSH-2.0-libssh2_1.10.0 | 1279 |
| SSH-2.0-phpseclib_1.0 (openssl) | 417 |
| SSH-2.0-libssh_0.11.1 | 357 |
| SSH-2.0-phpseclib_1.0 (openssl, gmp) | 348 |
| SSH-2.0-JSCH_0.2.24_RAPIDSEVEN | 267 |
| SSH-2.0-OpenSSH_10.0 | 238 |
| SSH-2.0-libssh2_1.11.1 | 208 |
| SSH-2.0-libssh2_1.9.0 | 113 |
| SSH-2.0-OpenSSH_8.9p19 | 72 |

**Observation:** The most frequent client string was `SSH-2.0-Go`, consistent with large-scale scanning bots. A small amount of non-SSH noise was also observed (e.g., `GET / HTTP/1.1`), likely from opportunistic scanners hitting open ports.

---

## Post-login behavior (command telemetry)

After a successful honeypot login, many actors immediately run host fingerprinting commands.

### Top observed commands (sample)

| Command | Count |
|---|---:|
| echo -e "\x6F\x6B" | 5750 |
| uname -s -v -n -m 2 > /dev/null | 2747 |
| uname -m 2 > /dev/null | 2747 |
| cat /proc/uptime 2 > /dev/null \| cut -d. -f1 | 2747 |
| uname -s -v -n -r -m | 788 |
| uname -a | 338 |
| /bin/./uname -s -v -n -r -m | 300 |
| uname -s -v -n -r | 117 |
| grep "model name" /proc/cpuinfo \| cut -d ' ' -f3- \| awk '{print $1,$2,$3,$4,$5,$6,$7,$8,$9,$10}' \| head -1 | 117 |
| nproc | 117 |

### Reconnaissance patterns
Common follow-up actions included:
- Kernel/OS fingerprinting (`uname ...`)
- CPU enumeration (`/proc/cpuinfo`, `nproc`)
- Host identity checks (`hostname`, `whoami`, `id`)

### Suspicious patterns (high level, redacted)
A smaller subset attempted actions consistent with persistence or payload staging:
- `wget`/`curl` download-and-execute style one-liners *(URLs and payloads redacted in this report)*
- Modifying `~/.ssh/authorized_keys` to establish persistence *(redacted)*

---

## Defensive takeaways

1. **Public SSH is continuously attacked.** Once a host is reachable, it will be scanned repeatedly.
2. **Passwords are the primary target.** Weak/default credentials are immediately tested at scale.
3. **Key-based SSH + rate limiting is essential.** Use SSH keys, disable password auth, and add controls such as fail2ban and allowlists.
4. **Separate admin access from exposure.** Moving the real SSH daemon to a non-default port (plus firewall allowlisting) reduces noise and risk; honeypots capture the noise safely.

---

## Appendix: How to reproduce this analysis (recommended)

- Store raw logs under `raw-logs/`
- Parse `cowrie.json*` (JSON-lines) into counts for:
  - connections per day
  - usernames/passwords attempted
  - client version strings
  - post-login command sequences

