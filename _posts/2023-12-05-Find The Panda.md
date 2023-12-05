---
title: Find The panda Machine Write up
categories: [Write-ups, MachinesCTF]
tags: [CTF,Pentesting]     # TAG names should always be lowercase
comments: false
published: true
---

# **Find The Panda**
## **Enumeration**
We start by scanning the machine with nmap.

Discover all open ports:

![Alt text](/assets/img/machinesCTF/find-the-panda/image.png)

Check what is exactly running on these ports:

![Alt text](/assets/img/machinesCTF/find-the-panda/image-1.png)

## **Foothold**
### Port 1337
Port 1337 is running vsftpd 2.3.4, login as anonymous is allowed and apperantely the root direectory of the services is the root dierctory of the machine!!

Either we login thru ftp as anonymous and download user panda ssh private key or we exploit the vulnerbility on this version of vsftpd.

Using search sploit we find a CVE for this version of vsftpd.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-2.png)

Get the poc to our working directory.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-3.png)

Reading the code to see if we need to do some changes.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-4.png)

We should change the port because ftp is not running on the default port.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-5.png)

After running the code now we have RCE on the target and run id command.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-6.png)

Get user panda ssh key.

Copy it to our attakcing machine.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-8.png)

![Alt text](/assets/img/machinesCTF/find-the-panda/image-9.png)

![Alt text](/assets/img/machinesCTF/find-the-panda/image-10.png)

Set up the needed permissions to the file.

```bash
chmod 600 id_rsa
```

Login using the key.
```bash
ssh -i id_rsa panda@10.0.0.133
```
![Alt text](/assets/img/machinesCTF/find-the-panda/image-12.png)

Now we can find the user.txt flag in /home/panda.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-13.png)

## **Privilege Escalation**

Running `sudo -l` to check what the user panda can run using sudo.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-14.png)

Searching for binaries that have SUID bit set.

```bash
find / -perm -04000 2>/dev/null
```

We find base64.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-16.png)

We can use it to read files.

```bash
base64 <file> | base64 -d
```
https://gtfobins.github.io/gtfobins/base64/#suid

Let us try to read `/etc/shadow` file.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-18.png)

We can directly read the root flag since we know what its name.

![Alt text](/assets/img/machinesCTF/find-the-panda/image-17.png)
