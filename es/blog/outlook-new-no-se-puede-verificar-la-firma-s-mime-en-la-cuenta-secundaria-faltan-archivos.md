---
title: "Outlook New: no se puede verificar la firma S/MIME en la cuenta secundaria, faltan archivos adjuntos"
navTitle: "S/MIME en cuenta secundaria"
description: "El nuevo Outlook indica en el buzón compartido que la firma S/MIME no se puede verificar en la cuenta secundaria y no muestra archivos adjuntos. El artículo explica la diferencia entre Clear Signing y Opaque Signing, por qué desaparecen los adjuntos en los correos firmados de forma opaca, por qué el nuevo Outlook solo procesa S/MIME en la cuenta principal y qué alternativas existen, incluido extraer smime.p7m con PowerShell u OpenSSL."
date: "2026-09-03"
kategorie: "Outlook"
timeToRead: "8 min de lectura"
themen:
  - outlook
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "outlook"
protokolle:
  - "verschluesselung"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
slug: "outlook-new-no-se-puede-verificar-la-firma-s-mime-en-la-cuenta-secundaria-faltan-archivos"
translationId: "article-f1e9d4ab5be67349"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
translationOf: outlook-smime-sekundaeres-konto-anhaenge-fehlen
url: https://rafaelpfister.ch/es/blog/outlook-new-no-se-puede-verificar-la-firma-s-mime-en-la-cuenta-secundaria-faltan-archivos
translationSourceHash: ee167a56424fa3ffe1d8e79c62a748cd68c7864d7a95d3d9fdc8989a33cd4283
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:12:21.432Z
translationReview: automatic
---

# Outlook New: no se puede verificar la firma S/MIME en la cuenta secundaria, faltan archivos adjuntos

Al abrir un correo firmado digitalmente en un buzón compartido en el nuevo Outlook para Windows, aparece una barra roja: "No se puede verificar la firma S/MIME al visualizarla en la cuenta secundaria." El correo se muestra, pero no los archivos adjuntos, aunque el remitente los haya enviado. Las compañeras y los compañeros que utilizan el mismo buzón como cuenta principal ven los adjuntos sin problemas.

Detrás de esto hay dos factores que se refuerzan mutuamente: el nuevo Outlook solo procesa S/MIME en la cuenta principal y el remitente ha firmado el correo de forma opaca. Con esta forma de firma, todo el contenido, incluidos los adjuntos, está dentro de un único contenedor criptográfico. Si el cliente no puede abrir el contenedor, los adjuntos permanecen invisibles. Ambos problemas pueden resolverse por separado.

## Qué significa el mensaje

"Cuenta secundaria" significa en el nuevo Outlook cualquier buzón que no sea la cuenta con la que ha iniciado sesión. Esto se aplica a los buzones compartidos (Shared Mailboxes) que se muestran mediante acceso completo y automapping, a los buzones que haya agregado mediante "Agregar buzón compartido" y a las delegaciones. El procesamiento de S/MIME en el nuevo Outlook está vinculado de forma fija a la cuenta principal. Si un mensaje firmado llega a otra cuenta, el cliente no verifica la firma y muestra el mensaje en su lugar.

No se trata de una afirmación sobre la validez de la firma ni de un problema con el certificado del remitente. El mismo correo puede verificarse y abrirse en la cuenta principal o en el Outlook clásico.

## Clear Signing y Opaque Signing

El estándar S/MIME (RFC 8551) conoce dos formatos para mensajes firmados. Ambos proporcionan la misma firma, pero empaquetan el mensaje de forma diferente.

| | Clear Signing | Opaque Signing |
|---|---|---|
| Tipo MIME | `multipart/signed` con `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` con `smime-type=signed-data` |
| Estructura | Dos partes: el texto legible del mensaje con los adjuntos y, por separado, la firma | Una parte: texto del mensaje, adjuntos y firma juntos en un contenedor CMS-SignedData, codificado en Base64 |
| Adjunto que ve un cliente sin S/MIME | `smime.p7s` (solo la firma, unos pocos KB) | `smime.p7m` (el mensaje completo) |
| Legible sin compatibilidad con S/MIME | Sí, el texto y los adjuntos se muestran normalmente | No, el cliente solo ve el contenedor |
| Sensibilidad durante el transporte | La firma deja de ser válida si un servidor de correo o gateway modifica saltos de línea, codificación o espacios en blanco | El contenedor protege el contenido frente a esos cambios |
| Sección de RFC 8551 | 3.5.3 | 3.5.2 |

