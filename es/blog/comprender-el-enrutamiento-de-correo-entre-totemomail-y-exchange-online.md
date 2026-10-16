---
title: "Comprender el enrutamiento de correo entre totemomail y Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Cómo totemomail almacena y procesa mensajes, cómo el Apache James subyacente cambia entre processors y qué es importante para un bucle de correo seguro con Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min de lectura"
themen:
  - totemomail
slug: "comprender-el-enrutamiento-de-correo-entre-totemomail-y-exchange-online"
translationOf: "totemomail-m365"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: d92dbf8dac2f4fe0cd6e28f70e9e5575c76b56acb4afa5c5444f9e10895b4abf
translatedAt: 2026-10-14T08:58:09.966Z
url: https://rafaelpfister.ch/es/blog/comprender-el-enrutamiento-de-correo-entre-totemomail-y-exchange-online
translationModel: gpt-5.6-terra
---

# Comprender el enrutamiento de correo entre totemomail y Exchange Online

En un bucle de correo entre Exchange Online y totemomail, cada sistema asume una tarea claramente delimitada. Exchange Online proporciona los buzones. Totemomail, o el actual Kiteworks Email Protection Gateway, se encarga del cifrado, las firmas, las políticas y las reglas especiales de enrutamiento.

Para lograr un flujo de correo fiable, no basta con configurar dos conectores SMTP. Para solucionar problemas también debe quedar claro qué ocurre dentro del gateway después de aceptar un mensaje: ¿dónde se encuentra? ¿Qué regla se ejecutará a continuación? ¿Y por qué un mensaje puede esperar en una cola aunque el diálogo SMTP ya haya finalizado correctamente?

