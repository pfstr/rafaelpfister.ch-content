---
title: "Proofpoint Protection Server 8.22.4: El parche puede restablecer la configuración del certificado para SMTP Client"
navTitle: "Certificado PPS 8.22.4"
description: "Si, en Proofpoint Protection Server, en System > Certificates > Services, la asignación «SMTP Client» aparece como «-none-», Sendmail realiza las entregas sin certificado de cliente TLS. Los destinos que verifican certificados lo rechazan, por ejemplo Exchange Online con 5.7.51 o un Smarthost con «Relay access denied». Así puede comprobarlo y solucionarlo."
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
url: https://rafaelpfister.ch/es/blog/proofpoint-protection-server-8-22-4-el-parche-puede-restablecer-la-configuracion-del
translationSourceHash: a6ce9c6e7e8b62910100ae9ff2323179510735b20326f5b916838ea5df74de02
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:27:17.536Z
translationReview: automatic
---

Hoy investigué una interrupción en el flujo de correo saliente de un Proofpoint Protection Server (PPS) 8.22.4. Conclusión breve: tras instalarse automáticamente un parche, la asignación **SMTP Client** en **System > Certificates > Services** pasó a ser `-none-`. A continuación, Sendmail estableció conexiones salientes sin certificado de cliente TLS. Todos los destinos que identifican Proofpoint mediante este certificado rechazaron los correos. Los destinatarios habituales con TLS oportunista siguieron aceptándolos, por lo que el problema solo se detectó cuando algunos socios dejaron de estar accesibles.

Si utiliza PPS con parches automáticos y tiene socios que exigen un certificado de cliente, compruebe esta configuración después de cada parche.

## Síntoma

Solo se ven afectadas las conexiones en las que el destino verifica el certificado de cliente de Proofpoint. Dos casos típicos:

- **Exchange Online con un conector de socios**, restringido a un certificado mediante `RestrictDomainsToCertificate`. El rechazo es `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost que solo permite relay con certificado de cliente.** La respuesta es `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Todos los demás destinos siguen recibiendo sus correos. Por ello, la matriz de pruebas muestra una imagen desigual: los correos externos se entregan, pero los correos a determinados socios o a través de un Smarthost concreto fallan.

En el registro de correo del agente aparece uno de los dos siguientes mensajes en cada conexión saliente. Si el archivo no existe:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Si el archivo existe, pero está vacío:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Inmediatamente después, Sendmail establece la conexión con TLS de todos modos (`verify=OK`), pero sin certificado de cliente. Para el destino, esto parece un remitente desconocido.

## Causa

PPS asigna un certificado a cada servicio por servidor. En **System > Certificates > Services**, cada servidor incluye, entre otros, `SMTP Server`, `SMTP Client`, `Administration Server` y `End User Web Application / Encryption`. A partir de la asignación en `SMTP Client`, PPS genera el archivo `/etc/pki/tls/certs/sendmail-client.pem`, que Sendmail utiliza como certificado de cliente. Si figura `-none-`, PPS deja el archivo vacío o no lo crea.

En el entorno investigado, tras un parche automático la asignación aparecía como `-none-` en todos los servidores. En **System > Config History**, el parche era visible como entrada `configuration changes for patch-…` con el creador `system`, y los archivos bajo `/etc/pki/tls/certs/` tenían la misma marca de tiempo. Las notas de versión disponibles públicamente no indican si Proofpoint introdujo la asignación con este parche o si el parche restableció una asignación existente. Actualizaré el artículo en cuanto Proofpoint se pronuncie al respecto.

Revertir en **Config History** a la instantánea del parche no ayudó. PPS volvió a escribir el archivo, pero con 0 bytes.

## Comprobar

En la GUI de administración del maestro, en **System > Certificates > Services**, seleccione sucesivamente en la parte superior, en **Server**, cada agente que realiza entregas al exterior y revise la línea `SMTP Client`.

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
| `--time-style=full-iso` | Marca de tiempo con fecha, hora y zona horaria, para compararla con el momento del parche |
| `grep -E` | Expresiones regulares ampliadas, para combinar ambos términos de búsqueda con `\|` |
| `tail -5` | Mostrar solo las cinco últimas coincidencias |

</details>

Si falta `sendmail-client.pem` o el archivo tiene 0 bytes, mientras que `sendmail.pem` tiene varios miles de bytes, no hay ningún certificado asignado al servicio `SMTP Client`.

## Solucionar

En **System > Certificates > Services**, seleccione un certificado válido para `SMTP Client` en cada agente afectado y guarde con **Save Changes**. Por regla general, es el mismo certificado público que figura en `SMTP Server`.

La lista de selección muestra todos los certificados importados, incluidos los caducados con el mismo nombre. Solo pueden distinguirse mediante el número de serie entre paréntesis. Antes de guardar, compare el número de serie con la lista en **System > Certificates > Certificates**, donde figuran las fechas de emisión y caducidad.