En el código fuente del mensaje puede reconocer ambos formatos por la cabecera `Content-Type`. Un correo firmado de forma clara comienza así:

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

Un correo firmado de forma opaca, así:

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

La diferencia explica por completo el comportamiento del nuevo Outlook. En un correo firmado de forma clara, el cliente muestra el texto y los adjuntos incluso si no verifica la firma; solo falta el estado de la firma. En un correo firmado de forma opaca, el cliente primero debe extraer el contenedor mediante el procesamiento S/MIME para acceder al texto y a los adjuntos. Si se niega a hacerlo porque el mensaje está en una cuenta secundaria, el contenedor permanece cerrado. Que el texto siga siendo legible se debe a Exchange Online: el servicio representa la parte de texto en el servidor, pero no los adjuntos del contenedor.

Ninguno de los dos formatos cifra nada. Incluso la variante opaca solo está codificada en Base64 y puede ser leída por cualquiera que obtenga el mensaje. Microsoft lo indica expresamente en la documentación de Exchange Online.

## Qué formato elige el remitente

En el Outlook clásico, la opción "Enviar mensajes firmados como texto sin formato" (Archivo > Opciones > Centro de confianza > Seguridad de correo electrónico) controla el formato. Está activada de forma predeterminada, por lo que Outlook firma de forma clara. Quien desactive la opción envía de forma opaca. El nuevo Outlook y Outlook en la Web no ofrecen esta configuración.

Los gateways de correo que firman de forma centralizada tienen su propia configuración para el formato de firma. Algunos productos firman de forma opaca de manera predeterminada por motivos de robustez, ya que la firma sigue siendo válida incluso después de una modificación por sistemas posteriores. Si recibe regularmente correos de un remitente concreto con adjuntos ausentes, merece la pena revisar la configuración de su gateway.

## Por qué el nuevo Outlook solo procesa S/MIME en la cuenta principal

Microsoft documenta la limitación, pero no indica ningún motivo técnico. La razón se desprende de la arquitectura del cliente.

El nuevo Outlook es, en esencia, el cliente web de Outlook en la Web dentro de una envoltura nativa. En el navegador, JavaScript no puede acceder al almacén de certificados de Windows. Por ello, Outlook en la Web necesitó durante años una extensión de navegador S/MIME independiente. El nuevo Outlook sustituye esta extensión por un puente integrado entre la interfaz web y la criptografía de Windows. Este puente se inicializa al iniciar sesión en la cuenta principal y recibe su sesión de buzón, sus certificados y sus configuraciones S/MIME desde Configuración > Correo > S/MIME.

Los buzones compartidos y las cuentas secundarias utilizan otras rutas de datos. Las cuentas secundarias tienen sus propias sesiones; los buzones compartidos se cargan mediante la delegación de la cuenta principal. Hasta ahora, el puente no estaba conectado a estas rutas. Esto también se aplica a la mera verificación de una firma, aunque para ello no sea necesaria una clave privada: el análisis y la extracción de la estructura PKCS#7 se realizan mediante el mismo componente.

En el Outlook clásico el problema no se produce, porque el procesamiento S/MIME se realiza en la pila MAPI por mensaje, independientemente del almacén del que proceda el mensaje.

Microsoft está añadiendo la conexión que falta: la entrada 565861 de la hoja de ruta amplía S/MIME en el nuevo Outlook a buzones compartidos y delegados vinculados a la cuenta principal. La disponibilidad general está anunciada para julio de 2026 y la implementación será gradual. Si continúa viendo el mensaje, el cambio aún no ha llegado a su tenant o a su versión de cliente. La entrada no cubre las cuentas secundarias agregadas por separado con inicio de sesión propio.

## Alternativas

La vía adecuada depende de cómo esté integrado el buzón y de si necesita verificar la firma o solo acceder a los adjuntos.