Por ello, este artículo explica el modelo de procesamiento de [Apache James](https://james.apache.org/), en el que se basa totemomail. La configuración concreta del enrutamiento depende de cada entorno; sin embargo, los processors, matchers, mailets y repositorios descritos constituyen la base técnica de toda instalación.

Una regla de seguridad importante se aplica independientemente de los detalles: si totemomail es el gateway previo, Exchange Online solo debe aceptar correo de Internet de este gateway. Para ello se necesita un conector de partner restrictivo. Un registro MX por sí solo no bloquea la vía de entrega directa. El artículo [Un registro MX no es un firewall](/blog/ghost-sender-exchange-online-nebeneingang) muestra cómo surge esta entrada secundaria y cómo cerrarla.

## De la entrada SMTP al procesamiento

La lógica de procesamiento de Apache James consta de cuatro componentes:

- **Matchers** comprueban condiciones y determinan para qué destinatarios se aplica una regla.
- **Mailets** ejecutan la acción propiamente dicha, como modificar encabezados, cifrar, entregar o finalizar el procesamiento posterior.
- **Processors** agrupan matchers y mailets en pasos de procesamiento ordenados.
- **Repositorios de correo** almacenan mensajes durante el procesamiento o después de un error.

Esta separación es decisiva para el análisis: el repositorio responde a la pregunta de **dónde** se encuentra un mensaje. El processor determina **cómo** se sigue procesando.

![Uso de James como relay SMTP](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

El servidor SMTP acepta la conexión y lee el mensaje hasta el final de la sección `DATA`. Después, James crea un objeto `MailImpl`. Este contiene el contenido MIME como `MimeMessage`, así como los datos necesarios para el procesamiento: remitente, destinatarios, estado y otros atributos.

En un repositorio basado en archivos, James almacena esta información por separado:

- `FileStreamStore` contiene el mensaje RFC-822/MIME completo como flujo de bytes.
- `FileObjectStore` contiene el objeto `MailImpl` serializado con el estado y los metadatos.

Por tanto, un mensaje puede ya haber sido aceptado y almacenado por completo, aunque su procesamiento funcional todavía esté pendiente.

## Repositorios y colas bajo `/var/mail`

Los distintos repositorios aparecen en el sistema de archivos como directorios. En funcionamiento normal, un mensaje permanece allí solo durante muy poco tiempo. Si una cola se acumula, normalmente indica una regla defectuosa, un destino inaccesible o un servicio backend caído.

El siguiente ejemplo contiene, además de las colas estándar, directorios opcionales para una conexión HIN. HIN proporciona el espacio de comunicación seguro para el sector sanitario suizo.

> Si necesita ayuda para conectar HIN-Mailgateway o para migrar a la nueva solución HIN-Stargate, encontrará los expertos correspondientes en [adeptio](https://adeptio.ch/).  
>   
> **adeptio** es socio oficial de [Health Info Net AG](https://www.hin.ch/de/index.cfm) y, como tal, también dispone de contactos directos con el fabricante.  
> [➜ Reserve una cita hoy mismo.](/kontakt)

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

El archivo `*.FileStreamStore` contiene el mensaje RFC-822/MIME completo. Con `cat` se pueden leer los encabezados y el cuerpo:

```text
From:
To:
Subject:
...
Body
```

El formato de mensaje subyacente se describe en [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: estado y metadatos

El archivo `*.FileObjectStore` es un objeto Java serializado de tipo `org.apache.james.core.MailImpl`. Entre sus campos se incluyen:

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

## El estado selecciona el siguiente processor

La estructura de directorios muestra únicamente el repositorio. El estado de procesamiento real se encuentra en el campo `state` de `FileObjectStore`. Su valor hace referencia al atributo `name` de un processor.

Después de cada mailet, el SpoolManager comprueba este estado:

1. Si el estado no cambia, sigue el siguiente par matcher-mailet en el mismo processor.
2. Si un mailet cambia el estado, James finaliza el processor actual y salta al processor del mismo nombre.
3. El estado especial `ghost` finaliza por completo el procesamiento.

Los processors obligatorios `root` y `error` tienen tareas fijas. Los mensajes nuevos comienzan en `root`; los errores internos y los mailets configurados en consecuencia redirigen a `error`. Sin embargo, el orden de los elementos `<processor>` en el archivo XML **no** determina el orden de ejecución.

## Estructura de processors en `totemomail_config.xml`

Antes de cualquier modificación, se debe exportar y guardar la configuración actual de `totemomail_config.xml`:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

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

El processor `root` contiene una lista ordenada de pares matcher-mailet:

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

El archivo XML configura las clases, pero no las implementa. Por ejemplo, `SimpleLogger` es una clase proporcionada por totemomail o Kiteworks cuyo código fuente no está disponible en la appliance. Sin embargo, la ayuda de la GUI de administración explica sus parámetros:

- `log-message` define el texto de registro y es obligatorio.
- `showSenderEmailAddress` añade opcionalmente la dirección del remitente.
- `showRecipientsEmailAddress` añade las direcciones de los destinatarios.
- `showSubject` añade el asunto.

El orden **dentro** de un processor es vinculante. Un matcher puede seleccionar ninguno, todos o solo una parte de los destinatarios. En caso de un subconjunto, James divide el mensaje: los destinatarios coincidentes pasan por el mailet y los demás se siguen procesando por separado. Si un mailet cambia después el estado, el procesamiento salta inmediatamente al processor indicado; se omiten las reglas restantes del processor actual.

De ello se deriva un procedimiento fiable para solucionar problemas:

1. Determinar el repositorio y los archivos `FileStreamStore` y `FileObjectStore` correspondientes.
2. Determinar el `state` actual en `FileObjectStore`.
3. Buscar el processor del mismo nombre en `totemomail_config.xml`.
4. Comprobar los matchers y mailets en su orden real.
5. En caso de cambio de estado, continuar en el processor de destino.

De este modo, se puede seguir un flujo de correo paso a paso sin leer erróneamente el archivo XML de arriba abajo como un programa lineal.

## Fuentes

1.  [Apache James – Página del proyecto](https://james.apache.org/): MTA de código abierto en el que se basa técnicamente totemomail o Kiteworks EPG.
    
2.  [Apache James – «Spool Manager»](https://james.apache.org/server/head/spoolmanager.html): procesamiento de correos entrantes, spool y colas.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): configuración de processors y orden de las reglas.
    
4.  [Apache James – «Mailet API»](https://james.apache.org/server/head/mailet_api.html): concepto de mailets y matchers detrás de las reglas.
    
5.  [Apache James – «MailImpl» (documentación de API)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): modelo de objetos de correo detrás de FileStreamStore y FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): formato de los mensajes de texto de Internet (encabezados y cuerpo).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): configuración del conector para el flujo de correo seguro entre Exchange Online y el gateway.
