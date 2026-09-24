---
title: "Bucle de correo con pasarela de cifrado detrás de EXO: evitar problemas de suplantación"
navTitle: "Bucle de pasarela"
description: "Si la pasarela de cifrado (HIN en el ejemplo) se encuentra detrás de Exchange Online, cada mensaje entrante llega una segunda vez a EOP: con un dominio de remitente ajeno, una firma DKIM no válida y la IP de la pasarela como origen. El resultado es un veredicto de suplantación y correo no deseado. Cuatro configuraciones lo evitan sin desactivar el filtrado mediante SCL -1: registro PTR, Enhanced Filtering desactivado, excepción de suplantación para la infraestructura de la pasarela y CloudServicesMailEnabled en ambos conectores."
date: "2026-09-23"
kategorie: "Flujo de correo y SMTP"
timeToRead: "9 min de lectura"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
  - hin-gateway
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "hybrid-mailfluss"
  - "hin"
protokolle:
  - "mail-auth"
  - "smtp"
  - "powershell"
slug: "bucle-de-correo-con-pasarela-de-cifrado-detras-de-exo-evitar-problemas-de-suplantacion"
translationId: "article-b773c8d303aee87a"
aiPrompt: |
  Du bist mein Exchange-Online-Assistent. Ich betreibe ein Verschlüsselungsgateway hinter Exchange Online (Mail-Schlaufe: EXO, Gateway, EXO). Prüfe mit mir die vier Einstellungen: PTR-Record der Gateway-IP, Enhanced Filtering auf dem Inbound-Connector, Spoof-Ausnahme in der Tenant Allow/Block List für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren. Erkläre mir zu jeder Einstellung, warum sie nötig ist, und hilf mir, das Ergebnis anhand der Authentication-Results-Header einer Testnachricht zu verifizieren.
translationOf: verschluesselungsgateway-hinter-exchange-online
url: https://rafaelpfister.ch/es/blog/bucle-de-correo-con-pasarela-de-cifrado-detras-de-exo-evitar-problemas-de-suplantacion
translationSourceHash: ac3ae7b36a2682c2913479a6d9d0356d1abd96c68459bcfb4912362884e0a708
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:35:49.039Z
translationReview: automatic
---

# Bucle de correo con pasarela de cifrado detrás de EXO: evitar problemas de suplantación

Muchas organizaciones del sector sanitario suizo utilizan una pasarela HIN; otras usan SEPPmail o totemomail para S/MIME y PGP. Si el registro MX apunta a Microsoft y la pasarela está detrás de Exchange Online, cada mensaje entrante pasa dos veces por el filtrado. En la segunda pasada, EOP ve un mensaje con un dominio de remitente ajeno, una firma DKIM no válida y la IP de la pasarela como origen. Desde la perspectiva del filtro, este es el patrón de una falsificación de remitente, y el correo legítimo termina en la carpeta de correo no deseado. He configurado esta arquitectura varias veces y aquí describo las cuatro configuraciones con las que el bucle funciona sin un veredicto de suplantación, así como por qué es necesaria cada una de ellas. El ejemplo utiliza la pasarela HIN; el mecanismo es idéntico con cualquier otra pasarela en la misma posición.

## La arquitectura

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

Una regla de transporte redirige los mensajes entrantes a la pasarela mediante un conector de salida. La pasarela descifra, comprueba las firmas y entrega de nuevo el mensaje a Exchange Online mediante un conector de entrada. Un campo de encabezado establecido por la pasarela evita que la regla vuelva a aplicarse.

Esta arquitectura tiene buenas razones: Microsoft filtra primero, la pasarela recibe únicamente correo preevaluado y la operación no necesita exponer su propio MX a Internet. El precio es la segunda pasada, que no se puede desactivar, sino solo configurar correctamente.

## Por qué SCL -1 es la respuesta equivocada

