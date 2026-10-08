---
title: "Kiteworks: el fabricante recomienda apagar los sistemas el 26 de septiembre: lo que se sabe hasta ahora"
navTitle: "Apagado de Kiteworks"
description: "Kiteworks pidió a sus clientes que apagaran todos los sistemas el sábado 26/09/2026 de 04:00 a 10:00. Informe final: durante el apagado, el fabricante encontró y corrigió una vulnerabilidad crítica sin CVE; el 30/09 siguieron 125 avisos, entre ellos CVE-2026-54154 (CVSS 10.0) en Email Protection Gateway. Totemomail no está afectado."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 min de lectura"
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
translationSourceHash: 15d32c4610fafdeff76397f77ace28ebf6f4aab220c10dc03f5dd4713fab9e48
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:54:16.904Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/kiteworks-el-fabricante-recomienda-apagar-los-sistemas-el-26-de-septiembre-lo-que-se-sabe-hasta
---

# Kiteworks: el fabricante recomienda apagar los sistemas el 26 de septiembre: lo que se sabe hasta ahora

El 25 de septiembre de 2026, Kiteworks pidió a sus clientes por correo electrónico que apagaran todos los sistemas Kiteworks el sábado 26 de septiembre, de 04:00 a 10:00 (hora de Europa Central). Según la carta del CISO Frank Balonis, el fabricante había recibido indicios de las fuerzas de seguridad de que podría producirse un ataque contra sistemas Kiteworks ese fin de semana. El soporte al cliente justificó el apagado como protección frente a posibles ataques de día cero. heise online confirmó la autenticidad del mensaje por teléfono con el soporte.

<div class="update-hinweis">
<p class="update-hinweis__titel">Informe final del 7 de octubre de 2026</p>
<p>Desde la perspectiva del fabricante, el incidente está cerrado. La recomendación de apagado dejó de aplicarse el 27 de septiembre y, hasta la fecha, no se conoce ningún ataque contra sistemas de Kiteworks o de clientes. Los resultados más importantes:</p>
<ul>
<li><strong>Vulnerabilidad crítica encontrada durante el apagado:</strong> Según el comunicado de prensa del 28 de septiembre, Kiteworks descubrió, durante el análisis con las autoridades federales, una vulnerabilidad crítica hasta entonces desconocida en una función activada en menos del 1 % de los clientes. El fabricante desarrolló e implementó una corrección durante la ventana de apagado y activó además una capa de protección en todos los entornos. No se ha publicado qué función se vio afectada; hasta hoy tampoco existe un número CVE para ello.</li>
<li><strong>125 avisos el 30 de septiembre:</strong> Dos días después, Kiteworks publicó en GitHub 125 avisos de seguridad para Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28) y MFT Server (3); 12 de ellos críticos y 49 altos. Todos están corregidos en versiones hasta la 9.5.1 inclusive; la mayoría se notificaron a través del programa de recompensas por errores de YesWeHack. Según el estado actual, no tienen relación con la vulnerabilidad de la ventana de apagado.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0):</strong> La vulnerabilidad más grave afecta a Email Protection Gateway anterior a la versión 9.4.1. Un atacante no autenticado puede ejecutar código con privilegios de root mediante puntos finales accesibles públicamente. Diez de los 12 avisos críticos afectan a Email Protection Gateway.</li>
<li><strong>Sin explotación conocida:</strong> No hay informes de ataques para ninguna de las vulnerabilidades; a fecha del 7 de octubre, el catálogo CISA KEV no contiene ninguna entrada de Kiteworks de 2026. Según BleepingComputer, Shadowserver cuenta casi 400 instancias de Kiteworks accesibles desde Internet.</li>
<li><strong>Totemomail:</strong> No aparece en ninguno de los avisos y, según el fabricante, no se vio afectado por el apagado.</li>
</ul>
<p><strong>Medidas necesarias:</strong> Quien opere Kiteworks por su cuenta debería actualizar todos los componentes a la versión 9.5.1; Email Protection Gateway tiene prioridad. Los detalles se encuentran en la sección <a href="#abschlussbericht">Informe final</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Actualización del 28 de septiembre de 2026: Kiteworks retira la recomendación de apagado</p>
<p>Kiteworks añadió una nota al comunicado de prensa: desde el 27 de septiembre, la recomendación de apagado deja de aplicarse a todos los clientes.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Quien aún no haya vuelto a poner en marcha sus sistemas puede hacerlo ahora. Quien opere Advanced Forms por su cuenta debe ponerse en contacto con el soporte de Kiteworks antes de reiniciar. Las instancias alojadas por Kiteworks vuelven a estar operativas. Sigue sin haber un número CVE, ninguna nueva versión posterior a la 9.5.1, indicadores de una vulneración ni información sobre si se intentó un ataque o qué motivó la advertencia. La página de actualizaciones de seguridad y los avisos de GitHub no han cambiado.</p>
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

