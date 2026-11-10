---
title: "Proofpoint Protection Server 8.22.4: el parche 5807 deja SMTP Client sin certificado"
navTitle: "PPS 8.22.4 certificado"
description: "Los parches 5806 y 5807 introducen en Proofpoint Protection Server 8.22.4 una asignación propia «SMTP Client» en System > Certificates > Services. Si está en «-none-», Sendmail realiza las entregas sin certificado TLS de cliente. Los sistemas remotos que verifican certificados lo rechazan, por ejemplo Exchange Online con 5.7.51 o un smarthost con «Relay access denied». Así puede comprobarlo y solucionarlo."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min de lectura"
themen:
  - proofpoint
  - smtp-mailflow
produkte:
  - "proofpoint-pps"
  - "exchange-online"
protokolle:
  - "tls"
  - "smtp"
  - "troubleshooting"
slug: "proofpoint-protection-server-8-22-4-el-parche-puede-restablecer-la-configuracion-del"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
translationSourceHash: fcb36b2ae8af7420156aca54a99d61f4702aeb47781e1081e27edf7419e77db8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:21:17.457Z
translationReview: automatic
aktualisiert: 2026-11-06
url: https://rafaelpfister.ch/es/blog/proofpoint-protection-server-8-22-4-el-parche-puede-restablecer-la-configuracion-del
---

Hoy he investigado una interrupción en el flujo de correo saliente de un Proofpoint Protection Server (PPS) 8.22.4. Conclusión breve: tras un parche instalado automáticamente, la asignación **SMTP Client** en **System > Certificates > Services** estaba en `-none-`. A partir de entonces, Sendmail estableció conexiones salientes sin certificado TLS de cliente. Todos los sistemas remotos que identifican Proofpoint mediante este certificado rechazaron los correos. Los destinatarios habituales con TLS oportunista siguieron aceptándolos, por lo que el error solo se detectó cuando determinados socios dejaron de ser accesibles.

Si utiliza PPS con parches automáticos y tiene socios que exigen un certificado de cliente, compruebe esta configuración después de cada parche.

## Síntoma

Solo se ven afectadas las conexiones en las que el sistema remoto verifica el certificado de cliente de Proofpoint. Dos casos típicos:

- **Exchange Online con conector de socio**, limitado a un certificado mediante `RestrictDomainsToCertificate`. El rechazo es `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost que solo permite relay con certificado de cliente.** La respuesta es `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Todos los demás destinos siguen recibiendo sus correos. Por eso, la matriz de pruebas ofrece una imagen desigual: los correos externos se entregan, mientras que los correos a determinados socios o a través de un smarthost concreto fallan.

En el registro de correo del agente aparece uno de los dos mensajes siguientes para cada conexión saliente. Si el archivo falta por completo:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Si el archivo existe pero está vacío:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Inmediatamente después, Sendmail establece la conexión con TLS de todos modos (`verify=OK`), pero sin certificado de cliente. Para el sistema remoto, esto parece un remitente desconocido.

## Causa

PPS asigna un certificado a cada servicio por servidor. En **System > Certificates > Services** aparecen por servidor, entre otros, `SMTP Server`, `SMTP Client`, `Administration Server` y `End User Web Application / Encryption`. PPS genera el archivo `/etc/pki/tls/certs/sendmail-client.pem` a partir de la asignación de `SMTP Client`, que Sendmail utiliza como certificado de cliente. Si allí figura `-none-`, PPS deja el archivo vacío o ni siquiera lo escribe.

La asignación `SMTP Client` solo existe en 8.22.4 desde los parches 5806 y 5807, publicados por Proofpoint el 29.10.2026. Ambos se titulan «TLS client and server certificate authentication improvement» y, según la Knowledge Base, corrigen que PPS 8.22.4 no pueda asignar certificados separados para SMTP Server y SMTP Client. En la versión 8.23.0, la función ya se incluye de forma regular (Release Notes, sección «SMTP Client Certificate»).

| Parche | Contenido según la Knowledge Base |
|---|---|
| 5806 | Actualiza, entre otros, `SysCertificates.pm`, `Certificate.pm`, `TLS.pm`, `pp-sendmail.pl` y la plantilla `sendmail.mc`, reinicia la GUI de administración (`admind`) y actualiza el RPM `pps-pfpt` |
| 5807 | Requiere 5806, no modifica archivos ni reinicia procesos; solo aplica los cambios de configuración necesarios para certificados separados de servidor y cliente |

Por tanto, Proofpoint introdujo una nueva asignación y no restableció una existente. El parche no adopta ningún certificado: tras la instalación, la nueva línea queda en `-none-`, y Sendmail deja de utilizar un certificado de cliente a partir de ese momento. Las descripciones de los parches no lo indican.