La solución habitual es una regla de transporte en el trayecto de regreso que establece el Spam Confidence Level en `-1`. Con ello, Exchange Online omite por completo la comprobación de contenido en la segunda pasada. Funciona de inmediato y tiene dos inconvenientes: la segunda pasada deja de aportar nada y la regla depende de una condición (conector o IP) que cambia con cada modificación. Microsoft también describe SCL -1 como una entrada para el filtrado, no como una decisión definitiva; el valor sellado puede diferir. Las cuatro configuraciones siguientes resuelven el problema en el punto donde se origina: en la evaluación de la infraestructura de entrega.

## Las cuatro configuraciones

1. Publicar un registro PTR para la IP pública de la pasarela HIN en el DNS público. Sin esta entrada, toda la ruta no funciona.
2. Desactivar completamente Enhanced Filtering en el conector de entrada mediante el cual HIN entrega a Exchange Online.
3. Crear una excepción de suplantación para la infraestructura de entrega de HIN en la Tenant Allow/Block List.
4. Activar `CloudServicesMailEnabled` en el conector de salida hacia HIN y en el conector de entrada desde HIN.

Para el punto 2, primero compruebe qué conectores están afectados:

```powershell
Get-InboundConnector |
    Select-Object Name, Enabled, ConnectorType, EFSkipLastIP, EFSkipIPs, EFUsers, EFTestMode |
    Format-List
```

A continuación, desactívelo en el conector HIN:

```powershell
$efAus = @{
    Identity     = "<Inbound-Connector HIN>"
    EFSkipLastIP = $false
    EFSkipIPs    = $null
    EFUsers      = $null
}
Set-InboundConnector @efAus
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Parámetro | Efecto |
|---|---|
| `EFSkipLastIP = $false` | El último salto deja de omitirse automáticamente. Si al mismo tiempo no hay ninguna IP en `EFSkipIPs`, Enhanced Filtering queda desactivado en el conector. |
| `EFSkipIPs = $null` | Vacía la lista de direcciones IP que deben omitirse. |
| `EFUsers = $null` | Elimina la restricción a destinatarios concretos; sin Enhanced Filtering activo, el valor no tiene relevancia, pero así sigue siendo inequívoco. |
| `EFTestMode` | Solo en la consulta: muestra si el conector está en modo de prueba. Microsoft clasifica el parámetro como interno, pero se puede leer. |

</details>

Punto 3, la excepción de suplantación:

```powershell
$spoof = @{
    Identity              = "Default"
    Action                = "Allow"
    SpoofedUser           = "*"
    SendingInfrastructure = "gateway.example.com"
    SpoofType             = "External"
}
New-TenantAllowBlockListSpoofItems @spoof
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Parámetro | Efecto |
|---|---|
| `Identity = "Default"` | La lista propiamente dicha; solo existe esta. |
| `Action = "Allow"` | Permite la combinación. `Block` la clasifica en cambio como phishing. |
| `SpoofedUser = "*"` | La dirección visible en el campo De. El comodín representa cualquier remitente. Se permite un comodín en un lado del par, pero no en ambos. |
| `SendingInfrastructure` | El origen: el dominio del registro PTR de la IP de entrega (punto 1). Sin registro PTR, la lista solo acepta `<IP>/24`. |
| `SpoofType = "External"` | Se aplica a dominios de remitente ajenos. `Internal` cubre los dominios aceptados propios y requiere una segunda entrada. |

</details>

Punto 4, los encabezados Cross-Premises:

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

Explico más abajo, en el punto 4, por qué `TreatMessagesAsInternal` aparece en el mismo comando para el conector de entrada.

## Sobre el punto 1: registro PTR

EOP identifica la infraestructura de entrega mediante la búsqueda inversa de la IP de origen. El valor PTR aparece en el encabezado `Authentication-Results` como infraestructura de envío, y precisamente a este valor se refiere la excepción de suplantación del punto 3. Si falta el registro PTR, Exchange Online evalúa cada mensaje en el trayecto de regreso con `PTR:InfoDomainNonexistent`, y la excepción debe aplicarse a una `/24` completa en lugar de a un nombre. Además, la falta de una entrada DNS inversa es desde hace tiempo una señal negativa establecida en cualquier filtro de spam. La entrada debe resolver hacia delante de nuevo a la misma IP.

