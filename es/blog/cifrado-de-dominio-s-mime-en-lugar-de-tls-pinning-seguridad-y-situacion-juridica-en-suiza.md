---
title: "Cifrado de dominio S/MIME en lugar de TLS pinning: seguridad y situación jurídica en Suiza"
navTitle: "Dominio en lugar de pinning"
description: "Por qué el cifrado de dominio S/MIME protege mejor el tráfico de correo entre dos organizaciones que el TLS pinning, qué puede hacer un atacante con un mensaje S/MIME interceptado, cómo influyen la DSG, la DSV y los secretos profesionales en la elección y por qué MTA-STS y DANE asumen la protección del transporte."
date: "2026-11-16"
kategorie: "Cifrado de correo electrónico"
timeToRead: "10 min de lectura"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "cifrado-de-dominio-s-mime-en-lugar-de-tls-pinning-seguridad-y-situacion-juridica-en-suiza"
translationId: "article-92db7a4587660dec"
translationOf: domain-verschluesselung-statt-tls-pinning
url: https://rafaelpfister.ch/es/blog/cifrado-de-dominio-s-mime-en-lugar-de-tls-pinning-seguridad-y-situacion-juridica-en-suiza
translationSourceHash: 8eeb8d5552f8680381710c1a052edd3e62b69b34e1af9fd032c4bd481e8d65a1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:59:17.561Z
translationReview: automatic
---

# Cifrado de dominio S/MIME en lugar de TLS pinning: seguridad y situación jurídica en Suiza

Conclusión breve: el TLS pinning protege la conexión entre dos servidores de correo; el cifrado de dominio S/MIME protege el contenido del mensaje durante todo el trayecto entre dos organizaciones. Si hay que elegir entre ambos, el cifrado de dominio es la mejor opción desde el punto de vista técnico y jurídico. MTA-STS y DANE asumen la protección del transporte que hasta ahora proporcionaba el pinning.

En un proyecto actual, una conexión de correo existente entre dos organizaciones, protegida hasta ahora mediante TLS forzado y pinning de certificados, debe migrarse al cifrado de dominio S/MIME a través de un servicio de correo seguro. El servicio no puede aplicar ambos simultáneamente a la misma conexión: o se aplica el pinning o el cifrado de dominio. La pregunta de si esto supone una pérdida de seguridad está justificada. La respuesta es clara en cuanto se analiza qué protege cada uno de los dos métodos.

**Enfoque en Suiza:** La clasificación jurídica se basa en el derecho suizo, en particular la DSG, la DSV y el StGB. Cuando las normas de la UE difieren, se incluye una indicación identificada como tal. Este artículo no sustituye el asesoramiento jurídico en un caso concreto.

## Qué protege el TLS pinning

Sin medidas adicionales, TLS entre servidores de correo es oportunista. De ello resultan tres posibilidades de ataque:

- **Downgrade:** Un atacante en la ruta de transmisión elimina la oferta STARTTLS de la sesión SMTP. El servidor emisor entrega entonces el mensaje en texto claro.
- **Cualquier certificado:** Los servidores de correo emisores no verifican por defecto el certificado de la contraparte. Se acepta un servidor con un certificado autofirmado.
- **MX falsificado:** Quien manipula la respuesta DNS redirige el correo a un servidor propio que ofrece TLS correctamente configurado.

El pinning cierra las tres brechas para un socio determinado. La pasarela emisora fuerza TLS y acepta únicamente un certificado o una clave definidos. Sin embargo, la protección termina con la conexión. En el servidor receptor, en las colas, los filtros antispam y antivirus, los sistemas de journaling y archivo, el mensaje está disponible en texto claro. Si atraviesa varios servidores, el pinning solo se aplica al tramo para el que se ha configurado.

A esto se suma la operación. El pin es una regla manual del remitente. Si el destinatario renueva su certificado o modifica su enrutamiento, por ejemplo mediante un nuevo servicio de filtrado o un MX de respaldo, la entrega se detiene hasta que se actualiza la regla. Por ello, en caso de incidencia, la regla se suele relajar temporalmente y no siempre se vuelve a activar.

## Qué protege el cifrado de dominio S/MIME

