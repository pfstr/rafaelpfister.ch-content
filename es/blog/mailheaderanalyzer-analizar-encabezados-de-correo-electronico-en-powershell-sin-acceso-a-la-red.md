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
  - "uebergreifend"
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
url: https://rafaelpfister.ch/es/blog/mailheaderanalyzer-analizar-encabezados-de-correo-electronico-en-powershell-sin-acceso-a-la-red
translationSourceHash: 7c01d42e83d3aa175486c74e2f88855bc7d9dee519026ce748edb239fdd355cc
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T18:54:05.082Z
translationReview: automatic
---

# MailHeaderAnalyzer: analizar encabezados de correo electrónico en PowerShell sin acceso a la red

MailHeaderAnalyzer es un módulo de PowerShell con dos cmdlets. `Get-MailHeaderAnalysis` analiza el encabezado de un correo electrónico: la cadena de entrega con retrasos e información de TLS, los resultados de SPF, DKIM, DMARC y ARC, incluida la comprobación de si estos resultados proceden del servidor receptor, la alineación de DMARC, la clasificación híbrida de Exchange Online, las evaluaciones de Microsoft Defender, SpamAssassin y Rspamd, así como anomalías como líneas `From` duplicadas o caracteres de control Unicode. `ConvertTo-MailHeaderReport` genera a partir de ello un informe para tickets. El módulo funciona completamente sin conexión: sin consultas DNS ni conexiones HTTP. Es la versión de línea de comandos del [analizador de encabezados de este sitio web](/tools/header-analyzer) y utiliza la misma lógica de análisis.

