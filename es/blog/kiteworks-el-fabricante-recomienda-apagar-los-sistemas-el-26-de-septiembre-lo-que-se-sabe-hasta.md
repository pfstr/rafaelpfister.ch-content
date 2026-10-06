---
title: "Kiteworks: el fabricante recomienda apagar los sistemas el 26 de septiembre; lo que se sabe hasta ahora"
navTitle: "Apagado de Kiteworks"
description: "Kiteworks ha pedido por correo electrónico a sus clientes que apaguen todos los sistemas el sábado 26/09/2026, de 04:00 a 10:00. El motivo es una advertencia de las fuerzas de seguridad sobre un posible ataque. Desde el 27/09 se ha levantado la recomendación; no hay CVE ni un nuevo parche. Totemomail no está afectado."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min de lectura"
themen:
  - totemomail
  - sicherheitsluecken
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
translationSourceHash: 93bc9f973258d524a87baa5fe75957444b339bcac669281db049e3f1e5817813
translationModel: gpt-5.6-terra
translatedAt: 2026-09-28T09:59:49.018Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/kiteworks-el-fabricante-recomienda-apagar-los-sistemas-el-26-de-septiembre-lo-que-se-sabe-hasta
---

# Kiteworks: el fabricante recomienda apagar los sistemas el 26 de septiembre; lo que se sabe hasta ahora

Kiteworks pidió a sus clientes por correo electrónico el 25 de septiembre de 2026 que apagaran todos los sistemas de Kiteworks el sábado 26 de septiembre, de 04:00 a 10:00 (hora de Europa Central). Según la carta del CISO Frank Balonis, el fabricante cuenta con indicios de las fuerzas de seguridad de que este fin de semana podría producirse un ataque contra sistemas de Kiteworks. El soporte al cliente justifica el apagado como protección frente a posibles ataques de día cero. heise online confirmó por teléfono con el soporte la autenticidad del mensaje.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Asistencia de emergencia para redirigir el flujo de correo</p>
<p>Si necesita ayuda para redirigir el flujo de correo antes del apagado y restablecerlo después, utilice el <a href="https://adeptio.ch/">formulario de contacto en adeptio.ch</a>. También responderé con poca antelación.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Actualización del 28 de septiembre de 2026: Kiteworks levanta la recomendación de apagado</p>
<p>Kiteworks ha añadido una nota al comunicado de prensa: desde el 27 de septiembre, la recomendación de apagado deja de aplicarse a todos los clientes.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Quienes aún no hayan vuelto a poner en marcha sus sistemas pueden hacerlo ahora. Quienes operen Advanced Forms por su cuenta deben ponerse en contacto con el soporte de Kiteworks antes de reiniciar. Las instancias alojadas por Kiteworks vuelven a estar en funcionamiento. Sigue sin haber número CVE, ninguna versión nueva posterior a la 9.5.1, indicadores de compromiso ni información sobre si se intentó un ataque o qué motivó la advertencia. La página de actualizaciones de seguridad y los avisos de GitHub no han cambiado.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Actualización del 25 de septiembre de 2026: declaración de Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail no está afectado.</strong> Queda por aclarar si Kiteworks EPG (Email Protection Gateway) está afectado.</p>
</div>

## Cronología

Todas las horas están expresadas en horario de verano de Europa Central (CEST). Cuando no se indica hora, no hay una indicación horaria fiable.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Aviso a los clientes</p>
<p>El CISO Frank Balonis informa a los clientes por correo electrónico sobre indicios de las fuerzas de seguridad de un posible ataque ese fin de semana y recomienda un apagado de seis horas. Según el aviso, todas las vulnerabilidades conocidas están corregidas en la versión 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Primeras informaciones en medios</p>
<p>heise online informa de que el soporte de Kiteworks confirma la autenticidad del mensaje y justifica el apagado como protección frente a posibles ataques de día cero. Poco después siguen TechCrunch, BleepingComputer, Computer Weekly y otros medios.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre, 17:41</p>
<p class="timeline__titel">La BKA no se pronuncia</p>
<p>heise añade que la BKA rechaza hacer declaraciones por razones tácticas de investigación. La BSI no responde y el FBI declina hacer comentarios a TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Declaración y comunicado de prensa</p>
<p>Kiteworks califica el apagado como una medida de precaución sin compromiso conocido. El comunicado de prensa cita a «autoridades federales de inteligencia» como fuente y enumera las filiales no afectadas, entre ellas totemo. El fabricante apaga por sí mismo las instancias alojadas por Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sáb., 26 de septiembre, de 04:00 a 10:00</p>
<p class="timeline__titel">Ventana de apagado</p>
<p>La ventana tiene lugar simultáneamente en todo el mundo: de 02:00 a 08:00 UTC, de 12:00 a 18:00 en Sídney y de viernes 22:00 a sábado 04:00 en Nueva York.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sáb., 26 de septiembre, 10:00</p>
<p class="timeline__titel">Fin de la ventana</p>
<p>Finaliza la ventana indicada en el correo electrónico a los clientes. La revocación formal de la recomendación se produce el 27 de septiembre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Dom., 27 de septiembre</p>
<p class="timeline__titel">Recomendación levantada</p>
<p>Kiteworks añade al comunicado de prensa que la recomendación de apagado queda levantada para todos los clientes y que los sistemas pueden volver a funcionar. Las instancias alojadas vuelven a estar operativas. Los clientes con Advanced Forms operados por cuenta propia deben ponerse en contacto con el soporte.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Situación a lunes, 28 de septiembre</p>
<p class="timeline__titel">Sigue sin aclararse</p>
<p>No hay aviso público, número CVE, versión nueva, indicadores, datos sobre la vulnerabilidad ni informes de un ataque consumado o intentado.</p>
</li>
</ol>

