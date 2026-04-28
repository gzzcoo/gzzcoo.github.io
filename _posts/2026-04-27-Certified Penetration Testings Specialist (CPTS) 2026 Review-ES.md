---
title: "[ES] Certified Penetration Testing Specialist (HTB CPTS) 2026 Review"
date: 2026-04-27 00:00:00 +0000
categories: [Certifications]
tags: [CPTS, HTB-Academy, Pentesting, Active-Directory, Web, Pivoting, Reporting]
image: /assets/img/certifications/cpts/cpts_logo.png
---

> English version available: [Certified Penetration Testing Specialist (CPTS) Review - English Version](http://blog.gzzcoo.com/posts/Certified-Penetration-Testings-Specialist-(CPTS)-2026-Review/)

## 1. ¿Qué es el CPTS?

La certificación **Certified Penetration Testing Specialist (CPTS)** de HackTheBox se ha convertido oficialmente en el "estándar de oro" de la seguridad ofensiva en 2026. A diferencia de otras certificaciones que parecen un simple juego de "captura la bandera" (CTF), el CPTS es una evaluación brutal, 100% práctica, que simula un entorno empresarial real.

No es solo un examen, es la culminación del `Penetration Tester Job Role Path`. Valida que no solo sabes ejecutar herramientas, sino que posees la metodología, la profundidad técnica y las habilidades de reporte que se le exigen a un consultor profesional.

---

## 2. El Precio

El modelo de precios es probablemente el más flexible del mercado. Dependiendo de tu situación, puedes certificarte por una fracción de lo que cuesta el OSCP. 

**Importante:** Independientemente del plan que elijas, debes completar el **100% del Penetration Tester Job Role Path** para desbloquear el examen. Aquí no hay atajos, el voucher solo se activa una vez que el path está completado al completo. Esto exige que al menos el candidato que quiera presentarse deberá asumir todos los conocimientos que se muestran en el camino.

### Plan de Estudiante

Es la forma más asequible de certificarte si actualmente estás estudiando.

* **Precio:** **$8/mes**.
* **Acceso:** Acceso directo a todos los módulos hasta el Nivel II (Tier II), lo que incluye todo el path del CPTS.
* **Setup:** No necesitas cambiar tu cuenta principal. Solo añade tu correo educativo como _Secondary Email_ en los ajustes de tu cuenta de HTB. Si tu dominio no está en su lista, abre un ticket de soporte, son muy rápidos metiendo instituciones en la lista blanca.

### Planes Anuales y Mensuales
Estos planes te dan el acceso o los "Cubos" necesarios para ir desbloqueando los módulos del path:

* **Silver Annual (€410/año):** La mejor opción. Te da acceso a Tier II e **incluye un voucher de examen** (CPTS, CDSA, CWES, etc.).
* **Gold Annual (€1055/año):** Incluye acceso a Tier III y un voucher de examen (incluyendo CAPE, CWEE o CWPE).
* **Cubos Mensuales:** Si prefieres pagar sobre la marcha, los planes van desde **Silver (€16/mes)**, **Gold (€33/mes)** hasta **Platinum (€58/mes)**.

### Voucher del Examen
Si ya has terminado el path y solo necesitas el intento:
* **Voucher CPTS:** 180€
* **Reintento de Examen:** Algo que hace que el CPTS destaque frente al OSCP y que se agradece a HTB, es que **obtienes dos intentos por voucher**. El precio indicado arriba cubre ambos, lo que significa que tienes una segunda oportunidad para aprobar sin pagar extra. Esto ayuda muchísimo a reducir la ansiedad de examenm, pero no te confíes. Además el laboratorio en el segundo intento, es el mismo.

---

## 3. El Contenido (Academy)

El path es una maratón, no un sprint. Es denso, técnico y exige profundizar en cada tema. Aquí tienes el desglose:

* **Web Attacks**: Agrupa varios módulos (SQLi, XSS, File Upload, IDOR, XXE, etc.). Esencial, ya que el examen casi siempre empieza con un **foothold Web**.
* **Attacking Common Apps & Services**: Ataques a objetivos como WordPress, Tomcat, IIS, MSSQL, NFS y SMB.
* **Pivoting**: ¿Sinceramente? Este módulo se hace pesado porque enseña herramientas algo anticuadas. Mi consejo: aprende los conceptos básicos pero domina [Ligolo-ng](https://blog.gzzcoo.com/posts/ligolo-ng/) por tu cuenta.
* **Privilege Escalation**: Centrado en malas configuraciones del mundo real tanto en Windows como en Linux.
* **Active Directory**: Inmersión profunda en ACLs, Kerberoasting, AS-REP Roasting, y ataques complejos de Domain Trusts/Cross-Forest. También cubre interceptación de tráfico con Inveigh/Responder.
* **Attacking Enterprise Networks (AEN)**: El "Final Boss" del path. **Hazlo 100% a ciegas.** Superar AEN sin ayuda es buena señal de que tu metodología funciona, pero no te confíes: el examen es significativamente más duro.
* **Documentation & Reporting**: La parte más crítica. Puedes sacar 100/100 en el laboratorio, pero **sin un reporte profesional, suspendes.** Recomiendo ver el [vídeo de Cyber Ryan](https://www.youtube.com/watch?v=cQFXuMPv2KE) y la [guía de Bruno Rocha](https://www.brunorochamoura.com/posts/cpts-report/) para entender el nivel que espera HTB.

---

## 4. Tiempo de Estudio y Experiencia

Mi viaje con el CPTS no fue una línea recta. Cuando empecé el path, yo no tenía ninguna certificación y llevaba medio año metido en el campo de la ciberseguridad mientras cursaba un máster de ciberseguridad. Logré completar el 50% en unos 3 meses (6-7 horas al día), pero me costó bastante en módulos como el de AD porque me faltaban las bases.

Dejé el path a medias porque encontrar un hueco de 10 días para el examen era complicado, y me dediqué a otras certificaciones de AD como CRTP y luego CRTO para ganar algo de experiencia.

![image](/assets/img/certifications/cpts/spiderman.png)

Al volver a CPTS a finales de septiembre de 2025, toda esa experiencia me vino genial. Además, actualmente cuento con más de 220 máquinas resueltas en HTB, 3 Pro Labs y 9 Mini ProLabs de HTB que hice antes de presentarme, así que tengo algo de experiencia en la plataforma.

Desde noviembre hasta principios de enero hice el 50% restante dedicando unas 5-6 horas al día. En total, me llevó unos **5-6 meses de estudio efectivo** completar el 100% con muchas idas y venidas.

**DONT RUSH.** Asegúrate de entender bien cada concepto antes de pasar al siguiente. Los Skill Assessments son "mini-exámenes" en entornos black-box; practica hasta que la metodología se vuelva algo natural. Si te saltas las bases, el examen te lo hará notar. Es mejor ir paso a paso y con buena letra que correr demasiado para acabar dándote contra un muro.

---

## 5. Cómo Me Preparé

Recomiendo hacer solo las máquinas del path y ya, no hacer nada extra en sí que no se vea o toque técnicas que enseñan en el path, porque te vas a cargar de información adicional que probablemente no veas. El Path de CPTS y Track es más que suficiente si sabes los conceptos de cada módulo y puedes hacer los skills assessments y AEN tú solo. Enfócate en construir tu propia metodología, a cada uno le funciona una diferente, crea la tuya propia para que tú mismo puedas volver a ubicarte cuando te sientas perdido en mitad del examen.

Para estar realmente listo para el examen, utilicé estos recursos:

* **Pro Labs (Zephyr, Dante & Offshore)**: 
    * Recomiendo **Zephyr** para asentar Active Directory. 
    * **Dante** está bien para practicar tunneling y acostumbrarse a un entorno con varias máquinas. 
    * **Offshore** es más difícil que el CPTS, pero el lab es muy divertido y preparatorio.
* **El Track de CPTS en HTB Labs**: Haz el [CPTS Track](https://app.hackthebox.com/tracks/76). Algunas máquinas pueden parecer más difíciles que el propio examen, pero si están ahí es por algo. Lista de máquinas de CPTS Track y HTB Labs que pondría de prioridad: **TombWatcher, Administrator, Media, Trick, Heist, Snoopy, Pov, Postman**.
* **Attacking Enterprise Networks (AEN)**: Lo traté como un simulacro de examen real. Cero walkthroughs, solo mis propias notas para probar la metodología.
* **Playlist de CPTS de IppSec**: Me vi [esta playlist](https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5). La metodología de IppSec es muy buena para estructurar cómo pensar frente a una máquina.
* **Mis Apuntes**: Me hice una **cheatsheet específica para cada módulo** en Obsidian. Después de terminar AEN, documenté cada truco y lección aprendida que te enseñan en el módulo.

---

## 6. La Experiencia del Examen

El examen de CPTS es duro, sobre todo si no tienes experiencia previa en algunos sectores. En mi caso la web es lo peor que llevaba y efectivamente el foothold es web, ese era uno de mis mayores miedos, pero no te deprimas, ¡sigue intentándolo hasta que lo consigas!


Este fue mi progreso real durante el examen:

* **Día 1:** 0 flags.
* **Día 2:** 3 flags.
* **Día 3:** 0 flags.
* **Día 4:** 7 flags (10 en total).
* **Noche del Día 4 a la madrugada del Día 5:** 4 flags (14/14 en total - 100/100 puntos).

Perdí todo el primer día porque no enumeré correctamente e intenté ir con prisas. **DONT RUSH.** Tómate tu tiempo para enumerar bien.

### Mis errores
1. **La VPN:** Mi mayor error fue la VPN. En Europa funciona extremadamente mal a ciertas horas. Era imposible enumerar o hacer fuzzing correctamente; tenía falsos positivos y tuve que rehacer escaneos. (La Pwnbox funcionaba relativamente mejor que OpenVPN).
2. **No intentar lo obvio:** Me fui a dormir frustrado porque no entendía qué hacer el primer día. 

El segundo día, con la cabeza más clara, rehice mis escaneos y conseguí la primera flag. La Flag 1 es un filtro; es una cadena (vuln chain) y tienes que preguntarte: "¿Por qué tengo esto? Igual me sirve luego".

Después, las dos siguientes fueron bastante fáciles. Pero entonces llegó mi gran bloqueo... la Flag 4. Me llevó un día y medio sacar una de las flags más fáciles (por eso tuve 0 flags el Día 3). ¿Por qué? Porque no intenté lo más obvio. Creía que lo había probado, pero no era así, e incluso me metí en un camino no intencionado (unintended path). **¡Piensa más simple! (Think dumber)**.

![image](/assets/img/certifications/cpts/milhouse.gif)

Luego, pasé de 3 flags a 10 en apenas 4 horas. Eso incluye la famosa Flag 8, que tiene muchos pasos y si no controlas el tema te puede arruinar. Por suerte, la experiencia me ayudó a ver el camino claro. **Haz el CPTS track**, lo agradecerás.

Saqué las últimas 4 flags a las 2 de la madrugada tras volver de trabajar.

![image](/assets/img/certifications/cpts/cpts_final.png)

### Pensamientos Finales
Necesitas 12/14 flags + un reporte profesional. Yo conseguí 14/14. 

Empecé el sábado 11 de abril a las 07:00 de la mañana, y saqué la última flag el 15 de abril a las 02:00 de la madrugada. Con una media estimada entre 9h al día. Terminé el laboratorio sobrándome unos 6 días, pero el reporte me llevó 3 días.

El reporte no tiene mínimo de páginas, he visto que oscila entre las 170-200 páginas de media; en mi caso fueron 184. Mientras esté todo explicado como espera HTB no debería haber problema si haces uno de unas páginas menos o de más, que esté lo que debería estar y ya. 

Como el inglés no es mi lenguaje nativo, yo utilicé Gemini para el reporte CPTS. Yo redactaba qué quería poner y Gemini lo embellecía de la manera que CPTS espera, que no fuera tan IA ni tan vulgar, un lenguaje técnico y preciso que cualquiera pueda leerlo. También lo usé para algún script custom rápido.

Practica hacer el reporte durante el módulo AEN. Yo no lo hice y perdí mucho tiempo buscando cómo resaltar código en Sysreptor y haciendo pequeños ajustes de cómo me gustaría que se viera mi reporte final. Creo que el reporte fue lo más duro del examen, se hace muy eterno.

---

## 7. Tips para Futuros Candidatos

* **AD y Herramientas:** En la parte de AD no todo es Bloodhound, recomiendo encarecidamente la enumeración manual con `bloodyAD`. La sintaxis es muy simple y ayuda un montón (de hecho, tengo un [manual/cheatsheet de bloodyAD en mi blog](https://blog.gzzcoo.com/posts/bloodyAD/)). `NetExec` y la suite `Impacket` también son esenciales. Yo no usé Windows para AD, lo hice todo desde Linux y funciona perfectamente.
* **Si estás estancado:** Mantén la calma, prueba otra herramienta, otra wordlist, otro método, revisa de nuevo tus apuntes, prueba lo básico y poco a poco vas escalando de dificultad. El examen en sí no es difícil, es difícil si no sabes organizarte, si no tienes esa capacidad de hilar esas cosas que construyen la vuln chain. Trata de mejorar tu metodología, de crearte una checklist de puntos a revisar primero. 
* **Uso de IA:** También puedes consultar con Gemini o Claude si estás muy atascado y que te propongan ideas nuevas, pero siempre y cuando que sepan qué temas se tocan en CPTS. Es decir, si le digo X cosa de AD a lo mejor me dice de hacer un Kerberos Relay cuando ni en CAPE dan eso; pídele cosas realistas que SÍ puedas ver y te hayan enseñado en el CPTS Path o Track.
* **Pivoting:** Recomiendo echar un ojo a mi [post sobre Ligolo-ng](https://blog.gzzcoo.com/posts/ligolo-ng/) para aprender a usarlo bien.
* **Scope del Examen:** El examen te da el orden de las máquinas, así que no vas a ciegas. Comprueba siempre el scope; si caes en un rabbit hole, mira si la IP forma parte de él. **Solo están permitidas las subredes indicadas.**
* **Piensa más simple (Think Dumber):** Prueba siempre lo básico primero: `admin:admin`, credenciales en texto plano, reutilización de contraseñas, etc.
* **Post-Explotación:** Cuando consigas privilegios máximos en una máquina, **dumpea todo lo que puedas**.
* **Problemas de Conectividad:** Si tienes problemas de conexión al enumerar (sobretodo en la fase inicial, luego no tuve problemas en el resto de examen en hacerlo desde Exegol), hazlo desde la **Pwnbox**.
* **Tip de Resaltado en Sysreptor:** También quiero añadir un tip para el reporte de cómo poner una línea en rojo o azul para remarcar algo sobre el texto. 

  Primero, hay que editar el CSS Style de Sysreptor de la siguiente manera:

  ```css
  span.texto-azul {
      color: #0000FF !important;
  }
  span.texto-rojo {
      color: #dc2626 !important;
      word-break: break-all; 
  }
  ```

  ![image](/assets/img/certifications/cpts/sysreptor3.png)

  Luego, hay que poner el codeblock con el lenguaje `txt highlight-manual`. Lo que queramos que sea azul o rojo debe empezar y terminar así:

  `§<span class="texto-rojo">§` y acabando el rojo con `§§`

  `§<span class="texto-azul">§` y acabando el azul con `§§`

  Esto sería un ejemplo de codeblock donde el comando lo ponemos en azul y lo que queremos resaltar en rojo:

```sh
  $ §<span class="texto-azul">§ffuf -u [http://lab.local](http://lab.local) -w /usr/share/seclists/Discovery/DNS/namelist.txt -H 'Host: FUZZ.lab.local' -c -ac -t 150 -fs 0§§

  ________________________________________________

   :: Method           : GET
   :: URL              : [http://lab.local](http://lab.local)
   :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/namelist.txt
   :: Header           : Host: FUZZ.lab.local
   :: Follow redirects : false
   :: Calibration      : true
   :: Timeout          : 10
   :: Threads          : 150
   :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
   :: Filter           : Response size: 0
  ________________________________________________

  gitlab                  [Status: 302, Size: 428, Words: 78, Lines: 14, Duration: 2112ms]
  jupyter                 [Status: 200, Size: 15742, Words: 4210, Lines: 412, Duration: 1876ms]
  §<span class="texto-rojo">§vpn                      [Status: 200, Size: 893, Words: 210, Lines: 28, Duration: 945ms]§§
  jenkins                 [Status: 403, Size: 587, Words: 112, Lines: 18, Duration: 1563ms]
  monitoring              [Status: 200, Size: 31022, Words: 8901, Lines: 534, Duration: 2781ms]
  §<span class="texto-rojo">§backup                   [Status: 401, Size: 512, Words: 98, Lines: 16, Duration: 1024ms]§§
  wiki                    [Status: 200, Size: 8453, Words: 2231, Lines: 209, Duration: 1345ms]
  mail                    [Status: 200, Size: 12442, Words: 3340, Lines: 298, Duration: 1983ms]
  status                  [Status: 200, Size: 2112, Words: 498, Lines: 62, Duration: 765ms]
```

  Como se ve en la siguiente imagen:

  ![image](/assets/img/certifications/cpts/sysreptor.png)

  Y así es como se ve renderizado en el PDF:

  ![image](/assets/img/certifications/cpts/sysreptor2.png)

---

## 8. Conclusión

¿Merece la pena el CPTS en 2026? Absolutamente. Te aleja de la "mentalidad de CTF" y te lleva a la "mentalidad de auditor profesional". El path te da una base muy sólida para un pentest real.

En mi opinión (y muchos compañeros coinciden), el CPTS es el nuevo OSCP, pero mejor y más barato. El salto de un examen de 24h a uno de 10 días lo cambia todo. Es un nivel diferente que requiere más habilidad, paciencia y estándares más altos. 

En cuanto a lo que viene ahora, me estoy preparando para el **Certified Active Directory Pentesting Expert (CAPE)** e intentaré examinarme a finales de año. Será un viaje largo, ¡pero tengo muchas ganas de sacarlo!

![image](/assets/img/certifications/cpts/cpts.png)

----

## 9. Recursos Útiles y Referencias

Si vas a por el CPTS, guarda estos enlaces:

* **Track Oficial de CPTS en HTB**: [Las máquinas que tienes que hacer](https://app.hackthebox.com/tracks/76).
* **Excel de Preparación CPTS**: [Lista de máquinas y su temática](https://docs.google.com/spreadsheets/d/1F8D5x2IHmyPvE4LjTeSu7b-IoLa-H5L4-RA2eWEA9X8/edit?pli=1&gid=1458525923#gid=1458525923).
* **Guía de Reporte (Bruno Rocha)**: [El estándar para el reporte CPTS](https://www.brunorochamoura.com/posts/cpts-report/).
* **Vídeo de Reporte (Cyber Ryan)**: [Desglose de lo que espera HTB](https://www.youtube.com/watch?v=cQFXuMPv2KE).
* **The Hacker Recipes**: [Guías prácticas de ataques](https://www.thehacker.recipes/).
* **Internal All The Things**: [Apuntes de pentesting interno de Swissky](https://swisskyrepo.github.io/InternalAllTheThings/).
* **NetExec Wiki**: [Documentación oficial](https://www.netexec.wiki/).
* **Playlist de CPTS de IppSec**: [Preparación en vídeo](https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5).
* **Guía de Pivoting**: [Mi cheatsheet de Ligolo-ng](https://blog.gzzcoo.com/posts/ligolo-ng/).
* **Guía de bloodyAD**: [Mi cheatsheet de bloodyAD](https://blog.gzzcoo.com/posts/bloodyAD/).

> _Happy Hacking :)_