Con el cifrado de dominio, la pasarela del remitente cifra el mensaje con la clave pública del certificado de dominio del dominio destinatario. Solo la pasarela del destinatario vuelve a descifrarlo. Todos los sistemas intermedios ven únicamente el contenido cifrado, independientemente de cuántos servidores atraviese el mensaje y de dónde se almacene temporalmente. Además, la pasarela del remitente firma el mensaje con su propio certificado de dominio.

| Protección | TLS pinning | Cifrado de dominio S/MIME |
|---|---|---|
| Texto del correo y adjuntos | solo durante la transmisión en el tramo fijado | durante todo el trayecto, también en servidores intermedios y colas |
| Asunto, remitente, destinatario, fecha | cifrados durante la transmisión | legibles, salvo que esté activa la protección de cabeceras |
| Sobre, tamaño, momento | cifrados durante la transmisión | legibles |
| Autenticidad del servidor receptor | sí, se rechaza un servidor incorrecto | no, el correo se entrega al servidor que indique el enrutamiento MX |
| Forward Secrecy | sí, con intercambio de claves ECDHE | no |
| Integridad y procedencia tras la entrega | no | sí, mediante firma |
| Dependencia del enrutamiento y la infraestructura | alta, el pin se aplica a un servidor o clave | baja, la protección depende del certificado de dominio |

Por tanto, el pinning tiene tres propiedades que no ofrece el cifrado de dominio: protege los metadatos, evita que un servidor incorrecto acepte el correo y proporciona Forward Secrecy. Para el contenido, el cifrado de dominio es claramente superior.

## Qué puede hacer un atacante con un mensaje S/MIME interceptado

Supongamos que un atacante introduce su propio servidor de correo en la ruta de transmisión mediante registros MX falsificados, secuestro BGP o un relay comprometido.

No puede hacer lo siguiente:

- **Leer el contenido:** El texto y los adjuntos están cifrados con la clave pública del dominio destinatario. Sin la clave privada de la pasarela receptora, permanecen ilegibles.
- **Modificar el contenido sin ser detectado:** Si el mensaje está firmado y la pasarela receptora exige una firma válida, cualquier modificación se detecta.
- **Generar mensajes en nombre del remitente:** Para ello necesitaría la clave privada de firma del dominio remitente.

Sí puede hacer lo siguiente:

- **Analizar metadatos:** El remitente, el destinatario, el momento, el tamaño y, por regla general, el asunto son legibles. Un asunto como «Informe de alta Max Muster» ya constituye una comunicación de datos personales.
- **Guardar mensajes y descifrarlos más adelante:** S/MIME no ofrece Forward Secrecy. Si la clave privada del dominio se ve comprometida posteriormente, todos los mensajes almacenados serán legibles. Lo mismo se aplica a largo plazo si RSA se vuelve vulnerable a los ordenadores cuánticos.
- **Suprimir o retrasar mensajes:** El remitente no necesariamente lo advertirá. La disponibilidad se ve afectada, lo cual es relevante en caso de plazos.
- **Modificar cabeceras no protegidas:** Puede, por ejemplo, reescribir el asunto o `Reply-To` para desviar respuestas. El contenido cifrado permanece inalterado.
- **Manipular el texto cifrado de forma dirigida:** En mensajes que solo están cifrados y no firmados y que utilizan AES-CBC, el texto cifrado puede modificarse de modo que partes del texto claro se filtren tras el descifrado. Este es el principio de los ataques EFAIL de 2018. Una verificación obligatoria de la firma o el cifrado autenticado con AES-GCM conforme a S/MIME 4.0 lo impiden.

Un texto claro interceptado con cifrado exclusivo de transporte constituye una filtración completa de datos. Un mensaje S/MIME interceptado afecta esencialmente a los metadatos y a la disponibilidad.

## Clasificación jurídica

Ninguna norma suiza prescribe S/MIME, TLS pinning u otro procedimiento específico. Los requisitos son tecnológicamente neutrales y establecen objetivos de protección. Sin embargo, la elección del procedimiento debe evaluarse frente a estos objetivos de protección y documentarse.