En HIN, la IP de entrega es la de la pasarela de correo HIN mediante la cual los mensajes regresan a Exchange Online. Quién gestiona el registro PTR de esta IP depende del modelo operativo; la responsabilidad debe aclararse antes del cambio.

## Sobre el punto 2: desactivar Enhanced Filtering

Enhanced Filtering for Connectors (Skip Listing) está diseñado para la arquitectura inversa: pasarela delante de Microsoft 365 y MX apuntando a la pasarela. En ese caso, EOP omite el último salto y evalúa el origen real del mensaje.

No funciona así en la configuración con bucle. Microsoft indica en la documentación que Enhanced Filtering no está pensado para servicios que procesan correo después de Microsoft 365, y enumera el enrutamiento no lineal (Internet, Microsoft 365, sistema externo, Microsoft 365) como no compatible. Como consecuencia, la documentación menciona exactamente el síntoma observado en la práctica: Microsoft 365 vuelve a comprobar el correo que regresa, asigna un valor `compauth`, y el mensaje puede clasificarse como spam.

Si Enhanced Filtering sigue activo, EOP omite la IP de HIN y evalúa la IP anterior. En el bucle, esa IP es una IP propiedad de Microsoft de la primera pasada o el remitente externo original. Ambas opciones son incorrectas:

* La evaluación se aplica a una IP que en la segunda pasada ya no permite extraer conclusiones sobre el mensaje.
* La excepción de suplantación del punto 3 nunca se aplica, porque está vinculada a la infraestructura HIN que EOP acaba de omitir.

Por eso, Enhanced Filtering debe estar desactivado en este conector: solo entonces HIN es visible y puede identificarse como infraestructura de entrega.

## Sobre el punto 3: excepción de suplantación

Una entrada de suplantación siempre es un par formado por Spoofed user (la dirección From o su dominio) y Sending infrastructure (el origen). Solo se permite esta combinación.

`SpoofedUser = "*"` junto con la infraestructura HIN significa, por tanto: cualquier dirección From puede entregar a través de HIN sin que se aplique el veredicto de suplantación. Cualquier otro origen que use los mismos remitentes se seguirá comprobando. La excepción no es una omisión del filtro de spam: las comprobaciones de spam, contenido y amenazas continúan sin cambios en la segunda pasada. Por tanto, un mensaje puede seguir descartándose por su contenido; simplemente deja de considerarse una falsificación.

El comando anterior incluye deliberadamente solo `SpoofType = "External"`. Los correos propios, es decir, mensajes con remitentes de los dominios aceptados propios, no deberían regresar a través de la pasarela. Van desde sistemas internos o desde Exchange local directamente a Exchange Online, sin pasar por HIN. La evaluación de Spoof Intelligence muestra si esto es realmente así en su entorno:

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

Si allí aparecen dominios propios con la infraestructura HIN y `SpoofType Internal`, se necesita una segunda entrada con `SpoofType = "Internal"` o, mejor aún, la causa por la que el correo interno pasa por la pasarela.

Las entradas de suplantación no caducan por sí solas. Si la pasarela deja de utilizarse o cambia su infraestructura, la entrada debe eliminarse.

## Sobre el punto 4: CloudServicesMailEnabled

Este punto es el único que Microsoft no documenta expresamente para este escenario. Es mi deducción a partir del comportamiento documentado del parámetro: conserva los encabezados Cross-Premises a lo largo del bucle.

El parámetro controla el tratamiento de los encabezados internos `X-MS-Exchange-Organization-*`. En el conector de salida se transforman en `X-MS-Exchange-CrossPremises-*` y así sobreviven al trayecto a través de HIN. En el conector de entrada se vuelven a escribir como `X-MS-Exchange-Organization-*` y, al hacerlo, sustituyen encabezados del mismo nombre que ya estén en el mensaje. Si el parámetro está establecido en `$false`, el conector elimina estos encabezados.

