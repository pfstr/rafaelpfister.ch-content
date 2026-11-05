---
title: "Buzón de voz de Teams por correo electrónico: nuevo remitente y nuevo asunto a partir de marzo de 2027 (MC1511072)"
navTitle: "Remitente del buzón de voz"
description: "Microsoft cambia la entrega de buzones de voz de Teams por correo electrónico: en adelante, el remitente será «Microsoft Teams» desde el dominio del propio tenant, y los datos de la persona que llama pasarán al asunto. El artículo muestra a quién afecta, cómo revisar reglas de transporte, listas de permitidos y automatizaciones, y qué debe adaptarse antes de marzo de 2027."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 min de lectura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "buzon-de-voz-de-teams-por-correo-electronico-nuevo-remitente-y-nuevo-asunto-a-partir-de-marzo"
translationId: "article-b75c975f086dd0d5"
translationOf: teams-voicemail-mail-absender
url: https://rafaelpfister.ch/es/blog/buzon-de-voz-de-teams-por-correo-electronico-nuevo-remitente-y-nuevo-asunto-a-partir-de-marzo
translationSourceHash: 32089440ce20d39865ec421317bed6588c48eace59eb8f114015c626da412eba
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:37:13.349Z
translationReview: automatic
---

Con el anuncio del Centro de mensajes MC1511072 del 16 de octubre de 2026, Microsoft anuncia una nueva entrega de los correos electrónicos de buzón de voz de Teams. Según el plan actual, la implementación comenzará a principios de marzo de 2027 y debería completarse a finales de junio de 2027; Microsoft se reserva el derecho de aplazarla y comunicará las fechas definitivas y las posibles acciones de los clientes en otro anuncio. El objetivo es admitir dominios de remitente propios (Bring Your Own Domain), de modo que los correos de buzón de voz se envíen conforme a las directivas SPF, DKIM y DMARC de la organización.

## Qué cambia

En adelante, el campo «De» no contendrá ni el nombre, ni el número de teléfono ni la dirección de correo electrónico de la persona que llama. Estos datos pasarán al asunto y seguirán figurando en el cuerpo del mensaje. La visualización en Teams no cambia.

| Característica | Actualmente | Tras la implementación |
|---|---|---|
| Remitente (buzón de voz personal) | Nombre o número de la persona que llama | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` o dominio propio, si está configurado |
| Dominio del remitente | `skype.voicemail.microsoft.com` en llamadas PSTN, dominio del tenant en llamadas de Teams a Teams | Dominio predeterminado del tenant o dominio propio |
| Asunto (buzón de voz personal) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` o bien `Unidentified: +19371817273 - Voicemail (00:22)` |
| Cola de llamadas (buzón de voz compartido) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Operadora automática (buzón de voz compartido) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

Para los usuarios, esto tiene dos consecuencias visibles: en la bandeja de entrada, todos los buzones de voz aparecerán con el mismo remitente, «Microsoft Teams», y, dado que la dirección es de no respuesta, la función «Responder» ya no permitirá contactar con la persona que llama.

## A quién afecta

Afecta a todas las organizaciones que utilizan Teams Calling con buzón de voz, es decir, Teams Phone con Calling Plan, Operator Connect o Direct Routing, así como colas de llamadas y operadoras automáticas con buzón de voz compartido. El anuncio no menciona limitaciones de licencia ni de nube; la implementación se ha anunciado a nivel mundial.

Es necesario actuar allí donde algo dependa del remitente o del asunto actuales:

- Reglas de transporte de Exchange que comprueban `skype.voicemail.microsoft.com`, a la persona que llama como remitente o el asunto `Voice mail` o `Shared Voicemail`
- Entradas en listas de permitidos: Tenant Allow/Block List, autorizaciones de suplantación, `AllowedSenderDomains` en directivas antispam
- Reglas de bandeja de entrada de usuarios y buzones compartidos de las colas de llamadas que clasifican o reenvían buzones de voz según el remitente o el asunto
- Flujos de Power Automate, sistemas de tickets e integraciones de CRM que obtienen el número de teléfono de la persona que llama a partir del remitente
- Puertas de enlace previas o posteriores (por ejemplo, en caso de flujo de correo centralizado en un entorno híbrido) que contienen reglas o excepciones para el dominio de buzón de voz