| Normativa | Contenido | Importancia para elegir el procedimiento |
|---|---|---|
| DSG art. 8, DSV art. 1 a 3 | seguridad de los datos adecuada, objetivos de protección, control del transporte | protección en la comunicación, medida según el riesgo y el estado de la técnica |
| DSG art. 61 letra c | multa por incumplimiento de los requisitos mínimos de seguridad de los datos | responsabilidad penal personal de la persona responsable |
| DSG art. 9 | tratamiento por encargados del tratamiento | debe tenerse en cuenta todo sistema con acceso al texto claro |
| DSG art. 16 y 17 | comunicación al extranjero | tratamiento de texto claro en sistemas extranjeros |
| DSG art. 24 | notificación de vulneraciones de la seguridad de los datos | el riesgo para los afectados depende de si los datos filtrados son legibles |
| StGB art. 321, DSG art. 62, ATSG art. 33 | secreto profesional, deber de confidencialidad profesional, deber de secreto en los seguros sociales | el contenido debe protegerse frente a todo conocimiento no autorizado |
| ZPO art. 177 | archivos electrónicos como documentos | la firma refuerza la prueba de procedencia e integridad |

### Seguridad de los datos conforme a la DSG y la DSV

El art. 8 DSG exige una seguridad de los datos adecuada al riesgo. La DSV concreta los objetivos de protección de confidencialidad, disponibilidad, integridad y trazabilidad y menciona en el art. 3, entre otras medidas, el control del transporte: los datos personales no pueden leerse, copiarse, modificarse ni eliminarse sin autorización al comunicarlos. Para los datos personales especialmente sensibles según el art. 5 letra c DSG, como los datos de salud o los datos sobre medidas de ayuda social, se aplican requisitos correspondientemente más elevados.

El TLS pinning cumple el control del transporte en un tramo de la transmisión. Sin embargo, el mensaje se almacena en varios sistemas antes de llegar a su destino, y allí puede leerse en texto claro. Con el cifrado de dominio, el contenido está protegido durante todo el trayecto entre las dos organizaciones. La firma cubre además el objetivo de protección de integridad, que TLS ya no puede demostrar tras la entrega.

### Responsabilidad penal personal

Según el art. 61 letra c DSG, quien incumpla deliberadamente los requisitos mínimos de seguridad de los datos conforme al art. 8 párr. 3 DSG será castigado, previa denuncia, con una multa de hasta CHF 250'000. La multa se dirige a la persona física responsable, no a la empresa. Un cifrado del contenido conforme al estado actual de la técnica, documentado en una evaluación de riesgos, es una prueba más sólida para estas personas que una regla de transporte para servidores individuales.

**Indicación sobre la UE:** El RGPD también exige en su art. 32 medidas técnicas y organizativas adecuadas y menciona expresamente el cifrado. Sin embargo, las multas conforme al art. 83 RGPD se dirigen contra la empresa y, en caso de infracciones del art. 32, ascienden hasta 10 millones de euros o el 2 % del volumen de negocio anual mundial.

### Encargados del tratamiento y comunicación al extranjero

Todo sistema en la ruta que procese el texto claro debe considerarse jurídicamente: servicios de filtrado, relays, pasarelas en la nube, servicios de archivado. Si terceros operan estos sistemas, normalmente se trata de un tratamiento por encargo conforme al art. 9 DSG, con los contratos y obligaciones de control correspondientes. Si los sistemas se encuentran en el extranjero, se añaden las normas sobre comunicación al extranjero de los art. 16 y 17 DSG.

Con el cifrado de dominio, estos sistemas solo procesan texto cifrado. El círculo de entidades que pueden acceder a los contenidos se limita a las dos pasarelas. Sin embargo, los metadatos siguen siendo datos personales y las obligaciones mencionadas continúan aplicándose a ellos.

### Obligación de notificar vulneraciones de la seguridad de los datos

Una vulneración de la seguridad de los datos que previsiblemente implique un alto riesgo para las personas afectadas debe notificarse al EDÖB conforme al art. 24 DSG. Si los mensajes se filtran a través de un servidor intermedio comprometido, la evaluación depende de si los contenidos son legibles. Si están fuertemente cifrados y las claves no se ven afectadas, el riesgo para las personas afectadas suele ser bajo, siempre que no haya datos sensibles en el asunto. Con cifrado exclusivo de transporte, los contenidos están disponibles para el atacante en texto claro.

