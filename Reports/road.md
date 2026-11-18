# Penetration Testing Report

## TryHackMe - Road

**Assessment Type:** Web Application and Linux Host Penetration Test

**Platform:** TryHackMe

**Target:** Road

**Difficulty:** Medium

**Operating System:** Linux

**Assessment Status:** Completed

**Report Classification:** Technical Security Assessment - Lab Environment

## 1. Executive Summary

This report documents a penetration test conducted against the Road machine hosted on the TryHackMe platform. The assessment focused on identifying exposed services, enumerating the web application, exploiting security vulnerabilities, obtaining initial access, and escalating privileges to the root user.

The assessment identified several weaknesses that, when chained together, enabled complete compromise of the target system. The initial attack vector was an Insecure Direct Object Reference (IDOR) vulnerability in the web application's password reset functionality. This allowed an authenticated user to reset another user's password, including the administrator's account.

Following administrative account compromise, an unrestricted file upload vulnerability enabled the execution of a PHP reverse shell, providing initial access to the underlying Linux host. Further enumeration revealed a locally accessible MongoDB instance containing a backup of user data, from which credentials for the *webdeveloper* account were recovered.

Finally, a misconfiguration involving a privileged backup utility and the LD_PRELOAD environment variable allowed the *webdeveloper* account to escalate privileges to root.

The attack chain demonstrates how weaknesses in application-level authorization, file upload validation, credential storage, and privilege management can combine to compromise an entire system.

**Overall assessment:** The target was fully compromised within the authorized lab environment, with both user-level and root-level accesss obtained.

# 2. Scope and Objectives

## 2.1 Scope

The assessment was limited to the TryHackMe Road machine.

**Attribute** | **Details**

Target | Road

Target IP | TARGET-IP

Operating system | Linux

Exposed services | SSH, HTTP

Application path | /v2/

Database service | MongoDB on localhost

Assessment environment | Authorized CTF lab

## 2.2 Objectives

The primary objectives were to:

* Identify exposed network services and potential attack vectors.

* Enumerate the web application and assess its authentication and authorization controls.

* Identify and exploit vulnerabilities that could provide initial access.

* Investigate local services and recover potentially exposed credentials.

* Evaluate privilege escalation opportunities.

* Demonstrate the impact of the identified weaknesses using proof-of-compromise evidence.

# 3. Methodology 

The assessment followed a structured penetration testing workflow consisting of the following phases:

1. **Reconnaissance:** Identification of open ports and running services.

2. **Enumeration:** Discovery of web application endpoints and analysis of HTTP requests.

3. **Exploitation:** Abuse of the password reset authorization flaw and file upload functionality.

4. **Post-exploitation:** Enumeration of the compromised host and collection of the user flag.

5. **Credential discovery:** Examination of locally accessible MongoDB data.

6. **Privilege escalation:** Abuse of the privileged backup utility to obtain root access.

7. **Verification:** Confirmation of elevated privileges and retrieval of the root flag.

**Tool Used**

* Nmap

* Gobuster

* Burp Suite

* Netcat

* MongoDB client utilities

* SSH

* GCC

* Linux command-line utilities

# 4. Reconnaissance and Service Enumeration

## 4.1 Port Scanning

An initial Nmap scan was conducted to identify open ports, service versions, and common service-related information.

**Command:** nmap -sV -sC -p- TARGET-IP -oN road.txt

**Results:**

**Port** | **Service** | **Status**

22/tcp | SSH | Open

80/tcp | HTTP | Open

The scan identified two externally accessible services: SSH and HTTP.

SSH was initially deferred because valid credentials had not yet obtained. The HTTP service was selected as the primary attack surface for further investigation.

**Evidence:** Nmap scan output.

<img width="935" height="419" alt="{DFDF2551-702A-4442-8F1B-1E9704C3F22C}" src="https://github.com/user-attachments/assets/6b566123-6a32-4672-a30a-f12b88d5ad3a" />