## Lo que se sabe

La recomendación es válida en todo el mundo; el correo electrónico indica la ventana para todas las zonas horarias, desde AEST hasta PDT. Kiteworks aconseja apagar los sistemas antes del inicio de la ventana, incluso si no son accesibles desde Internet.

Por ahora, casi todo lo demás sigue sin aclararse: no hay un aviso público de seguridad, número CVE, parche ni información sobre qué productos o versiones están afectados. El comunicado de prensa cita como fuente a «autoridades federales de inteligencia», probablemente organismos federales estadounidenses; se desconoce cuáles. A fecha del 28 de septiembre, no hay ninguna entrada en Security Updates ni en los avisos de GitHub de Kiteworks; la última entrada de GitHub es del 27 de mayo de 2026. Públicamente están disponibles la declaración citada arriba y el comunicado de prensa del 25 de septiembre.

Ante TechCrunch, el CISO de Kiteworks Frank Balonis emitió la declaración con el mismo texto. La BKA declinó hacer declaraciones ante heise por razones tácticas de investigación, y la BSI no respondió. El FBI no quiso pronunciarse ante TechCrunch, y un portavoz de CISA no quiso hacer declaraciones públicas. Según TechCrunch, un cliente del sector sanitario desconectó inmediatamente su servidor, con restricciones perceptibles en sus operaciones: durante un tiempo, los médicos solo pudieron contactar con sus pacientes con retraso. Según un investigador de seguridad citado por TechCrunch, al menos 1.000 sistemas de Kiteworks son accesibles desde Internet; BornCity habla de más de 1.000 organizaciones que recibieron la advertencia.

El comunicado de prensa difiere del correo electrónico a los clientes en un aspecto: habla de una ventana de apagado de nueve horas, mientras que el aviso a los clientes habla de seis horas. Según el comunicado, la recomendación solo afecta a instalaciones operadas por el propio cliente (On-Premises, AWS, Azure). Según el fabricante, no están afectadas las filiales Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai y 123FormBuilder.

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

Para clústeres con varios servidores, Kiteworks establece un orden fijo:

1.  **Activar el modo de mantenimiento** en System Setup > Maintenance Mode para que ningún usuario pueda acceder.

2.  **Crear una copia de seguridad:** una instantánea de cada nodo o una copia de seguridad de la base de datos de Kiteworks (System Setup > Cluster Configuration > System Configuration). Solo se conserva una copia de seguridad de la base de datos; cada nueva sustituye a la anterior.

3.  **Registrar las funciones:** en System Setup > Locations, la columna Assigned Roles muestra qué nodos tienen la función Application; el nodo Application principal está marcado con un asterisco. Anote los nodos y sus direcciones IP, pues se necesitarán para el reinicio.

4.  **Apagar en este orden:** primero todos los nodos sin función Application, después los demás nodos Application y, por último, el nodo Application principal. Puede hacerse desde la pestaña Shut Down del nodo correspondiente o mediante la consola del hipervisor (por ejemplo, VMware o AWS) si la interfaz de Kiteworks deja de estar accesible.

