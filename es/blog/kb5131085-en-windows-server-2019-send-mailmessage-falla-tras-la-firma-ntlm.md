---
title: "KB5131085 en Windows Server 2019: Send-MailMessage falla tras la firma NTLM"
navTitle: "Envío de correo KB5131085"
description: "Tras la actualización de noviembre KB5131085, el envío de correo mediante Send-MailMessage a servidores Postfix falla en Windows Server 2019. La causa se considera la nueva firma NTLM de KB5135557. El artículo explica por qué un cliente SMTP utiliza NTLM, cómo comprobar el caso y qué soluciones alternativas existen: forzar AUTH LOGIN, desactivar NTLM en el servidor de correo o revertirlo temporalmente mediante el Registro."
date: "2026-11-17"
kategorie: "Windows y SMTP"
timeToRead: "7 min de lectura"
themen:
  - windows-client
  - smtp-mailflow
produkte:
  - "windows-client"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
  - "releases"
slug: "kb5131085-en-windows-server-2019-send-mailmessage-falla-tras-la-firma-ntlm"
translationId: "article-ca3e77cefa049fca"
translationOf: kb5131085-send-mailmessage-ntlm
url: https://rafaelpfister.ch/es/blog/kb5131085-en-windows-server-2019-send-mailmessage-falla-tras-la-firma-ntlm
translationSourceHash: 6416155fbbec934196848b06f3468ff46dbca040310b92cf8fd3171db1858227
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:19:57.438Z
translationReview: automatic
---

# KB5131085 en Windows Server 2019: Send-MailMessage falla tras la firma NTLM

Desde el Patch Tuesday del 10 de noviembre de 2026, los administradores informan de que los scripts en Windows Server 2019 ya no pueden enviar correos. Se ven afectados informes de copia de seguridad, alertas de monitorización y tareas similares que desde hace años envían mensajes a un servidor Postfix con el cmdlet de PowerShell `Send-MailMessage`. Tras desinstalar la actualización acumulativa KB5131085 (OS Build 17763.9487), el envío vuelve a funcionar. Günter Born describió el caso el 17 de noviembre de 2026 en Borns IT- und Windows-Blog, a raíz del aviso de un lector.

Microsoft aún no incluye el problema como Known Issue. La única entrada de Known Issue para KB5131085 afecta a comandos de activación con el error `0x80041008`. Por ello, la causa no está confirmada oficialmente, pero puede deducirse bien de los cambios introducidos por la actualización y del comportamiento de la clase .NET `SmtpClient`.

## Qué cambia KB5131085 en NTLM

La actualización incluye medidas de protección contra ataques de retransmisión NTLM, que Microsoft describe en KB5135557 («NTLM signing security improvements in Windows»). Se aplican a Windows 11, Windows 10 Enterprise LTSC 2021 y Windows Server 2016 a 2025. El registro de cambios de KB5131085 indica que, a partir de ahora, los clientes ofrecen de forma predeterminada la firma de mensajes NTLM para todas las conexiones no anónimas.

Técnicamente, esto significa que un cliente Windows ahora establece en la negociación NTLM la marca `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) y el `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`) necesario para ello, incluso si la aplicación no ha solicitado integridad. Hasta ahora, tales aplicaciones solo recibían `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, un marcador de posición sin protección criptográfica. Microsoft distribuye el cambio en dos fases:

| Fase | Fecha | Comportamiento |
|---|---|---|
| 1 | 10 de noviembre de 2026 | Los clientes ofrecen firma en todas las conexiones. Los servidores aceptan clientes firmados y sin firmar, y registran los no firmados. |
| 2 | Mayo de 2027 | Los servidores Windows exigen firma de forma predeterminada y rechazan clientes sin firma con `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`). |

Según Microsoft, un cliente Windows vuelve sin errores al comportamiento anterior si el servidor no confirma la firma. Todos los servidores Windows lo hacen. Los extremos con una implementación NTLM propia, como un servidor de correo Linux, no se tienen en cuenta en esta afirmación.

## Por qué un cliente SMTP utiliza NTLM

`Send-MailMessage` en Windows PowerShell 5.1 utiliza internamente la clase .NET `System.Net.Mail.SmtpClient`. Esta comprueba la línea `AUTH` en la respuesta EHLO del servidor y prueba los métodos en un orden fijo: primero `GSSAPI` (Negotiate), luego `NTLM`, después `WDigest` y, por último, `LOGIN`. Si el servidor ofrece NTLM, `SmtpClient` usa NTLM, incluso cuando se proporcionan el nombre de usuario y la contraseña mediante `-Credential`.

