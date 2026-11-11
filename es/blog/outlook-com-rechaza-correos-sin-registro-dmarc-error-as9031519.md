---
title: "Outlook.com rechaza correos sin registro DMARC: error AS9031519"
navTitle: "Outlook.com: AS9031519"
description: "Desde el 5 de noviembre de 2026, Outlook.com, Hotmail y Live rechazan mensajes de dominios sin registro DMARC con 550 5.7.1 y el añadido AS9031519, incluso con un volumen de envío bajo. Qué documenta Microsoft, qué han medido los operadores, cómo comprobar el propio dominio y qué muestra el índice de seguridad del correo de los municipios suizos."
date: "2026-11-10"
kategorie: "SMTP y flujo de correo"
timeToRead: "8 min de lectura"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "dns"
  - "smtp"
  - "troubleshooting"
related:
  - dns-records-e-mail-stolpersteine
  - microsoft-365-compauth-reason-codes
slug: "outlook-com-rechaza-correos-sin-registro-dmarc-error-as9031519"
translationId: "article-84656f0bf3da2006"
translationOf: outlook-com-dmarc-ablehnung-as9031519
url: https://rafaelpfister.ch/es/blog/outlook-com-rechaza-correos-sin-registro-dmarc-error-as9031519
translationSourceHash: afec1593a28f2ec6ab14b2af5b064bae9b03ef23d0d091f8b659713eff887222
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T03:20:42.235Z
translationReview: automatic
---

# Outlook.com rechaza correos sin registro DMARC: error AS9031519

Desde el 5 de noviembre de 2026, los mensajes a buzones de Outlook.com, Hotmail, Live y MSN se devuelven con un nuevo error si el dominio del remitente no ha publicado un registro DMARC:

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

Datos actualizados al 10 de noviembre de 2026.

## Lo que documenta Microsoft

Las directrices para postmasters de Outlook.com siguen incluyendo únicamente la regla del 5 de mayo de 2025: los dominios que envían más de 5000 mensajes al día a cuentas de Outlook.com deben superar SPF y DKIM y publicar un registro DMARC con al menos `p=none`, alineado mediante SPF o DKIM. Los mensajes que no cumplan estos requisitos llegan primero a la carpeta de correo no deseado y posteriormente pueden ser rechazados con el siguiente error:

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

Para todos los demás remitentes, la misma página solo formula una recomendación. El añadido `AS9031519` y el texto de error `Message lacks SPF/DKIM/DMARC Auth Records` no aparecen ni en las directrices para postmasters ni en el artículo de soporte sobre 5.7.515.

Microsoft solo se ha pronunciado sobre la situación actual en los foros de Microsoft Q&A. Un moderador identificado como «Microsoft External Staff» escribió el 9 de noviembre que existe una incidencia en curso en el servicio Exchange en la entrega a Outlook.com, Hotmail.com y Live.com. Está clasificada como degradación del servicio y Microsoft está investigando la causa. La publicación no menciona número de incidente, causa ni fecha de resolución. No hay ninguna entrada de blog, actualización en las directrices para postmasters ni aviso público sobre el estado del servicio.

## Lo que han medido los operadores

