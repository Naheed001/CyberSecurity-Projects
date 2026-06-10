# 🛡️ Reverse Shell Network Detection — Fundamentals

# Overview

This page covers all the fundamental concepts needed for the **Reverse Shell Network Detection** lab. It includes theory on reverse shells, tools used, traffic analysis, real-world attack delivery methods, and detection strategies.

> 💡 **Lab Tools:** Kali Linux · Netcat · Metasploit (msfvenom + msfconsole) · Wireshark
> 

---

# 1. Bind Shell vs Reverse Shell

## What Is a Bind Shell?

The **victim machine opens a port and waits** for the attacker to connect inward. The attacker initiates the connection.

## What Is a Reverse Shell?

The **victim machine reaches out and connects to the attacker**. The attacker listens, and the victim calls home.

<aside>

🔑 **Why does this matter?** Firewalls typically block *inbound* connections but are far more permissive with *outbound* ones. Reverse shells exploit this — a victim behind a firewall can call out to an attacker on port 80 or 443 and the firewall waves it through without question.

</aside>

## The Real World Analogy

> A kidnapper can't walk into someone's house and grab them — security (firewall) stops them at the door. So instead they trick the person into **walking outside themselves** and getting into the kidnapper's car. The security guard sees someone leaving voluntarily and does nothing.
> 

## Why Attackers Prefer Reverse Shells

- **Firewalls become useless** — outbound traffic is rarely restricted
- **Works behind NAT** — attacker doesn't need to know victim's real IP
- **Blends in with normal traffic** — especially on ports 80/443
- **Full control with no authentication** — no username prompt, no password, no login screen

---

# 2. Netcat — The Swiss Army Knife

Netcat (`nc`) is a raw TCP/UDP utility. It can open listening ports, connect to remote ports, and send/receive data — including a shell. It is deliberately simple by design, which makes it one of the most versatile tools in both offensive and defensive security.

## Two Key Modes

| Mode | Command | Purpose |
| --- | --- | --- |
| **Listener** | `nc -lvnp <port>` | Sit and wait for a connection |
| **Connect** | `nc <ip> <port>` | Reach out to a remote host |

## Important Flags

| Flag | Meaning |
| --- | --- |
| `-l` | Listen mode — wait for an incoming connection |
| `-v` | Verbose output — show connection events in the terminal |
| `-n` | No DNS lookups — use raw IP addresses only |
| `-p` | Specify the port number (listener mode only) |
| `-e` | Attach a program (e.g. `/bin/bash`) to the connection |
| `-u` | UDP mode instead of TCP |
| `-w` | Set a timeout in seconds — connection closes after this if idle |
| `-z` | Zero-I/O mode — sends no data, used for port scanning |
| `-k` | Keep listening after a client disconnects (persistent listener) |

<aside>

⚠️ The `-e` flag is only available in **netcat-traditional**. The OpenBSD version deliberately removes it for security reasons. Always install `netcat-traditional` for lab work.

</aside>

## How to Use Netcat — Common Workflows

Netcat's power comes from combining its flags and piping it with other tools. Below are the main workflows you should understand.

### Workflow 1 — Reverse Shell (Attacker + Victim)

The core workflow of this lab. Two steps always required:

**Step 1 — Attacker sets up the listener first.**

Think: which flags open a port, keep it verbose, skip DNS, and specify the port number?

**Step 2 — Victim connects outward and attaches a shell.**

Think: you need the attacker's IP, the port, and the flag that hands over `/bin/bash`.

> Always start the listener *before* triggering the victim — if nothing is listening when the victim connects, the connection is refused immediately.
> 

---

### Workflow 2 — Simple Connectivity Test

Before running a full reverse shell, verify two machines can reach each other on a specific port. Netcat lets you do this cleanly — no need for a full shell setup.

**Attacker:** open a listener on a port

**Victim:** connect to that port

If text you type on one side appears on the other — the network path is clear. No `-e` flag needed here.

> This is equivalent to `ping` but for a specific TCP port. Useful when a firewall allows TCP but blocks ICMP.
> 

