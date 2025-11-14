---
title: "Certified Red Team Operator (CRTO) 2025 Review"
date: 2025-11-13 00:35:00 +0000
categories: [Certifications]
tags: [CRTO, Zero-PointSecurity, CobaltStrike, OPSEC, ActiveDirectory, Beacons]
image: /assets/img/certifications/crto/crto.png
---
## Introduction

La **Certified Red Team Operator (CRTO)** de _Zero-Point Security_ es una certificación práctica centrada en operaciones de Red Team y simulación de adversarios en entornos Windows Active Directory. Lo que realmente diferencia a esta certificación es su enfoque dual: combina teoría y práctica con un fuerte énfasis en **OPSEC** (_Operations Security_) en cada acción, preparando a los estudiantes para entornos reales donde evitar la detección es crucial.

Impartido por Daniel Duggan (_RastaMouse_), el curso fue actualizado a mediados de 2025 con un nuevo formato que exploraremos en detalle más adelante.

La certificación nos guía a través del ciclo completo de un ataque ofensivo:

- **Fase Inicial**: Acceso inicial y enumeración del dominio
- **Expansión**: Movimiento lateral, escalada de privilegios y persistencia
- **Dominio**: Toma de control mediante ataques a Kerberos
- **Enterprise**: Abuso de confianzas de dominio y bosque

El curso establece una base sólida en el funcionamiento del C2 - **Cobalt Strike**, framework que utilizaremos de manera extensiva tanto en la formación como en la evaluación final. Una vez que se ha disfrutado y asimilado todo el contenido del curso, el siguiente paso es enfrentarse a un **examen práctico de 48 horas** para conseguir la certificación **Certified Red Team Operator (CRTO)**.

![images](/assets/img/certifications/crto/cobaltstrike.png)

---

## The Course

En mi opinión, el curso ha sido una experiencia muy gratificante. El contenido es excelente para introducirse en el mundo del Red Teaming. Estos son los módulos que más he disfrutado:

- **Malware Essentials**: Aunque la certificación no se centra en malware, este módulo complementa muy bien conocimientos básicos de malware que más adelante estuve probando por mi cuenta para ponerlo en práctica.

- **Cobalt Strike Primer**  
Mi primera experiencia con un C2 framework, y qué mejor que fuera con Cobalt Strike. Me ha fascinado aprender conceptos fundamentales como creación de listeners, BOFs y otros conocimientos transferibles a otras plataformas similares.

- **User Impersonation**  
Un módulo muy disfrutable y crucial para el examen. Abarca técnicas como Pass-the-Hash (aunque no recomendada), Pass-The-Ticket, Process Injection y Token Impersonation, entre otras.

- **Kerberos**  
Sin duda uno de los módulos más importantes, con conceptos avanzados sobre ataques de delegación: RBCD, Unconstrained Delegation, Constrained Delegation, etc. Esencial para preparar el examen.

- **Microsoft SQL Server**  
Aunque ya había trabajado este tema en la certificación CRTP de Altered Security, en este caso me resultó interesante ver su implementación mediante BOFs.

- **Forest & Domain Trusts**  
Uno de los módulos que más conocimientos me ha aportado. Es muy interesante aprender a explotar los diferentes tipos de confianzas existentes. Fundamental para el examen.

- **Defence Evasion**  
El módulo más importante para la preparación del examen, ya que proporciona consejos y técnicas valiosas para configurar tu malleable C2 y evadir defensas.

El contenido del curso es excelente. Combina perfectamente teoría concisa con práctica en laboratorio, sin resultar abrumador.


El único punto que no me acaba de convencer son los laboratorios en la nueva plataforma. Antes tenías un laboratorio compartido donde podías pasar horas probando cosas: técnicas de Defence Evasion, los distintos ataques de Kerberos, viendo en tiempo real cómo respondía el entorno a cada acción...

Ahora, aunque te dan una instancia privada, solo dura 30-45 minutos y está limitada a la sección que estés viendo en ese momento. Se queda corto cuando ya has terminado el temario y quieres practicar con más libertad. 

Por ejemplo, herramientas como Kibana y Elastic Search solo estaban disponibles en laboratorios específicos, no había una integración completa donde poder hacer pruebas reales y ver qué acciones concretas generaban alertas. Y esto es importante porque con la actualización del CRTO, el OPSEC se convirtió en el tema más importante para el examen.

---
## The Exam

Con la actualización de 2025, el examen del CRTO cambió en varios aspectos importantes.

Para mí, el cambio más significativo son los **intentos ilimitados** (con un cooldown de 7 días entre intentos). Esto es un win-win total: el precio es razonable comparado con otras certificaciones, tiene mucho reconocimiento laboral, y el conocimiento que adquieres es igual de valioso. Me parece genial que ZeroPoint haya tomado esta decisión, especialmente para quienes no podemos permitirnos certificaciones más caras como las de OffSec.

El segundo gran cambio es la **modalidad del examen**. Antes había que conseguir varias flags, pero ahora solo hay una flag en la última máquina. Conseguirla vale 50 puntos de los 100 totales. Los otros 50 puntos dependen completamente del **OPSEC**: no levantar alertas de Defender, evitar que AppLocker bloquee tus ejecutables, y no generar actividades sospechosas detectables. Para aprobar hay que conseguir un mínimo de 85 puntos, es decir, que aunque obtengas la flag pero tu puntuación de OPSEC baje demasiado, no conseguirás aprobar el examen.

