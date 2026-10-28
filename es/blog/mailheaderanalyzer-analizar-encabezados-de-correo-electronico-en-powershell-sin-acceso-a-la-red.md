---
title: "MailHeaderAnalyzer: analizar encabezados de correo electrónico en PowerShell sin acceso a la red"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referencia del módulo de PowerShell MailHeaderAnalyzer: resumen de parámetros, sintaxis, descripción, ejemplos y propiedades de parámetros de Get-MailHeaderAnalysis y ConvertTo-MailHeaderReport, además del objeto de salida con cadena de entrega, resultados de autenticación, clasificación de Exchange Online y todos los códigos de hallazgos."
date: "2026-09-24"
kategorie: "SMTP y flujo de correo"
timeToRead: "16 min de lectura"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "powershell"
  - "mail-auth"
  - "smtp"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
slug: "mailheaderanalyzer-analizar-encabezados-de-correo-electronico-en-powershell-sin-acceso-a-la-red"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
translationSourceHash: 7f1ed02181dcd5665641de26ab47fc5acaeaa73a230c60093063ab8bb99e15d3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:32:31.339Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/mailheaderanalyzer-analizar-encabezados-de-correo-electronico-en-powershell-sin-acceso-a-la-red
---

# MailHeaderAnalyzer: analizar encabezados de correo electrónico en PowerShell sin acceso a la red

MailHeaderAnalyzer es un módulo de PowerShell con dos cmdlets. `Get-MailHeaderAnalysis` analiza el encabezado de un correo electrónico: cadena de entrega con retrasos y datos de TLS, resultados de SPF, DKIM, DMARC y ARC, incluida la comprobación de origen frente al authserv-id de su gateway, alineación de DMARC, clasificación híbrida de Exchange Online, evaluaciones de Microsoft Defender, SpamAssassin y Rspamd, así como anomalías como líneas `From` duplicadas o caracteres de control Unicode. `ConvertTo-MailHeaderReport` genera a partir de ello un informe para tickets. El módulo funciona completamente sin conexión: sin consultas DNS ni conexiones HTTP. Es la versión de línea de comandos del [analizador de encabezados de este sitio web](/tools/header-analyzer) y utiliza la misma lógica de análisis.