# 5. Web Application Enumeration

## 5.1 Directory and Endpoint Discovery

The web application was initially inspected manually, including its page source, cookies, network activity, and available functionality. This did not immediately reveal an exploitable weakness.

Directory enumeration was subsequently performed using Gobuster. 

**Command:** gobuster dir -u http://TARGET-IP -w WORDLIST

**Discovered paths:**

* /assets

* /v2

The */assets* directory did not reveal any useful information. Accessing */v2* redirected the browser to the administrative login page: http://TARGET-IP/v2/admin/login.html

Attempts to authenticate using common default credentials were unsuccessful.

## 5.2 Account Registration and Password Reset Analysis

The application provided a registration feature, allowing a new user account to be created. After authentication, the *ResetUser* functionality was identified in the application's navigation menu.

Testing the password reset feature revealed that the corresponding HTTP request contained the target user's email address and the new password.

The request was captured in Burp Suite and forwarded to Repeater for controlled testing.

## 5.3 Insecure Direct Object Reference (IDOR)

To determine whether the password reset function enforced appropriate authorization, a second test account was created.

The captured request was modified by replacing the email address associated with the authenticated account with the email address of the second account.

The modified request successfully reset the second account's password, despite the operation being initiated from the first account's authenticatd session.

This demonstrated that the application relied on a client-controlled identifier without adequately verifying whether the authenticated user was authorized to modify the specified account.

**Finding:** Insecure Direct Object Reference (IDOR) in the password reset functionality.

**Security impact:**

* Unauthorized password changes for other users.

* Potential account takeover.

* Escalation of impact if privileged accounts are affected.

<img width="739" height="502" alt="{FD28883F-1B1B-446E-AD71-F54F0433C8F6}" src="https://github.com/user-attachments/assets/c23a534d-4823-4f81-8d34-205f061adedb" />

# 6. Initial Access

## 6.1 Administrator Account Compromise

Further inspection of the application revealed a profile page accessible through the username menu in the upper-right corner of the interface.

A note near the bottom of the profile page disclosed the administrator's email address: *admin@sky.thm*

The previously identified IDOR vulnerability was then used to reset the administrator's password.

After changing the administrator's password, authentication with the administrator account was successful.

This confirmed that the IDOR vulnerability could be used to compromise a privileged application account, rather than only a standard user account.

## 6.2 Unrestricted PHP File Upload

After gaining administrative access, the profile functionality was examined further. The application allowed users to change their profile pictures.

Testing revealed that the upload mechanism accepted a PHP file rather than restricting uploads to legitimate image formats.

This indicated insufficient server-side validation of uploaded files and insufficient protection against execution of uploaded content.

A PHP reverse shell obtained from the following public repository was used for validation: https://github.com/pentestmonkey/php-reverse-shell

The file was uploaded through the profile picture functionality.

A Netcat listener was started on the attack machine: nc -lvnp 4444

The uploaded PHP file was then accessed through the web application: http://TARGET-IP/v2/profileimages/php-reverse-shell.php

The request triggered the reverse shell, establishing an interactive connection to the target.

**Result:** Initial operating system access was obtained through the web server's execution context.

**Finding:** Insufficient file upload validation combined with executable uploaded content.

<img width="258" height="99" alt="image" src="https://github.com/user-attachments/assets/de8b2d63-838e-450d-833b-6094f360f9f7" />

# 8. MongoDB Enumeration and Credential Discovery

## 8.1 Identifying the Database Service

Further investigation revealed a MongoDB instance accessible locally on the target host.

A connection was established using: mongo "mongodb://127.0.0.1:27017"

The available databases were enumerated using: show dbs

**Observed databases:**

* admin

* backup

* config

* local

The presence of a backup database warranted further investigation because it could contain sensitive application data. 

## 8.2 Extracting User Data

The *user* collection in the *backup* database was exported for analysis

**Command:** mongodump --host 127.0.0.1 --port 27017 -db backup --collection user --out /.dump