Tras guardar, PPS distribuye la configuración a los agentes. Comprobación en el agente:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `ls -l` | Muestra el tamaño y los permisos del archivo; tras la corrección ya no estará vacío |
| `grep -E 'STARTTLS=client\|stat='` | Filtra los mensajes TLS de las conexiones salientes y el resultado de la entrega |
| `tail -20` | Mostrar solo las veinte últimas líneas |

</details>

Los mensajes `SSL_CTX_… failed` y `unsafe` ya no deben aparecer después de guardar, y las entregas a los socios deben terminar con `stat=Sent`. Sendmail entrega por sí mismo desde la cola los correos que anteriormente se aplazaron con un código 4xx. Los correos rechazados con un código 5xx deben reenviarlos los remitentes.

Copiar manualmente `sendmail.pem` a `sendmail-client.pem` también permite sortear la interrupción. Sin embargo, PPS vuelve a sobrescribir el archivo con el siguiente envío de configuración mientras falte la asignación en la GUI.

## Comprobar después de cada parche

Hasta que se aclare por qué el parche modificó la asignación, recomiendo comprobar la página **Services** después de cada parche. Más sencillo es implementar una supervisión que alerte cuando aparezca el mensaje `sendmail-client.pem` en el registro de correo de los agentes. En **Licenses and Updates > Settings** también se puede definir cuándo se instalan los parches. Una ventana de tiempo en la que alguien pueda comprobar el flujo de correo reduce considerablemente una interrupción de este tipo.

## Client Authentication desaparece de los certificados públicos

Independientemente del parche, un segundo cambio afecta precisamente a estas conexiones. Hasta ahora, los certificados TLS públicos solían incluir ambos usos en la extensión `Extended Key Usage`: `TLS Web Server Authentication` y `TLS Web Client Authentication`. Por ello, las pasarelas de correo utilizan su certificado de servidor también como certificado de cliente.

El Chrome Root Program exige que los certificados públicos emitidos a partir del 15.03.2027 contengan únicamente `serverAuth`. Desde el 01.10.2025, DigiCert emite certificados TLS públicos sin Client Authentication de forma predeterminada; previa solicitud, este propósito seguirá siendo posible hasta el 01.03.2027. Otras CA tienen calendarios similares. OpenSSL muestra si un certificado incluye el propósito de cliente:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `x509` | Subcomando para mostrar y editar certificados X.509 |
| `-in zertifikat.pem` | Archivo de certificado en formato PEM |
| `-noout` | No volver a mostrar el certificado en sí |
| `-subject` | Mostrar el titular (CN y otros campos) |
| `-enddate` | Mostrar la fecha de caducidad |
| `-ext extendedKeyUsage` | Mostrar únicamente la extensión con los propósitos de uso |

</details>

En Windows, un doble clic en el archivo muestra lo mismo: pestaña **Details**, campo **Enhanced Key Usage**.

Lo que esto significa para las conexiones salientes depende del destino:

| El destino verifica | Consecuencia con un certificado sin Client Authentication |
|---|---|
| Nombres en el certificado (p. ej., Exchange Online mediante `TlsSenderCertificateName`) | No está documentado si Exchange Online verifica el propósito de uso; pruébelo antes de cambiar el certificado |
| Propósito de uso del certificado de cliente | Rechazo, aunque el nombre y la CA sean correctos |
| Huella digital del certificado de cliente | Rechazo en cada cambio de certificado hasta que se registre la nueva huella digital |

A esto se suma la menor vigencia de los certificados públicos. Tras la decisión SC-081v3 del CA/Browser Forum, se reduce gradualmente de 398 días a 47 días hasta marzo de 2029; la primera etapa, de 200 días, está en vigor desde marzo de 2026. Cada cambio de certificado es un momento en el que un destino puede no conocer el nuevo certificado.

Por tanto, para las conexiones en las que los socios identifican Proofpoint mediante un certificado de cliente, recomiendo:

1.  **Comprobar los certificados existentes:** ¿Los certificados actuales de las pasarelas aún contienen el propósito de cliente y cuándo caducan?

2.  **Planificar un certificado de cliente propio** que incluya permanentemente el propósito de cliente, por ejemplo de una PKI privada o de una oferta de la CA para conexiones servidor a servidor. Los socios deben conocerlo.

3.  **Preguntar a los socios qué verifican:** nombre, propósito de uso, emisor o huella digital. De ello depende si un cambio de certificado supone un cambio para ellos.

4.  **Gestionar los cambios de certificado como un cambio**, con una lista de todos los destinos que deben conocer el certificado y una prueba para cada destino.

## Fuentes

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): parámetros `RestrictDomainsToCertificate` y `TlsSenderCertificateName` para conectores de socios en Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): sección 1.3.2 sobre jerarquías TLS dedicadas para servidores; certificados emitidos a partir del 15.03.2027 únicamente con `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): estándar sin Client Authentication desde el 01.10.2025, previa solicitud hasta el 01.03.2027, alternativas para conexiones servidor a servidor.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): resumen de las fechas aplazadas para Client Authentication y de las etapas de vigencia en DigiCert y Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): decisión sobre la reducción gradual de la vigencia de los certificados TLS públicos hasta marzo de 2029.
