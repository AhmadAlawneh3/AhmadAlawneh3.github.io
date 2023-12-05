---
title: Hacker's Dungeon Machine Write up
categories: [Write-ups, MachinesCTF]
tags: [CTF,Pentesting]     # TAG names should always be lowercase
comments: false
published: true
---

# Hacker's Dungeon
IP: 10.0.0.133
## **Enumeration**
We start by scanning the machine with nmap.

Discover all open ports:

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image.png)

Check what is exactly running on these ports:

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-1.png)

## **Foothold**

### Port 111 and 2049
Check HackTricks for pentesting rpcbin and nfs.

https://book.hacktricks.xyz/network-services-pentesting/pentesting-rpcbind#rpcbind-+-nfs

https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting#enumeration

List NFS mounts:
```bash
nmap --script="nfs-showmount" 10.0.0.133
```
![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-13.png)

There is one folder `/dungeon`.

Mount this folder. 
```bash
mkdir /tmp/mount
```
```bash
sudo mount -t nfs 10.0.0.133:dungeon /tmp/mount -o nolock
```

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-14.png)

Nothing interesting.

### Port 80
As we saw in our nmap scan that the robots.txt file has one disallowed directory which is `wp-admin` which indecates that there is a wordpress running on port 80.

Visiting the site.

We know the version of wordpress and apache this might be helpfull, but let us look around more.
![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-4.png)

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-5.png)
Nothing seems interesting.

Checking the post by cliking on the date.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-6.png)
We notice that the post was written by  user kasi.

We can confirm that the user kasi exists by trying to login with any random password on `wp-login.php` adn reading the error message.

The different weeor message will be displayed if we try random username and password `test:test`.

![Alt text]/assets/img/machinesCTF/Hackers-Dungeon/(image-7.png)

We try to brute force kasi's password using hydra.

```bash
hydra -l kasi -P /usr/share/wordlists/rockyou.txt 10.0.0.133 http-form-post "/wp-login.php:log=^USER^&pwd=^PASS^:F=incorrect" 
```
![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-8.png)

We got the user's password `dogcat`.

After logging in we land on the profile page of the user, there is nothing interesting there.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-9.png)

Going around the dashboard we find out that we are normal user, so we cant do much.

Let us try and login with the same credentials thru ssh, there is a great chance the password is reused.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-10.png)

And we are in!!

We can find the user.txt flag in the user's home directory.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-11.png)

## **Privilege Escalation**

Running `sudo -l` to check what the user kasi can run using sudo.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-12.png)

Run python simple web server on our attacking machine to host linpeas.sh, to transfer it to the machine.

```bash
python3 -m http.server 80
```

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-15.png)
![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-16.png)

What is root_squash?

root_squash will allow the root user on the client to both access and create files on the NFS server as root. Technically speaking, this option will force NFS to change the client’s root to an anonymous ID and, in effect, this will increase security by preventing ownership of the root account on one system migrating to the other system. This is needed if you are hosting root filesystems on the NFS server (especially for diskless clients); with this in mind, it can be used (sparingly) for selected hosts, but you should not use no_root_squash unless you are aware of the consequences.

[Read about the vulnerability.](https://www.thegeekdiary.com/basic-nfs-security-nfs-no_root_squash-and-suid/)

We need to make a binary that will set our user id to 0 and givves us a shell as root.

```C
#include <stdio.h>
int main(void){
    setreuid(0,0);
    system("/bin/bash");
    return 0;
      }
```

Compile it using gcc.

```bash
gcc test.c -o test 
```

As the root user on the attacking machine copy the binray `test` to the mounted directory, and set the SUID bit for it.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-17.png)
![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-18.png)

Back to the target machine on /dungeon we can find the binary and run it.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-19.png)

We can find the root.txt flag in the root user's home directory.

![Alt text](/assets/img/machinesCTF/Hackers-Dungeon/image-20.png)