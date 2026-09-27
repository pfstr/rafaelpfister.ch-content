---
title: "Kiteworks: el fabricante recomienda apagar los sistemas el 26 de septiembre; lo que se sabe hasta ahora"
navTitle: "Apagado de Kiteworks"
description: "Kiteworks ha pedido a sus clientes por correo electrónico que apaguen todos los sistemas el sábado 26/09/2026, de 04:00 a 10:00. El motivo es una advertencia de las autoridades policiales sobre un posible ataque. Totemomail no se ve afectado."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min de lectura"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-el-fabricante-recomienda-apagar-los-sistemas-el-26-de-septiembre-lo-que-se-sabe-hasta"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
url: https://rafaelpfister.ch/es/blog/kiteworks-el-fabricante-recomienda-apagar-los-sistemas-el-26-de-septiembre-lo-que-se-sabe-hasta
translationSourceHash: c7274a068cc60b422ffcdf30dbaef2d90eac1fe72cf3f768b454a71c676aa046
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:46:28.294Z
translationReview: required
---

# Kiteworks: el fabricante recomienda apagar los sistemas el 26 de septiembre; lo que se sabe hasta ahora

Kiteworks pidió a sus clientes por correo electrónico el 25 de septiembre de 2026 que apagaran todos los sistemas Kiteworks el sábado 26 de septiembre, de 04:00 a 10:00 (hora central europea). Según el escrito del CISO Frank Balonis, el fabricante dispone de indicios de las autoridades policiales de que este fin de semana podría producirse un ataque contra sistemas Kiteworks. El soporte al cliente justifica el apagado como protección frente a posibles ataques de día cero. heise online confirmó por teléfono con el soporte la autenticidad del mensaje.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Asistencia de emergencia para cambiar el flujo de correo</p>
<p>Si necesita ayuda para redirigir el flujo de correo antes del apagado y restaurarlo después, utilice el <a href="https://adeptio.ch/">formulario de contacto en adeptio.ch</a>. También responderé con poca antelación.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Actualización del 25 de septiembre de 2026: declaración de Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail no se ve afectado.</strong> Queda por aclarar si Kiteworks EPG (Email Protection Gateway) se ve afectado.</p>
</div>

## Cronología

Todas las horas corresponden al horario de verano de Europa Central (CEST). Cuando no se indica una hora, no se dispone de una indicación horaria fiable.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Aviso a los clientes</p>
<p>El CISO Frank Balonis informa a los clientes por correo electrónico sobre indicios de las autoridades policiales de un posible ataque este fin de semana y recomienda un apagado de seis horas. Según el aviso, todas las vulnerabilidades conocidas están corregidas en la versión 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Primeras informaciones en medios</p>
<p>heise online informa que el soporte de Kiteworks confirma la autenticidad del mensaje y justifica el apagado como protección frente a posibles ataques de día cero. Poco después se suman TechCrunch, BleepingComputer, Computer Weekly y otros.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre, 17:41</p>
<p class="timeline__titel">La BKA no se pronuncia</p>
<p>heise añade: la BKA rechaza hacer una declaración por motivos tácticos de investigación. La BSI no responde y el FBI declina hacer comentarios a TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Declaración y comunicado de prensa</p>
<p>Kiteworks califica el apagado como una medida de precaución sin compromisos conocidos. El comunicado de prensa cita a «autoridades federales de inteligencia» como fuente y enumera las filiales no afectadas, entre ellas totemo. El fabricante apaga por sí mismo las instancias alojadas por Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sáb., 26 de septiembre, de 04:00 a 10:00</p>
<p class="timeline__titel">Ventana de apagado</p>
<p>La ventana es simultánea en todo el mundo: de 02:00 a 08:00 UTC, de 12:00 a 18:00 en Sídney y de las 22:00 del viernes a las 04:00 del sábado en Nueva York.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Situación al sáb., 26 de septiembre</p>
<p class="timeline__titel">Sigue sin aclararse</p>
<p>No hay aviso público, número CVE, información sobre la vulnerabilidad ni informes de un ataque producido.</p>
</li>
</ol>

## Lo que se sabe

