---
title: HTB Devvortex Machine Write up
categories: [Write-ups, HackTheBox]
tags: [HTB]     # TAG names should always be lowercase
comments: false

---

<img src="/assets/img/HTB/devvortex/2565d292772abc4a2d774117cf4d36ff.png" style="margin-left: 20px; zoom: 60%;" align=left alt="Machine image" >

# Devvortex
# Difficulty: easy
<b>4<sup>th</sup> DEC 2023</b><br>
<b>IP: 10.10.11.242</b>

## **Enumeration**

```bash
nmap -Pn -T4 -sVC 10.10.11.242 
```

![](/assets/img/HTB/devvortex/20231203225603.png)

## **Foothold**

When accessing the IP directly thru the browser, it showed that we should resolve the domain devvortex.htb.
![](/assets/img/HTB/devvortex/20231203224326.png)

Adding `10.10.11.242 devvortex.htb` to /etc/hosts file.

```bash
sudo nano /etc/hosts
```
![](/assets/img/HTB/devvortex/20231204124522.png)

Then we refresh the page.

![](/assets/img/HTB/devvortex/20231204124157.png)

A well designed website. Going around the site, checking linked pages and forms.
It is just a static html, there is nothing interesting.
### Directory enumeration
```bash
gobuster dir -u http://devvortex.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,txt -t 50
```
![](/assets/img/HTB/devvortex/20231203231220.png)

No new results, we can check images, js and css directories but we will get 403 Forbidden.

![](/assets/img/HTB/devvortex/20231204125123.png)

### VHOST enumeration
There might me another sites running on the same server.
```bash
gobuster vhost -u devvortex.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt  --append-domain
```
![](/assets/img/HTB/devvortex/20231204125550.png)

We get a hit, virtual host `dev.devvortex.htb` exists.

Adding it to `/etc/hosts` file.

Visiting it.

A slightly different website. Going around wont lead to anything.

Checking if `robots.txt`exists.

![](/assets/img/HTB/devvortex/20231204130829.png)

 Joomla! is a well known CMS.

 Checking the administrator directory, we get a log in form.

 Searching for default credentials for joomla CMS we find that the username is admin with no password.

 Trying it the login will fail.

 ![](/assets/img/HTB/devvortex/20231204130428.png)

 Usually CMSs have CVEs but we have first to find out the version of Joomla.

 Check [hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/joomla#version) on how to find joomla version.
 
 By visiting `/administrator/manifests/files/joomla.xml` we find that it is version 4.2.6.

 ![](/assets/img/HTB/devvortex/20231204132353.png)

 Search for `joomla  v 4.2.6 exploit github`, we find that there is a recent CVE for this version.

 ![](/assets/img/HTB/devvortex/20231204132531.png)

 https://github.com/Acceis/exploit-CVE-2023-23752

 Joomla! < 4.2.8 - Unauthenticated information disclosure.

 Download the Ruby script and run it. 

 ```bash
 ruby exploit.rb http://dev.devvortex.htb
```
 ![](/assets/img/HTB/devvortex/20231204132952.png)

 We found 2 users and the database password.

 The password could be reused, so we try to login.

![](/assets/img/HTB/devvortex/20231204150756.png)

And we are in!!
We already know user lewis is the super user so we can alter PHP pages to get a reverse shell.
In Joomla, such a page can typically be found within installed templates. To locate it, we’ll navigate through System -> Templates -> Site Templates.

![](/assets/img/HTB/devvortex/20231204151036.png)

![](/assets/img/HTB/devvortex/20231204151102.png)

![](/assets/img/HTB/devvortex/20231204151119.png)

We can see all the pages related to this template.

Chose error.php, and we can add a PHP code to give us a reverse shell.

Replace the whole code with PHP PentestMonkey reverse shell.

(Get reverse shell payload form https://www.revshells.com/ )

![](/assets/img/HTB/devvortex/20231204152452.png)

After saving it, start a listener.

```bash
nc -nvlp 1234
```

![](/assets/img/HTB/devvortex/20231204151959.png)

Now execute it by visiting the page thru the path written above the editor.

![](/assets/img/HTB/devvortex/20231204152649.png)

Back to our listener.

![](/assets/img/HTB/devvortex/20231204152720.png)

We are in as www-data.

List all users with shell on the machine.
```bash
cat /etc/passwd | grep sh$
```

![](/assets/img/HTB/devvortex/20231204153044.png)

## **Privilege Escalation**
### www-data to logan
To read user.txt we need to be user logan.
We already know the database credentials, we can get user logan password hash and crack it.

```bash
mysql -u lewis -p -D joomla
```

![](/assets/img/HTB/devvortex/20231204153306.png)

List all tables available.

```sql
show tables;
```

Return all data from table sd4fg_users.

```sql
select * from sd4fg_users;
```

![](/assets/img/HTB/devvortex/20231204153550.png)

Copy hash to local machine into  a file called hash, and crack it using john with rockyou.txt wordlist.

![](/assets/img/HTB/devvortex/20231204154003.png)

Switch user to logan, and read user.txt flag.

```bash
su logan
```

![](/assets/img/HTB/devvortex/20231204154330.png)

### logan to root
Running `sudo -l` to check what the user logan can run using sudo.

![](/assets/img/HTB/devvortex/20231204154505.png)

Apport intercepts Program crashes, collects debugging information about the crash and the operating system environment, and sends it to bug trackers in a standardized form. It also offers the user to report a bug about a package, with again collecting as much information about it as possible.

Search for Apport exploits, we find a recent CVE CVE-2023-1326, on the official apport github repository we cam find a PoC.
https://github.com/canonical/apport/commit/e5f78cc89f1f5888b6a56b785dddcb0364c48ecb

![](/assets/img/HTB/devvortex/20231204155153.png)

So we just need to make a fake crash file called `xxx.crash` inside `/var/crash `directory.

```bash
touch /var/crash/xxx.crash
```

![](/assets/img/HTB/devvortex/20231204155507.png)

After entering V, type `!id`.

![](/assets/img/HTB/devvortex/20231204155552.png)

Get a shell as root. `!bash`

![](/assets/img/HTB/devvortex/20231204155726.png)