### Secretos profesionales y deberes de confidencialidad

Cuando los datos están sujetos a un secreto profesional conforme al art. 321 StGB, por ejemplo en el caso de médicos, abogados o sus auxiliares, o a un deber legal de secreto como el art. 33 ATSG en el ámbito de los seguros sociales, el contenido debe protegerse frente a todo conocimiento no autorizado. Esto incluye también a los operadores de sistemas intermedios. El art. 62 DSG castiga además la vulneración del deber de confidencialidad profesional de personas que no están sujetas al art. 321 StGB. Este requisito solo lo cumple el cifrado del contenido.

### Valor probatorio de la firma

Según el art. 177 ZPO, los archivos electrónicos son documentos cuyo valor probatorio el tribunal aprecia libremente. Una firma S/MIME válida del dominio remitente prueba que el mensaje procede de ese dominio y que no ha sido modificado después de la firma. Esto refuerza la prueba en relación con plazos, compromisos o instrucciones. No sustituye una firma manuscrita; los requisitos formales del OR solo se cumplen mediante una firma electrónica cualificada conforme a ZertES. Con TLS pinning no se puede demostrar ni la procedencia ni la integridad después de la entrega.

## MTA-STS y DANE como sustitutos del TLS pinning

Al eliminar el pinning, faltan las propiedades que solo proporciona la protección del transporte: TLS forzado, un servidor receptor autenticado y, con ello, la protección de los metadatos frente a la lectura y el redireccionamiento. Precisamente estas propiedades las proporcionan MTA-STS y DANE, como estándar que el destinatario publica una vez para todos los remitentes.

**MTA-STS (RFC 8461):** El destinatario publica en `https://mta-sts.<domain>/.well-known/mta-sts.txt` una política con los nombres de host MX permitidos y el modo `enforce`. Un remitente compatible con MTA-STS obtiene la política, la almacena y entrega en adelante únicamente mediante TLS con un certificado válido a uno de los MX indicados. Se rechazan un MX falsificado o un servidor sin un certificado válido para el nombre MX.

**DANE (RFC 7672):** El destinatario publica en su zona firmada con DNSSEC un registro TLSA con el hash de su certificado o clave. El remitente verifica el certificado del MX frente a dicho registro. Técnicamente, DANE es un pinning que el propio destinatario publica en DNS y protege con DNSSEC.

| | TLS pinning | MTA-STS | DANE |
|---|---|---|---|
| Gestionado por | remitente, manualmente por socio | destinatario, una vez para todos los remitentes | destinatario, una vez para todos los remitentes |
| Se aplica a | un socio configurado | todos los remitentes compatibles con MTA-STS | todos los remitentes compatibles con DANE |
| Renovación de certificado | el pin debe actualizarse en el remitente | sin ajustes, mientras el nuevo certificado sea válido | sin ajustes con la misma clave; de lo contrario, publicar previamente un nuevo registro TLSA |
| Base de confianza | exactamente este certificado o esta clave | autoridades de certificación públicas | DNSSEC |
| Punto débil | esfuerzo operativo, relajación ante incidencias | primera consulta y caché caducada (Trust on First Use), emisión incorrecta por una CA | requiere DNSSEC |
| Informes de errores | ninguno | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

Para la documentación conforme al art. 8 DSG, MTA-STS y DANE ofrecen otra ventaja: mediante TLS-RPT, el destinatario recibe informes diarios sobre si las entregas se realizaron cifradas o fallaron. El pinning no proporciona esta retroalimentación. El artículo [Registros DNS para correo electrónico](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) describe la configuración de ambos procedimientos en la sección sobre MTA-STS, TLS-RPT y DANE.

## Quién debe comprobar MTA-STS y DANE

Siempre debe comprobarlos el sistema que entrega el mensaje directamente al MX del destinatario. El operador del MX debe publicar los registros.

Con un servicio de correo seguro, la ruta saliente suele ser la siguiente:

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Hasta ahora, el pinning protegía el último tramo. En este punto, el servicio debe comprobar MTA-STS y DANE del socio; el propio servidor de correo no influye en este tramo. Usted protege el tramo entre su propio servidor de correo y el servicio mediante TLS forzado y verificación del certificado en el conector.