| | |
|---|---|
| Módulo | [MailHeaderAnalyzer en la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Código fuente | [pfstr/MailHeaderAnalyzer en GitHub](https://github.com/pfstr/MailHeaderAnalyzer), licencia MIT |
| Se aplica a | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlets | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Resumen de parámetros

| Cmdlet | Parámetro | Tipo | Obligatorio | Canalización | Efecto |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Sí (conjunto de parámetros Text) | Sí, por valor | El encabezado como texto. Las líneas de la canalización se combinan en un encabezado |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Sí (conjunto de parámetros Path) | Sí, por nombre de propiedad | Archivo con el encabezado o mensaje `.eml` completo; acepta objetos de `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Sí (conjunto de parámetros Clipboard) | No | Lee el encabezado del portapapeles (solo Windows) |
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
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [<CommonParameters>]
```

### Descripción

El cmdlet descompone el encabezado sin procesar en campos, despliega los plegados RFC 5322 y descodifica valores RFC 2047 en el asunto y las direcciones. A partir de las líneas `Received` forma la cadena de entrega en orden cronológico, calcula el retraso de cada estación y lee la versión de TLS, el cifrado y la clase de protocolo según RFC 3848. A partir de `Authentication-Results`, `Received-SPF`, `DKIM-Signature` y la cadena ARC determina los resultados de autenticación y comprueba si la línea de verificación procede realmente de una estación de la cadena de entrega (RFC 8601, sección 5). También incluye la alineación de DMARC, la clasificación híbrida de Exchange Online, las evaluaciones de los filtros antispam y una lista de anomalías.

El cmdlet no realiza consultas DNS ni abre conexiones de red. Por ello, `Spf`, `Dkim` y `Dmarc` son siempre el veredicto del servidor receptor, complementado con la comprobación de si dicho veredicto procede de él. Las firmas DKIM no se verifican criptográficamente.

Las entradas se leen de forma tolerante: una línea en blanco finaliza el encabezado e ignora el cuerpo del mensaje posterior. Las líneas sin nombre de campo y sin espacio inicial, como las que se producen al copiar desde diálogos de clientes, pertenecen al campo anterior. Se omite una línea separadora mbox `From ...` antes del primer campo y se elimina una marca de orden de bytes. Se analizan como máximo 200 líneas `Received`, contadas desde la entrega.

### Ejemplos

#### Ejemplo 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analiza el encabezado que se encuentra en el portapapeles. En Outlook para Windows encontrará el encabezado en Archivo, Propiedades, Encabezados de Internet; en Outlook en la web, en las opciones del mensaje, en «Ver detalles del mensaje».

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

Lee el encabezado línea por línea desde un archivo de texto y muestra solo las anomalías. `-Raw` en `Get-Content` no es necesario, ya que el cmdlet combina las líneas por sí mismo.

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
| `Get-ChildItem -Path .\export\*.eml` | Devuelve los objetos de archivo; `-Path` toma su propiedad `FullName` |
| `Select-Object … @{ Name; Expression }` | Columna calculada que reúne todos los códigos de hallazgos en una cadena |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV sin encabezado de tipo, UTF-8 para caracteres acentuados en las líneas de asunto |

#### Ejemplo 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Enumera los mensajes cuya línea `Authentication-Results` no procede del sistema de entrega. En análisis de phishing, es un primer filtro rápido.

#### Ejemplo 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Muestra la estación con el mayor retraso. `SlowestHopIndex` empieza en 1, mientras que el array `Hops` empieza en 0.

#### Ejemplo 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Escribe el análisis completo como JSON. `-Depth 6` es necesario porque `Hops`, `DkimSignatures` y `Exchange` son objetos anidados; el valor predeterminado 2 solo los mostraría como nombres de tipo.

### Parámetros

#### -Header

El encabezado como texto. El parámetro acepta una única cadena con todo el encabezado o varias cadenas; las entradas de canalización se recopilan y se combinan al final en un encabezado, por lo que `Get-Content datei | Get-MailHeaderAnalysis` funciona sin `-Raw`. Para analizar varios encabezados por separado, use `-Path` con varios archivos.

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

Ruta a un archivo que contiene el encabezado o un mensaje `.eml` completo. Las rutas relativas se resuelven respecto al directorio actual. El archivo se lee con `[System.IO.File]::ReadAllText`: se tiene en cuenta una marca de orden de bytes; sin BOM se usa UTF-8. Cada archivo genera su propio objeto de resultado; los archivos inexistentes producen un error no terminante.

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

Lee el encabezado del portapapeles con `Get-Clipboard -Raw`. El parámetro solo está disponible en Windows; en Linux y macOS el cmdlet termina con un mensaje de error, igual que si el portapapeles está vacío.

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

### Entradas

`System.String`: líneas de encabezado o el encabezado completo, para `-Header`.

`System.IO.FileInfo`: objetos de archivo de `Get-ChildItem`, cuya propiedad `FullName` se vincula a `-Path`.

### Salidas

`MailHeaderAnalyzer.Analysis`: un objeto por cada encabezado analizado. Las propiedades se describen en la sección [Objeto de salida](#ausgabeobjekt).

### Notas

El texto de análisis (las explicaciones en `Findings`, `CompAuthReasonMeaning` y los campos de significado) está en inglés para que pueda incorporarse sin cambios a tickets internacionales. La salida de la vista predeterminada se puede mostrar íntegramente con `Format-List *`.

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

El informe incluye asunto, remitente, fecha y Message-ID, los resultados de autenticación con indicación de origen y alineación, todos los hallazgos, la cadena de entrega, la clasificación de Exchange y los valores de los filtros antispam. Los caracteres de control Unicode de dirección de escritura permanecen visibles en el informe como `<U+...>`, para que no lleguen a un sistema de tickets a través del informe. La última línea indica la versión del módulo.

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

Genera el informe como texto sin formato Markdown, por ejemplo para correos electrónicos o registros de consola.

#### Ejemplo 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Uso en Exchange Management Shell. Si `$env:PSModulePath` está restringido allí por directivas de grupo, cargue el módulo con `Import-Module` y la ruta completa al archivo `.psd1`.

### Parámetros

#### -Analysis

El objeto de análisis de `Get-MailHeaderAnalysis`. El cmdlet rechaza otros tipos de objeto con un error de vinculación.

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
- `Text`: encabezados en mayúsculas, líneas con sangría y la cadena de entrega como lista numerada.

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

`Get-MailHeaderAnalysis` devuelve un objeto de tipo `MailHeaderAnalyzer.Analysis` por entrada. La vista predeterminada muestra el resumen del ejemplo 1; se puede acceder a todas las propiedades mediante `Select-Object`, `Format-List *` o `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Propiedad | Tipo | Contenido |
|---|---|---|
| `Source` | String | Ruta de archivo, `Clipboard` o `Text` |
| `Subject` | String | Asunto, descodificado según RFC 2047 |
| `From`, `ReplyTo`, `ReturnPath` | Objeto de dirección | `Name`, `Address`, `Domain`, `Display`; `$null` si falta el campo |
| `Date` | DateTime (UTC) | Valor del campo `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Dominio del remitente de sobre de `smtp.mailfrom` de la comprobación SPF; de lo contrario, de `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Resultado según la línea de `Authentication-Results` determinante (`pass`, `fail`, `none`, `softfail` y otros); `$null` si no se comprobó |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Código de motivo de la autenticación compuesta de Microsoft 365 y su significado |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` o `None`, véase [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id de la línea de verificación determinante |
| `AuthenticationResults` | Objeto[] | Todas las líneas `Authentication-Results` con `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objeto | La línea `Received-SPF` con `Result` y `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` o `$null` |
| `Hops` | Hop[] | Cadena de entrega en orden cronológico; véase [objeto Hop](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Métricas de la cadena |
| `DeliveredBy` | String | Host `by` de la línea `Received` más reciente, es decir, la estación de entrega |
| `DkimSignatures` | Objeto[] | Por firma: `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Objeto[], Bool | Instancias ARC con `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` es `$null` sin cadena ARC |
| `Exchange` | Objeto | Clasificación híbrida de Exchange Online; véase [objeto Exchange](#mailheaderanalyzerexchangeclassification); `$null` sin encabezados correspondientes |
| `Spam` | Objeto | Evaluaciones de los filtros antispam; véase [objeto Spam](#mailheaderanalyzerspamassessment); `$null` sin encabezados correspondientes |
| `List` | Objeto | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` sin encabezados de lista |
| `Findings` | Finding[] | Anomalías con `Severity`, `Code`, `Message`; véase [hallazgos](#findings) |
| `Fields` | Objeto[] | Todos los campos con `Name`, `Value` (desplegado) y `Raw` |
| `HadBody` | Bool | Indica si siguió un cuerpo de mensaje después del encabezado |

### MailHeaderAnalyzer.Hop

Cada entrada en `Hops` corresponde a una línea `Received`. El orden es cronológico, es decir, inverso al orden en el encabezado.

| Propiedad | Contenido |
|---|---|
| `Index` | Número correlativo, 1 = envío inicial |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Datos del sistema remitente de la parte `from`; la IP procede del corchete en el comentario y el nombre rDNS del comentario anterior |
| `ByHost`, `Software` | Sistema receptor y su software (comentario tras `by`) |
| `Protocol`, `ProtocolClass` | Valor `with` y clase según RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS o indicación TLS presente), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | De las notaciones de Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) y Exim |
| `Id`, `For`, `Via` | Otros componentes de `Received` |
| `Date` | Marca de tiempo tras el punto y coma, UTC |
| `Delay` | TimeSpan hasta el salto anterior; negativo si hay desfase de relojes |
| `Provider` | Proveedor o puerta de enlace detectado a partir de los nombres de host (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast y otros) |
| `Attested` | `$true` solo en el último salto: solo esta línea fue escrita por el propio sistema receptor; todas las inferiores ya estaban en el mensaje |
| `Raw` | La línea original |

### AuthTrust: origen de los resultados de verificación

Cualquier remitente puede escribir una línea `Authentication-Results` en un mensaje. Según RFC 8601, sección 5, solo es determinante la línea de la organización receptora, y su authserv-id debe poder asignarse a una estación de la cadena de entrega. El cmdlet compara el authserv-id de cada línea con los hosts `by` de la cadena `Received` (mismo dominio o subdominio, siempre en el límite de punto, sin subcadenas).

| Valor | Significado |
|---|---|
| `Matched` | El authserv-id aparece como host `by` en la cadena. Solo estas líneas se incorporan a `Spf`, `Dkim`, `Dmarc` cuando existe al menos una |
| `Unmatched` | El authserv-id no aparece en la cadena. Se muestran los resultados, pero se consideran una afirmación sin respaldo; el hallazgo `AuthUnverified` lo indica |
| `Absent` | La línea no contiene authserv-id. Microsoft 365 escribe su línea de verificación de esta forma; comienza directamente con `spf=` |
| `None` | No hay ninguna línea de verificación |

Si el encabezado contiene líneas de verificación de varios orígenes, el hallazgo `AuthMixedOrigins` informa de ello. Si falta una línea de verificación, pero hay una línea `Received-SPF`, se adopta su resultado como `Spf` y se identifica con `ReceivedSpfOnly`.

### Alineación de DMARC

`SpfAlignment` compara el dominio del remitente de sobre con el dominio `From`; `DkimAlignment` compara el dominio `d=` de la firma verificada con el dominio `From`. `Strict` significa dominio idéntico, `Relaxed` el mismo dominio organizativo y `None` ninguna coincidencia. El dominio organizativo se determina de forma heurística: las dos últimas etiquetas; en terminaciones compuestas conocidas como `co.uk` o `com.au`, las tres últimas. No se incluye una Public Suffix List completa.

### MailHeaderAnalyzer.ExchangeClassification

Si el encabezado contiene los campos `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` o `X-MS-Exchange-CrossTenant-*`, el cmdlet rellena la propiedad `Exchange`. Los significados siguen el artículo «Demystifying hybrid mail flow» del equipo de Exchange; los antecedentes se explican en el artículo [Encabezados híbridos de Exchange: ¿interno o externo?](/blog/exchange-hybrid-header-intern-extern).

| Propiedad | Contenido |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` o `Incoming` con explicación |
| `AuthAs`, `AuthAsMeaning` | `Internal` o `Anonymous` con las consecuencias para el filtrado EOP |
| `AuthSource` | Servidor que realizó la clasificación |
| `AuthMechanism`, `AuthMechanismMeaning` | Código de mecanismo. Solo el valor 10 (Externally Secured) está documentado públicamente; para todos los demás códigos, el módulo indica expresamente que Microsoft no los documenta |
| `OriginatorOrg` | Dominio predeterminado del tenant remitente, la característica de tenant no falsificable al recibir desde Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Clasificación e ID de tenant en el límite del tenant |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` o `HybridOnPrem` con explicación |
| `WrongTenantAttribution` | Valor de `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` si el mensaje se asignó a un tenant ajeno |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marcadores de encabezados de organización recibidos o eliminados por el conector de envío |

### MailHeaderAnalyzer.SpamAssessment

La propiedad `Spam` resume las evaluaciones de los filtros conocidos. Los valores proceden de sistemas ajenos y se descodifican, pero no se evalúan.

| Propiedad | Fuente | Contenido |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level con significado (-1 fiable, 0/1 no spam, 5/6 sospecha de spam, 9 muy probablemente spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level de 0 a 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Clasificación como `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Resultado del filtro como `NSPM`, `SPM`, `SKA` (lista de permitidos), `SKI` (intraorganizativo) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP en la lista de permitidos de conexión) o `NLI` (sin reputación) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | IP de entrega y país de origen |
| `Forefront` | `X-Forefront-Antispam-Report` | Todos los pares clave-valor del campo |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Puntuación y pruebas activadas |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Símbolos con puntuación |

La lista completa de códigos de motivo `compauth` se encuentra en el artículo [Microsoft 365 compauth: códigos de motivo](/blog/microsoft-365-compauth-reason-codes).

### Hallazgos

El cmdlet proporciona las anomalías como objetos en `Findings`, cada uno con `Severity` (`Info`, `Warning`, `Fail`), un `Code` estable para filtros y scripts, y una explicación en `Message`.

| Código | Gravedad | Significado |
|---|---|---|
| `DuplicateField` | Warning | Un campo que RFC 5322 limita a una instancia (`From`, `Subject`, `Date`, `Message-ID` y otros) aparece varias veces. Los clientes de correo y filtros pueden elegir instancias distintas; es un patrón conocido en falsificaciones |
| `BidiControls` | Warning | Carácter de control Unicode de dirección de escritura en un campo. Invierte la dirección de lectura, por lo que `fdp.exe` aparece como `exe.pdf`. El módulo lo muestra como `<U+202E>` |
| `HopOverflow` | Warning | Más de 200 líneas `Received`; las adicionales no se analizaron |
| `AuthUnverified` | Warning | Los resultados de verificación contienen un authserv-id que no aparece en la cadena de entrega |
| `AuthMixedOrigins` | Warning | Hay líneas de verificación de varios orígenes |
| `ReceivedSpfForeign` | Warning | El `receiver=` de la línea `Received-SPF` no aparece en la cadena |
| `ReceivedSpfOnly` | Info | El resultado SPF procede solo de `Received-SPF`, no de una línea de verificación |
| `NoAuthResults` | Info | No hay resultados de verificación en el encabezado |
| `DmarcFail` | Fail | DMARC no se superó según el servidor receptor |
| `SpfNotPass` | Warning | Resultado SPF `fail`, `softfail`, `permerror` o `temperror` |
| `DkimNotPass` | Warning | Resultado DKIM `fail`, `permerror` o `temperror` sin testigo ARC |
| `DkimBrokenAfterForward` | Info | DKIM no se superó en el destinatario, pero un sello ARC del mismo dominio acredita una firma válida anterior: típico de reenvíos y listas de correo |
| `DkimWeakHash` | Warning | Firma con `rsa-sha1` (RFC 8301 clasifica SHA-1 como obsoleto) |
| `DkimBodyLength` | Warning | La etiqueta `l=` limita la longitud firmada del cuerpo del mensaje; el contenido adjunto no está cubierto |
| `DkimExpired` | Warning | El momento `x=` está en el pasado |
| `DkimFromUnsigned` | Warning | El campo `From` no está incluido en `h=`, aunque RFC 6376 lo exige |
| `ClockSkew` | Info | Un salto tiene una marca de tiempo anterior a la de su predecesor; los retrasos son solo aproximados |
| `ReplyToMismatch` | Info | El dominio `Reply-To` difiere del dominio `From`; habitual en boletines, pero también un patrón de phishing |
| `SpfNotAligned` | Info | El dominio del remitente de sobre y el dominio `From` pertenecen a organizaciones diferentes; por tanto, SPF no contribuye a DMARC |
| `ExchangeWrongTenant` | Warning | El mensaje se asignó a un tenant ajeno; una causa clásica es un conector de entrada de otro tenant con el mismo certificado o las mismas direcciones IP |
| `ExchangeHeadersFiltered` | Warning | El conector de envío eliminó los encabezados Cross-Premises (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: entrada mediante un conector de recepción con «Externally Secured», filtrado EOP omitido |
| `SpamCategory` | Warning | Microsoft asignó una categoría distinta de `NONE` |
| `SpamConfidence` | Warning | SCL 5 o superior |

## Funcionamiento y límites

El módulo lee lo que contiene el encabezado e infiere de ello lo que puede demostrarse sin consultas externas. De ello se derivan algunos límites:

- **Sin verificación criptográfica.** Las firmas DKIM no se recalculan y no se consultan registros DNS. `Spf`, `Dkim` y `Dmarc` son siempre el veredicto del servidor receptor, complementado con la comprobación de si dicho veredicto procede realmente de él.
- **Solo la última línea `Received` está acreditada.** El remitente proporcionó todas las líneas inferiores y puede haberlas creado arbitrariamente. `Attested` marca esta diferencia; los retrasos de saltos anteriores se basan en los datos de estas líneas.
- **Dominios organizativos heurísticos.** Para la alineación relajada, el módulo utiliza una lista corta de terminaciones compuestas, no una Public Suffix List completa.
- **AuthMechanism solo está parcialmente documentado.** Salvo el valor 10, Microsoft no ha publicado los códigos; el módulo no inventa significados.
- **Conjuntos de caracteres.** Los valores RFC 2047 se descodifican con las codificaciones que .NET reconoce en el sistema correspondiente. Los conjuntos de caracteres desconocidos permanecen sin cambios.

## Protección de datos

Un encabezado completo contiene nombres de host internos, direcciones IP, remitentes, destinatarios y asuntos. El motivo por el que estos datos no pertenecen a una herramienta en línea se explica en el artículo [Analizar encabezados de correo electrónico sin subir el mensaje](/blog/e-mail-header-analysieren-ohne-upload). Para el módulo se aplica la misma promesa que para la versión de navegador: sin accesos a la red. La suite de pruebas del módulo incluye una prueba que falla en cuanto aparece un cmdlet de red o DNS en el código fuente.

## Código fuente y versiones

El código fuente está disponible bajo licencia MIT en [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). La lógica de análisis es una adaptación de la biblioteca que también utiliza el analizador de encabezados de este sitio web; ambos comparten los casos de prueba. La integración continua comprueba cada cambio con PSScriptAnalyzer y Pester en Windows PowerShell 5.1, PowerShell 7 en Windows, Ubuntu y macOS. Las publicaciones en la PowerShell Gallery se realizan automáticamente a partir de etiquetas versionadas; los cambios de cada versión se encuentran en el [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Recibo errores y solicitudes de ampliación como [issue en GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonimice los encabezados de prueba antes de enviar informes de errores; los casos de prueba incluidos utilizan exclusivamente dominios de ejemplo según RFC 2606 y direcciones según RFC 5737.

## Fuentes

1.  [MailHeaderAnalyzer en la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): página del paquete con comando de instalación e historial de versiones.

2.  [pfstr/MailHeaderAnalyzer en GitHub](https://github.com/pfstr/MailHeaderAnalyzer): código fuente, suite de pruebas, changelog e issues.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): modelo para la estructura de esta referencia (sintaxis, descripción, ejemplos, propiedades de parámetros).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): estructura de `Authentication-Results` y la regla de que solo la línea de la organización receptora es determinante (sección 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): estructura de las líneas `Received` y recomendación de un límite máximo como protección ante bucles (sección 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): los valores `with` `ESMTPS`, `ESMTPA` y `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): etiquetas de `DKIM-Signature`, obligación de firmar `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): clasificación de `rsa-sha1` como obsoleto.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): alineación Strict y Relaxed.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): `ARC-Seal`, `ARC-Authentication-Results` y la comprobación de cadena `cv=`.

11.  [Microsoft Learn: encabezados de mensajes antispam en Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): significado de SCL, BCL, CAT, SFV, IPV y los códigos de motivo `compauth`.

12.  [Blog del equipo de Exchange: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): origen de los significados de MessageDirectionality, AuthAs y AuthMechanism.

13.  [Analizador de encabezados en rafaelpfister.ch](/tools/header-analyzer): la versión de navegador con la misma lógica de análisis.
