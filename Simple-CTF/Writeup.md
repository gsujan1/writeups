This is my writeup of the Simple CTF room on TryHackMe. This box utilizes a CVE within the Simple CMS application (hence the name) to gain the initial foothold and then a SUID abuse for the privilege escalation.

Let's get started!

---

# Nmap scan
As always we start with our nmap scan:

```bash
sudo nmap -sC -sV -oN initial 10.10.121.100 && sudo nmap -T5 -p- -oN allports 10.10.121.100
```

### Output:
Detailed scan of the top 1000 ports:

![Pasted image 20210327195457.png](https://github.com/gsujan1/writeups/blob/main/Simple-CTF/Pasted%20image%2020210327195457.png)

Quick scan of all 65,535 (TCP) ports:

![Pasted image 20210327195629.png]

*Aside: I've begun to use rustscan for the allports scan as the speed at which it scans ports seems orders of magnitudes faster than nmap. I highly reccomend checking it out, especially for scans of all possible ports.*

---

# Begin poking at the box
Let's start by going down the nmap output

## FTP server
So first up, that FTP server, which nmap conveniently tells us allows anonymous logins

Logging into the FTP server as anonymous and listing the directories/files shows us a "pub" directory:

![Pasted image 20210327200815.png]

Moving into the "pub" directory shows us a juicy looking file:

![Pasted image 20210327200929.png]

Downloading the file to our machine and reading it shows us a note from an (annoyed) sysadmin regarding a dev's weak password:

![Pasted image 20210327201124.png]

So we now have a potential user (mitch) as well as a clue that this user reuses his password. We'll keep this in the back of our mind for now...

Nothing else seems to come from the FTP server, and looking up vulnerabilities for the FTP server software does not return anything useful, so let's move on to the next service.

## http server
Nmap again proves to be a time saver here by showing us two things:
1. That the web server is just serving the default apache "it works."
2. The site has a "/robots.txt" page we can look for interesting directories the site owner's do not want web crawlers to see.

Browsing to the robots.txt page shows us there is one disallowed directory:

![Pasted image 20210327202144.png]

However, trying to browse to this directory gives us a 404 Not Found, so this looks like a dead end.

Since our manual enumeration hasn't turned up anything too noteworthy, let's start directory busting. I'm using feroxbuster to quickly (and recursively) start bruteforcing common directories.

Feroxbuster output:

![Pasted image 20210327202544.png]

Well, it looks like the name of the box is starting to make more sense! Browsing to the /simple directory brings us to the homepage of the "CMS Made Simple" site. After scrolling through and exploring the site for anything interesting, we find the specific version:

![Pasted image 20210327202753.png]

Doing a quick searchsploit for "CMS Made Simple" shows us this service has had quite a few CVEs. Looking closer, we see an exploit made for a version immediately after ours, which means it's a solid bet to assume the exploit works. (This is also indicated by searcsploit):

![Pasted image 20210327203156.png]

Looking through the script, we see it seems to be using a time-based blind SQL injection attack that seems to query the database. Basically, a request is sent via a pre-crafted malicious URL and we can verify whether the result of our query is TRUE or FALSE based on the response time. Specifically, this exploit seems to spam every possible character in an attempt to enumerate user information such as name, email, salt, and hash of the password which is incredibly cool.

Great, let's run this baby and get some creds!

![Pasted image 20210327203902.png]

Well, it looks like the tool was written in python2, indicated by the error thrown by the interpreter around the use of quotes without parentheses. 

*NOTE: while 2to3 (a tool for converting python2 to python3) has a good chance of properly converting python2 code to python3, this script seems to be a bit finnicky (it's never that easy), and I seem to get a slightly different output everytime with maybe a warped email one time or no password (which I know is in rockyou) other attempts. So save yourself the time and just run it with python2 (and don't forget to install required libraries with pip2!)*

The program does it's magic and begins slowly revealing user info!!! Watching a time-based SQL injection happen in real time was super cool, and you could see the script work through every character for every field in the database.

Apart from recovering the hashed password, this program will also crack the password with the '-c' flag and '-w' flag to specify the wordlist.

My exact command:
```bash
python2 exploit.py -u https://10.10.121.100/simple/ -c -w /usr/share/wordlists/rockyou.txt
```


Output of this (pretty much) autopwn script:

![Pasted image 20210327214000.png]

---

# Initial Foothold
Trying the creds on the admin login page fails so let's try using these creds other places like the ssh server since we know that mitch, who we just got creds for, likes to reuse his password.

Logging in as mitch via SSH on port 2222, we get the user.txt flag: 

![Pasted image 20210327205621.png]

---

# Privilege Escalation
The first thing I like to do on Linux boxes is to look for any commands we can run as root with the ```sudo -l``` command.

![Pasted image 20210327214552.png]

We can run Vim as root, and we don't even need to enter a password!

I know we can run a shell from Vim, so let's create a new file (as root) with Vim and try calling a shell from within it:

![Pasted image 20210327214826.png]

And we're root!

![Pasted image 20210327214857.png]

Nice, looking the /root directory, we find our flag:

![Pasted image 20210327214945.png]

---

And we're done! That was a super fun and *simple* box. I struggled a little bit with getting python2 working on my machine, but other than python version issues, this was a nice, fun box to practice my methodology and get comfortable with pentesting tools.