## Cómo comprobar si le afecta

Las siguientes consultas se ejecutan en una sesión de PowerShell de Exchange Online (`Connect-ExchangeOnline`).

### Buscar reglas de transporte

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `$muster` | Expresión regular para el dominio de remitente y los textos de asunto actuales |
| `Format-List *` | Devuelve todas las propiedades de la regla para que también se busquen las condiciones y excepciones |
| `Out-String` | Convierte la salida en texto al que se aplica `-match` |
| `Select-Object Name, State, Priority` | Muestra únicamente el nombre, el estado y el orden de las reglas encontradas |

</details>

Revise cada regla encontrada en busca de condiciones como `FromAddressContainsWords`, `SenderDomainIs` o `SubjectContainsWords`.

### Listas de permitidos y directivas antispam

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-ListType Sender` | Lee las entradas de remitentes de la Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filtra las entradas cuyo valor contiene el dominio del buzón de voz |
| `Get-TenantAllowBlockListSpoofItems` | Lista las autorizaciones y bloqueos de suplantación |
| `AllowedSenderDomains, AllowedSenders` | Autorizaciones directamente en las directivas antispam |

</details>

### Determinar el volumen real de correo

El seguimiento de mensajes muestra si los buzones de voz se entregan actualmente y a dónde:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-StartDate` / `-EndDate` | Periodo de búsqueda; Exchange Online proporciona como máximo los últimos 90 días y, por consulta, un máximo de 10 días |
| `-ResultSize 5000` | Número máximo de resultados por consulta |
| `Where-Object …` | Conserva los correos del dominio de buzón de voz PSTN o con asunto de buzón de voz |
| `Group-Object RecipientAddress` | Cuenta los buzones de voz por destinatario |

</details>

La lista de destinatarios muestra qué buzones y buzones compartidos reciben buzones de voz. Conviene revisar allí las reglas de bandeja de entrada (`Get-InboxRule -Mailbox <Adresse>`) y los flujos conectados.

## Qué hacer

1.  Cambie las reglas para basarlas en el asunto en lugar del remitente: oriente las nuevas condiciones a los futuros patrones de asunto `- Voicemail (` y `- Shared voicemail in `. Hasta que la implementación alcance su propio tenant, mantenga en paralelo la condición antigua y la nueva.

2.  Elimine tras la implementación las autorizaciones para `skype.voicemail.microsoft.com`. Los correos procedentes del dominio propio del tenant no necesitan una excepción; permitir el dominio propio supondría un riesgo de seguridad porque facilitaría la suplantación.

3.  Adapte las automatizaciones que leen el número de la persona que llama desde el remitente. En adelante, el número aparecerá entre paréntesis en el asunto o, en el caso de personas que llaman desconocidas, directamente después de `Unidentified:`.

4.  Prepare su propio dominio de remitente: Microsoft anunciará en el mensaje posterior las instrucciones de configuración para el envío desde un dominio propio. Si DMARC para ese dominio está configurado como `p=reject`, SPF y DKIM deben cubrir el envío desde Exchange Online; de lo contrario, los buzones de voz acabarán en cuarentena.

5.  Informe a los usuarios y al service desk: remitente «Microsoft Teams», datos de la persona que llama en el asunto y ya no será posible responder a la persona que llama por correo electrónico.

## Fuentes

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): texto archivado del anuncio del Centro de mensajes con el calendario y la comparación entre el remitente y el asunto actuales y futuros.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): parámetros y límites del seguimiento de mensajes en Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): lectura de la Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): aspectos básicos del buzón de voz de Teams y su entrega por correo electrónico.