Postfix no implementa por sí mismo la autenticación SMTP, sino que la delega en Cyrus SASL o Dovecot. Ambos pueden ofrecer NTLM: Cyrus SASL mediante un complemento propio; Dovecot, hasta la versión 2.3, mediante `ntlm` en `auth_mechanisms` (Dovecot 2.4 ya no incluye NTLM). Si NTLM está activo, Postfix anuncia, por ejemplo, `AUTH PLAIN LOGIN NTLM`, y cada cliente Windows se autentica mediante NTLM sin que nadie lo advierta. Con KB5131085, el cliente ofrece adicionalmente firma e intercambio de claves en esta negociación. Si la implementación NTLM del servidor de correo no puede gestionar esto, el inicio de sesión falla y `Send-MailMessage` se interrumpe. Un comentarista del artículo de Born menciona la firma NTLM como causa, y el hallazgo encaja: la solución alternativa del lector funciona porque excluye NTLM.

## Comprobar el caso

Primero, determine si el servidor de correo ofrece NTLM. Desde un equipo Linux, la respuesta EHLO tras STARTTLS muestra los métodos disponibles:

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-starttls smtp` | Establece la conexión en texto sin cifrar y cambia a TLS mediante STARTTLS |
| `-crlf` | Envía finales de línea como CRLF, tal como requiere SMTP |
| `-quiet` | Suprime detalles del certificado y de la sesión |
| `-connect mail.example.com:587` | Host de destino y puerto de submission |

</details>

Tras establecer la conexión, introduzca `EHLO test`. Si la línea `250-AUTH` contiene la entrada `NTLM`, el servidor está afectado. En el propio servidor Postfix, dos comandos muestran qué backend SASL está activo y qué métodos ofrece Dovecot:

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

En el servidor Windows, el registro NTLM documenta desde Windows Server 2019 las conexiones con las marcas negociadas. El evento 4022 es la entrada del cliente:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Lee el registro operativo de NTLM |
| `-MaxEvents 50` | Limita la salida a las 50 entradas más recientes |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filtra los eventos de auditoría para cliente (4022) y servidor (4023) |
| `Format-List TimeCreated, Id, Message` | Muestra la hora, el ID de evento y el texto con las Negotiated Flags |

</details>

Una entrada en el momento del intento de envío con el servidor de correo como destino confirma que `SmtpClient` ha utilizado NTLM.

## Solución alternativa 1: forzar AUTH LOGIN en el script

`Send-MailMessage` no ofrece ningún parámetro para definir el método de inicio de sesión. La clase `SmtpClient`, invocada directamente, sí puede hacerlo: un `CredentialCache` proporciona las credenciales solo para un método específico. Si únicamente está registrado `Login`, `SmtpClient` omite NTLM y usa `AUTH LOGIN`. El siguiente script se basa en el ejemplo del lector del artículo de Born:

```powershell
$server = 'mail.example.com'
$port   = 587
$cred   = Import-Clixml -Path 'C:\Skripte\smtp.cred'

$cache = New-Object System.Net.CredentialCache
$cache.Add($server, $port, 'Login', $cred.GetNetworkCredential())

$smtp = New-Object System.Net.Mail.SmtpClient($server, $port)
$smtp.EnableSsl   = $true
$smtp.Credentials = $cache

$msg = New-Object System.Net.Mail.MailMessage
$msg.From = 'backup@example.com'
$msg.To.Add('admin@example.com')
$msg.Subject      = 'Backup-Report'
$msg.Body         = 'Sicherung abgeschlossen.'
$msg.BodyEncoding = [System.Text.Encoding]::UTF8