5.  **Reiniciar en orden inverso** mediante el hipervisor, ya que la consola de administración solo estará accesible cuando haya suficientes nodos en ejecución (anexo E de la Administrator Guide): primero el nodo Application principal, después los demás nodos Application uno a uno y únicamente cuando el anterior esté completamente operativo, para que los servidores de base de datos puedan formar un quórum. Después, los servidores de almacenamiento, luego las demás funciones (Repositories Gateway, Search, SFTP, Antivirus) y, por último, los servidores web.

6.  **Desactivar el modo de mantenimiento** en cuanto todos los nodos aparezcan en verde en el Cluster Health Dashboard de la página de estado de la consola de administración.

Además, el soporte de Kiteworks confirmó, tras ser consultado, que ninguna de las filiales de Kiteworks está afectada.

## Posibles causas: teorías

Mientras Kiteworks no publique detalles, la causa seguirá sin aclararse. Las explicaciones siguientes son hipótesis que pueden deducirse de los datos conocidos; algunas también se discuten en los comentarios sobre la noticia de heise. Ninguna está confirmada.

Tres datos limitan el abanico de posibilidades. En primer lugar, la advertencia menciona una ventana fija en lugar de un apagado indefinido hasta que haya un parche. En segundo lugar, también deben desconectarse los sistemas que no son accesibles desde Internet. En tercer lugar, la ventana se sitúa a la misma hora en todo el mundo (de 02:00 a 08:00 UTC), en vez de coincidir con la noche local. Una vulnerabilidad clásica explotable a través de Internet no explicaría los dos primeros puntos: basta con desconectar el sistema de Internet hasta que esté disponible el parche.

### 1. Las autoridades conocen una fecha prevista

Las fuerzas de seguridad conocen ocasionalmente de antemano la fecha de una campaña prevista, por ejemplo, a partir de comunicaciones vigiladas de un grupo de delincuentes o de infraestructura incautada. Las explotaciones masivas de productos de intercambio de archivos suelen producirse en una ventana breve y coordinada, a menudo durante fines de semana o días festivos, cuando hay menos personal disponible. Kiteworks es el sucesor de Accellion, cuya File Transfer Appliance fue atacada precisamente de esta manera en 2020 y 2021, entonces atribuida al grupo Clop: se extrajeron datos a través de varias vulnerabilidades y posteriormente se extorsionó a las organizaciones afectadas.

A favor de esta teoría está la ventana acotada durante el fin de semana. En contra está la objeción planteada por varios comentaristas en heise: la advertencia se envió a todos los clientes, por lo que los atacantes probablemente lo sabrán y podrán simplemente aplazar el ataque. Sin embargo, un aplazamiento daría tiempo al fabricante para crear un parche.

### 2. El fabricante aún no conoce la vulnerabilidad

También es posible que, aparte del aviso de las autoridades, Kiteworks no disponga de detalles técnicos; es decir, que no conozca ni el componente afectado ni pueda recomendar un parche o un cambio de configuración. En ese caso, el apagado sería la única medida eficaz sin conocer la vulnerabilidad, y el final fijo un compromiso que los clientes estarían más dispuestos a aceptar. En los comentarios de heise se plantea la conjetura de que el fabricante podría dejar algunos sistemas en línea como señuelo durante la ventana para observar el ataque. No hay pruebas de ello.

A favor está que no se menciona ni un aviso ni una mitigación. En contra está que Kiteworks afirma colaborar con Mandiant y que, ante una advertencia de las autoridades, por lo general se dispone al menos de indicadores.

### 3. Una puerta trasera ya implantada con activación programada

La recomendación de apagar también los sistemas internos encaja con un escenario en el que el ataque no procede del exterior, sino que ya se ha preparado en los appliances: por ejemplo, una puerta trasera derivada de un compromiso anterior, que se activa a una hora determinada o se conecta a un servidor de control. Un sistema apagado no puede ejecutar nada en ese momento.

A favor está que la accesibilidad desde Internet no importa en este escenario. En contra está que, en ese caso, un fabricante probablemente recomendaría comprobar si hay compromiso y reinstalar, en vez de reiniciar después de seis horas.

### 4. Compromiso del lado del fabricante

Otra vía que alcanza sistemas internos son las conexiones que el appliance establece con el fabricante, por ejemplo para actualizaciones, comprobación de licencias o mantenimiento remoto. Si un canal de este tipo está comprometido, un firewall no protege frente al tráfico entrante. En este escenario, el apagado daría al fabricante una ventana para limpiar su propia infraestructura, cambiar claves o certificados y permitir de nuevo las conexiones solo después.

A favor está la hora uniforme a nivel mundial, que encaja con una acción coordinada del lado del fabricante. En contra está que el fabricante probablemente recomendaría bloquear las conexiones salientes en lugar de apagar totalmente los sistemas.

