---
title: "Comprender el enrutamiento de correo entre totemomail y Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Cómo totemomail almacena y procesa los mensajes, cómo cambia el Apache James subyacente entre procesadores y qué es importante para un bucle de correo seguro con Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min de lectura"
themen:
  - totemomail
slug: "comprender-el-enrutamiento-de-correo-entre-totemomail-y-exchange-online"
translationOf: "totemomail-m365"
url: "https://rafaelpfister.ch/es/blog/comprender-el-enrutamiento-de-correo-entre-totemomail-y-exchange-online"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: 8dabf54e50de750dbd1e13baf487ccb1fa9db0d7bd98afcd1933e87bdb57f0af
translatedAt: 2026-07-29T12:29:38.950Z
---

# Comprender el enrutamiento de correo entre totemomail y Exchange Online

En un bucle de correo entre Exchange Online y totemomail, cada sistema asume una tarea claramente delimitada. Exchange Online proporciona los buzones. Totemomail, o el actual Kiteworks Email Protection Gateway, se encarga del cifrado, las firmas, las directivas y las reglas de enrutamiento especiales.

Para que esto se convierta en un flujo de correo fiable, no basta con configurar dos conectores SMTP. Para la resolución de problemas, también debe quedar claro qué sucede dentro de la puerta de enlace después de aceptar un mensaje: ¿Dónde se encuentra? ¿Qué regla se ejecuta a continuación? ¿Y por qué un mensaje puede esperar en una cola aunque el diálogo SMTP ya haya finalizado correctamente?