try {
    $smtp.Send($msg)
}
finally {
    $msg.Dispose()
    $smtp.Dispose()
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Import-Clixml -Path …` | Carga un PSCredential guardado previamente con `Get-Credential \| Export-Clixml`; vinculado mediante DPAPI al usuario y al equipo |
| `$cache.Add($server, $port, 'Login', …)` | Registra las credenciales únicamente para el host, el puerto y el método `Login` |
| `SmtpClient($server, $port)` | Crea el cliente SMTP para el servidor y el puerto de submission |
| `EnableSsl = $true` | Cambia a TLS mediante STARTTLS antes de transmitir las credenciales |
| `Credentials = $cache` | Pasa el CredentialCache en lugar de una NetworkCredential simple |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Codifica el texto como UTF-8 para que los caracteres con diéresis lleguen correctamente |

</details>

`EnableSsl` es obligatorio aquí: `AUTH LOGIN` transmite el nombre de usuario y la contraseña solo codificados en Base64, por lo que sin TLS se enviarían en texto sin cifrar. La solución alternativa supone una intervención por script, pero no modifica la configuración del sistema y mantiene intacta la protección NTLM de la actualización.

## Solución alternativa 2: desactivar NTLM en el servidor de correo

Si administra el propio servidor Postfix, puede resolver el problema de forma centralizada: elimine `ntlm` de `auth_mechanisms` en la configuración de Dovecot (normalmente `/etc/dovecot/conf.d/10-auth.conf`), de modo que solo permanezca `plain login`, y recargue Dovecot:

```bash
sudo systemctl reload dovecot
```

En Cyrus SASL, `mech_list: plain login` en el archivo `smtpd.conf` del directorio de configuración de SASL restringe los métodos de forma correspondiente. Después, Postfix deja de anunciar NTLM y `Send-MailMessage` vuelve a `LOGIN` sin cambiar el script. Compruebe antes si otros clientes dependen de NTLM, por ejemplo, perfiles antiguos de Outlook con la opción «Autenticación de contraseña segura (SPA)». Para submission mediante TLS, NTLM no aporta ninguna mejora de seguridad frente a `PLAIN` y `LOGIN`.

## Solución alternativa 3: Registro, solo temporalmente

KB5135557 documenta un valor del Registro con el que el cliente deja de ofrecer firma:

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Clave del paquete de autenticación NTLM |
| `/v ClientAllowUnsignedNtlm` | Nombre del valor; `0` (predeterminado) ofrece firma, `1` restaura el comportamiento anterior |
| `/t REG_DWORD` | Tipo de datos del valor |
| `/d 1` | Establece el valor en 1 |
| `/f` | Sobrescribe sin solicitar confirmación |

</details>

Según Microsoft, no es necesario reiniciar. No obstante, el valor se aplica a todas las conexiones NTLM del servidor, no solo al envío de correo, y por tanto elimina la protección contra retransmisión de la actualización para todo el sistema. Microsoft lo describe como una medida exclusivamente de compatibilidad y se reserva el derecho de retirar posteriormente los valores del Registro. Como mucho, cubre el tiempo hasta implementar la solución alternativa 1 o 2. Desinstalar KB5131085 no es una alternativa, ya que con ello faltan todas las correcciones de seguridad del Patch Tuesday de noviembre.

## A medio plazo: sustituir Send-MailMessage

Microsoft considera `Send-MailMessage` obsoleto desde hace años y desaconseja su uso porque el cmdlet no garantiza conexiones seguras. También para `SmtpClient`, la documentación de .NET recomienda otras bibliotecas, como MailKit, para desarrollos nuevos. Para scripts que envían correos a través de Exchange Online, la API de Microsoft Graph con `Send-MgUserMail` y el registro de aplicaciones es la vía compatible; la configuración se describe en el artículo [Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph](/blog/microsoft-graph-powershell-postfach-anbindung). A más tardar con la fase 2 en mayo de 2027, conviene revisar qué scripts y servicios siguen utilizando NTLM: los eventos de auditoría 4022 y 4023 proporcionan la lista.

## Fuentes

1.  [Windows Server 2019: KB5131085 interfiere con el envío de correo mediante PowerShell (Borns IT- und Windows-Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/): Informe de un lector del 17 de noviembre de 2026 con el síntoma, la prueba de desinstalación, el script CredentialCache y la referencia a la firma NTLM en los comentarios.

2.  [KB5131085 (OS Build 17763.9487)](https://support.microsoft.com/help/5131085): Registro de cambios de la actualización de noviembre para Windows Server 2019 y Windows 10 Enterprise LTSC 2019 con la entrada NTLM y el Known Issue relativo al error 0x80041008.

3.  [KB5135557: NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557): Marcas, calendario de las dos fases, eventos de auditoría 4022/4023 y los valores del Registro ClientAllowUnsignedNtlm y ServerRequireNtlmSign.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs): Orden de registro de los módulos de inicio de sesión SMTP Negotiate, NTLM, Digest y Login en SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Aviso de que el cmdlet está obsoleto y no garantiza conexiones seguras.

6.  [SmtpClient Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Recomendación de utilizar MailKit u otras bibliotecas para desarrollos nuevos.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html): Autenticación SMTP en Postfix mediante Cyrus SASL o Dovecot, incluida la restricción de los métodos ofrecidos.

8.  [Dovecot: Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html): Lista actual de métodos SASL en Dovecot, que ya no incluye NTLM.