Readable strings were extracted from the resulting BSON data using: strings /dump/backup/user.bson

The output contained credentials associated with the *webdeveloper* account.

This demonstrated that sensitive authentication data was recoverable from the database backup accessible to the compromised environment.

**Finding:** Exposure of user credentials through an accessible database backup.

**Security impact:**

* Recovery of credentials belonging to another system account.

* Potential lateral movement from the web server context to SSH.

* Increased risk of privilege escalation if the recovered account has additional privileges.

<img width="339" height="457" alt="{66B9F498-B65B-4259-8996-E09D18082E34}" src="https://github.com/user-attachments/assets/cc37c111-4a39-4177-8463-75eaad21220d" />

# 9. Privilege Escalation

## 9.1 SSH Access as webdeveloper

The recovered credentials were used to authenticate to the target through SSH.

**Command:** ssh webdeveloper@TARGET-IP

After establishing the session, the account's available sudo privileges were examined.

**Command:** sudo -l

The output identified a privileged backup utility: /usr/bin/sky_backup_utility

The relevant sudo configuration permitted the utility to be executed with elevated privileges. This configuration was investigated as a potential privilege escalation vector.

<img width="1203" height="127" alt="{C24672C9-654D-49EC-8261-4EA44F587F2B}" src="https://github.com/user-attachments/assets/82207cf4-ed1b-4c98-b890-ac4fead96746" />

## 9.2 Exploiting LD_PRELOAD

The LD_PRELOAD environment variable can instruct the dynamic linker to load a specified shared library before other libraries. If a privileged program permits this mechanism to be used with attacker-controlled code, it may allow execution with elevated privileges.

To test this condition, a C source file named *exploit.c* was created in /tmp.

**Source code:**

<img width="190" height="202" alt="image" src="https://github.com/user-attachments/assets/80f15a08-6b06-4b98-b042-f7e5874108d9" />

The source code was compiled into a shared library: gcc -fPIC -shared -o /tmp/exploit.so /tmp/exploit.c -nostartfiles

The backup utility was then executed with the compiled library specified through LD_PRELOAD: sudo LD_PRELOAD=/tmp/exploit.so /usr/bin/sky_backup_utility

The library was designed to execute code in the context of th privileged process.

## 9.3 Verification of Root Access

Following execution, the current security context was checked using: id

The resulting identity information confirmed successful escalation to root privileges.

The root-level flag, *root.txt*, was subsequently retrieved.

**Result:** Full privilege escalation from the *webdeveloper* account to root was achieved.

**Finding:** Privilege escalation through an unsafe privileged execution path involving LD_PRELOAD.

<img width="993" height="94" alt="{F96E22BF-6964-4567-AC58-73F1264077F8}" src="https://github.com/user-attachments/assets/ce6380fb-03d5-4368-8e41-52683176710b" />

# 10. Findings and Risk Assessment

The following findings summarize the security weaknesses identified during the assessment. Severity ratings are provisional qualitative assessments based on demonstrated impact in the lab environment; they are not formal CVSS scores.

## F-01: IDOR in Password Reset Functionality

**Suggested severity: High**

**Description:** Th application allowed an authenticated user to reset another user's password by modifying the email address in the HTTP request.

**Impact:** Unauthorized account modification and account takeover, including compromise of the administrator account.

**Evidence:** A password reset request modified in Burp Suite successfully changed the password of a different account.

**Recommendations:**

* Derive the target account from the authenticated session wherever possible.

* Enforce server-side authorization checks for every account modification.

* Do not rely user-supplied email addresses or identifiers as proof of authorization.

* Implement audit logging and alerting for password resets affecting privileged accounts.

* Require appropriate verification for administrative password changes.

## F-02: Unrestricted Upload of Executable PHP Files

**Suggested severity: Critical**

**Description:** The profile image upload functionality accepted PHP files that could be executed through a web-accessible path.