La recomendación se aplica en todo el mundo; el correo electrónico indica la ventana para todas las zonas horarias, de AEST a PDT. Kiteworks aconseja apagar los sistemas antes incluso del inicio de la ventana, también si no son accesibles desde Internet.

Por ahora, casi todo lo demás sigue sin aclararse: no hay un aviso público de seguridad, número CVE, parche ni indicación de qué productos o versiones se ven afectados. El comunicado de prensa cita como fuente a «autoridades federales de inteligencia», presumiblemente organismos federales estadounidenses; no se sabe cuáles. A fecha de 26 de septiembre, no hay ninguna entrada en Security Updates ni en los avisos de GitHub de Kiteworks. Públicas son la declaración citada arriba y el comunicado de prensa del 25 de septiembre.

Ante TechCrunch, el CISO de Kiteworks Frank Balonis hizo la declaración con el mismo tenor literal. La BKA rechazó hacer una declaración a heise por motivos tácticos de investigación, y la BSI no respondió. El FBI no quiso pronunciarse ante TechCrunch y no hubo respuesta de CISA. Según TechCrunch, un cliente del sector sanitario desconectó inmediatamente su servidor de la red, con restricciones perceptibles en sus operaciones.

El comunicado de prensa difiere de la comunicación por correo a los clientes en un punto: habla de una ventana de apagado de nueve horas, mientras que el aviso a los clientes habla de seis horas. Según el comunicado, la recomendación afecta únicamente a instalaciones operadas por los propios clientes (On-Premises, AWS, Azure). Según el fabricante, no se ven afectadas las filiales Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai y 123FormBuilder.

## El aviso a los clientes

El correo electrónico a los clientes del 25 de septiembre incluye, además de la advertencia, un calendario por zona horaria e instrucciones para clústeres. Al convertir las horas a UTC, todas las regiones tienen la misma ventana, de 02:00 a 08:00 UTC.

| Zona horaria | Ciudad | Inicio | Fin |
|---|---|---|---|
| AEST (UTC+10) | Sídney | Sáb., 12:00 | Sáb., 18:00 |
| SGT (UTC+8) | Singapur | Sáb., 10:00 | Sáb., 16:00 |
| IDT (UTC+3) | Tel Aviv | Sáb., 05:00 | Sáb., 11:00 |
| CEST (UTC+2) | Ámsterdam, Zúrich | Sáb., 04:00 | Sáb., 10:00 |
| BST (UTC+1) | Londres | Sáb., 03:00 | Sáb., 09:00 |
| EDT (UTC−4) | Nueva York | Vie., 22:00 | Sáb., 04:00 |
| CDT (UTC−5) | Chicago | Vie., 21:00 | Sáb., 03:00 |
| MDT (UTC−6) | Denver | Vie., 20:00 | Sáb., 02:00 |
| PDT (UTC−7) | San Francisco | Vie., 19:00 | Sáb., 01:00 |

Para los clústeres con varios servidores, Kiteworks especifica un orden fijo:

1.  **Activar el modo de mantenimiento** en System Setup > Maintenance Mode, para que los usuarios ya no puedan acceder.

2.  **Crear una copia de seguridad:** una instantánea de cada nodo o una copia de seguridad de la base de datos de Kiteworks (System Setup > Cluster Configuration > System Configuration). Solo se conserva una copia de seguridad de la base de datos; cada nueva sustituye a la anterior.

3.  **Registrar los roles:** en System Setup > Locations, la columna Assigned Roles muestra qué nodos tienen el rol Application; el nodo Application principal está marcado con un asterisco. Anote los nodos y sus direcciones IP, pues serán necesarios para el reinicio.

4.  **Apagar en este orden:** primero todos los nodos sin rol Application, después los demás nodos Application y, por último, el nodo Application principal. Puede hacerse mediante la pestaña Shut Down de cada nodo o mediante la consola del hipervisor (por ejemplo, VMware o AWS), si la interfaz de Kiteworks ya no está accesible.