Los datos más sólidos proceden de proveedores de correo que han analizado sus registros y los han publicado en la lista de correo mailop y en sus propios blogs (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Las observaciones coinciden:

- **Desencadenante:** El dominio del remitente no tiene registro DMARC. En los casos documentados, SPF y DKIM eran válidos y estaban alineados. Aunque el texto del error menciona los tres procedimientos, el rechazo se debe a la falta del registro DMARC.
- **Volumen:** También están afectados dominios con pocos mensajes por semana, muy por debajo del umbral de 5000 al día.
- **Destinatarios:** Solo los buzones de consumidores (outlook.com, hotmail.com, live.com, msn.com y variantes nacionales como hotmail.de u outlook.be). Según los informes disponibles, los tenants de Exchange Online como destinatarios no rechazan con este error.
- **Irregularidad:** Solo se rechaza una parte de los mensajes. Reenviar el mismo mensaje puede funcionar.
- **Solución:** En los casos documentados, bastó un registro con `v=DMARC1; p=none`. Tras su publicación, Outlook.com volvió a aceptar mensajes entre unos pocos minutos y alrededor de hora y media después.

Dos análisis delimitan aún más el desencadenante: en one.com solo se rechazaron dominios cuya consulta de `_dmarc.<domain>` terminaba con `NXDOMAIN`. Los dominios en los que el nombre existe pero no hay ningún registro TXT (`NOERROR` sin respuesta) no se vieron afectados allí. MXroute encontró en un registro 367 dominios de remitentes rechazados, 286 de ellos con `NXDOMAIN` en `_dmarc`. Esta delimitación se basa en datos de operadores individuales y no está aclarada de forma concluyente.

Desde el mediodía del 9 de noviembre, algunos afectados informan de que los rechazos han cesado. Microsoft no lo ha confirmado.

## Caso especial: Microsoft 365 como remitente

Los tenants de Microsoft 365 se ven afectados de forma desproporcionada como remitentes. Microsoft 365 no crea un registro DMARC para un dominio propio y, mientras DKIM no esté activado para el dominio, Exchange Online firma con el dominio `onmicrosoft.com` del tenant. El informe de no entrega también encapsula el rechazo en un segundo código de estado que hace referencia a SPF:

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

Por lo general, el registro SPF es correcto en estos casos. Lo determinante es el error interno tras la flecha.

## Comprobar el propio dominio

Se consulta el registro TXT en `_dmarc.<domain>`:

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `+short` | `dig` muestra únicamente el contenido del registro |
| `TXT` o `-Type TXT` | Tipo de registro de la consulta |
| `_dmarc.example.com` o `-Name` | Nombre consultado: la etiqueta `_dmarc` antes del dominio del remitente |

</details>

Una salida vacía con `dig +short` o el error `DNS name does not exist` con `Resolve-DnsName` significa que no existe ningún registro DMARC. Para los subdominios se aplica lo siguiente: si `news.example.com` envía y no tiene su propio registro, según RFC 7489 cuenta el registro del dominio organizativo `example.com`.

La [comprobación DNS de correo](https://rafaelpfister.ch/tools/mail-check) de esta página verifica DMARC junto con SPF, DKIM y la alineación en una sola pasada.

## Medida inmediata y el camino posterior

Si falta el registro, esta entrada restablece la entrega a Outlook.com:

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` no cambia el tratamiento de sus correos por parte de otros destinatarios y, por tanto, puede publicarse sin trabajos previos. La etiqueta `rua` no es necesaria para la aceptación en Outlook.com. Sin `rua`, sin embargo, no recibirá informes agregados, y sin estos informes falta la base para cualquier paso posterior. La dirección debe llegar a un buzón o servicio de análisis que procese los informes XML.

`p=none` no protege el dominio frente al uso indebido. El registro solo cumple la condición de aceptación. Para una protección efectiva, siga estos pasos:

1. Inventariar todos los sistemas que envían con el dominio (servidor de correo, servicio de newsletter, ERP, tienda web, escáner).
2. Mantener SPF completo y válido para estos sistemas, con un máximo de diez consultas DNS.
3. Activar DKIM con el propio dominio, en Microsoft 365 para cada dominio en el portal Defender.
4. Analizar los informes agregados durante algunas semanas y corregir las fuentes no alineadas.
5. Elevar la política a `p=quarantine` y, posteriormente, a `p=reject`.

La [guía para administradores de DNS](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) describe los errores habituales al publicar estos registros (registros SPF duplicados, claves DKIM demasiado largas, nombre de zona añadido automáticamente).

## El impacto en Suiza: 152 municipios en cinco días

El [índice de seguridad del correo de Suiza](https://rafaelpfister.ch/open-data/mail-security-index) comprueba cada noche los registros DNS de 3364 organizaciones suizas, entre ellas 2078 municipios. El [registro de cambios](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) muestra desde el 6 de noviembre una oleada de registros DMARC recién publicados. La fecha corresponde en cada caso a la comparación nocturna.

| Comparación | Organizaciones nuevas con registro DMARC | De ellas, municipios |
|---|---|---|
| Del 19 de octubre al 5 de noviembre | como máximo 5 al día | como máximo 4 al día |
| 6 de noviembre | 4 | 4 |
| 7 de noviembre | 22 | 21 |
| 8 de noviembre | 1 | 1 |
| 9 de noviembre | 2 | 2 |
| 10 de noviembre | 140 | 124 |

En cinco días, 169 organizaciones publicaron por primera vez un registro DMARC, 152 de ellas municipios. El número de organizaciones sin DMARC en el índice descendió de 817 a 650. El momento coincide con los rechazos en Outlook.com. No he preguntado a los municipios los motivos.

Los registros presentan las siguientes características:

- **Política:** 139 de los 152 municipios han establecido `p=none`, cinco `p=quarantine` y tres `p=reject`. En cinco casos, el registro solo consta de `v=DMARC1` sin la etiqueta `p=` y, por tanto, no es válido.
- **Informes:** 114 registros contienen exclusivamente `v=DMARC1; p=none`. En total, 124 de los 152 municipios no han indicado una dirección `rua` y, por tanto, no reciben informes.
- **Cambios centralizados:** Las modificaciones se producen de forma agrupada. El 10 de noviembre se sumaron 32 municipios ginebrinos, cuyos servidores de correo funcionan todos bajo `acg-geneve.ch`, además de 26 municipios de Zúrich y 19 de Friburgo. El 7 de noviembre fueron ocho municipios de Nidwalden y siete del Tesino. La agrupación por servidor de correo y cantón indica que los proveedores de servicios informáticos configuraron los registros para todos los municipios atendidos en una sola operación.

Estos registros son suficientes para la entrega a Outlook.com. Sin embargo, todavía no protegen los dominios municipales frente a remitentes falsificados: sin `rua` faltan los datos para elevar posteriormente la política. 537 de los 2078 municipios del índice siguen sin tener registro DMARC.

## Contexto

No está claro si Microsoft retirará el rechazo o lo confirmará como norma. Para la planificación, esto tiene poca importancia: Google y Yahoo exigen DMARC desde febrero de 2024 a los remitentes masivos, Microsoft desde mayo de 2025, y las directrices para postmasters de Outlook.com lo recomiendan a todos los remitentes. En los últimos días, un dominio sin registro DMARC ya no podía entregarse de forma fiable a uno de los mayores proveedores de correo. El registro con `p=none` y `rua` se publica en pocos minutos y no tiene efectos secundarios en la entrega.

## Fuentes

1.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies): Requisitos oficiales para remitentes; menciona el umbral de 5000 mensajes al día, el error 5.7.515 y la recomendación para todos los remitentes.

2.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com): Artículo de soporte sobre el error documentado para remitentes de gran volumen.

3.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error): Hilo del foro con la respuesta del moderador del 9 de noviembre de 2026 sobre la incidencia en curso; publicación en un foro, no aviso oficial sobre el estado del servicio.

4.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc): Análisis de los registros salientes de un proveedor de correo desde el 5 de noviembre de 2026.

5.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/): Informe de un proveedor de hosting con más de 500 rechazos el primer día y la solución mediante `p=none`.

6.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html): Hilo de la lista de correo de operadores con las mediciones de MXroute, Fastmail y one.com, incluida la delimitación a `NXDOMAIN`.

7.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/): Prueba reproducida con SPF y DKIM válidos y alineados, sin registro DMARC.

8.  [heise online: Outlook.com: Die Mail ist echt und sie kommt trotzdem nicht an](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html): Resumen en alemán de los informes del 9 de noviembre de 2026.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Especificación, entre otros aspectos, sobre la determinación del registro mediante el dominio organizativo.

10.  [Índice de seguridad del correo de Suiza: registro de cambios](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen): Cambios diarios en los registros DNS de 3364 organizaciones suizas; base de las cifras sobre los municipios.