En el lado receptor, depende del MX. Si el MX apunta al servicio, este debe proporcionar certificados adecuados, y usted publica registros MTA-STS y TLSA que se ajusten a sus servidores. En el caso de DANE, esto requiere que el servicio anuncie los cambios de clave con suficiente antelación. Si el MX apunta a la pasarela propia, por ejemplo Exchange Online, configure allí MTA-STS y DANE. Si ambas organizaciones utilizan el mismo servicio, este puede entregar el mensaje dentro de su propia plataforma; entonces no existe un tramo SMTP público entre las dos organizaciones.

Si el servicio no admite ni MTA-STS ni DANE al enviar, el contenido sigue protegido por S/MIME. En ese caso, quedan expuestos la protección de los metadatos y la autenticación del servidor receptor en el último tramo. Este riesgo residual debe incluirse en la evaluación de riesgos.

## Riesgos residuales y medidas

| Riesgo residual | Medida |
|---|---|
| Los datos personales en el asunto son legibles | Regla: no incluir datos personales en el asunto, acordado por escrito con el socio |
| Falta de Forward Secrecy | almacenar la clave privada del dominio sin posibilidad de exportación, idealmente en un HSM; registrar los accesos |
| Manipulación de mensajes no firmados | obligación de firma en ambos lados, AES-GCM si ambas pasarelas lo admiten |
| Entrega en texto claro en caso de errores | retener mensajes cuando falle el cifrado en lugar de entregarlos en texto claro |
| Metadatos en la ruta de transporte | MTA-STS en modo `enforce` y, donde exista DNSSEC, DANE; evaluar TLS-RPT |
| Certificados caducados | notificación mutua al menos 30 días antes de la caducidad, cotejar huellas digitales a través de un segundo canal |

La decisión a favor del cifrado de dominio y los riesgos restantes deben incluirse en el concepto de seguridad de los datos o en la documentación de las actividades de tratamiento.

## Preguntas para el proveedor de la pasarela

1.  ¿El servicio comprueba al enviar las políticas MTA-STS en modo `enforce` y los registros DANE del dominio destinatario?

2.  ¿Desde qué servidores se realiza la entrega y pueden consultarse informes TLS-RPT o registros de entrega relativos al cifrado de transporte?

3.  ¿El servicio como MX admite MTA-STS y DANE para mensajes entrantes y cómo se anuncian los cambios de clave?

4.  ¿Puede operarse el conector desde el propio servidor de correo al servicio con TLS forzado y verificación de certificado?

5.  ¿La entrega entre dos clientes del servicio se realiza mediante SMTP público o dentro de la plataforma?

6.  ¿Qué algoritmos se utilizan para el cifrado y la firma, y se admite AES-GCM?

## Fuentes

1.  [Ley federal de protección de datos (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sobre seguridad de los datos, art. 9 sobre tratamiento por encargo, art. 16 y 17 sobre comunicación al extranjero, art. 24 sobre obligación de notificación, art. 61 y 62 sobre disposiciones penales.

2.  [Ordenanza de protección de datos (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 a 3 con principios, objetivos de protección y medidas técnicas y organizativas, incluido el control del transporte.

3.  [Código Penal Suizo (StGB)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): art. 321 sobre vulneración del secreto profesional.

4.  [Ley federal sobre la parte general del derecho de los seguros sociales (ATSG)](https://www.fedlex.admin.ch/eli/cc/2002/510/de): art. 33 sobre el deber de secreto de los órganos de ejecución.

5.  [Código Procesal Civil (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 177 sobre el concepto de documento, incluidos los archivos electrónicos.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): especificación de la política, las reglas de consulta y el modo `enforce`.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672): DANE para SMTP, registros TLSA y reglas de verificación para servidores emisores.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): formato y entrega de los informes TLS-RPT.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): especificación actual de S/MIME con AuthEnvelopedData y AES-GCM.

10.  [EFAIL](https://efail.de/): descripción de los ataques publicados en 2018 contra mensajes S/MIME y OpenPGP no firmados en modo CBC.

11.  [Reglamento General de Protección de Datos (RGPD)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679): art. 32 sobre seguridad del tratamiento, art. 83 sobre multas (indicación sobre la UE).