Por ello, este artículo explica el modelo de procesamiento de [Apache James](https://james.apache.org/), en el que se basa totemomail. La configuración concreta del enrutamiento depende de cada entorno; sin embargo, los procesadores, matchers, mailets y repositorios descritos constituyen la base técnica de cualquier instalación.

Una importante regla de seguridad se aplica independientemente de los detalles: si totemomail es la puerta de enlace previa, Exchange Online solo debe aceptar correo de Internet procedente de esta puerta de enlace. Para ello se necesita un conector de socio restrictivo. Una entrada MX por sí sola no bloquea la vía de entrega directa. El artículo [Un registro MX no es un firewall](/blog/ghost-sender-exchange-online-nebeneingang) muestra cómo surge esta entrada secundaria y cómo cerrarla.

## De la entrada SMTP al procesamiento

La lógica de procesamiento de Apache James consta de cuatro componentes:

- **Los matchers** comprueban condiciones y determinan a qué destinatarios se aplica una regla.
- **Los mailets** ejecutan la acción propiamente dicha, por ejemplo modificar cabeceras, cifrar, entregar o finalizar el procesamiento posterior.
- **Los procesadores** agrupan matchers y mailets en pasos de procesamiento ordenados.
- **Los repositorios de correo** almacenan mensajes durante el procesamiento o después de un error.

Esta separación es decisiva para el análisis: el repositorio responde a la pregunta de **dónde** se encuentra un mensaje. El procesador determina **cómo** se sigue procesando.

![Uso de James como relé SMTP](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

El servidor SMTP acepta la conexión y lee el mensaje hasta el final de la sección `DATA`. A continuación, James crea un objeto `MailImpl`. Este contiene el contenido MIME como `MimeMessage`, así como la información necesaria para el procesamiento: remitente, destinatarios, estado y otros atributos.

En un repositorio basado en archivos, James almacena esta información por separado:

- `FileStreamStore` contiene el mensaje RFC 822/MIME completo como flujo de bytes.
- `FileObjectStore` contiene el objeto `MailImpl` serializado con estado y metadatos.

Por tanto, un mensaje puede haber sido aceptado y almacenado completamente aunque su procesamiento funcional siga pendiente.

## Repositorios y colas en `/var/mail`

Los distintos repositorios aparecen como directorios en el sistema de archivos. En funcionamiento normal, un mensaje permanece allí solo durante muy poco tiempo. Si una cola se acumula, normalmente indica una regla incorrecta, un destino inaccesible o un servicio de backend caído.

El siguiente ejemplo contiene, además de las colas estándar, directorios opcionales para una conexión HIN. HIN proporciona el espacio de comunicación seguro para el sector sanitario suizo.

> Si necesita ayuda para conectar HIN-Mailgateway o para migrar a la nueva solución HIN-Stargate, encontrará a los expertos adecuados en [adeptio](https://adeptio.ch/).  
>   
> **adeptio** es socio oficial de [Health Info Net AG](https://www.hin.ch/de/index.cfm) y, como tal, también dispone de contactos directos con el fabricante.  
> [➜ Reserve hoy mismo una cita.](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)

```text
Root-Folder:
~/mailer/apps/james/var/mail

├── spool/
│   → Eingehende Mails (initiale Queue, noch nicht verarbeitet)
│
├── incoming/
│   → Mails, die als intern zuzustellen erkannt wurden (Standardfolder)
│
├── incomingHIN/
│   → Eingehende Mails für HIN-Netzwerk (Optional)
│
├── outgoing/
│   → Normale ausgehende Mails (Standardfolder)
│
├── outgoingHIN/
│   → Ausgehende Mails über HIN-Netzwerk (Optional)
│
├── outgoingNotifications/
│   → System- oder Zertifikatsbenachrichtigungen
│
├── error/
│   → Fehlgeschlagene Mails (z. B. Policy, Encryption, Routing)
│
├── DBUnavailable/
│   → Mails, die wegen Backend-/DB-Problemen nicht verarbeitet werden konnten
```

## Cómo se almacena un mensaje en el sistema de archivos

Cada mensaje almacenado consta de dos archivos.

### `FileStreamStore`: contenido del mensaje

El archivo `*.FileStreamStore` contiene el mensaje RFC 822/MIME completo. Con `cat` se pueden leer las cabeceras y el cuerpo:

```text
From:
To:
Subject:
...
Body
```

El formato de mensaje subyacente se describe en [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: estado y metadatos

El archivo `*.FileObjectStore` es un objeto Java serializado de tipo `org.apache.james.core.MailImpl`. Sus campos incluyen:

```text
attributes: HashMap
errorMessage: String
lastUpdated: Date
message: MimeMessage
name: String
state: String
recipients: Collection
remoteAddr
remoteHost
sender
```

La [documentación de la API de `MailImpl`](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) describe el modelo de objetos en detalle.

## El estado selecciona el siguiente procesador

La estructura de directorios solo muestra el repositorio. El estado real de procesamiento se encuentra en el campo `state` de `FileObjectStore`. Su valor hace referencia al atributo `name` de un procesador.

Después de cada mailet, el SpoolManager comprueba este estado:

1. Si el estado no cambia, se ejecuta el siguiente par matcher-mailet en el mismo procesador.
2. Si un mailet cambia el estado, James finaliza el procesador actual y salta al procesador con el mismo nombre.
3. El estado especial `ghost` finaliza completamente el procesamiento.

Los procesadores obligatorios `root` y `error` tienen tareas fijas. Los mensajes nuevos comienzan en `root`; los errores internos y los mailets configurados en consecuencia redirigen a `error`. Sin embargo, el orden de los elementos `<processor>` en el archivo XML **no** determina el orden de ejecución.

## Estructura de procesadores en `totemomail_config.xml`

Antes de cualquier modificación, se debe exportar y guardar la configuración actual `totemomail_config.xml`:

![Configuración / Abrir actual / Exportar a archivo](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

Los distintos procesadores y los mailets que contienen se muestran en totemomail\_config.xml. Aquí hay de nuevo un ejemplo práctico:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<spoolmanager>
    <multiparamformat>XML</multiparamformat>

    <processor name="addExtSender">
    <processor name="decrypt">
    <processor name="error">
    <processor name="externalDelivery">
    <processor name="externalDeliveryToHIN">
    <processor name="incoming">
    <processor name="internalDelivery">
    <processor name="internalDeliveryToHIN">
    <processor name="outgoing">
    <processor name="outgoingCheckRecipientCertificate">
    <processor name="outgoingProcessAutoGeneratedMessages">
    <processor name="outgoingProcessEncryptionTriggers">
    <processor name="outgoingProcessEncryptionTriggersRemoval">
    <processor name="outgoingProcessExceptionTriggers">
    <processor name="processIncoming">
    <processor name="processOutgoing">
    <processor name="processOutgoingCertificateExchange">
    <processor name="processOutgoingDomainEncryptionPGP">
    <processor name="processOutgoingDomainEncryptionSMIME">
    <processor name="processOutgoingNotifications">
    <processor name="root">

</spoolmanager>
```

Aunque `root` aparece al final de este extracto, cada mensaje nuevo comienza allí. Lo decisivo es el nombre, no la posición en el documento.

El procesador `root` contiene una lista ordenada de pares matcher-mailet:

```xml
   <processor name="root">
      <mailet class="SimpleLogger" match="All">
         <log-message>totemomail: New Mail</log-message>
         <showSenderEmailAddress>true</showSenderEmailAddress>
         <showRecipientsEmailAddress>true</showRecipientsEmailAddress>
         <showSubject>false</showSubject>
      </mailet>
      <mailet class="ToRepository" match="RelayLimit?Limit=20">
         <repositoryPath>file://var/mail/error</repositoryPath>
         <passThrough>false</passThrough>
         <notifySender>false</notifySender>
         <takeSenderInfoFrom>SMTP</takeSenderInfoFrom>
      </mailet>
      <mailet class="ToProcessor" match="HostIsLocal?includeSubdomains=no">
         <processor>incoming</processor>
      </mailet>
      <mailet class="ToProcessor" match="All">
         <processor>outgoing</processor>
      </mailet>
   </processor>
```

El archivo XML configura las clases, pero no las implementa. `SimpleLogger` es, por ejemplo, una clase proporcionada por totemomail o Kiteworks cuyo código fuente no está disponible en el appliance. Sin embargo, la ayuda de la interfaz gráfica de administración explica sus parámetros:

- `log-message` define el texto del registro y es obligatorio.
- `showSenderEmailAddress` añade opcionalmente la dirección del remitente.
- `showRecipientsEmailAddress` añade las direcciones de los destinatarios.
- `showSubject` añade el asunto.

El orden **dentro** de un procesador es vinculante. Un matcher puede seleccionar ninguno, todos o solo una parte de los destinatarios. En caso de un subconjunto, James divide el mensaje: los destinatarios coincidentes pasan por el mailet, mientras que los demás se procesan por separado. Si un mailet cambia posteriormente el estado, el procesamiento salta inmediatamente al procesador indicado; se omiten las reglas restantes del procesador actual.

Para la resolución de problemas, de ello se deriva un procedimiento fiable:

1. Determinar el repositorio y los archivos `FileStreamStore`/`FileObjectStore` correspondientes.
2. Determinar el `state` actual en `FileObjectStore`.
3. Buscar el procesador con el mismo nombre en `totemomail_config.xml`.
4. Comprobar los matchers y mailets en su orden real.
5. Si hay un cambio de estado, continuar en el procesador de destino.

De este modo, se puede seguir un flujo de correo paso a paso sin interpretar erróneamente el archivo XML de arriba abajo como un programa lineal.

## Fuentes

1.  [Apache James – Página del proyecto](https://james.apache.org/): MTA de código abierto en el que se basa técnicamente totemomail o Kiteworks EPG.
    
2.  [Apache James – «Spool Manager»](https://james.apache.org/server/head/spoolmanager.html): procesamiento de correos entrantes, spool y colas.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): configuración de procesadores y orden de las reglas.
    
4.  [Apache James – «Mailet API»](https://james.apache.org/server/head/mailet_api.html): concepto de mailet y matcher detrás de las reglas.
    
5.  [Apache James – «MailImpl» (documentación de API)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): modelo de objetos Mail detrás de FileStreamStore y FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): formato de mensajes de texto de Internet (cabeceras y cuerpo).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): configuración de conectores para el flujo de correo seguro entre Exchange Online y la puerta de enlace.
