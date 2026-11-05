---
title: "Exchange Server: Permitir solo remitentes externos seleccionados en una lista de distribución"
navTitle: "Distribución: remitentes externos"
description: "Una lista de distribución en Exchange Server puede configurarse para que solo determinados remitentes externos puedan contactarla. Dos métodos con PowerShell: restricción de entrega con contactos de correo electrónico o regla de transporte. También se explican los límites de la comprobación del remitente y cómo DMARC los resuelve."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min de lectura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-message-tracking-und-receive-connectoren-analysieren
slug: "exchange-server-permitir-solo-remitentes-externos-seleccionados-en-una-lista-de-distribucion"
translationId: "article-b3dc70052f5e8729"
translationOf: exchange-verteiler-externe-absender
url: https://rafaelpfister.ch/es/blog/exchange-server-permitir-solo-remitentes-externos-seleccionados-en-una-lista-de-distribucion
translationSourceHash: cff4b579945841f2469f23b5e44d834f6be7528cf9817f9d7ade0fd613f1c582
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:22:45.084Z
translationReview: automatic
---

# Exchange Server: Permitir solo remitentes externos seleccionados en una lista de distribución

De forma predeterminada, una lista de distribución en Exchange Server solo acepta mensajes de remitentes autenticados, es decir, internos. Si también debe poder contactar con la lista un socio, proveedor o cliente, sin dejarla abierta a todo Internet, existen dos opciones: una restricción de entrega en el propio grupo o una regla de transporte. Ambas funcionan en Exchange Server 2016, 2019 y Subscription Edition.

## Situación inicial: RequireSenderAuthenticationEnabled

La propiedad `RequireSenderAuthenticationEnabled` controla si un grupo acepta mensajes externos. Si está establecida en `$true` (valor predeterminado para los nuevos grupos de distribución), Exchange rechaza todos los mensajes de remitentes no autenticados. Por tanto, las dos variantes siguientes requieren desactivar esta comprobación:

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity` | El grupo de distribución (nombre, alias o dirección de correo electrónico). |
| `-RequireSenderAuthenticationEnabled $false` | El grupo también acepta mensajes de remitentes externos no autenticados. |

</details>

Sin restricciones adicionales, tras este paso cualquier remitente externo puede escribir a la lista. La restricción a remitentes individuales se aplica en el siguiente paso, idealmente durante la misma ventana de mantenimiento.

## Variante 1: Restricción de entrega con contactos de correo electrónico

La lista de remitentes permitidos (`AcceptMessagesOnlyFromSendersOrMembers`) solo acepta objetos del directorio propio: buzones, usuarios de correo electrónico, contactos de correo electrónico y grupos. No se puede introducir una dirección SMTP sin más. Por ello, cree un contacto de correo electrónico para cada remitente externo. Para evitar que el contacto aparezca en la lista global de direcciones, ocúltelo:

```powershell
$kontakt = @{
    Name                 = "Extern Max Muster"
    ExternalEmailAddress = "max.muster@partner.example"
    OrganizationalUnit   = "example.local/Kontakte/Extern"
}
New-MailContact @kontakt
Set-MailContact -Identity "Extern Max Muster" -HiddenFromAddressListsEnabled $true
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Name` | Nombre para mostrar del contacto en Active Directory. Un prefijo uniforme como «Externo» facilita el mantenimiento posterior. |
| `ExternalEmailAddress` | La dirección externa desde la que escribe el remitente. |
| `OrganizationalUnit` | OU de destino en Active Directory. Si no se especifica, el contacto se crea en el contenedor predeterminado `Users`. |
| `-HiddenFromAddressListsEnabled $true` | Oculta el contacto de la lista global de direcciones y de todas las listas de direcciones. |

</details>

A continuación, añada el contacto como remitente permitido. La tabla hash con `Add` amplía la lista existente en lugar de sobrescribirla:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | Lista de remitentes que pueden contactar con el grupo. Se admiten remitentes individuales y grupos; en el caso de los grupos, pueden enviar todos sus miembros. |
| `@{ Add = ... }` | Añade entradas sin eliminar las existentes. Con `@{ Remove = ... }` se eliminan entradas; un valor sin tabla hash sustituye toda la lista. |

</details>

En cuanto esta lista contiene al menos una entrada, Exchange rechaza **todos los demás remitentes**, incluidos los internos. Si los empleados deben seguir pudiendo escribir a la lista, añada también un grupo adecuado, por ejemplo, el propio grupo de distribución (así podrán enviar todos los miembros) o un grupo con todos los empleados. Los remitentes rechazados reciben un informe de no entrega con el estado `5.7.1`.

Puede comprobar el estado actual de esta forma:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` es un parámetro de recopilación: Exchange distribuye las entradas entre las propiedades `AcceptMessagesOnlyFrom` (remitentes individuales) y `AcceptMessagesOnlyFromDLMembers` (grupos). Por ello, la consulta muestra estas dos propiedades. La misma configuración se encuentra en Exchange Admin Center, en las propiedades del grupo, en Administración de entrega (Delivery management).

Esta variante es adecuada cuando solo se añaden unos pocos remitentes externos y la lista de remitentes permitidos debe ser visible en el propio objeto de grupo. La desventaja es el esfuerzo de mantenimiento: cada nuevo remitente necesita un contacto en Active Directory.