**Impact:** Remote code execution in the web server's security context, resulting in initial host compromise.

**Evidence:** An uploaded PHP reverse shell was triggered through an HTTP request and established a connection to the assessment machine.

**Recommendations:**

* Allow only explicitly approved image formats.

* Validate file content using server-side MIME detection and file signature checks.

* Generate server-side filenames and disregard user-supplied extensions.

* Store uploaded files outside the web root wherever possible.

* Disable script execution in upload directories.

* Apply least-privilege permissions to the web server account.

* Consider image re-encoding and content scanning before accepting uploaded files.

## F-03: Exposure of Credentials in MongoDB Backup Data

**Suggested severity: High**

**Description:** The locally accessible MongoDB backup database contained recoverable credentials associated with the *webdeveloper* account.

**Impact:** Credentials could be extracted following initial host compromise and reused to access SSH, facilitating lateral movement and subsequent privilege escalation.

**Evidence:** Strings extracted from the BSON collection revealed credentials for the *webdeveloper* account.

**Recommendations:**

* Restrict database and backup access to authorized service accounts.

* Avoid storing plaintext passwords; use appropriately salted, adaptive password hashing for application authentication.

* Encrypt sensitive backups at rest and protect backup access with dedicated permissions.

* Remove unnecessary or outdated sensitive data from backup collections.

* Rotate any exposed credentials and assess whether they have been reused across services.

* Separate database, application, and operating system privileges.

## F-04: Root Privilege Escalation Through LD_PRELOAD

**Suggested severity: Critical**

**Description:** A user permitted to execute a privileged backup utility could exploit the program's execution environment to load a malicious shared library through LD_PRELOAD.

**Impact:** Execution of attacker-controlled code with root privileges, resulting in complete host compromise.

**Evidence:** The compiled shared library was supplied through LD_PRELOAD when invoking the backup utility, and the resulting security context was verified with *id*.

**Recommendations:**

* Remove unnecessary sudo permissions for the backup utility.

* Ensure privileged programs do not accept attacker-controlled environment variables or library paths. 

* Review sudoers configuration, including environment preservation and permitted command arguments.

* Use a restricted, explicitly defined execution environment for privileged backup operations.

* Avoid invoking external commands or loading dynamic libraries from writable locations in privileged programs.

* Audit custom backup utilities for unsafe environment handling and excessive privileges.

# 11. Attack Chain Summary

The successful attack can be summarized as follows:

1. **Reconnaissance:** Identified SSH and HTTP as externally accessible services.

2. **Web enumeration:** Discovered the application under */v2/* and its account management functionality.

3. **Account takeover:** Exploited the IDOR vulnerability to reset the administrator's password.

4. **Initial access:** Uploaded and executed a PHP reverse shell through the profile image functionality.

5. **Credential discovery:** Extracted *webdeveloper* credentials from the MongoDB backup database.

6. **Lateral movement:** Authenticated to the host through SSH as *webdeveloper*.

7. **Privilege escalation:** Abused the privileged backup utility and LD_PRELOAD to obtain root access.

8. **Objective completion:** Retrieved both the user and root flags.

# 12. Conclusion 

The assessment demonstrated that multiple security weaknesses in the Road environment could be chained to achieve complete system compomise.

The initial application-level authorization flaw enabled administrator account takeover. Insufficient upload validation then provided an avenue for remote code execution on the host. Exposed credentials in the MongoDB backup facilitated access through SSH, while unsafe privileged execution allowed escalation to root.

The most significant lesson from this assessment is that application security, credential management, and operating system hardening must be addressed together. A single vulnerability may provide limited access, but several weaknesses across different layers can create a direct path to full compromise.

The recommended remediation priorities are to correct the password reset authorization flaw, prevent execution of uploaded files, secure database backup data, and remove the unsafe privilege escalation path.

**Final result**: Successful compromise of the application and underlying Linux host, including verified root-level access, within the authorized TryHackMe lab environment.