5.  **Reiniciar en el orden inverso** mediante el hipervisor, ya que la consola de administración solo estará accesible cuando haya suficientes nodos en ejecución (apéndice E de la Administrator Guide): primero el nodo Application principal, después los demás nodos Application individualmente y solo cuando el anterior esté completamente en funcionamiento, para que los servidores de bases de datos puedan formar un quórum. Después los servidores de almacenamiento, luego los demás roles (Repositories Gateway, Search, SFTP, Antivirus) y, por último, los servidores web.

6.  **Desactivar el modo de mantenimiento** tan pronto como todos los nodos aparezcan en verde en el Cluster Health Dashboard de la página de estado de la consola de administración.

Además, ante una consulta, el soporte de Kiteworks confirmó que ninguna de las filiales de Kiteworks se ve afectada.

## Posibles causas: teorías

Mientras Kiteworks no publique detalles, la causa seguirá sin aclararse. Las siguientes explicaciones son hipótesis que pueden deducirse de los datos conocidos; algunas también se debaten en los comentarios de la noticia de heise. Ninguna está confirmada.

Tres datos delimitan el espacio de posibilidades. Primero, la advertencia menciona una ventana fija en vez de un apagado indefinido hasta que haya un parche. Segundo, también deben desconectarse los sistemas no accesibles desde Internet. Tercero, la ventana tiene lugar mundialmente a la misma hora (de 02:00 a 08:00 UTC), y no durante la noche local de cada lugar. Una vulnerabilidad clásica explotable a través de Internet no explicaría los dos primeros puntos: desconectar el sistema de Internet ayudaría, y debería hacerse hasta que estuviera disponible el parche.

### 1. Las autoridades conocen una hora prevista

Las autoridades policiales conocen ocasionalmente de antemano el momento de una campaña planificada, por ejemplo a partir de comunicaciones vigiladas de un grupo de delincuentes o de infraestructura incautada. Las explotaciones masivas de productos de intercambio de archivos suelen ejecutarse en una ventana breve y coordinada, a menudo durante fines de semana o festivos, cuando hay menos personal disponible. Kiteworks es el sucesor de Accellion, cuya File Transfer Appliance fue atacada precisamente de este modo en 2020 y 2021: se extrajeron datos mediante varias vulnerabilidades y posteriormente se extorsionó a las organizaciones afectadas.

A favor de esta hipótesis está la ventana acotada del fin de semana. En contra está la objeción planteada por varios comentaristas en heise: la advertencia se envió a todos los clientes, por lo que los atacantes probablemente lo sepan y puedan simplemente aplazar el ataque. Sin embargo, un aplazamiento daría tiempo al fabricante para desarrollar un parche.

### 2. El fabricante aún no conoce la vulnerabilidad

También es posible que Kiteworks, aparte del aviso de las autoridades, no disponga de detalles técnicos; es decir, que no conozca ni el componente afectado ni pueda recomendar un parche o un cambio de configuración. En ese caso, el apagado es la única medida que funciona sin conocer la vulnerabilidad, y el final fijo es un compromiso que los clientes estarán más dispuestos a aceptar. En los comentarios de heise se plantea la sospecha de que el fabricante podría dejar algunos sistemas en línea como cebo durante la ventana para observar el ataque. No hay pruebas de ello.

A favor está que no se menciona ningún aviso ni mitigación. En contra está que Kiteworks afirma colaborar con Mandiant y que, ante una advertencia de las autoridades, normalmente se dispone al menos de indicadores.

### 3. Una puerta trasera ya implantada con activación temporal

La recomendación de apagar también los sistemas internos encaja con un escenario en el que el ataque no llega desde fuera, sino que ya está preparado en los dispositivos: por ejemplo, una puerta trasera procedente de una comprometida anterior que se activa a una hora fija o contacta con un servidor de control. Un sistema apagado no puede ejecutar nada en ese momento.

A favor está que la accesibilidad desde Internet no desempeña ningún papel en este escenario. En contra está que, en ese caso, un fabricante recomendaría más bien comprobar si hubo una comprometida y reinstalar, en lugar de reiniciar al cabo de seis horas.

### 4. Compromiso en el lado del fabricante

