# CTF Write-Up - Road

## 1. Basic Information

**Platform: TryHackMe**

**Machine name: Road**

**Difficulty level: Medium**

**OS: Linux**

## 2. Reconnaissance

First, I performed a scan of the target, to identify open ports, running services, and their versions, and potential entry points.

**Command:** nmap -sV -sC -p- <TARGET-IP> -oN road.txt

**Results:** **22/tcp open ssh, 80/tcp open http**

<img width="624" height="281" alt="{BEBA7C37-8D70-4803-9214-10B8C0EBDF0E}" src="https://github.com/user-attachments/assets/cd7f9207-3be4-4cf4-90aa-1b06f06c9608" />

**Analysis:**

I decided to leave SSH for later, once I had obtained the credentials required to log in. HTTP was the first service I focused on enumerating.

## 3. Enumeration

After visiting the website, I performed basic enumeration by inspecting the source code, cookies, and network activity, as well as testing available functionality. However, I did not find anything useful.

I then moved on to directory and file enumeration.

**Command:** gobuster dir -u http://**<TARGET_IP>** -w **WORDLIST**

**Results:** /assets /v2

Using the proxy in Burp Suite to inspect the HTTP history, I visited /assets, but there was nothing useful there.

When I navigated to http://<TARGET_IP>/v2, I was redirected to http://<TARGET_IP>/v2/admin/login.html. I tried logging in with the most common default credentials, but they did not work.

I then navigated to the registration page, created my own account, and logged in.

After clicking "ResetUser" in the left-hand menu, I found an option to reset a user's password. I reset my own password and noticed something interesting in the HTTP history: the request contained the user's email address and the new password.

<img width="562" height="376" alt="{F28F9959-5E1F-4FCA-83FE-FEF9E9881893}" src="https://github.com/user-attachments/assets/9408e604-717d-450b-aee1-eb056c3dfbc4" />

I sent the request to Burp Suite Repeater using Ctrl+R. I created another account and modified the request in Repeater, replacing the email address of the first account with the email address associated with the second account.

This allowed me to reset the second user's password while authenticated as the first user. This confirmed the presence of an **Insecure Direct Object Reference (IDOR)** vulnerability.

## 4. Initial Access

With the ability to reset other users passwords, I continued enumerating the website after logging in.

By clicking on the username in the top-right corner, selecting **Profile** and scrolling to the bottom of the page, I found a note from the administrator containing their email address:**admin@sky.thm**.

Using the IDOR vulnerability, I reset the administrator's password and logged in to the administrator account.

During further enumeration, I also discovered that the profile picture could be changed. I tested whether the upload functionality accepted only common image extensions such as .png and .jpg. It turned out that .php files were also accepted.

I therefore prepared a PHP reverse shell and uploaded it to the server.

**Reverse shell source:** https://github.com/pentestmonkey/php-reverse-shell

After uploading the file, I started a listener: **nc -lvnp 4444**

I then navigated to the uploaded PHP file: http://<TARGET_IP>/v2/profileimages/php-reverse-shell.php

This successfully established a reverse shell, giving me initial access to the target machine.

## 5. Post-Exploitation Enumeration

After obtaining the reverse shell, the first thing I did was check the available users and inspect accessible files.

During this process, I obtained the **user.txt** flag.

<img width="258" height="99" alt="{39892CBD-A014-4768-9136-F8F0D04ECF9C}" src="https://github.com/user-attachments/assets/759c5ebf-2e80-44d5-9793-79194f2e6824" />

## 6. MongoDB Enumeration

While examining the target environment, I discovered that MongoDB was running locally.

**Connecting to the database:** 

mongo "mongodb://127.0.0.1:27017"

**Listing the available databases:**

show dbs

**Results:** admin, backup, config, local

I then dumped the user collection from the **backup** database and inspected the output.

**Dumping the collection:**

mongodump --host 127.0.0.1 --port 27017 -db backup --collection user --out /.dump

**Extracting readable strings from the BSON file:**

strings dump/backup/user.bson

**Results:** I recovered the credentials for the **webdeveloper** user.

<img width="250" height="329" alt="{31F59A7F-22EC-4361-8FCD-129DA4850DD6}" src="https://github.com/user-attachments/assets/a16b6010-6384-4a68-a8d2-dca19a99e942" />

## 7. Privilege Escalation

Using the previously recovered credentials, I logged in as the **webdeveloper** user via SSH.

**Command:** ssh webdeveloper@<TARGET_IP>

I then checked which commands the user was allowed to execute with sudo.

**Command:** sudo -l

**Results:** 

<img width="871" height="112" alt="{FBD8446F-5A5C-48A4-A2F4-1DAC76A9A3BB}" src="https://github.com/user-attachments/assets/4a816ee1-9640-46c4-933b-7eb0eacab9d2" />

**Creating the Exploit and Escalating Privileges**

I navigated to /tmp and created a file named exploit.c

**Source code:**

<img width="190" height="202" alt="{33497B10-AF60-4C89-84D3-C1D7E7543677}" src="https://github.com/user-attachments/assets/fc93ae0f-7f7d-4f87-a1aa-3cfa32405d45" />

I compiled the source code into a shared library.

**Command:** gcc -fPIC -shared -o /tmp/exploit.so /tmp/exploit.c -nostartfiles

I then executed the backup utility with the **LD_PRELOAD** environment variable pointing to the compiled library.

**Command:** sudo LD_PRELOAD=/tmp/exploit.so /usr/bin/sky_backup_utility

**Confirming Privilege Escalation**

To verify that the privilege escalation had succeeded, I ran the following command: **id**

After confirming that i had obtained root privileges, i read the **root.txt** flag.

<img width="720" height="68" alt="{21ABF58B-2780-4DCC-A293-92E9B71804E0}" src="https://github.com/user-attachments/assets/a5ab476b-5270-4ced-93d8-a924ef4dce53" />
