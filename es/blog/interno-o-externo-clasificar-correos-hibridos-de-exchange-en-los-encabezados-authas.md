---
title: "¿Interno o externo? Clasificar correos híbridos de Exchange en los encabezados: AuthAs, MessageDirectionality y X-originatorOrg"
navTitle: "Leer encabezados híbridos"
description: "En entornos híbridos de Exchange, la clasificación de los encabezados determina si un correo se trata como interno. Qué encabezados determinan la clasificación, cómo funciona la asignación de tenants mediante certificado y conector, y cómo reconocer un mensaje enrutado incorrectamente."
date: "2026-08-26"
kategorie: "Exchange OnPrem / híbrido"
timeToRead: "10 min de lectura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-hybrid"
  - "hybrid-mailfluss"
  - "exchange-online"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - microsoft-365-compauth-reason-codes
  - einliefernde-ip-adressen-aggregieren
slug: "interno-o-externo-clasificar-correos-hibridos-de-exchange-en-los-encabezados-authas"
translationId: "article-c8d7859be8dbfe63"
translationOf: exchange-hybrid-header-intern-extern
url: https://rafaelpfister.ch/es/blog/interno-o-externo-clasificar-correos-hibridos-de-exchange-en-los-encabezados-authas
translationSourceHash: 5a0eccedd4b1a194461602319f5f1a8f59de204c1710e261c2358591bb720dfb
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:19:32.526Z
translationReview: automatic
---

# ¿Interno o externo? Clasificar correos híbridos de Exchange en los encabezados: AuthAs, MessageDirectionality y X-originatorOrg

En un entorno híbrido, los correos entre Exchange OnPrem y Exchange Online deben tratarse como correo interno: sin filtro de spam intermedio, sin indicación de «Externo», entrega a listas de distribución protegidas y nombres para mostrar resueltos. Que esto funcione no depende del dominio del remitente, sino de un conjunto de encabezados que deben conservarse durante el recorrido entre ambos mundos. Quien sabe leerlos puede responder directamente en el encabezado las preguntas híbridas más frecuentes: ¿llegó el correo a través del conector híbrido? ¿Por qué se clasificó como externo? ¿Y a qué tenant se asignó?

## Los encabezados implicados

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** indica la clasificación: `Internal` o `Anonymous`. Es el resultado de las demás señales y el indicador más directo de cómo Exchange ha tratado el mensaje.

**`AuthSource`** indica el FQDN del servidor que realizó la clasificación: un servidor OnPrem propio, un servidor de buzones de Exchange Online o un frontend de EOP. Esto permite determinar en qué lado tuvo lugar la evaluación.

**`MessageDirectionality`** distingue `Originating` (el mensaje se originó dentro de la organización, en Exchange Online o mediante un conector de entrada autenticado) de `Incoming` (el mensaje llegó desde fuera).

**`X-OriginatorOrg`** identifica la organización remitente desde la perspectiva de Exchange Online: el dominio aceptado predeterminado o correspondiente del tenant remitente. El encabezado se establece al enviar desde Exchange Online mediante la extensión SMTP XOORG y está vinculado a la combinación de certificado TLS de EOP, configuración del conector y dominio aceptado. Por ello, no puede falsificarse simplemente enviándolo: un `X-OriginatorOrg` entregado desde fuera sin la relación de confianza correspondiente no se reconoce como tal.

Además, están los encabezados `X-MS-Exchange-CrossTenant-*`, que Exchange Online inserta al pasar entre tenants, entre ellos `X-MS-Exchange-CrossTenant-AuthAs`. Reflejan la clasificación desde la perspectiva del tenant receptor.

## Cómo funciona técnicamente la relación de confianza

La clasificación interna a través del límite organizativo se basa en dos componentes que configura el Hybrid Configuration Wizard:

En primer lugar, el **Inbound Connector** de tipo OnPremises en Exchange Online, que identifica la fuente de entrega mediante el certificado TLS (`TlsSenderCertificateName`) o la dirección IP. Exchange Online también utiliza esta asignación para decidir a qué tenant se atribuye una entrega (attribution).