| | |
|---|---|
| Módulo | [MailHeaderAnalyzer en la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Código fuente | [pfstr/MailHeaderAnalyzer en GitHub](https://github.com/pfstr/MailHeaderAnalyzer), licencia MIT |
| Se aplica a | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlets | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Resumen de parámetros

| Cmdlet | Parámetro | Tipo | Obligatorio | Canalización | Efecto |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Sí (conjunto de parámetros Text) | Sí, por valor | El encabezado como texto. Las líneas de la canalización se ensamblan en un encabezado |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Sí (conjunto de parámetros Path) | Sí, por nombre de propiedad | Archivo con el encabezado o el mensaje `.eml` completo; acepta objetos de `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Sí (conjunto de parámetros Clipboard) | No | Lee el encabezado del portapapeles (solo Windows) |
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | No | No | authserv-id(s) de su gateway de entrada; solo se consideran acreditadas las líneas de comprobación con uno de estos ID |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Sí | Sí, por valor | El objeto de resultado de `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | No | No | `Markdown` (predeterminado) o `Text` |

Ambos cmdlets admiten los parámetros comunes `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` y los demás de [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Instalación

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Opción | Efecto |
|---|---|
| `-Name MailHeaderAnalyzer` | Nombre del módulo en la PowerShell Gallery |
| `-Scope CurrentUser` | Instala en el directorio de módulos del usuario, sin derechos de administrador |

En un sistema sin acceso a Internet, descargue el módulo en otro equipo con `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` y copie la carpeta `MailHeaderAnalyzer` en un directorio de `$env:PSModulePath`, por ejemplo `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) o `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). `Update-Module -Name MailHeaderAnalyzer` obtiene actualizaciones y `Get-Module -Name MailHeaderAnalyzer -ListAvailable` muestra la versión instalada.

## Get-MailHeaderAnalysis

Analiza el encabezado de un correo electrónico y devuelve un objeto de análisis.

### Sintaxis

#### Text (predeterminado)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

### Descripción

El cmdlet descompone el encabezado sin procesar en campos, despliega los plegados de RFC 5322 y decodifica los valores RFC 2047 en asunto y direcciones. A partir de las líneas `Received` forma la cadena de entrega en orden cronológico, calcula el retraso por estación y lee la versión TLS, el cifrado y la clase de protocolo conforme a RFC 3848. A partir de `Authentication-Results`, `Received-SPF`, `DKIM-Signature` y la cadena ARC, determina los resultados de autenticación y asigna un origen a cada línea de comprobación (RFC 8601, sección 5): acreditada si su authserv-id figura en `-TrustedAuthServId`, de lo contrario solo plausible o no acreditada; consulte [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). También añade la alineación DMARC, la clasificación híbrida de Exchange Online, las evaluaciones de los filtros antispam y una lista de anomalías.

El cmdlet no realiza consultas DNS ni abre conexiones de red. `Spf`, `Dkim`, `Dmarc` y `Arc` son, por tanto, siempre el veredicto del servidor receptor. Las firmas DKIM y ARC no se recalculan criptográficamente; para la cadena ARC, el cmdlet solo comprueba la estructura (`ArcStructure`).

Las entradas se leen de forma tolerante: una línea en blanco termina el encabezado y se ignora el texto del mensaje posterior. Las líneas sin nombre de campo y sin espacio inicial, como las que se originan al copiar desde diálogos de clientes, pertenecen al campo anterior. Se omite una línea separadora mbox `From ...` antes del primer campo y se elimina una marca de orden de bytes. Se analizan como máximo 200 líneas `Received`, contadas desde la entrega.

### Ejemplos

#### Ejemplo 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analiza el encabezado que se encuentra en el portapapeles. En Outlook para Windows encontrará el encabezado en Archivo, Propiedades, Encabezados de Internet; en Outlook en la Web, en las opciones del mensaje, en «Ver detalles del mensaje».

```text
Subject     : Service-Report für März
From        : Beispiel Newsletter <news@example.org>
Date        : 2026-08-03 09:14:27Z
SPF         : pass
DKIM        : pass
DMARC       : pass
ARC         : -
CompAuth    : pass (reason 100)
AuthTrust   : Absent (no authserv-id, Microsoft 365 style)
Hops        : 3 (total 00:00:42)
DeliveredBy : zr0p278mb0570.chep278.prod.outlook.com
Findings    : none
```

#### Ejemplo 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Analiza un mensaje guardado. El archivo puede contener solo el encabezado o el mensaje completo; el cuerpo del mensaje se ignora.

#### Ejemplo 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Muestra la cadena de entrega como tabla, con el primer salto primero.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Ejemplo 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Lee el encabezado línea por línea desde un archivo de texto y muestra solo las anomalías. `-Raw` en `Get-Content` no es necesario, pues el cmdlet ensambla las líneas por sí mismo.

#### Ejemplo 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analiza todos los mensajes de una carpeta y escribe un archivo CSV con una línea por mensaje. La columna calculada resume los códigos de hallazgos.

| Opción | Efecto |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Proporciona los objetos de archivo; `-Path` toma su propiedad `FullName` |
| `Select-Object … @{ Name; Expression }` | Columna calculada que reúne todos los códigos de hallazgos en una cadena |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV sin cabecera de tipo, UTF-8 para caracteres acentuados en las líneas de asunto |

#### Ejemplo 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Lista los mensajes cuyos resultados de comprobación no proceden del gateway de entrada propio `mx.example.org`. En análisis de phishing, constituye un primer filtro. Sin `-TrustedAuthServId` solo se puede filtrar por `Unmatched`; una falsificación que incluya además una línea `Received` adecuada aparecerá como `Matched` y no será detectada por el filtro.

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id que su gateway escribe en `Authentication-Results`; comparación exacta, sin subdominios |
| `Where-Object AuthTrust -ne 'Trusted'` | Conserva todos los mensajes cuya línea de comprobación determinante no lleva una authserv-id de confianza |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | Archivo, nivel de origen, authserv-id de la línea de comprobación y estación de entrega |

</details>

#### Ejemplo 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Muestra la estación con el mayor retraso. `SlowestHopIndex` está basado en 1, mientras que la matriz `Hops` está basada en 0.

#### Ejemplo 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Escribe el análisis completo como JSON. `-Depth 6` es necesario porque `Hops`, `DkimSignatures` y `Exchange` son objetos anidados; el valor predeterminado 2 solo los mostraría como nombres de tipo.

### Parámetros

#### -Header

El encabezado como texto. El parámetro acepta una sola cadena con todo el encabezado o varias cadenas; las entradas de la canalización se recopilan y se ensamblan en un encabezado al final, por lo que `Get-Content datei | Get-MailHeaderAnalysis` funciona sin `-Raw`. Si desea analizar varios encabezados por separado, utilice `-Path` con varios archivos.

Alias: `Text`, `Raw`, `InputObject`

| Propiedad del parámetro | Valor |
|---|---|
| Tipo | `String[]` |
| Valor predeterminado | Ninguno |
| Admite comodines | No |

| Conjunto de parámetros Text | Valor |
|---|---|
| Posición | 0 |
| Obligatorio | Sí |
| Valor desde la canalización | Sí |
| Valor desde la canalización por nombre de propiedad | No |

#### -Path

Ruta a un archivo que contiene el encabezado o un mensaje `.eml` completo. Las rutas relativas se resuelven respecto al directorio actual. El archivo se lee con `[System.IO.File]::ReadAllText`: se respeta una marca de orden de bytes; sin BOM se usa UTF-8. Se crea un objeto de resultado independiente para cada archivo; los archivos inexistentes generan un error no terminante.

Alias: `FullName`, `PSPath`, `LiteralPath`

| Propiedad del parámetro | Valor |
|---|---|
| Tipo | `String[]` |
| Valor predeterminado | Ninguno |
| Admite comodines | No |

| Conjunto de parámetros Path | Valor |
|---|---|
| Posición | Con nombre |
| Obligatorio | Sí |
| Valor desde la canalización | No |
| Valor desde la canalización por nombre de propiedad | Sí |

#### -FromClipboard

Lee el encabezado con `Get-Clipboard -Raw` desde el portapapeles. El parámetro solo está disponible en Windows; en Linux y macOS el cmdlet termina con un mensaje de error, al igual que si el portapapeles está vacío.

| Propiedad del parámetro | Valor |
|---|---|
| Tipo | `SwitchParameter` |
| Valor predeterminado | `False` |
| Admite comodines | No |

| Conjunto de parámetros Clipboard | Valor |
|---|---|
| Posición | Con nombre |
| Obligatorio | Sí |
| Valor desde la canalización | No |
| Valor desde la canalización por nombre de propiedad | No |

#### -TrustedAuthServId

El authserv-id o los authserv-id que su gateway de entrada escribe en `Authentication-Results`, por ejemplo `mx.example.org`. El cmdlet compara de forma exacta, sin distinguir mayúsculas de minúsculas y sin subdominios. Las líneas de comprobación con uno de estos ID reciben `AuthTrust = Trusted`, y solo ellas se incorporan a `Spf`, `Dkim`, `Dmarc` y `Arc`. Lo mismo se aplica al `receiver=` de una línea `Received-SPF`.

El resultado solo es tan fiable como el gateway: debe eliminar las líneas `Authentication-Results` entrantes que reclamen su propio authserv-id (RFC 8601, sección 5). No se puede determinar a partir de un encabezado si lo hace. Si dos líneas con ID de confianza se contradicen, el cmdlet informa `AuthTrustedConflict`. Microsoft 365 no escribe un authserv-id en su línea de comprobación; si Exchange Online recibe su correo, omita el parámetro.

Para toda una sesión, el valor puede configurarse como predeterminado, por ejemplo en el perfil de PowerShell:

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Propiedad del parámetro | Valor |
|---|---|
| Tipo | `String[]` |
| Valor predeterminado | Ninguno |
| Admite comodines | No |

| Conjunto de parámetros (todos) | Valor |
|---|---|
| Posición | Con nombre |
| Obligatorio | No |
| Valor desde la canalización | No |
| Valor desde la canalización por nombre de propiedad | No |

### Entradas

`System.String`: líneas de encabezado o el encabezado completo, para `-Header`.

`System.IO.FileInfo`: objetos de archivo de `Get-ChildItem`, cuya propiedad `FullName` se vincula a `-Path`.

### Salidas

`MailHeaderAnalyzer.Analysis`: un objeto por cada encabezado analizado. Las propiedades se describen en la sección [Objeto de salida](#ausgabeobjekt).

### Notas

El texto del análisis (explicaciones en `Findings`, `CompAuthReasonMeaning` y los campos de significado) está en inglés para que pueda incluirse sin cambios en tickets internacionales. La salida de la vista predeterminada puede mostrarse por completo con `Format-List *`.

## ConvertTo-MailHeaderReport

Genera un informe en Markdown o texto a partir de un objeto de análisis.

### Sintaxis

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Descripción

El informe incluye asunto, remitente, fecha e ID de mensaje, los resultados de autenticación con indicación de origen y alineación, todos los hallazgos, la cadena de entrega, la clasificación de Exchange y los valores de los filtros antispam. Los caracteres de control Unicode de dirección de escritura siguen siendo visibles en el informe como `<U+...>`, para que no lleguen a un sistema de tickets a través del informe. La última línea indica la versión del módulo.

### Ejemplos

#### Ejemplo 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Genera un informe Markdown y lo coloca en el portapapeles.

```markdown
# Email header analysis

- Subject: Service-Report für März
- From: Beispiel Newsletter <news@example.org>
- Date: 2026-08-03T09:14:27Z
- Message-ID: <20260803091427.4Wq2Wx5RbTz@mail.example.org>

## Authentication

- spf: pass
- dkim: pass
- dmarc: pass
- compauth: pass (reason=100: passed authentication)
- Results without authserv-id (Microsoft 365 style)
- DMARC alignment: SPF Strict, DKIM Strict

## Delivery chain (total: 42 s)

| # | From | By | Protocol | TLS | Time (UTC) | Delay |
|---|---|---|---|---|---|---|
| 1 | client.example.net (198.51.100.34) | mail.example.org | ESMTPSA | - | 2026-08-03T09:14:28Z | - |
| 2 | mail.example.org (203.0.113.25) | mx.eur02.prod.protection.outlook.com | Microsoft SMTP Server | TLS 1.3 | 2026-08-03T09:15:09Z | 41 s |
| 3 | AM0EUR02FT056.eop-EUR02.prod.protection.outlook.com | ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM | Microsoft SMTP Server | TLS 1.2 | 2026-08-03T09:15:10Z | 1 s |
```

#### Ejemplo 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Emite el informe como texto sin marcado Markdown, por ejemplo para correos electrónicos o registros de consola.

#### Ejemplo 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Uso en Exchange Management Shell. Si `$env:PSModulePath` está restringido allí por directivas de grupo, cargue el módulo con `Import-Module` y la ruta completa al archivo `.psd1`.

### Parámetros

#### -Analysis

El objeto de análisis de `Get-MailHeaderAnalysis`. El cmdlet rechaza otros tipos de objeto con un error de enlace.

| Propiedad del parámetro | Valor |
|---|---|
| Tipo | `MailHeaderAnalyzer.Analysis` |
| Valor predeterminado | Ninguno |
| Admite comodines | No |

| Conjunto de parámetros (todos) | Valor |
|---|---|
| Posición | 0 |
| Obligatorio | Sí |
| Valor desde la canalización | Sí |
| Valor desde la canalización por nombre de propiedad | No |

#### -Format

El formato de salida. Valores válidos:

- `Markdown`: encabezados, listas y la cadena de entrega como tabla. Predeterminado.
- `Text`: encabezados en mayúsculas, líneas con sangría y cadena de entrega como lista numerada.

| Propiedad del parámetro | Valor |
|---|---|
| Tipo | `String` |
| Valores permitidos | `Markdown`, `Text` |
| Valor predeterminado | `Markdown` |
| Admite comodines | No |

| Conjunto de parámetros (todos) | Valor |
|---|---|
| Posición | Con nombre |
| Obligatorio | No |
| Valor desde la canalización | No |
| Valor desde la canalización por nombre de propiedad | No |

### Entradas

`MailHeaderAnalyzer.Analysis`: el resultado de `Get-MailHeaderAnalysis`.

### Salidas

`System.String`: el informe, una cadena por objeto de análisis.

## Objeto de salida

`Get-MailHeaderAnalysis` devuelve un objeto de tipo `MailHeaderAnalyzer.Analysis` por cada entrada. La vista predeterminada muestra el resumen del ejemplo 1; se puede acceder a todas las propiedades mediante `Select-Object`, `Format-List *` o `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Propiedad | Tipo | Contenido |
|---|---|---|
| `Source` | String | Ruta de archivo, `Clipboard` o `Text` |
| `Subject` | String | Asunto, decodificado según RFC 2047 |
| `From`, `ReplyTo`, `ReturnPath` | Objeto de dirección | `Name`, `Address`, `Domain`, `Display`; `$null` si falta el campo |
| `Date` | DateTime (UTC) | Valor del campo `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Dominio del remitente del sobre a partir de `smtp.mailfrom` de la comprobación SPF; de lo contrario, a partir de `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Resultado según la línea `Authentication-Results` determinante (`pass`, `fail`, `none`, `softfail` y otros); `$null` si no se comprobó. `Arc` es el veredicto del receptor sobre la cadena ARC |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Código de motivo de la autenticación compuesta de Microsoft 365 y su significado |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` o `None`; consulte [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id de la línea de comprobación determinante |
| `AuthenticationResults` | Objeto[] | Todas las líneas `Authentication-Results` con `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objeto | La línea `Received-SPF` con `Result` y `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` o `$null` |
| `Hops` | Hop[] | Cadena de entrega en orden cronológico; consulte [Objeto Hop](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Métricas de la cadena |
| `DeliveredBy` | String | Host `by` de la línea `Received` más reciente, es decir, la estación de entrega |
| `DkimSignatures` | Objeto[] | Por firma: `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Objeto[] | Instancias ARC con `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | Estructura de la cadena ARC: `Consistent`, `Inconsistent` o `$null` sin encabezados ARC; además, las desviaciones detectadas. Comprobación meramente estructural, sin comprobación de firma; consulte [Cadena ARC](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Objeto | Clasificación híbrida de Exchange Online; consulte [Objeto Exchange](#mailheaderanalyzerexchangeclassification); `$null` sin encabezados correspondientes |
| `Spam` | Objeto | Evaluaciones de filtros antispam; consulte [Objeto Spam](#mailheaderanalyzerspamassessment); `$null` sin encabezados correspondientes |
| `List` | Objeto | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` sin encabezados de lista |
| `Findings` | Finding[] | Anomalías con `Severity`, `Code`, `Message`; consulte [Hallazgos](#findings) |
| `Fields` | Objeto[] | Todos los campos con `Name`, `Value` (desplegado) y `Raw` |
| `HadBody` | Bool | Si hubo un cuerpo de mensaje después del encabezado |

### MailHeaderAnalyzer.Hop

Cada entrada de `Hops` corresponde a una línea `Received`. El orden es cronológico, es decir, inverso al orden del encabezado.

| Propiedad | Contenido |
|---|---|
| `Index` | Número consecutivo; 1 = envío inicial |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Datos del sistema de envío de la parte `from`; la IP procede de los corchetes del comentario, y el nombre rDNS del comentario precedente |
| `ByHost`, `Software` | Sistema receptor y su software (comentario tras `by`) |
| `Protocol`, `ProtocolClass` | Valor `with` y clase según RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS o indicación TLS presente), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | De las notaciones de Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) y Exim |
| `Id`, `For`, `Via` | Otros componentes de `Received` |
| `Date` | Marca de tiempo tras el punto y coma, UTC |
| `Delay` | TimeSpan respecto al salto anterior; negativo en caso de desfase de relojes |
| `Provider` | Proveedor o gateway detectado por los nombres de host (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast y otros) |
| `Attested` | `$true` solo en el último salto: únicamente esta línea la escribió el propio sistema receptor; todas las inferiores ya estaban en el mensaje |
| `Raw` | La línea original |

### AuthTrust: origen de los resultados de comprobación

Cualquier remitente puede escribir una línea `Authentication-Results` en un mensaje, así como una línea `Received` adecuada. Por ello, no se puede demostrar únicamente a partir del encabezado quién escribió una línea de comprobación. Conforme a RFC 8601, sección 5, solo es determinante la línea cuyo authserv-id la organización receptora reconoce como propio, y el gateway de entrada debe eliminar las líneas entrantes con este ID. El cmdlet aplica esta regla con `-TrustedAuthServId`. Sin este parámetro, solo compara el authserv-id con los hosts `by` de la cadena `Received` (mismo dominio o subdominio, siempre en el límite del punto, sin subcadena); se trata de una comprobación de plausibilidad.

| Valor | Significado |
|---|---|
| `Trusted` | El authserv-id figura en `-TrustedAuthServId`. En cuanto exista una línea de este tipo, solo las líneas de este nivel se incorporan a `Spf`, `Dkim`, `Dmarc` y `Arc`. Fiable, siempre que el gateway elimine las líneas ajenas con este ID |
| `Matched` | El authserv-id aparece como host `by` en la cadena. Plausible, pero no acreditado: una falsificación puede incluir la línea `Received` adecuada. Sin `-TrustedAuthServId`, el hallazgo `AuthPlausibleOnly` lo indica. Las líneas de este nivel cuentan si no hay ninguna línea `Trusted` |
| `Unmatched` | El authserv-id no es de confianza ni aparece en la cadena. Los resultados se muestran, pero se consideran una afirmación no acreditada; el hallazgo `AuthUnverified` lo indica |
| `Absent` | La línea no lleva authserv-id. Microsoft 365 escribe su línea de comprobación de esta forma: comienza directamente con `spf=` |
| `None` | No hay línea de comprobación |

Si el encabezado contiene líneas de comprobación de varios orígenes, el hallazgo `AuthMixedOrigins` informa de ello. Si falta una línea de comprobación, pero hay una línea `Received-SPF`, se adopta su resultado como `Spf` y se identifica con `ReceivedSpfOnly`.

Con `-TrustedAuthServId` se añaden dos hallazgos: `AuthNotTrusted` si ninguna línea de comprobación lleva un ID de confianza, y `AuthTrustedConflict` si dos de esas líneas informan resultados diferentes para `spf`, `dmarc`, `arc` o `compauth`. El segundo caso implica que al menos una línea no procede del gateway y que este no la eliminó. En ese caso, el cmdlet adopta la línea superior; no se puede decidir a partir del encabezado cuál de las dos es auténtica. DKIM queda excluido de esta comparación, pues varias firmas pueden tener legítimamente resultados distintos.

### Cadena ARC: estructura y veredicto

Para ARC, el cmdlet proporciona dos datos independientes. `Arc` es el resultado `arc=` de la línea de comprobación determinante, es decir, el veredicto del receptor que comprobó las firmas de la cadena. `ArcStructure` es la comprobación propia del módulo y solo afecta a la estructura conforme a RFC 8617: números de instancia consecutivos de `i=1` a `i=n` (máximo 50), exactamente un `ARC-Seal`, un `ARC-Message-Signature` y un `ARC-Authentication-Results` por instancia, además de `cv=none` en la instancia 1 y `cv=pass` en todas las posteriores. El módulo no recalcula firmas; por tanto, `Consistent` no indica si la cadena es auténtica. Las desviaciones figuran en `ArcStructureIssues` y en el hallazgo `ArcStructureInconsistent`.

Los datos de `ARC-Authentication-Results` son afirmaciones del respectivo reenviador. Por ello, el hallazgo `DkimBrokenAfterForward` clasifica un error DKIM como consecuencia de un reenvío solo si el propio receptor informa `arc=pass` y una instancia anterior registró un DKIM-`pass` para el mismo dominio.

### Alineación DMARC

`SpfAlignment` compara el dominio del remitente del sobre con el dominio `From`; `DkimAlignment` compara el dominio `d=` de la firma comprobada con el dominio `From`. `Strict` significa dominio idéntico; `Relaxed`, el mismo dominio organizativo; y `None`, ninguna coincidencia. El dominio organizativo se determina heurísticamente: las dos últimas etiquetas o, en finales compuestos conocidos como `co.uk` o `com.au`, las tres últimas. No se incluye una Public Suffix List completa.

### MailHeaderAnalyzer.ExchangeClassification

Si el encabezado contiene los campos `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` o `X-MS-Exchange-CrossTenant-*`, el cmdlet rellena la propiedad `Exchange`. Los significados siguen el artículo «Demystifying hybrid mail flow» del equipo de Exchange; la información de fondo se encuentra en el artículo [Encabezados híbridos de Exchange: ¿interno o externo?](/blog/exchange-hybrid-header-intern-extern).

| Propiedad | Contenido |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` o `Incoming` con explicación |
| `AuthAs`, `AuthAsMeaning` | `Internal` o `Anonymous` con las consecuencias para el filtrado EOP |
| `AuthSource` | Servidor que realizó la clasificación |
| `AuthMechanism`, `AuthMechanismMeaning` | Código de mecanismo. Solo el valor 10 (Externally Secured) está documentado públicamente; para todos los demás códigos, el módulo declara expresamente que Microsoft no los documenta |
| `OriginatorOrg` | Dominio predeterminado del tenant emisor, el atributo de tenant no falsificable al recibir desde Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Clasificación e ID de tenant en el límite del tenant |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` o `HybridOnPrem` con explicación |
| `WrongTenantAttribution` | Valor de `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` si el mensaje se asignó a un tenant ajeno |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marcadores de encabezados de organización recibidos o eliminados por el conector de envío |

### MailHeaderAnalyzer.SpamAssessment

La propiedad `Spam` resume las evaluaciones de los filtros conocidos. Los valores proceden de sistemas externos y se decodifican, pero no se evalúan.

| Propiedad | Fuente | Contenido |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level con significado (-1 de confianza, 0/1 no spam, 5/6 sospecha de spam, 9 muy probablemente spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level de 0 a 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Clasificación como `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Resultado del filtro como `NSPM`, `SPM`, `SKA` (lista de permitidos), `SKI` (intraorganizacional) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP en la lista de permitidos de conexión) o `NLI` (sin reputación) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | IP de envío y país de origen |
| `Forefront` | `X-Forefront-Antispam-Report` | Todos los pares clave-valor del campo |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Puntuación y pruebas activadas |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Símbolos con puntuación |

La lista completa de códigos de motivo `compauth` se encuentra en el artículo [Microsoft 365 compauth: códigos de motivo](/blog/microsoft-365-compauth-reason-codes).

### Hallazgos

El cmdlet proporciona anomalías como objetos en `Findings`, cada uno con `Severity` (`Info`, `Warning`, `Fail`), un `Code` estable para filtros y scripts, y una explicación en `Message`.

| Código | Gravedad | Significado |
|---|---|---|
| `DuplicateField` | Warning | Un campo que RFC 5322 limita a una instancia (`From`, `Subject`, `Date`, `Message-ID` y otros) aparece varias veces. Los clientes de correo y filtros pueden elegir instancias distintas; es un patrón conocido en falsificaciones |
| `BidiControls` | Warning | Caracteres de control Unicode de dirección de escritura en un campo. Invierten la dirección de lectura: `fdp.exe` aparece entonces como `exe.pdf`. El módulo los muestra como `<U+202E>` |
| `HopOverflow` | Warning | Más de 200 líneas `Received`; las excedentes no se analizaron |
| `AuthPlausibleOnly` | Info | El authserv-id aparece en la cadena de entrega, pero no se indicó `-TrustedAuthServId`: plausible, no acreditado |
| `AuthNotTrusted` | Warning | Se indicó `-TrustedAuthServId`, pero ninguna línea de comprobación lleva uno de estos ID |
| `AuthTrustedConflict` | Warning | Dos líneas de comprobación con ID de confianza informan resultados distintos para el mismo método; aparentemente, el gateway no elimina las líneas ajenas |
| `AuthUnverified` | Warning | Los resultados de comprobación llevan un authserv-id que no es de confianza ni aparece en la cadena de entrega |
| `AuthMixedOrigins` | Warning | Hay líneas de comprobación de varios orígenes |
| `ReceivedSpfForeign` | Warning | El `receiver=` de la línea `Received-SPF` no aparece en la cadena |
| `ReceivedSpfOnly` | Info | El resultado SPF procede solo de `Received-SPF`, no de una línea de comprobación |
| `NoAuthResults` | Info | No hay resultados de comprobación en el encabezado |
| `DmarcFail` | Fail | DMARC no superado según el servidor receptor |
| `SpfNotPass` | Warning | Resultado SPF `fail`, `softfail`, `permerror` o `temperror` |
| `DkimNotPass` | Warning | Resultado DKIM `fail`, `permerror` o `temperror`, sin que el receptor confirme la cadena ARC |
| `DkimBrokenAfterForward` | Info | DKIM no superado en el receptor, pero el receptor informa `arc=pass`, y una instancia ARC anterior registró un DKIM-`pass` para el mismo dominio: típico de reenvíos y listas de correo |
| `DkimWeakHash` | Warning | Firma con `rsa-sha1` (RFC 8301 clasifica SHA-1 como obsoleto) |
| `DkimBodyLength` | Warning | La etiqueta `l=` limita la longitud firmada del cuerpo del mensaje; el contenido añadido no está cubierto |
| `DkimExpired` | Warning | El momento `x=` está en el pasado |
| `DkimFromUnsigned` | Warning | El campo `From` no está incluido en `h=`, aunque RFC 6376 lo exige |
| `ArcStructureInconsistent` | Warning | Los encabezados ARC no forman una cadena formalmente completa (lagunas, encabezados ausentes o duplicados, secuencia `cv=` incorrecta); comprobación meramente estructural |
| `ClockSkew` | Info | Un salto tiene una marca de tiempo anterior a la de su predecesor; los retrasos son solo valores aproximados |
| `ReplyToMismatch` | Info | El dominio `Reply-To` difiere del dominio `From`; habitual en boletines, un patrón en phishing |
| `SpfNotAligned` | Info | El dominio del remitente del sobre y el dominio `From` pertenecen a organizaciones distintas; entonces SPF no contribuye a DMARC |
| `ExchangeWrongTenant` | Warning | El mensaje se asignó a un tenant ajeno; una causa clásica es un conector entrante de otro tenant con el mismo certificado o las mismas direcciones IP |
| `ExchangeHeadersFiltered` | Warning | El conector de envío eliminó los encabezados entre instalaciones (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: entrada a través de un conector de recepción con «Externally Secured», filtrado EOP omitido |
| `SpamCategory` | Warning | Microsoft asignó una categoría distinta de `NONE` |
| `SpamConfidence` | Warning | SCL 5 o superior |

## Funcionamiento y límites

El módulo lee lo que figura en el encabezado y deduce de ello lo que puede acreditarse sin consultas externas. De ello se derivan algunos límites:

- **Sin comprobación criptográfica.** Las firmas DKIM y ARC no se recalculan y no se consultan registros DNS. `Spf`, `Dkim`, `Dmarc` y `Arc` son siempre el veredicto del servidor receptor; `ArcStructure` solo comprueba la estructura de la cadena.
- **El origen solo puede acreditarse con conocimiento del gateway.** Sin `-TrustedAuthServId`, `AuthTrust` es una comprobación de plausibilidad. Con el parámetro, la conclusión depende de que el gateway elimine líneas de comprobación ajenas con su authserv-id; el módulo no puede verificarlo.
- **Solo se acredita la última línea `Received`.** El remitente ya proporcionó todas las líneas inferiores y puede haberlas diseñado libremente. `Attested` marca esta diferencia; los retrasos de saltos anteriores se basan en los datos de dichas líneas.
- **Dominios organizativos heurísticos.** Para la alineación relajada, el módulo usa una lista breve de finales compuestos, no una Public Suffix List completa.
- **AuthMechanism solo está parcialmente documentado.** Salvo el valor 10, Microsoft no ha publicado los códigos; el módulo no inventa significados.
- **Conjuntos de caracteres.** Los valores RFC 2047 se decodifican con las codificaciones que .NET reconoce en el sistema respectivo. Los conjuntos de caracteres desconocidos permanecen sin modificar.

## Privacidad

Un encabezado completo contiene nombres de host internos, direcciones IP, remitentes, destinatarios y asuntos. El motivo por el que estos datos no pertenecen a una herramienta en línea se explica en el artículo [Analizar encabezados de correo sin subir el mensaje](/blog/e-mail-header-analysieren-ohne-upload). Para el módulo se aplica la misma promesa que para la versión de navegador: sin accesos a la red. La suite de pruebas del módulo contiene una prueba que falla en cuanto aparece un cmdlet de red o DNS en el código fuente.

## Código fuente y versiones

El código fuente está bajo licencia MIT en [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). La lógica de análisis es una adaptación de la biblioteca que también utiliza el analizador de encabezados de este sitio web; ambos comparten los casos de prueba. La integración continua comprueba cada cambio con PSScriptAnalyzer y Pester en Windows PowerShell 5.1, PowerShell 7 en Windows, Ubuntu y macOS. Las publicaciones en la PowerShell Gallery se realizan automáticamente a partir de etiquetas versionadas; los cambios de cada versión figuran en el [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

La versión 0.2.0 del 26 de septiembre de 2026 reforzó el modelo de origen tras una observación de revisión de @saltyslugga: parámetro `-TrustedAuthServId`, nivel `Trusted`, `Matched` solo como plausibilidad. La propiedad `ArcValid` se eliminó y se sustituyó por `ArcStructure` y `ArcStructureIssues`; los scripts que evalúan `ArcValid` deben adaptarse.

Acepto errores y solicitudes de mejora como [issue en GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonimice antes los encabezados de prueba para informes de errores; los casos de prueba incluidos usan exclusivamente dominios de ejemplo conforme a RFC 2606 y direcciones conforme a RFC 5737.

## Historial de cambios

- **25 de octubre de 2026:** Se sustituyó la fuente RFC 7489 por RFC 9989, que reemplaza la especificación DMARC desde mayo de 2026.

## Fuentes

1.  [MailHeaderAnalyzer en la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): página del paquete con comando de instalación e historial de versiones.

2.  [pfstr/MailHeaderAnalyzer en GitHub](https://github.com/pfstr/MailHeaderAnalyzer): código fuente, suite de pruebas, changelog e incidencias.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): modelo para la estructura de esta referencia (sintaxis, descripción, ejemplos, propiedades de parámetros).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): estructura de `Authentication-Results` y la regla de que solo es determinante la línea de la organización receptora (sección 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): estructura de las líneas `Received` y recomendación de un límite máximo como protección contra bucles (sección 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): los valores `with` `ESMTPS`, `ESMTPA` y `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): etiquetas de `DKIM-Signature`, obligación de firmar `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): clasificación de `rsa-sha1` como obsoleto.

9.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): alineación estricta y relajada; sustituye RFC 7489 desde mayo de 2026.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): estructura de la cadena formada por `ARC-Seal`, `ARC-Message-Signature` y `ARC-Authentication-Results`, números de instancia y valores `cv=`.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): significado de SCL, BCL, CAT, SFV, IPV y los códigos de motivo `compauth`.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): origen de los significados de MessageDirectionality, AuthAs y AuthMechanism.

13.  [Analizador de encabezados en rafaelpfister.ch](/tools/header-analyzer): la versión de navegador con la misma lógica de análisis.
