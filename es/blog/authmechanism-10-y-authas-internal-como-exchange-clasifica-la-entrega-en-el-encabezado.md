---
title: "AuthMechanism 10 y AuthAs Internal: cómo Exchange clasifica la entrega en el encabezado"
navTitle: "AuthMechanism 10"
description: "El encabezado X-MS-Exchange-Organization-AuthMechanism documenta cómo se ha autenticado un servidor que realiza una entrega. El valor 10 corresponde a un Receive Connector con Externally Secured y clasifica los correos externos como internos, con consecuencias para los filtros antispam, las reglas de flujo de correo y la protección contra spoofing."
date: "2026-08-26"
featured: "2026-08-27"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "8 min de lectura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-hybrid-header-intern-extern
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "authmechanism-10-y-authas-internal-como-exchange-clasifica-la-entrega-en-el-encabezado"
translationId: "article-0df383d5c49016da"
translationOf: exchange-authmechanism-10-authas-internal
url: https://rafaelpfister.ch/es/blog/authmechanism-10-y-authas-internal-como-exchange-clasifica-la-entrega-en-el-encabezado
translationSourceHash: 5a9335a90afc9bf7df78b908f71b679f64c29f3b9e96bd7f25bcc916123b82df
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:18:04.966Z
translationReview: automatic
---

# AuthMechanism 10 y AuthAs Internal: cómo Exchange clasifica la entrega en el encabezado

Al analizar casos de spam, spoofing y flujo de correo en entornos Exchange, hay tres encabezados decisivos que Exchange marca al recibirlos:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` registra cómo se presentó el remitente ante el transporte. `AuthSource` indica el servidor que realizó la evaluación. `AuthMechanism` documenta mediante qué mecanismo se produjo la autenticación. En conjunto determinan si Exchange trata un mensaje como interno o externo, y esta clasificación tiene consecuencias considerables.

## Por qué importa la clasificación

`AuthAs` tiene en la práctica dos valores: `Internal` y `Anonymous`. Un mensaje clasificado como `Internal` se trata de forma distinta al correo externo:

- Las reglas de flujo de correo con la condición «remitente fuera de la organización» no se aplican.
- El mensaje puede entregarse a listas de distribución y buzones que requieren remitentes autenticados (`RequireSenderAuthenticationEnabled`).
- Las comprobaciones antispam y antispoofing son menos estrictas o no se realizan; en entornos híbridos no se añade el descargo de responsabilidad externo y Outlook no muestra el aviso «Externo».
- Se resuelve el nombre para mostrar de la libreta de direcciones y el correo parece correo interno para los destinatarios.

Precisamente por ello, la pregunta «¿AuthAs Internal o Anonymous?» debe estar al principio de todo análisis de encabezados: permite aclarar por qué un correo de spoofing evidente superó el filtro antispam o por qué una regla de flujo de correo nunca se activó.

## Los valores de AuthMechanism

Microsoft no documenta públicamente de forma completa la codificación de `AuthMechanism`. Dos valores son relevantes y están bien documentados para la resolución de problemas:

| Valor | Significado |
|---|---|
| `04` | Tráfico Exchange autenticado: de buzón a buzón dentro de la organización, así como tráfico híbrido a través de los conectores configurados por Hybrid Configuration Wizard. |
| `10` | Receive Connector con la opción de autenticación `ExternalAuthoritative` («Protegido externamente» / «Externally secured»): la conexión se considera protegida fuera de Exchange; todo lo entregado a través de ella se trata como interno. |

Otros valores aparecen en los encabezados, pero carecen de referencia oficial. Para la práctica basta con distinguir: `04` significa autenticación real de Exchange; `10` significa confianza mediante la configuración del conector.

## Qué significa realmente Externally Secured

La opción `ExternalAuthoritative` en un Receive Connector le indica a Exchange: otro componente se encarga de proteger esta conexión, por ejemplo, un firewall, un segmento de red dedicado o IPsec. Entonces Exchange ya no comprueba nada y trata toda entrega a través de este conector como autenticada e interna, incluido el derecho a utilizar direcciones de remitente internas arbitrarias.

Está pensado para pocos escenarios, como un servidor de aplicaciones completamente fiable en el propio centro de datos. Se vuelve problemático cuando el conector apunta a una puerta de enlace de correo anterior o a un filtro antispam en la DMZ, a través del cual también entra correo de Internet. En ese caso, cada correo externo lleva tras su entrega:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

Las consecuencias: los correos externos se consideran internos, las reglas de flujo de correo para remitentes externos no se aplican, la protección contra spoofing para el dominio propio queda sin efecto y cualquiera que alcance la puerta de enlace puede entregar, usando direcciones de remitente internas, a destinatarios que en realidad exigen remitentes autenticados.

## Encontrar los conectores afectados

La Exchange Management Shell muestra qué Receive Connectors están configurados con `ExternalAuthoritative`:

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

Compruebe en cada resultado qué `RemoteIPRanges` están registrados y si los sistemas situados detrás realmente necesitan esta confianza. Una puerta de enlace que solo debe reenviar correos no la necesita.

## La alternativa para escenarios de relay

Si un sistema solo debe realizar relay anónimo a través de Exchange (impresoras, aplicaciones, monitorización), un conector de relay anónimo es la solución más limpia: entrega anónima más el derecho a entregar a cualquier destinatario, pero sin la clasificación Internal.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

Los correos a través de este conector permanecen `AuthAs: Anonymous`, pasan por las comprobaciones habituales y no pueden suplantar remitentes internos. `ExternalAuthoritative` debe reservarse para los sistemas a los que desee conceder deliberadamente el derecho a utilizar direcciones de remitente internas.

## Leer los encabezados en contexto

La forma más rápida de comprobar si un mensaje concreto se clasificó como interno o externo y por qué vía llegó es revisar el encabezado completo: `AuthAs`, `AuthMechanism` y `AuthSource`, junto con la cadena `Received`. El [analizador de encabezados de correo](/tools/header-analyzer) de este sitio web evalúa estos campos directamente en el navegador y marca la clasificación híbrida en la ruta de entrega; el encabezado no sale del navegador.

El artículo [¿Interno o externo? Clasificar correos híbridos de Exchange en el encabezado](/blog/exchange-hybrid-header-intern-extern) trata cómo se conserva la clasificación entre Exchange Online y OnPrem en entornos híbridos y cómo reconocer una asignación incorrecta.

## Fuentes

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): Asignación de AuthAs y AuthMechanism 10 a la configuración Externally Secured y su efecto sobre las reglas de flujo de correo.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Descripción oficial de la clasificación Internal y sus consecuencias en el flujo de correo híbrido.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): Valores observados de AuthAs, AuthSource y AuthMechanism en distintos escenarios de entrega.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): Configuración del conector de relay anónimo como alternativa a Externally Secured.