En la práctica, esto significa que lo que Exchange Online determinó en la primera pasada, incluido el estado de autenticación y la marca interna, sobrevive al bucle en vez de eliminarse al reingresar. En los encabezados del mensaje entregado aparece entonces `X-CrossPremisesHeadersPromoted`, y los resultados originales de las comprobaciones permanecen en `Authentication-Results-Original`. Esto por sí solo no evita un veredicto de suplantación (de ello se ocupan los puntos 1 a 3), pero conserva el veredicto de la primera pasada.

Hay que tener en cuenta tres aspectos:

* Si los conectores fueron creados por Hybrid Configuration Wizard (`ConnectorSource: HybridWizard`), una ejecución posterior de HCW sobrescribe la configuración. Por tanto, los conectores HIN deben ser conectores propios creados manualmente.
* En el conector de entrada, `CloudServicesMailEnabled $true` y `TreatMessagesAsInternal $true` se excluyen mutuamente. Si `TreatMessagesAsInternal` ya está en `$true`, Exchange Online rechaza el comando. Por ello, ambos parámetros deben incluirse en la misma llamada a `Set-InboundConnector`, como se muestra arriba.
* Microsoft recomienda establecer el parámetro únicamente siguiendo instrucciones del soporte o de la documentación del producto. Para el bucle no existe esa documentación; la decisión es suya y debe documentarse.

## Proteger la entrega

Como Exchange Online adopta los encabezados del trayecto de regreso tras el punto 4 y acepta cualquier dirección From de la infraestructura HIN tras el punto 3, el conector de entrada debe restringirse de modo que solo HIN pueda entregar a través de él. En un conector de tipo `Partner`, esto se consigue con `RestrictDomainsToCertificate` junto con `TlsSenderCertificateName` o con `RestrictDomainsToIPAddresses` junto con `SenderIPAddresses`, en cada caso junto con `RequireTls`. Sin esta restricción, cualquier origen que alcance el conector podría entregar usando remitentes arbitrarios y encabezados de organización adoptados.

```powershell
$absicherung = @{
    Identity                     = "<Inbound-Connector HIN>"
    RequireTls                   = $true
    RestrictDomainsToCertificate = $true
    TlsSenderCertificateName     = "gateway.example.com"
}
Set-InboundConnector @absicherung
```

## Comprobación

Tras el cambio, envíe un mensaje de prueba desde un dominio externo con una política DMARC a través de la pasarela y compruebe los encabezados del mensaje entregado. En el encabezado `Authentication-Results` de la segunda pasada, el dominio HIN debería figurar como infraestructura de envío y `compauth` debería estar en `pass` con una razón del ámbito de las entradas Tenant Allow, ya no en `fail reason=001`. `X-CrossPremisesHeadersPromoted` y `Authentication-Results-Original` muestran que el punto 4 funciona. El [analizador de encabezados de correo](/tools/header-analyzer) de este sitio muestra ambas pasadas como un diagrama de flujo y marca los saltos entre Exchange Online y la pasarela.

## Fuentes

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): delimitación entre enrutamiento lineal y no lineal, indicación sobre servicios detrás de Microsoft 365, consecuencia `compauth` y clasificación como spam, SCL -1 como entrada en vez de decisión, parámetros de PowerShell `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): comportamiento de `CloudServicesMailEnabled` (transformación y promoción de los encabezados Cross-Premises, eliminación con `$false`), exclusión con `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` y `RestrictDomainsToIPAddresses` para conectores de socios.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` en el conector de salida.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): sintaxis de las entradas de suplantación, reglas de comodines, determinación de la infraestructura de envío mediante registro PTR o `/24`, cobertura de veredictos de suplantación causados por DMARC.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): parámetros `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): evaluación de los pares de suplantación detectados durante los últimos siete días.