## Variante 2: Regla de transporte sin contactos

Si internamente todos deben poder seguir escribiendo a la lista y solo se deben restringir los remitentes externos, una regla de transporte es más sencilla. No requiere contactos y también se puede aplicar a dominios completos. `RequireSenderAuthenticationEnabled` también debe estar establecido en `$false` aquí.

```powershell
$regel = @{
    Name                              = "Projektteam: nur freigegebene externe Absender"
    SentTo                            = "projektteam@example.com"
    FromScope                         = "NotInOrganization"
    ExceptIfFromAddressMatchesPatterns = '^max\.muster@partner\.example$',
                                         '@lieferant\.example$'
    RejectMessageReasonText           = "Sie sind nicht berechtigt, an diese Adresse zu senden."
    RejectMessageEnhancedStatusCode   = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Name` | Nombre de la regla en la lista de reglas de transporte. |
| `SentTo` | Condición: el mensaje está dirigido a este grupo. |
| `FromScope NotInOrganization` | Condición: el remitente no pertenece a la propia organización. Por tanto, los mensajes internos no se ven afectados. |
| `ExceptIfFromAddressMatchesPatterns` | Excepción: direcciones de remitente que coinciden con una de las expresiones regulares. `^…$` ancla una dirección individual, `@domain$` habilita un dominio completo. Escape los puntos con `\.`. |
| `RejectMessageReasonText` | Texto del informe de no entrega para el remitente rechazado. |
| `RejectMessageEnhancedStatusCode` | Código de estado ampliado del informe de no entrega; `5.7.1` indica falta de autorización. |

</details>

Sin el anclaje con `^` y `$`, un patrón como `max\.muster@partner\.example` también coincidiría con `max.muster@partner.example.net`. Añada nuevos remitentes con `Set-TransportRule` y la lista completa y ampliada de patrones; el parámetro sustituye la lista existente.

## Límite de ambas variantes: la dirección del remitente se puede falsificar

Ambas variantes solo comprueban la dirección del remitente. Exchange no determina si el mensaje procede realmente del socio autorizado. Quien conozca la dirección de un remitente autorizado puede falsificarla y así contactar con la lista. En una lista con participantes internos, esto abre una puerta al phishing que, además, aparenta proceder de una dirección conocida.

SPF, DKIM y DMARC cierran esta brecha. Si el dominio del socio tiene una política DMARC con `p=reject` o `p=quarantine` y su puerta de enlace de correo aplica dicha política, los mensajes falsificados de ese dominio ni siquiera llegan al Exchange Server. En dominios sin política DMARC o con `p=none`, esta protección no existe.

Si su puerta de enlace escribe el resultado de la comprobación en la cabecera `Authentication-Results`, una segunda regla de transporte puede rechazar adicionalmente remitentes autorizados cuando DMARC no se haya superado:

```powershell
$regel = @{
    Name                            = "Projektteam: freigegebene Absender nur mit DMARC pass"
    SentTo                          = "projektteam@example.com"
    FromScope                       = "NotInOrganization"
    FromAddressMatchesPatterns      = '^max\.muster@partner\.example$',
                                      '@lieferant\.example$'
    ExceptIfHeaderMatchesMessageHeader = "Authentication-Results"
    ExceptIfHeaderMatchesPatterns   = 'dmarc=pass'
    RejectMessageReasonText         = "Absender konnte nicht verifiziert werden."
    RejectMessageEnhancedStatusCode = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `FromAddressMatchesPatterns` | Condición: la regla solo se aplica a los remitentes autorizados, con los mismos patrones que en la primera regla. |
| `ExceptIfHeaderMatchesMessageHeader` | Nombre de la cabecera que comprueba la excepción. |
| `ExceptIfHeaderMatchesPatterns` | Excepción: si la cabecera contiene `dmarc=pass`, se entrega el mensaje. Se rechazan todos los demás mensajes de los remitentes autorizados. |

</details>

Esta regla solo funciona de forma fiable si la puerta de enlace elimina o sobrescribe las cabeceras `Authentication-Results` entrantes desde Internet. De lo contrario, un remitente puede incluir él mismo la cabecera con `dmarc=pass`. La documentación de su puerta de enlace indica qué cabeceras establece y si elimina cabeceras externas; Exchange Server no evalúa DMARC por sí mismo.

## Qué variante elegir

La restricción de entrega con contactos es adecuada para listas con un círculo fijo y reducido de remitentes externos, y también permite restringir opcionalmente los remitentes internos. La regla de transporte es adecuada cuando todos los usuarios internos pueden enviar, se deben autorizar dominios completos de socios o no se deben crear contactos en Active Directory. Independientemente de la variante, compruebe la política DMARC de cada dominio autorizado y documente quién mantiene la lista de remitentes permitidos.

## Fuentes

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): Parámetros AcceptMessagesOnlyFromSendersOrMembers y RequireSenderAuthenticationEnabled, tipos de objetos permitidos.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): Cómo el Categorizador evalúa las restricciones de entrega y la autenticación del remitente.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): Creación de contactos de correo electrónico para direcciones externas.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): Condiciones, excepciones y acciones para reglas de transporte, incluidas FromScope y HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Especificación de las políticas DMARC p=none, quarantine y reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Estructura de la cabecera Authentication-Results e indicación de que los sistemas receptores deben eliminar instancias externas.