| Vía | Requisito | Resultado |
|---|---|---|
| Abrir el correo en la cuenta principal | Tiene el buzón como cuenta principal o el correo se le ha reenviado | Verificación de firma y adjuntos |
| Agregar el buzón como cuenta independiente en el nuevo Outlook (Configuración > Cuentas > Agregar cuenta) | El buzón tiene credenciales propias; no es posible con buzones compartidos sin contraseña | Verificación de firma y adjuntos al cambiar a esta cuenta |
| Outlook clásico | Aún instalado o se puede volver mediante el interruptor "Nuevo Outlook"; integrar allí el buzón como cuenta propia (Archivo > Configuración de la cuenta > Nuevo) | Verificación de firma y adjuntos en cada almacén |
| Outlook en la Web | Abrir el buzón directamente (`outlook.office.com/mail/<adresse>`), con la extensión S/MIME para Edge o Chrome instalada | Verificación de firma y adjuntos |
| Pedir al remitente Clear Signing | El remitente usa Outlook clásico o un gateway con formato seleccionable | Adjuntos visibles; el estado de la firma seguirá sin mostrarse en la cuenta secundaria |
| Extraer el contenedor manualmente | Guardar el correo como `.eml` o conservar `smime.p7m` | Adjuntos sin verificación de firma |

## Extraer el contenedor manualmente

Para un caso puntual basta con abrir el contenedor fuera de Outlook. La firma se verifica criptográficamente, pero no la cadena de confianza del certificado. Guarde el mensaje como `.eml` o guarde el adjunto `smime.p7m` en una carpeta.

En Windows basta con PowerShell. .NET Framework incluye la clase `SignedCms`, que lee el contenedor PKCS#7:

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Instrucción | Efecto |
|---|---|
| `Add-Type -AssemblyName System.Security` | Carga el ensamblado con las clases PKCS (necesario en Windows PowerShell 5.1; en PowerShell 7 ya está cargado) |
| `[IO.File]::ReadAllBytes(...)` | Lee el contenedor DER binario; el archivo `smime.p7m` guardado desde Outlook ya está decodificado |
| `$cms.Decode($bytes)` | Analiza la estructura CMS-SignedData |
| `$cms.CheckSignature($true)` | Verifica únicamente la firma sobre el contenido (`$true` = verifySignatureOnly); con `$false` también se verificaría la cadena de certificados. Si la firma no es válida, el comando se interrumpe con una excepción |
| `$cms.ContentInfo.Content` | El contenido firmado: un mensaje MIME completo con texto y adjuntos |
| `[IO.File]::WriteAllBytes(...)` | Escribe este mensaje MIME como `.eml`, que puede abrir en Outlook o Thunderbird |

</details>

En Linux, macOS o con Git para Windows, OpenSSL está disponible. Si tiene el correo completo como `.eml`, OpenSSL también realiza la decodificación Base64:

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `cms` | Herramienta CMS de OpenSSL; `smime` funciona de forma equivalente, `cms` es la interfaz más reciente |
| `-verify` | Verifica la firma y genera el contenido firmado |
| `-noverify` | Omite la verificación de la cadena de certificados; la firma en sí se sigue verificando |
| `-in nachricht.eml` | El correo completo en formato S/MIME (Base64 con cabeceras MIME); para un `smime.p7m` guardado, añadir además `-inform DER` |
| `-out inhalt.eml` | El contenido extraído como mensaje MIME |

</details>

El archivo `inhalt.eml` contiene el texto original del mensaje y todos los adjuntos como partes MIME normales. Al hacer doble clic se abre en Outlook, donde puede guardar los adjuntos como de costumbre.

## Fuentes

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): El caso práctico con el mismo mensaje en el buzón compartido; la respuesta confirma que el comportamiento es conocido y no ofrece solución en el nuevo Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Secciones 3.5.2 (application/pkcs7-mime con SignedData) y 3.5.3 (multipart/signed), con indicaciones sobre la legibilidad sin S/MIME y la robustez durante el transporte.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): La opción "Enviar mensajes firmados como texto sin formato" en Outlook clásico, activada de forma predeterminada; no está disponible en el nuevo Outlook.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): Configuración S/MIME en el nuevo Outlook en Configuración > Correo > S/MIME; los certificados deben instalarse manualmente o mediante directiva.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Indicación de que los mensajes firmados de forma opaca solo están codificados en Base64 y no son confidenciales.

6.  [Microsoft 365 Roadmap, entrada 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME para buzones compartidos y delegados en el nuevo Outlook para Windows, anunciado para julio de 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Qué tipos de cuentas admite el nuevo Outlook y cómo se integran los buzones compartidos.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature y ContentInfo para extraer el contenedor con PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Opciones `-verify`, `-noverify`, `-inform` y `-out`.