---

### Workflow 3 — File Transfer

Netcat can transfer files between machines without SCP or FTP — useful when those tools aren't available on the victim.

**Receiver side:** listen on a port and redirect incoming data into a file using output redirection (`>`)

**Sender side:** connect to the receiver and redirect a file into the connection using input redirection (`<`)

> Think about how Linux redirection (`<` and `>`) lets you pipe file contents into and out of commands. Netcat is just the pipe between two machines.
> 

---

### Workflow 4 — Banner Grabbing

Netcat can connect to any open port and read whatever the service sends back on connection — its *banner*. This reveals what software and version is running on that port.

Think: connect to a target IP and a well-known port (e.g. 80, 22, 25). What does the service say when you first connect?

> Defenders use this too — banner grabbing is a standard technique in both reconnaissance and service auditing. Web servers, mail servers, and SSH daemons all announce themselves on connection.
> 

---

### Workflow 5 — Port Scanning (Zero-I/O Mode)

With the `-z` flag, Netcat sends no data — it only checks whether a port is open or closed. Combined with a port range and `-v`, this gives basic scan output.

> This is far less capable than Nmap but useful to understand. The `-z` flag is the entire mechanism — think about what "zero I/O" means and why that makes it safe to use as a scanner.
> 

---

## Version Note

`nc` is the binary name — not `netcat`. This is common in Linux where the full tool name and the command you type differ. Even after installing `netcat-traditional`, the system may still point `nc` to the OpenBSD version via the alternatives system. Check what's installed and what's active with:

```bash
ls -la /usr/bin/nc*
```

If needed, call the traditional version directly by its full path rather than the `nc` shortcut.

---

# 3. Metasploit — Payloads & Listeners

Metasploit is the industry standard framework for penetration testing. It contains over 2,600 exploits, 1,700+ payloads, and hundreds of auxiliary and post-exploitation modules. For this lab, two components are used: **msfvenom** for generating payloads and **msfconsole** for catching connections and running post-exploitation.

## Two Key Components

### msfvenom — Payload Generator

Creates malicious executables or shellcode. Used to craft a reverse shell payload for the victim to run.

### msfconsole with multi/handler — The Listener

Catches the incoming connection on the attacker side when the victim executes the payload.

---

## How to Use msfvenom

msfvenom is a command-line tool run outside of msfconsole. It generates a self-contained payload file in a single command.

### Core Structure

Every msfvenom command follows the same pattern:

```
msfvenom -p [payload] [OPTIONS] -f [format] -o [output file]
```

### Key msfvenom Flags

| Flag | Purpose |
| --- | --- |
| `-p` | Payload to use (e.g. `linux/x86/meterpreter/reverse_tcp`) |
| `LHOST=` | Attacker IP — burned into the payload at generation time |
| `LPORT=` | Attacker port — burned into the payload at generation time |
| `-f` | Output format — `elf` for Linux, `exe` for Windows, `py` for Python, `raw` for shellcode |
| `-o` | Output file path where the payload is saved |
| `-e` | Encoder to apply — used for antivirus evasion |
| `-i` | Number of encoding iterations — more iterations = more obfuscation |
| `-x` | Template executable to inject the payload into |
| `--list payloads` | List all available payloads |
| `--list formats` | List all available output formats |
| `--list encoders` | List all available encoders |

### Payload Naming Convention

Metasploit payload names follow a strict pattern you should be able to read fluently:

```
[platform] / [architecture] / [payload type] / [connection type]

linux  / x86 / meterpreter / reverse_tcp
windows / x64 / meterpreter / reverse_https
```

> Before generating any payload, ask: what OS is the target running? What architecture? Staged or stageless? These decisions determine the entire payload string.
> 

---

## How to Use msfconsole

msfconsole is an interactive command-line interface. You navigate it like a structured menu — searching for modules, selecting them, setting options, and running them.

### Core Navigation Commands