### 5. Medida complementaria a una operación de las autoridades

Por último, es concebible que las autoridades actúen contra la infraestructura de los atacantes durante el mismo periodo y quieran evitar que estos ataquen rápidamente como reacción. Esto explicaría la ventana breve y el papel de las fuerzas de seguridad. Que la BKA rechace hacer declaraciones por razones tácticas de investigación apunta a pesquisas en curso, pero no demuestra esta posibilidad.

### Críticas a la comunicación

En los comentarios de heise predomina el escepticismo, y las objeciones son objetivamente comprensibles: sin información sobre la vulnerabilidad, no se puede valorar si habría bastado con desconectar Internet mediante un firewall. Una ventana temporal sin un parche anunciado deja abierto qué se aplica después de las 10:00. Y una advertencia enviada solo por correo electrónico a los clientes no llega a todos los operadores, por ejemplo, en el caso de socios, proveedores de servicios o cambios de personal. Independientemente de qué teoría sea correcta, quienes operen Kiteworks deberían revisar los registros tras volver a ponerlo en marcha y vigilar los canales del fabricante hasta que haya un aviso.

## Después de la ventana: qué pueden hacer ahora los operadores

Kiteworks levantó la recomendación de apagado el 27 de septiembre, pero no ha publicado detalles técnicos. Por tanto, no es posible valorar si se ha eliminado el peligro ni cómo. Al volver a poner los sistemas en marcha y después, resultan útiles los siguientes pasos:

1.  **Comprobar la versión:** ¿todos los nodos ejecutan la versión 9.5.1? Según el fabricante, todas las vulnerabilidades conocidas están corregidas en ella.

2.  **Comprobar el estado del clúster:** en el Cluster Health Dashboard, todos los nodos deberían aparecer en verde y el modo de mantenimiento debería estar desactivado.

3.  **Evaluar los registros:** revisar los inicios de sesión, las acciones de administrador y las descargas inusuales de archivos en torno a la ventana de apagado, especialmente en sistemas que no se apagaron o lo hicieron tarde.

4.  **Restringir la accesibilidad:** cuando sea posible, bloquear el acceso desde Internet a la interfaz de administración y habilitar solo los servicios necesarios.

5.  **Advanced Forms:** quienes operen el módulo por cuenta propia deben aclarar el reinicio previamente con el soporte de Kiteworks.

6.  **Vigilar los canales:** Security Updates, los avisos de GitHub, Newsroom y los correos electrónicos para clientes de Kiteworks, hasta que se publique un aviso con detalles técnicos.

## Fuentes

1.  [heise online: Ataque de día cero inminente: KiteWorks insta a los clientes a apagar los servidores](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): primera información con extractos del correo electrónico a los clientes y la ventana temporal; actualización del 25/09 a las 17:41 con la respuesta de la BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): versión en inglés con el texto original del CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): canal oficial del fabricante, sin entrada sobre la advertencia a fecha de 28/09/2026.

4.  [Kiteworks: Security Advisories en GitHub](https://github.com/kiteworks/security-advisories/security): lista de avisos del fabricante; a fecha de 28/09/2026, la última entrada es del 27/05/2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): comunicados oficiales, con el comunicado de prensa sobre el apagado desde el 25/09/2026.

6.  [Foro de heise: comentarios sobre la noticia](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): debate de lectores con objeciones sobre la ventana temporal fija y el apagado de sistemas internos, así como la teoría del señuelo.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): aviso sobre la explotación de Accellion FTA en 2020/2021 con extorsión posterior; Accellion es el nombre anterior de Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): información del fabricante sobre su colaboración con Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): declaración del CISO, hora de envío de la advertencia, reacciones del FBI y CISA (añadido), efectos para un cliente y número de sistemas accesibles desde Internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (comunicado de prensa)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): comunicado oficial del 25/09/2026 con datos sobre instancias alojadas, la versión 9.5.1 y las filiales no afectadas; ampliado con la nota del 27/09/2026 de que se ha levantado la recomendación de apagado.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): ventana temporal por regiones y contextualización de ataques anteriores contra productos de intercambio de archivos.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): valoración de watchTowr sobre la inusual recomendación de apagado.

13.  [BornCity: Kiteworks: más de 1.000 organizaciones deben apagar servidores](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): número de organizaciones notificadas y sectores en la región germanoparlante.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): contextualización de los ataques de Clop contra Accellion en 2020/2021 y cita de watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): informe del 28/09/2026 sobre la revocación de la recomendación y la operación de las instancias alojadas.