En el entorno investigado, la asignación quedó en todos los servidores en `-none-`. En **System > Config History**, el parche 5807 aparecía como entrada `configuration changes for patch-…` con el creador `system`, y los archivos en `/etc/pki/tls/certs/` tenían la misma marca de tiempo. En cambio, 5807 no aparecía en **Software Patch History**. Esto encaja con que 5807 no instala software, sino que solo cambia la configuración. Por ello, **Config History** muestra el momento del cambio, mientras que el cambio de software propiamente dicho corresponde al parche 5806.

Una reversión en **Config History** al snapshot del parche no ayudó. PPS volvió a escribir el archivo después, pero con 0 bytes.

## Comprobar

En la GUI de administración del máster, en **System > Certificates > Services**, seleccione uno tras otro en la parte superior, en **Server**, cada agente que entregue correo al exterior y revise la línea `SMTP Client`.

En el propio agente, dos comandos muestran el estado:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-l` | Formato largo con permisos, propietario, tamaño y fecha |
| `--time-style=full-iso` | Marca de tiempo con fecha, hora y zona horaria para compararla con el momento del parche |
| `grep -E` | Expresiones regulares extendidas para poder combinar ambos términos de búsqueda con `\|` |
| `tail -5` | Mostrar solo las cinco últimas coincidencias |

</details>

Si falta `sendmail-client.pem` o el archivo tiene 0 bytes, mientras que `sendmail.pem` ocupa varios miles de bytes, no hay ningún certificado asignado al servicio `SMTP Client`.

## Solucionar

En **System > Certificates > Services**, seleccione un certificado válido para cada agente afectado en `SMTP Client` y guarde con **Save Changes**. Por regla general, es el mismo certificado público que aparece en `SMTP Server`.

La lista de selección puede contener varios certificados con el mismo nombre, incluidos certificados caducados. Solo se pueden distinguir por el número de serie entre paréntesis. Antes de guardar, compare el número de serie con la lista de **System > Certificates > Certificates**, donde aparecen las fechas de emisión y caducidad.

La lista de `SMTP Client` no muestra necesariamente todos los certificados importados. En el entorno investigado faltaban algunos certificados, aunque eran válidos y los ejemplares caducados ya se habían eliminado. Ni los artículos de los parches ni las Release Notes 8.23 describen según qué criterios PPS filtra la lista. Si falta un certificado, compruebe dos aspectos:

- **Clave privada:** Si el certificado se importó como PEM sin clave, PPS no puede asignarlo a ningún servicio. Los archivos PKCS12 siempre contienen la clave.
- **Uso previsto:** ¿Contiene el certificado en `Extended Key Usage` el propósito `TLS Web Client Authentication`? Muchos certificados públicos emitidos desde el otoño de 2025 ya no lo contienen (véase más abajo). Para el servicio `SMTP Client`, un certificado sin este propósito tampoco es adecuado.

Cuando Proofpoint confirme los criterios, los añadiré aquí.

Después de guardar, PPS distribuye la configuración a los agentes. Comprobación en el agente:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `ls -l` | Muestra el tamaño y los permisos del archivo; tras la corrección ya no estará vacío |
| `grep -E 'STARTTLS=client\|stat='` | Filtra los mensajes TLS de conexiones salientes y el resultado de la entrega |
| `tail -20` | Mostrar solo las últimas veinte líneas |

</details>

Los mensajes `SSL_CTX_… failed` y `unsafe` ya no deben aparecer después de guardar, y las entregas a los socios deben terminar con `stat=Sent`. Sendmail entregará por sí mismo desde la cola los correos que antes se aplazaron con un código 4xx. Los remitentes deberán volver a enviar los correos rechazados con un código 5xx.

Copiar manualmente `sendmail.pem` a `sendmail-client.pem` también salva temporalmente la interrupción. Sin embargo, PPS volverá a sobrescribir el archivo en el siguiente envío de configuración mientras falte la asignación en la GUI.

## Comprobar después de cada parche

El parche 5807 modificó la configuración sin que la descripción del parche advierta de las consecuencias. Por ello, recomiendo revisar la página **Services** después de cada parche y consultar **Config History** para comprobar si el parche incluyó cambios de configuración. Si el parche 5807 aún no está instalado en un sistema 8.22.4, asigne un certificado a `SMTP Client` inmediatamente después de la instalación. Es más sencillo contar con una supervisión que alerte cuando aparezca el mensaje `sendmail-client.pem` en el registro de correo de los agentes. En **Licenses and Updates > Settings** también se puede definir cuándo se instalan los parches. Una ventana en la que alguien pueda comprobar el flujo de correo reduce considerablemente este tipo de interrupciones.

## Client Authentication desaparece de los certificados públicos

Independientemente del parche, un segundo cambio afecta precisamente a estas conexiones. Hasta ahora, los certificados TLS públicos solían contener ambos usos en la extensión `Extended Key Usage`: `TLS Web Server Authentication` y `TLS Web Client Authentication`. Por ello, las pasarelas de correo también utilizan su certificado de servidor como certificado de cliente.

El Chrome Root Program exige que los certificados públicos emitidos a partir del 15.03.2027 contengan únicamente `serverAuth`. DigiCert emite certificados TLS públicos sin Client Authentication de forma predeterminada desde el 01.10.2025; el propósito todavía puede solicitarse hasta el 01.03.2027. Otras CA tienen calendarios similares. OpenSSL permite comprobar si un certificado contiene el propósito de cliente:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `x509` | Subcomando para visualizar y editar certificados X.509 |
| `-in zertifikat.pem` | Archivo de certificado en formato PEM |
| `-noout` | No volver a mostrar el propio certificado |
| `-subject` | Mostrar el titular (CN y otros campos) |
| `-enddate` | Mostrar la fecha de caducidad |
| `-ext extendedKeyUsage` | Mostrar solo la extensión con los usos previstos |

</details>

En Windows, hacer doble clic en el archivo muestra lo mismo: pestaña **Details**, campo **Enhanced Key Usage**.

Lo que esto implica para las conexiones salientes depende del sistema remoto:

| El sistema remoto verifica | Consecuencia con un certificado sin Client Authentication |
|---|---|
| Nombre en el certificado (p. ej., Exchange Online mediante `TlsSenderCertificateName`) | No está documentado si Exchange Online verifica el uso previsto; pruebe antes de cambiar el certificado |
| Uso previsto del certificado de cliente | Rechazo, aunque el nombre y la CA sean correctos |
| Huella digital del certificado de cliente | Rechazo en cada cambio de certificado hasta que se registre la nueva huella digital |

A ello se suma la menor validez de los certificados públicos. Tras la decisión SC-081v3 del CA/Browser Forum, se reduce gradualmente de 398 días a 47 días hasta marzo de 2029; la primera etapa, con 200 días, está vigente desde marzo de 2026. Cada cambio de certificado es un momento en el que un sistema remoto puede no conocer el nuevo certificado.

Por ello, para las conexiones en las que los socios identifican Proofpoint mediante un certificado de cliente, recomiendo:

1.  **Comprobar los certificados existentes:** ¿Los certificados actuales de la pasarela todavía contienen el propósito de cliente y cuándo caducan?

2.  **Planificar un certificado de cliente propio** que incluya permanentemente el propósito de cliente, por ejemplo de una PKI privada o de una oferta de la CA para conexiones de servidor a servidor. Los socios deben conocerlo.

3.  **Preguntar a los socios qué verifican:** nombre, uso previsto, emisor o huella digital. De ello depende si un cambio de certificado supone un cambio para ellos.

4.  **Gestionar el cambio de certificado como un change**, con una lista de todos los sistemas remotos que deben conocer el certificado y una prueba para cada uno.

## Fuentes

1.  [Patch 5806 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, requiere inicio de sesión)](https://proofpoint.my.site.com/community/s/article/Patch-5806---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): artículo 000027579 del 29.10.2026, archivos modificados y reinicio de la GUI de administración.

2.  [Patch 5807 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, requiere inicio de sesión)](https://proofpoint.my.site.com/community/s/article/Patch-5807---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): artículo 000027580 del 29.10.2026, cambio de configuración únicamente con dependencia de 5806.

3.  [Proofpoint Enterprise Protection 8.23.X Release Notes (Proofpoint Knowledge Base, requiere inicio de sesión)](https://proofpoint.my.site.com/community/s/article/Email-Protection-PPS-PoD-Proofpoint-Enterprise-Protection-8-23-X-Release-Notes): artículo 000027117, sección «SMTP Client Certificate» en la página 5 de la revisión A.

4.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): parámetros `RestrictDomainsToCertificate` y `TlsSenderCertificateName` para conectores de socios en Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): sección 1.3.2 sobre jerarquías TLS de servidor dedicadas; certificados emitidos a partir del 15.03.2027 solo con `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): estándar sin Client Authentication desde el 01.10.2025, disponible previa solicitud hasta el 01.03.2027, alternativas para conexiones de servidor a servidor.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): resumen de las fechas aplazadas para Client Authentication y de las etapas de validez en DigiCert y Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): decisión sobre la reducción gradual de la validez de los certificados TLS públicos hasta marzo de 2029.