Todas las horas están en horario de verano de Europa Central (CEST). Cuando no se indica una hora, no existe una indicación horaria fiable.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Aviso a los clientes</p>
<p>El CISO Frank Balonis informa a los clientes por correo electrónico sobre indicios de las fuerzas de seguridad de un posible ataque durante ese fin de semana y recomienda un apagado de seis horas. Según el aviso, todas las vulnerabilidades conocidas están corregidas en la versión 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Primeras informaciones en medios</p>
<p>heise online informa de que el soporte de Kiteworks confirma la autenticidad del mensaje y justifica el apagado con la protección frente a posibles ataques de día cero. Poco después siguen TechCrunch, BleepingComputer, Computer Weekly y otros.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre, 17:41</p>
<p class="timeline__titel">La BKA no se pronuncia</p>
<p>heise añade: la BKA rechaza hacer una declaración por razones tácticas de investigación. La BSI no responde y el FBI declina hacer comentarios ante TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Vie., 25 de septiembre</p>
<p class="timeline__titel">Declaración y comunicado de prensa</p>
<p>Kiteworks describe el apagado como una medida de precaución sin vulneración conocida. El comunicado de prensa cita a «autoridades federales de inteligencia» como fuente y enumera las filiales no afectadas, entre ellas totemo. El fabricante apaga por sí mismo las instancias alojadas por Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sáb., 26 de septiembre, de 04:00 a 10:00</p>
<p class="timeline__titel">Ventana de apagado</p>
<p>La ventana se produce simultáneamente en todo el mundo: de 02:00 a 08:00 UTC, en Sídney de 12:00 a 18:00 y en Nueva York del viernes a las 22:00 al sábado a las 04:00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sáb., 26 de septiembre, 10:00</p>
<p class="timeline__titel">Fin de la ventana</p>
<p>Finaliza la ventana indicada en el correo electrónico a los clientes. La retirada formal de la recomendación se produce el 27 de septiembre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Dom., 27 de septiembre</p>
<p class="timeline__titel">Recomendación retirada</p>
<p>Kiteworks añade al comunicado de prensa: la recomendación de apagado se retira para todos los clientes y los sistemas pueden volver a funcionar. Las instancias alojadas vuelven a estar operativas. Los clientes con Advanced Forms autogestionado deben ponerse en contacto con el soporte.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lun., 28 de septiembre</p>
<p class="timeline__titel">Vulnerabilidad crítica encontrada y corregida</p>
<p>Kiteworks informa en otro comunicado de prensa de que, al trabajar con las autoridades federales durante el apagado, se descubrió una vulnerabilidad crítica hasta entonces desconocida. Afecta a una función activada en menos del 1 % de los clientes. Se han implementado la corrección y una capa de protección adicional; no hay indicios de una vulneración. El fabricante no indica la función ni el número CVE.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mié., 30 de septiembre, desde las 18:38</p>
<p class="timeline__titel">125 avisos de seguridad en GitHub</p>
<p>Kiteworks publica 125 avisos para Core, Email Protection Gateway, Secure Data Forms y MFT Server, todos corregidos hasta la versión 9.5.1. La vulnerabilidad más grave es CVE-2026-54154 en Email Protection Gateway anterior a 9.4.1 (CVSS 10.0, ejecución de código con privilegios de root sin autenticación).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Jue., 1 de octubre</p>
<p class="timeline__titel">Informaciones en medios y aviso de MS-ISAC</p>
<p>BleepingComputer, SecurityOnline y otros informan sobre los avisos; MS-ISAC (Center for Internet Security) publica su propio aviso sobre CVE-2026-54154. No se conoce explotación alguna.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Situación al mié., 7 de octubre</p>
<p class="timeline__titel">Cierre</p>
<p>No hay informes de un ataque realizado o intentado ni ninguna entrada de Kiteworks en el catálogo CISA KEV. Siguen sin conocerse la función afectada, un número CVE para la vulnerabilidad de la ventana de apagado y el trasfondo de la advertencia de las autoridades.</p>
</li>
</ol>