| Command | Purpose |
| --- | --- |
| `search [keyword]` | Search for modules by name, CVE, or platform |
| `use [module path]` | Select a module to work with |
| `info` | Show full details about the currently selected module |
| `show options` | Show all configurable options for the current module |
| `show payloads` | Show all compatible payloads for the current module |
| `set [OPTION] [value]` | Set a configuration value |
| `unset [OPTION]` | Clear a previously set value |
| `setg [OPTION] [value]` | Set a value globally — persists across module changes |
| `run` or `exploit` | Execute the current module |
| `back` | Go back to the main menu without losing settings |
| `sessions` | List all active sessions |
| `sessions -i [id]` | Interact with a specific session by its ID number |
| `sessions -k [id]` | Kill a specific session |
| `help` | Show all available commands |

### The Standard Workflow

Every Metasploit engagement follows the same sequence:

```
1. search      → find the right module
2. use         → select it
3. show options → see what needs configuring
4. set         → configure LHOST, LPORT, PAYLOAD etc.
5. run         → execute
```

> `show options` is the most important habit to build. It shows exactly what is required (marked `yes`) vs optional. Missing a required option is the most common reason modules fail silently with no clear error message.
> 

---

## Meterpreter Commands

Once a Meterpreter session is open, you have access to a rich set of built-in commands that go far beyond a basic shell. These run inside the Meterpreter agent itself — they don't spawn child processes, making them harder for endpoint tools to detect.

### System Information & User Context

| Command | What it tells you |
| --- | --- |
| `sysinfo` | OS, hostname, architecture, Meterpreter version |
| `getuid` | The user the session is currently running as |
| `getpid` | Process ID of the Meterpreter process on the victim |
| `ps` | List all running processes on the victim machine |
| `getsystem` | Attempt automatic privilege escalation to SYSTEM/root |
| `getprivs` | List current privileges held by the session |

### File System

| Command | What it does |
| --- | --- |
| `pwd` | Print current directory on the victim |
| `ls` | List files in the current directory |
| `cd [path]` | Change directory on the victim |
| `upload [file]` | Upload a file from attacker machine to victim |
| `download [file]` | Download a file from victim to attacker machine |
| `search -f [pattern]` | Search for files matching a pattern (e.g. `*.txt`, `*password*`) |
| `cat [file]` | Read a file's contents directly in the session |
| `rm [file]` | Delete a file on the victim |

### Shell & Execution

| Command | What it does |
| --- | --- |
| `shell` | Drop into a native system shell (bash on Linux, cmd on Windows) |
| `execute -f [cmd]` | Execute a command on the victim without dropping to a full shell |
| `migrate [pid]` | Move the Meterpreter process into another running process |

> `migrate` is a critical stealth technique — moving into a legitimate process like `explorer.exe` or `svchost.exe` makes Meterpreter much harder to detect and can survive reboots if the target process starts automatically.
> 

### Network

| Command | What it does |
| --- | --- |
| `ipconfig` | Show network interfaces on the victim machine |
| `arp` | Show ARP table — reveals other hosts on the victim's local network |
| `netstat` | Show active network connections on the victim |
| `portfwd` | Forward a port through the victim — used for pivoting to internal hosts |
| `route` | View or modify the victim's routing table |

### Post-Exploitation Modules

Meterpreter can run full Metasploit post-exploitation modules without leaving the session:

```
run post/[module path]
```

| Module | What it does |
| --- | --- |
| `post/multi/recon/local_exploit_suggester` | Scans the victim for known privilege escalation vulnerabilities |
| `post/linux/gather/hashdump` | Dump password hashes from a Linux system |
| `post/multi/gather/env` | Collect environment variables from the victim |
| `post/multi/manage/shell_to_meterpreter` | Upgrade a basic shell session into a full Meterpreter session |

### Session Management

| Command | What it does |
| --- | --- |
| `background` | Send session to background — returns to msfconsole prompt |
| `exit` | Close the Meterpreter session entirely |
| `help` | Show all Meterpreter commands available in this session |

<aside>

💡 **Always run `help` in a new Meterpreter session.** The available commands depend on the target OS and payload type — not every command above is available in every session. `help` shows exactly what your current context supports.