Aquí viene mi única queja real: que la mitad de la puntuación dependa del OPSEC sin tener un laboratorio dedicado donde poder practicar 2-3 horas con Kibana y ELK para ver exactamente qué genera alertas y cómo responde Defender. Esto es lo que realmente echaba en falta en el curso.

En mi caso, el examen me tomó 23 horas y en algunos momentos sufrí más de lo esperado. Como mencioné antes, no estaba familiarizado con C2 - siempre había trabajado con Exegol y lo que aprendí de Windows en el CRTP. Iba un poco "incómodo" porque sabía qué hacer, pero la dificultad estaba en hacerlo desde Cobalt Strike. Al final logré resolver todo en el primer intento.

![images](/assets/img/certifications/crto/funny-celebrate-56.gif)

La parte del OPSEC me daba más respeto que en el CRTP, porque no quería suspender por eso y siempre tenía ese miedo de ejecutar un comando y esperar a ver si funcionaba o si sería detectado.

El nivel de dificultad me pareció más alto que los labs (al contrario que en el CRTP), aunque en cuanto a ataques y conceptos de Active Directory lo llevaba bien, ya que es lo que mejor se me da. Lo más complicado para mí fue definitivamente el pivoting con C2.

Aparte de los intentos ilimitados, lo que más me gustó es que **puedes pausar el examen** para descansar o dormir, y las horas no cuentan. Tienes 7 días para completar las 48 horas efectivas de examen. Esto es ideal porque no te obliga a estar todas las horas seguidas, con el miedo de quedarte dormido y perder tiempo valioso.

----

## Tips

Lo primero que recomiendo es preparar un buen perfil de Malleable C2. Es crucial buscar información adicional sobre Cobalt Strike más allá del curso y practicar mucho en los laboratorios. Presta mucha atención a las secciones de OPSEC y al módulo 20 de Defence Evasion, donde te indican claramente lo que NO debes hacer.

- **Defence Evasion**: Revisa con [ThreatCheck](https://github.com/rasta-mouse/ThreatCheck) que archivos como `template.x64.ps1` o `compress.ps1` no sean detectados como maliciosos. Si los detecta, ofúscalos con [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation).

- **Personalización**: Cambia todos los nombres por defecto, incluyendo pipenames y otros identificadores.

- **Reconocimiento inicial**: Antes de ejecutar cualquier payload en una máquina nueva, revisa AppLocker, Constrained Language Mode en PowerShell y otras defensas.

- **Comandos seguros**: Evita comandos obvios como `whoami`. Mejor usa BOFs como `getuid` o desde winrs ejecuta `set username` o `$env:USERNAME` en PowerShell.

- **Evita LSASS**: No hagas dump de LSASS ni uses Pass-the-Hash.

- **Ejecución remota**: En lugar de PsExec o WmiExec, utiliza `winrs` (el cliente de WinRM), que además evade PSRemoting. Ejemplo: 

```sh
run winrs -r:ws01.lab.local "powershell -e AR3p6Y29vMTIzCg=="
```

- **Kerberos**: Prioriza el uso de tickets de Kerberos antes que Pass-the-Hash.

- **Práctica esencial**: Repasa y practica estos temas clave:
    - User Impersonation
    - Lateral Movement
    - Pivoting
    - Kerberos
    - Forest & Domain Trusts
    - Defence Evasion

---

## Conclusion

La CRTO ha sido, sin duda, la **certificación más valiosa que he realizado hasta la fecha**. No solo por el reconocimiento que tiene en el sector, sino por el enfoque 100% práctico que te obliga a aprender haciendo. 

Si tuviera que quedarme con algo, es la importancia brutal que le dan al OPSEC. Te cambia la mentalidad por completo: ya no se trata solo de entrar, sino de hacerlo sin que se enteren. Ese concepto lo aplico ahora en todo lo que hago.

El examen, aunque exigente, es justo. Los cambios que han implementado en 2025 - intentos ilimitados y la posibilidad de pausar - **demuestran que Zero-Point Security piensa en los estudiantes**. Saben que no todos podemos permitirnos pagar varios intentos de otras certificaciones, y eso se agradece.

**¿Recomendaría el CRTO?** Totalmente. Especialmente si ya tienes algo de experiencia con Active Directory y quieres dar el salto al Red Teaming profesional. Eso sí, ve preparado para dedicarle horas y horas de práctica, sobre todo si, como yo, no venías de usar Cobalt Strike, o de estar familiarizado con un C2 a diario. Ahora me han dado ganas de meterme de lleno con un Command & Control, así que he decidido enfocarme por mi cuenta en **Sliver**, ya que en **CAPE** (_Certified Active Directory Pentesting Expert_) de HTB se utiliza y en un futuro quisiera hacer dicha certificación con algo de soltura con el C2.

Al final, más allá del certificado, te quedas con conocimientos reales que puedes aplicar desde el primer día en un trabajo. Y eso, al menos para mí, es lo que realmente importa.

![images](/assets/img/certifications/crto/crto_jeremy.png)

> _Happy Hacking :)_