## Lo que se sabe

La recomendación se aplica en todo el mundo; el correo electrónico indica la ventana para todas las zonas horarias, desde AEST hasta PDT. Kiteworks aconseja apagar los sistemas antes del inicio de la ventana, incluso si no son accesibles desde Internet.

Hasta el 28 de septiembre, casi todo lo demás seguía sin aclararse: no había ningún aviso público de seguridad, número CVE, parche ni información sobre qué productos o versiones estaban afectados. El comunicado de prensa cita como fuente a «autoridades federales de inteligencia», probablemente autoridades federales estadounidenses; hasta hoy no se sabe cuáles. En los avisos de GitHub de Kiteworks, la última entrada hasta entonces era del 27 de mayo de 2026; los avisos del 30 de septiembre se resumen en la sección [Informe final](#abschlussbericht).

Ante TechCrunch, el CISO de Kiteworks, Frank Balonis, emitió la declaración con el mismo texto. La BKA rechazó hacer una declaración ante heise por razones tácticas de investigación y la BSI no respondió. El FBI no quiso pronunciarse ante TechCrunch, y un portavoz de CISA no quiso hacerlo públicamente. Según TechCrunch, un cliente del sector sanitario desconectó inmediatamente su servidor, con restricciones perceptibles en el funcionamiento: los médicos solo pudieron contactar temporalmente con sus pacientes con retraso. Según un investigador de seguridad citado por TechCrunch, al menos 1000 sistemas Kiteworks son accesibles desde Internet; BornCity habla de más de 1000 organizaciones que recibieron la advertencia.

El comunicado de prensa difiere en un punto del correo electrónico a los clientes: habla de una ventana de apagado de nueve horas, mientras que el aviso a los clientes habla de seis horas. Según el comunicado, la recomendación solo afecta a instalaciones autogestionadas (on-premises, AWS, Azure). Según el fabricante, no están afectadas las filiales Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai y 123FormBuilder.

## El aviso a los clientes

El correo electrónico a los clientes del 25 de septiembre contiene, además de la advertencia, un calendario por zona horaria e instrucciones para clústeres. Al convertir las horas a UTC, resulta la misma ventana para todas las regiones: de 02:00 a 08:00 UTC.

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

Para clústeres con varios servidores, Kiteworks especifica una secuencia fija:

1.  **Activar el modo de mantenimiento** en System Setup > Maintenance Mode, para que ningún usuario pueda acceder.

2.  **Crear una copia de seguridad:** una instantánea de cada nodo o una copia de seguridad de la base de datos de Kiteworks (System Setup > Cluster Configuration > System Configuration). Solo se conserva una copia de seguridad de la base de datos; cada nueva sustituye a la anterior.

3.  **Registrar las funciones:** en System Setup > Locations, la columna Assigned Roles muestra qué nodos tienen el rol Application; el nodo Application primario está marcado con una estrella. Anote los nodos y sus direcciones IP, pues se necesitarán para el reinicio.

4.  **Apagar en este orden:** primero todos los nodos sin rol Application, después los demás nodos Application y, por último, el nodo Application primario. Puede hacerse mediante la pestaña Shut Down de cada nodo o mediante la consola del hipervisor (por ejemplo, VMware o AWS) si la interfaz de Kiteworks deja de estar accesible.

5.  **Reiniciar en orden inverso** mediante el hipervisor, ya que la consola de administración solo estará accesible cuando haya suficientes nodos en funcionamiento (anexo E de la Administrator Guide): primero el nodo Application primario, después los demás nodos Application individualmente y solo cuando el anterior esté plenamente operativo, para que los servidores de base de datos puedan formar un cuórum. Después, los servidores de almacenamiento, luego las demás funciones (Repositories Gateway, Search, SFTP, Antivirus) y, por último, los servidores web.

6.  **Desactivar el modo de mantenimiento** en cuanto todos los nodos aparezcan en verde en el Cluster Health Dashboard de la página de estado de la consola de administración.

Además, el soporte de Kiteworks confirmó, en respuesta a una consulta, que ninguna de las filiales de Kiteworks está afectada.

## Posibles causas: teorías

Esta sección se elaboró antes del 28 de septiembre; la valoración según el estado actual figura en el [Informe final](#abschlussbericht). Las siguientes explicaciones son hipótesis que pueden deducirse de los datos conocidos; algunas también se debaten en los comentarios sobre la noticia de heise. Ninguna de ellas ha sido confirmada.

Tres datos limitan el escenario. En primer lugar, la advertencia menciona una ventana temporal fija, en vez de un apagado indefinido hasta disponer de un parche. En segundo lugar, también deben desconectarse los sistemas no accesibles desde Internet. En tercer lugar, la ventana coincide en todo el mundo a la misma hora (de 02:00 a 08:00 UTC), en lugar de producirse durante la noche local de cada región. Una vulnerabilidad clásica explotable a través de Internet no explicaría los dos primeros puntos: bastaría con desconectar el sistema de Internet hasta que estuviera disponible el parche.

### 1. Las autoridades conocen una hora prevista

Las fuerzas de seguridad conocen ocasionalmente con antelación el momento de una campaña planificada, por ejemplo mediante comunicaciones vigiladas de un grupo delictivo o infraestructura incautada. Las explotaciones masivas de productos de intercambio de archivos suelen desarrollarse en una ventana breve y coordinada, a menudo durante fines de semana o festivos, cuando hay menos personal disponible. Kiteworks es el sucesor de Accellion, cuyo File Transfer Appliance fue atacado precisamente de esta forma en 2020 y 2021, entonces atribuido al grupo Clop: se extrajeron datos mediante varias vulnerabilidades y posteriormente se extorsionó a las organizaciones afectadas.

A favor de ello está la ventana acotada durante el fin de semana. En contra está la objeción planteada por varios comentaristas en heise: la advertencia se envió a todos los clientes, por lo que los atacantes probablemente lo sabrían y podrían simplemente aplazar el ataque. Sin embargo, un aplazamiento daría tiempo al fabricante para desarrollar un parche.

### 2. El propio fabricante aún no conoce la vulnerabilidad

También es posible que, aparte del aviso de las autoridades, Kiteworks no disponga de detalles técnicos: ni conoce el componente afectado ni puede recomendar un parche o cambio de configuración. En tal caso, el apagado sería la única medida efectiva sin conocer la vulnerabilidad, y el final fijo constituiría un compromiso que los clientes tendrían más disposición a aceptar. En los comentarios de heise se plantea la conjetura de que el fabricante podría dejar algunos sistemas en línea como señuelo durante la ventana para observar el ataque. No hay pruebas de ello.

A favor está que no se mencionan ni un aviso ni una mitigación. En contra está que Kiteworks afirma colaborar con Mandiant y que, normalmente, cuando hay una advertencia de las autoridades, al menos se dispone de indicadores.

### 3. Una puerta trasera ya implantada con activador temporal

La recomendación de apagar también los sistemas internos encaja con un escenario en el que el ataque no llega desde fuera, sino que ya está preparado en los appliances: por ejemplo, una puerta trasera derivada de una vulneración anterior que se activa a una hora fija o establece contacto con un servidor de control. Un sistema apagado no puede ejecutar nada en ese momento.

A favor está que la accesibilidad desde Internet no importa en este escenario. En contra está que, en tal caso, un fabricante probablemente recomendaría comprobar si hubo una vulneración y reinstalar, en lugar de reiniciar después de seis horas.

### 4. Vulneración del lado del fabricante

Otra vía que alcanza los sistemas internos son las conexiones establecidas desde el appliance hacia el fabricante, por ejemplo para actualizaciones, verificación de licencias o mantenimiento remoto. Si uno de esos canales está comprometido, un cortafuegos no protege frente al tráfico entrante. En este escenario, el apagado daría al fabricante una ventana para limpiar su propia infraestructura, cambiar claves o certificados y permitir conexiones de nuevo solo después.

A favor está la hora uniforme mundial, compatible con una acción coordinada del fabricante. En contra está que el fabricante probablemente recomendaría bloquear las conexiones salientes, en vez de apagar por completo los sistemas.

### 5. Medida complementaria a una operación de las autoridades

Por último, cabe la posibilidad de que las autoridades actúen contra la infraestructura de los atacantes durante el mismo periodo y quieran impedir que estos ataquen rápidamente como reacción. Eso explicaría la ventana corta y el papel de las fuerzas de seguridad. Que la BKA rechace hacer una declaración por razones tácticas de investigación apunta a investigaciones en curso, pero no demuestra esta variante.

### Críticas a la comunicación

En los comentarios de heise predomina el escepticismo, y las objeciones son objetivamente comprensibles: sin información sobre la vulnerabilidad, no se puede evaluar si bastaba con desconectar de Internet mediante un cortafuegos. Una ventana temporal sin un parche anunciado deja sin aclarar qué se aplica después de las 10:00. Y una advertencia enviada únicamente por correo electrónico a los clientes no llega a todos los operadores, por ejemplo, en socios, proveedores de servicios o tras cambios de personal. Independientemente de qué teoría sea correcta: quien opere Kiteworks debería revisar los registros tras volver a poner los sistemas en marcha y vigilar los canales del fabricante hasta que haya un aviso.

## Informe final

A fecha del 7 de octubre de 2026, el incidente está cerrado desde la perspectiva del fabricante. Los acontecimientos posteriores a la ventana de apagado pueden separarse en dos líneas: la vulnerabilidad encontrada durante el apagado y la publicación conjunta de avisos dos días después.

### La vulnerabilidad de la ventana de apagado

El 28 de septiembre, Kiteworks publicó un segundo comunicado de prensa. Según este, el fabricante colaboró con las autoridades federales durante todo el fin de semana; en ese proceso se descubrió una vulnerabilidad crítica hasta entonces desconocida, limitada a una función activada en menos del 1 % de los clientes. Kiteworks desarrolló e implementó una corrección durante la ventana de apagado y activó además una capa de protección en todos los entornos. La monitorización continua no mostró actividades sospechosas y no hay indicios de una vulneración de sistemas de Kiteworks o de clientes. Todos los demás productos de Kiteworks no están afectados.

No se han publicado la función afectada, un número CVE, las versiones con la corrección ni si las instalaciones autogestionadas recibieron automáticamente la corrección. La retirada del 27 de septiembre contenía una única excepción: los clientes con Advanced Forms autogestionado debían contactar con el soporte antes de reiniciar. Kiteworks no ha confirmado si esa era la función afectada.

Respecto a las teorías anteriores: el comunicado describe una vulnerabilidad que solo se descubrió durante la ventana. Esto encaja con la teoría 2 (el fabricante no conocía previamente la vulnerabilidad) combinada con la teoría 1 (las autoridades conocían una hora prevista). No hay confirmación para las teorías 3 a 5. Sigue sin conocerse qué sabían concretamente las autoridades y si se intentó un ataque.

### 125 avisos de seguridad del 30 de septiembre

El 30 de septiembre, a partir de las 18:38, Kiteworks publicó de una vez 125 avisos de seguridad en GitHub. Se distribuyen de la siguiente forma:

| Producto | Avisos | de ellos críticos |
|---|---|---|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Total** | **125** | **12** |

Por gravedad, hay 12 clasificaciones críticas, 49 altas, 52 medias y 12 bajas. Todas las vulnerabilidades están corregidas en versiones hasta la 9.5.1 inclusive; las entradas más antiguas afectan a la versión 9.2.1. Se trata, por tanto, de una divulgación posterior de correcciones ya entregadas, no de una versión nueva. Los avisos mencionan mayoritariamente como informantes a participantes del programa de recompensas por errores de YesWeHack. Kiteworks no establece ninguna relación con la vulnerabilidad de la ventana de apagado; una semana después de la ventana, el estado de los avisos sigue correspondiendo a la declaración del 25 de septiembre de que todas las vulnerabilidades conocidas están corregidas en la versión 9.5.1.

Los avisos críticos:

| CVE | Producto | CVSS 3.1 | corregido desde | Impacto |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Ejecución de código con privilegios de root sin autenticación |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Toma de control de cuentas |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Toma de control de cuentas |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Toma de control de cuentas mediante el restablecimiento de contraseña |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Toma de control de cuentas |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Toma de control de cuentas |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Elusión de funciones de seguridad |
| CVE-2026-102095, CVE-2026-102102 a 102105 | EPG | 9.1 | 9.5.0 | Acceso a recursos de red internos (SSRF) |

CVE-2026-54154 es la vulnerabilidad más grave: según el aviso, una combinación de errores de validación de entradas en puntos finales accesibles públicamente de Email Protection Gateway permite a un atacante no autenticado ejecutar código y, mediante otras debilidades locales, obtener privilegios de root en el appliance. MS-ISAC publicó su propio aviso al respecto el 1 de octubre. Para los administradores de correo, el gateway es la parte relevante de la publicación: normalmente se encuentra directamente en el flujo de correo y es accesible desde Internet.

### Explotación y distribución

No existen informes de explotación ni exploits públicos para ninguna de las vulnerabilidades. A fecha del 7 de octubre, el catálogo CISA KEV contiene únicamente las cuatro entradas de Accellion FTA de 2021. Según BleepingComputer, Shadowserver cuenta casi 400 instancias de Kiteworks accesibles desde Internet; no se sabe cuántas de ellas ya ejecutan la versión 9.5.1. Totemomail no aparece en ninguno de los avisos.

## Después de la ventana: qué pueden hacer ahora los operadores

La recomendación de apagado ha sido retirada y las vulnerabilidades conocidas están corregidas en la versión 9.5.1. Para las instalaciones autogestionadas, resultan razonables los siguientes pasos:

1.  **Comprobar la versión:** ¿todos los nodos y componentes (Core, Email Protection Gateway, Secure Data Forms, MFT Server) ejecutan la versión 9.5.1? Un Email Protection Gateway anterior a 9.4.1 está afectado por CVE-2026-54154 y debe actualizarse primero.

2.  **Comprobar el estado del clúster:** en el Cluster Health Dashboard, todos los nodos deberían estar en verde y el modo de mantenimiento debería estar desactivado.

3.  **Analizar los registros:** revisar inicios de sesión, acciones de administrador y descargas de archivos inusuales alrededor de la ventana de apagado, especialmente en sistemas que no se apagaron o se apagaron tarde.

4.  **Restringir la accesibilidad:** cuando sea posible, bloquear el acceso desde Internet a la interfaz de administración y habilitar únicamente los servicios necesarios.

5.  **Advanced Forms:** quien opere el módulo por su cuenta y aún no haya contactado con el soporte debe aclarar con Kiteworks si la corrección de la ventana de apagado llegó a su propia instalación.

6.  **Comparar los avisos:** los avisos de GitHub se pueden filtrar por producto (prefijo `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). Para cada componente utilizado, compruebe si la versión instalada es inferior a la versión de corrección indicada en cada caso.

7.  **Vigilar los canales:** los avisos de GitHub, la sala de prensa y los correos electrónicos de Kiteworks a clientes, por si el fabricante publica finalmente un aviso con número CVE para la vulnerabilidad de la ventana de apagado. El [rastreador de CVE](/cve) de esta página también incluye nuevas CVE de Kiteworks Email Protection Gateway y Totemomail; allí se puede suscribir una alerta por correo electrónico.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Ayuda con la actualización</p>
<p>Si necesita ayuda para actualizar un gateway de Kiteworks o Totemomail, por ejemplo para redirigir el flujo de correo durante la ventana de mantenimiento o para analizar los registros, utilice el <a href="https://adeptio.ch/">formulario de contacto de adeptio.ch</a>.</p>
</div>

## Fuentes

1.  [heise online: Ataque de día cero inminente: KiteWorks insta a los clientes a apagar los servidores](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): primera información con extractos del correo electrónico a los clientes y la ventana temporal; actualización del 25/09 a las 17:41 con la respuesta de la BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): versión en inglés con el texto original del CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): antigua página de actualizaciones del fabricante; a fecha de 07/10/2026, sin entrada sobre la advertencia ni los avisos del 30/09/2026.

4.  [Kiteworks: Security Advisories en GitHub](https://github.com/kiteworks/security-advisories/security): lista de avisos del fabricante; hasta el 28/09/2026, la última entrada era del 27/05/2026; el 30/09/2026 hubo 125 avisos nuevos para Core, EPG, SDF y MFT. Las cifras de este artículo se contabilizaron mediante la API de GitHub.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): comunicados oficiales, desde el 25/09/2026 con el comunicado de prensa sobre el apagado.

6.  [Foro de heise: comentarios sobre la noticia](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): debate de los lectores con objeciones a la ventana temporal fija y al apagado de sistemas internos, así como a la teoría del señuelo.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): aviso sobre la explotación de Accellion FTA en 2020/2021 con posterior extorsión; Accellion es el nombre anterior de Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): información del fabricante sobre la colaboración con Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): declaración del CISO, hora de envío de la advertencia, reacciones del FBI y CISA (actualización), efectos en un cliente y número de sistemas accesibles desde Internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (comunicado de prensa)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): comunicado oficial del 25/09/2026 con información sobre instancias alojadas, versión 9.5.1 y filiales no afectadas; complementado con la nota del 27/09/2026 de que la recomendación de apagado fue retirada.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): ventana temporal por regiones y contextualización de ataques anteriores contra productos de intercambio de archivos.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): valoración de watchTowr sobre la inusual recomendación de apagado.

13.  [BornCity: Kiteworks: más de 1.000 organizaciones deben apagar servidores](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): número de organizaciones notificadas y sectores de la región de habla alemana.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): contextualización de los ataques de Clop contra Accellion en 2020/2021 y cita de watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): informe del 28/09/2026 sobre la retirada de la recomendación y el funcionamiento de las instancias alojadas.

16.  [Kiteworks: Kiteworks Restores Systems After Credible Threat (comunicado de prensa)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): comunicado del 28/09/2026 sobre la vulnerabilidad crítica encontrada durante el apagado, la corrección y la capa de protección adicional.

17.  [The Hacker News: Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): resumen del segundo comunicado de prensa con citas del CISO.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6: CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): información del fabricante sobre la ejecución de código en Email Protection Gateway anterior a 9.4.1, CVSS 10.0, notificación mediante YesWeHack.

19.  [BleepingComputer: Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): informe del 01/10/2026 sobre CVE-2026-54154 y número de instancias contabilizadas por Shadowserver.

20.  [MS-ISAC Advisory 2026-107: A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): aviso del Center for Internet Security del 01/10/2026 con recomendaciones.

21.  [SecurityOnline: Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): contextualización de las vulnerabilidades de toma de control de cuentas en Core, incluida CVE-2026-102115; el recuento difiere de la lista de avisos.

22.  [CISA: Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): a fecha de 07/10/2026, solo las cuatro entradas de Accellion FTA de 2021, ninguna entrada de Kiteworks de 2026.