</aside>

---

## Key Vocabulary

| Term | Meaning |
| --- | --- |
| **Payload** | The code that runs on the victim machine |
| **LHOST** | Attacker IP — where the victim calls back to |
| **LPORT** | Attacker port — where you're listening |
| **Meterpreter** | Advanced shell payload — runs in memory, TLS encrypted, harder to detect |
| **ELF** | Linux executable format (`.elf`) — equivalent to `.exe` on Windows |
| **Module** | Any individual component in Metasploit — exploit, auxiliary, post, or payload |
| **Stage** | The second part of a staged payload — downloaded from the handler at runtime |
| **Handler** | The `multi/handler` listener that catches incoming payload connections |
| **Session** | An active connection between Metasploit and a compromised machine |
| **Pivot** | Using a compromised machine to attack further machines on its internal network |


## Netcat vs Meterpreter

| Feature | Netcat Shell | Meterpreter Shell |
| --- | --- | --- |
| **Encryption** | None — plain text | TLS encrypted |
| **Wireshark visibility** | Every command readable | Unreadable gibberish |
| **Runs in memory** | No | Yes — never touches disk after execution |
| **Detectability** | Trivially easy | Much harder |
| **Built-in commands** | None — basic shell only | Rich built-in toolkit |
| **File transfer** | Requires manual piping | Built-in `upload` / `download` |
| **Privilege escalation** | Manual — run system commands yourself | `getsystem`  • automated post modules |
| **Real world use** | Rarely by serious attackers | Industry standard for post-exploitation |

---

# 4. Wireshark — Packet Capture Analysis

Wireshark captures and analyses raw network packets in real time.

## Key Concepts

| Concept | Description |
| --- | --- |
| **PCAP** | Packet Capture file format |
| **Capture filter** | Filters traffic *as it's being recorded* |
| **Display filter** | Filters what you *see* after capture (e.g. `tcp.port == 4444`) |
| **Follow TCP Stream** | Reconstructs the full conversation between two hosts |

## Follow TCP Stream

Right-click any packet → **Follow → TCP Stream**

This stitches all packets back into a readable conversation. For a Netcat reverse shell, you can **literally read every command the attacker typed and every response the victim's machine sent back**.

## Colour Coding in TCP Stream

- 🔴 **Red text** — data sent from the attacker (commands typed)
- 🔵 **Blue text** — data sent from the victim (responses returned)

---

# 5. What Suspicious Reverse Shell Traffic Looks Like

This is the detection side — what to train your eye to spot as a defender.

## Key Indicators of Compromise (IOCs)

- **Unusual outbound connections** on non-standard ports (e.g. 4444, 1234, 9001)
- **Persistent long-lived TCP connections** — a shell session stays open for minutes or hours
- **Interactive-looking traffic** — small packets back and forth (someone typing commands)
- **Asymmetric data** — small requests, larger responses (command sent, output returned)
- **Unexpected processes initiating connections** — `/bin/bash` or a random `.elf` file connecting outbound is a massive red flag

---

# 6. Real World Attack Delivery Methods

In a real attack, the victim doesn't type the reverse shell command themselves — they're tricked into running a payload. Here are the main delivery methods.

## Method 1 — Malicious Email Attachment (Most Common)

- How it works
    
    The attacker embeds a payload inside a convincing attachment:
    
    - **Macro-enabled Word/Excel document** — victim opens file, enables macros, macro silently executes payload
    - **PDF with embedded JavaScript** — exploits vulnerabilities in PDF readers
    - **Zipped executable** — renamed to look like an invoice (`Invoice_2024.pdf.exe`)
    
    The email creates urgency or legitimacy:
    
    > *"Your account will be suspended — action required immediately"*
    > 

## Method 2 — Malicious Link