Otra vía que alcanza sistemas internos son las conexiones establecidas desde el dispositivo hacia el fabricante, por ejemplo para actualizaciones, verificación de licencias o mantenimiento remoto. Si un canal de este tipo está comprometido, un cortafuegos no protege frente al tráfico entrante. En este escenario, el apagado daría al fabricante una ventana para limpiar su propia infraestructura, cambiar claves o certificados y permitir conexiones de nuevo solo después.

A favor está el momento uniforme a escala mundial, que encaja con una acción coordinada por parte del fabricante. En contra está que el fabricante recomendaría más bien bloquear las conexiones salientes que apagar completamente los sistemas.

### 5. Medida complementaria a una operación de las autoridades

Por último, cabe la posibilidad de que las autoridades actúen contra la infraestructura de los atacantes en el mismo periodo y quieran impedir que estos ataquen rápidamente como reacción. Eso explicaría la breve ventana y el papel de las autoridades policiales. Que la BKA rechace hacer una declaración por motivos tácticos de investigación apunta a investigaciones en curso, pero no prueba esta variante.

### Críticas a la comunicación

En los comentarios de heise predomina el escepticismo, y las objeciones son objetivamente comprensibles: sin información sobre la vulnerabilidad, no puede evaluarse si bastaba con desconectar de Internet mediante un cortafuegos. Una ventana temporal sin un parche anunciado deja abierto qué se aplica después de las 10:00. Y una advertencia enviada únicamente por correo electrónico a los clientes no llega a todos los operadores, por ejemplo en el caso de socios, proveedores de servicios o cambios de personal. Con independencia de cuál de las teorías sea correcta: quien opere Kiteworks debería revisar los registros tras volver a poner los sistemas en marcha y vigilar los canales del fabricante hasta que haya un aviso.

## Fuentes

1.  [heise online: Ataque de día cero inminente: KiteWorks insta a los clientes a apagar los servidores](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): primera información con extractos del correo electrónico a los clientes y la ventana temporal; actualización del 25/09, 17:41, con la respuesta de la BKA.

2.  [heise online (EN): Ataque de día cero inminente: KiteWorks insta a los clientes a apagar los servidores](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): versión en inglés con el texto original literal del CISO.

3.  [Kiteworks: Actualizaciones de seguridad](https://www.kiteworks.com/company/security-updates/): canal oficial del fabricante, sin entrada sobre la advertencia a fecha de 26/09/2026.

4.  [Kiteworks: Avisos de seguridad en GitHub](https://github.com/kiteworks/security-advisories/security): lista de avisos del fabricante, última entrada del 27/05/2026.

5.  [Kiteworks: Sala de prensa](https://www.kiteworks.com/newsroom/): comunicados oficiales, desde el 25/09/2026 con el comunicado de prensa sobre el apagado.

6.  [Foro de heise: Comentarios sobre la noticia](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): debate de lectores con las objeciones sobre la ventana temporal fija y el apagado de sistemas internos, así como la teoría del cebo.

7.  [CISA: Explotación de Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): aviso sobre la explotación de Accellion FTA en 2020/2021 con extorsión posterior; Accellion es el nombre anterior de Kiteworks.

8.  [Kiteworks: Caso de estudio Mandiant](https://www.kiteworks.com/case-study-mandiant/): declaración del fabricante sobre la colaboración con Mandiant.

9.  [TechCrunch: Kiteworks insta a los clientes a apagar sus servidores ante la amenaza «inminente» de un ciberataque](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): declaración del CISO, hora de envío de la advertencia, reacciones del FBI y CISA, y efectos en un cliente.

10.  [Kiteworks: Aviso de apagado preventivo (comunicado de prensa)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): comunicado oficial del 25/09/2026 con información sobre instancias alojadas, la versión 9.5.1 y las filiales no afectadas.

11.  [BleepingComputer: Kiteworks insta a apagar los servidores durante 6 horas por posibles ataques de día cero](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): ventana temporal por regiones y contexto de ataques anteriores contra productos de intercambio de archivos.

12.  [Computer Weekly: Ante la expectativa de un ciberataque, Kiteworks pide a los usuarios que apaguen los servidores](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): valoración de watchTowr sobre la inusual recomendación de apagado.