En segundo lugar, el indicador **`CloudServicesMailEnabled`** en los conectores de ambos lados. Garantiza que los encabezados `X-MS-Exchange-Organization-*` (encabezados entre instalaciones) se conserven durante la transición, en lugar de eliminarse como ocurre con el correo externo. Si falta el indicador o el correo sigue una ruta sin esta configuración, los encabezados se pierden y el correo llega como `Anonymous`.

De ello se deriva la regla de diagnóstico más importante: un correo híbrido solo es interno si realmente ha seguido la ruta configurada por el HCW.

## Caso 1: El correo llega como Anonymous aunque debería ser interno

Este es el problema más frecuente: los correos de buzones OnPrem aparecen en Exchange Online como externos, con comprobación de spam, etiquetado «Externo» o rechazo en listas de distribución protegidas. Las causas, por frecuencia descendente:

- **Ruta incorrecta:** El correo no pasó por el conector híbrido, sino por el MX (es decir, a través de EOP como correo de Internet) o por una puerta de enlace previa que elimina los encabezados entre instalaciones o termina la conexión TLS. Se aprecia en la cadena `Received`: en lugar de la entrega directa de OnPrem a `*.mail.protection.outlook.com` mediante el conector, aparecen estaciones intermedias.
- **Cambio de certificado:** Se renovó el certificado OnPrem, pero no se actualizó `TlsSenderCertificateName` en el Inbound Connector. La identificación mediante el certificado deja de funcionar.
- **Configuración del conector modificada:** `CloudServicesMailEnabled` se desactivó durante la resolución de problemas o un conector creado manualmente sustituyó al conector HCW sin los ajustes necesarios.

La comprobación en el lado de Exchange Online:

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

En el seguimiento de mensajes, el campo `ConnectorName` indica si el mensaje se entregó realmente a través del conector esperado.

## Caso 2: Asignación al tenant incorrecto

Exchange Online asigna cada mensaje entrante a un tenant; el encabezado `X-EOPTenantAttributedMessage` contiene el GUID del tenant atribuido. Si dos tenants utilizan el mismo `TlsSenderCertificateName` o los mismos `SenderIPAddresses` en sus Inbound Connectors, por ejemplo con un proveedor común de servicios de puerta de enlace o tras una migración, un mensaje puede atribuirse al tenant equivocado. Entonces no aparece en el seguimiento de mensajes del tenant propio y queda sujeto a reglas de transporte ajenas.

`Get-OrganizationConfig | Select-Object GUID` proporciona el GUID del tenant propio; si no coincide con el encabezado, deben separarse los identificadores de los conectores: un certificado propio o rangos IP propios por tenant.

## Caso 3: Un correo clasificado como externo se trata de todos modos como interno

El caso inverso se produce en OnPrem: un Receive Connector con la opción `ExternalAuthoritative` («Externally secured») clasifica como interno todo lo entregado a través de él, identificable por `AuthAs: Internal` junto con `AuthMechanism: 10`. Si un conector de este tipo apunta a una puerta de enlace por la que también circula correo de Internet, el correo externo se considera interno, con todas las consecuencias para los filtros de spam y la protección contra suplantación. Los detalles y las medidas correctivas se explican en el artículo [AuthMechanism 10 y AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Leer el encabezado en su conjunto

Para clasificar un mensaje concreto se necesitan todas las señales juntas: la cadena `Received` para la ruta real, `AuthAs`/`AuthSource`/`MessageDirectionality` para la clasificación, y `X-OriginatorOrg` junto con los encabezados CrossTenant para la organización de origen. El [analizador de encabezados de correo](/tools/header-analyzer) de este sitio web analiza estos campos directamente en el navegador y marca la transición entre tenants y la clasificación híbrida en la ruta de entrega; el encabezado no abandona el navegador.

## Fuentes

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Descripción oficial de la clasificación Internal, los encabezados implicados y los requisitos de los conectores.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): Funcionamiento de XOORG y X-OriginatorOrg en el enrutamiento entre Exchange Online y OnPrem.

3.  [Microsoft Learn (archivo): Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage y el procedimiento en caso de asignación al tenant incorrecto.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Referencia sobre tipos de Inbound Connector, TlsSenderCertificateName y attribution.