- How it works
    
    The payload isn't in the link — the link is a **door** leading to a server configured to compromise whoever visits it:
    
    - **Forced download** — server automatically pushes a file when victim visits
    - **Drive-by download** — just visiting the page executes the payload (no click needed)
    - **Fake login page** — pixel-perfect clone of a legitimate site harvests credentials
    - **URL obfuscation** — layers of redirects hide the malicious destination
 
  ⚠️ **Critical misconception:** The padlock in a browser means the *connection is encrypted* — it says nothing about whether the website itself is malicious.

## Method 3 — Trojanised Software

- How it works
    
    Legitimate software repackaged with a payload bundled inside:
    
    - Cracked software / games
    - Fake VPN clients
    - Pirated productivity tools
    
    Software works perfectly — giving the victim no reason to be suspicious.
    

## Method 4 — USB Drop Attack

- How it works
    
    USB drive loaded with a payload left somewhere the target will find it:
    
    - Office car park
    - Reception desk
    - Conference room
    
    Labelled something enticing like *"Salary Review 2024"*. Human curiosity does the rest.
    

## Method 5 — Watering Hole Attack

- How it works
    1. Identify websites the target regularly visits
    2. Compromise that website and inject a payload
    3. Wait for the target to visit naturally
    
    The victim trusts the site completely — highly effective against high-value targets.
    

## Method 6 — Spear Phishing

- How it works
    
    Highly personalised email crafted using OSINT from LinkedIn, Twitter, company websites:
    
    > *"Hi [Name], following up from the CyberSec conference last week — here's the research paper we discussed."*
    > 
    
    Far more effective than mass phishing because it feels legitimate and personal.
    

---

## Antivirus Evasion Techniques

| Technique | Description |
| --- | --- |
| **Obfuscation** | Scrambles payload code so it looks different but does the same thing |
| **Encryption** | Payload only decrypts in memory at runtime — never touches disk in recognisable form |
| **Polymorphism** | Payload changes its own code every time it runs |
| **Living off the land** | Uses legitimate Windows tools (`PowerShell`, `certutil`) to execute payload |

---

# 7. Attack Tools Reference

| Tool | Purpose |
| --- | --- |
| **msfvenom** | Generates payloads — what we used to create `shell.elf` |
| **SET (Social Engineering Toolkit)** | Automates phishing campaigns, creates malicious documents, clones websites |
| **GoPhish** | Professional phishing simulation framework for legitimate pentesters |
| **Empire / Covenant** | Advanced post-exploitation frameworks with built-in payload delivery |

---

# 8. Defensive Countermeasures

## By Attack Vector

| Attack | Defence |
| --- | --- |
| Malicious email | Email filtering, disable macros by default, user training |
| Drive-by download | Keep browser patched, use ad blockers, browser sandboxing |
| Trojanised software | Only install from trusted sources, application whitelisting |
| USB drop | Disable AutoRun, physical security policies |
| Watering hole | Web filtering, network monitoring |
| Smishing/Vishing | User awareness training, callback verification procedures |

## By Detection Method

| Indicator | What It Tells You |
| --- | --- |
| Non-standard port (4444, 4445) | Likely C2 communication |
| Long-lived TCP connection | Interactive shell session |
| `/bin/bash` making outbound connections | Process should never do this |
| Encrypted traffic on unusual port | Possible Meterpreter or similar |
| Small requests, large responses | Command-response pattern of a shell |

---

# 9. Legal & Ethical Boundaries

<aside>

⚠️ Everything covered in this lab is **illegal without explicit written permission** from the system owner. In professional security this requires a signed **Rules of Engagement** document before a single command is run.

In the UK, unauthorised access is a criminal offence under the **Computer Misuse Act 1990**.

Lab work is legal because **you own both machines**.

</aside>

---

# 10. Key Takeaways

- The **weakest link is always the human**, not the technology — this is why security awareness training is one of the most effective defensive measures
- **Meterpreter is significantly harder to detect** than Netcat because it encrypts all traffic and runs entirely in memory
- A **padlock in a browser ≠ safe** — it only means the connection is encrypted
- **Reverse shells exploit outbound firewall permissiveness** — the victim walks out to the attacker
- Even if a defender captures every packet of a Meterpreter session, they **cannot read what commands were run** without deeper tooling

---
