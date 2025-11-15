---
title: "Certified Red Team Analyst (CRTA) 2025 Review"
date: 2025-11-14 02:35:00 +0000
categories: [Certifications]
tags: [CRTA, CyberWarafareLabs, ActiveDirectory]
image: /assets/img/certifications/crta/crta_logo.png
---
## Introduction

El **Certified Red Team Analyst (CRTA)** de _CyberWarfare Labs_ es una certificación básica e introductoria que te sumerge en el mundo de Active Directory.

Para mí, el CRTA fue mi primera certificación oficial en ciberseguridad. El curso te guía a través de todos los conceptos fundamentales: desde el **acceso inicial** y el **movimiento lateral**, hasta la **escalada de privilegios** y los **ataques al dominio**.

---
## The Course

El contenido del PDF está bastante bien para lo que cuesta la certificación (**unos $10**). Te enseñan a hacer las cosas desde Windows, con comandos de `Mimikatz` y herramientas nativas, lo cual se agradece. En cuanto a los videos, como ya he dicho no soy de ver mucho contenido en video, pero lo poco que vi se me hizo bastante pesado. Al final no llegué a ver ninguno completo por eso mismo.

**El lab está bien para el precio de $10**. Tiene tres partes principales: web/enumeración, pivoting y la parte de Active Directory. 

Aquí viene lo surrealista: en el writeup que te dan para el laboratorio de práctica... **¡lo hacen todo desde Linux!** Después de todo el curso enseñándote en el PDF cómo hacerlo desde Windows, resulta que las soluciones oficiales usan herramientas de Linux sin haberte enseñado nada sobre ello. Por suerte yo ya tenía una base sólida en AD, así que no fue un problema real para mí. Pero para alguien nuevo debe ser desconcertante. Es como estudiar para ser médico y acabar reparando coches... no tiene sentido. A pesar de esto, el lab es lo mejor de la certificación.

----
## The Exam

El examen más surrealista que he realizado en mi vida. Cosas sin sentido, enumeración absurda, nada del otro mundo... y cosas tan ilógicas que ni caerías en ellas. 

Pero esto no es todo. La mayor decepción para mí fue que, siendo una certificación que se vende como tu primer paso en AD, el examen resulta ser:

- **90% Web/Enumeración** 
- **10% AD** (o incluso menos)

![images](/assets/img/certifications/crta/lagrimas-de-macho-homero.gif)

En el laboratorio al menos había algo más de AD y era más complejo. Pero el examen no tiene sentido como certificación de Active Directory. Y por si fuera poco, después de enseñarte todo desde Windows... ¡en el examen tienes que hacerlo desde Linux porque no te proporcionan ningún equipo Windows! 

Pero esto no es lo peor. Lo peor sin duda no es llegar a ser Administrador del dominio... lo peor es contestar las **preguntas de las flags**. Tardas más tiempo en **descifrar qué respuesta buscan que en resolver el lab entero.** Las pistas no te dejan claro el formato o qué es exactamente lo que quieren. Son flags sin sentido que arruinan la experiencia.

![images](/assets/img/certifications/crta/desesperacion.gif)

Tengo conocidos que han suspendido por esto mismo: tenían el laboratorio hecho en menos de 1 hora, pero no pudieron contestar las preguntas porque no se entienden. Y así suspendieron.

Por suerte tienes **2 intentos** para el examen, pero igualmente me parece absurdo que el mayor desafío no sea técnico, sino adivinar qué quieren en las respuestas.

---
## Tips

De cara al examen, recomendaría revisar los siguientes puntos:

- **[Local File Inclusion](https://blog.nody.cc/posts/container-breakouts-part1/)**
- **[Misconfiguración de Sudo](https://gtfobins.github.io/)** para escalada de privilegios
- **[Ataque DCSync](https://iam0xc4t.medium.com/dcsync-attack-gaining-domain-admin-via-active-directory-replication-3280a759cd95)**
- **[Pass The Hash](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/lateral-movement/alternate-authentication-material/wip-pass-the-hash)**
- **[[Credential Dumping](https://wadcoms.github.io/)]**
- **[Abuso de ACLs mal configuradas](https://www.thehacker.recipes/ad/movement/dacl/)**

Por otro lado, una serie de tips para el examen:

- **Enumera con calma**: Haz escaneos completos y metódicos
- **Mentalidad CTF**: En la parte Web/Enumeración, piensa como en un CTF, no como en un escenario realista
- **Revisa todo**: Examina cada archivo, por muy inofensivo que parezca o por raro que sea su nombre
- **Con las flags**: Lee las preguntas con atención, enumera repetidamente y prueba diferentes formatos de respuesta hasta que des con el correcto

----

## Conclusión

¿Recomiendo el CRTA? La verdad es que solo por dos razones:

1. **Está muy barata** ($10 en oferta)
2. **Para tenerla en el CV** y sumar una certificación más

Fuera de eso, no le veo mucho más valor. Si buscas aprender AD de verdad, mejor ve directo al CRTP. Esta certificación promete ser de Active Directory pero al final es 90% web y enumeración, con un toque de AD muy básico.

Eso sí, por el precio... ¿por qué no? Pero no esperes una certificación seria de Active Directory. Es más un "intro a la ciberseguridad" que otra cosa.

![images](/assets/img/certifications/crta/crta_jeremy.png)

> _Happy Hacking :)_