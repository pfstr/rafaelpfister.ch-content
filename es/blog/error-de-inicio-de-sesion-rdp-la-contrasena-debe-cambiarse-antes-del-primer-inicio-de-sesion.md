---
title: "Error de inicio de sesión RDP: la contraseña debe cambiarse antes del primer inicio de sesión"
navTitle: "RDP: cambiar contraseña"
description: "Una cuenta nueva con contraseña inicial y la conexión de Escritorio remoto se interrumpe con «You must change your password before logging on the first time». La causa es la autenticación de nivel de red, que no permite cambiar la contraseña. Aun así, la contraseña puede cambiarse: mediante otra sesión de Windows, desde el propio PC, con PowerShell o a través de RD Web Access."
date: "2026-11-11"
kategorie: "Cliente Windows"
timeToRead: "6 min de lectura"
themen:
  - windows-client
  - active-directory-entra
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "error-de-inicio-de-sesion-rdp-la-contrasena-debe-cambiarse-antes-del-primer-inicio-de-sesion"
translationId: "article-96f19bf85c2f7ccd"
draft: false
translationOf: rdp-kennwort-aendern-nla
url: https://rafaelpfister.ch/es/blog/error-de-inicio-de-sesion-rdp-la-contrasena-debe-cambiarse-antes-del-primer-inicio-de-sesion
translationSourceHash: 016256f24e5c0c258d7cbf225d3f62ac577cf37c42844ad4a5f5084f85e7e8bf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:32:31.374Z
translationReview: automatic
---

# Error de inicio de sesión RDP: la contraseña debe cambiarse antes del primer inicio de sesión

Con una cuenta recién configurada en el entorno de un cliente, me he vuelto a encontrar con esta situación: la contraseña inicial es correcta, pero la conexión de Escritorio remoto se desconecta inmediatamente. El cliente de Windows `mstsc` muestra el siguiente mensaje:

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

Otros clientes RDP y administradores de conexiones indican el mismo estado con su propia redacción, por ejemplo «The user password must be changed before logging on for the first time». Cuando la contraseña ha caducado, el mensaje es «This user account's password has expired. The password must change in order to logon». La causa y la solución son las mismas en ambos casos.

En resumen: en esta situación no es posible cambiar la contraseña mediante la conexión RDP. Debe cambiarse previamente por otra vía y, en la mayoría de los casos, basta con tener conexión de red a un controlador de dominio.

## Causa: NLA autentica antes de establecer la sesión

En la cuenta está activada en Active Directory la opción **El usuario debe cambiar la contraseña en el siguiente inicio de sesión** (técnicamente: `pwdLastSet` = 0). En un equipo de trabajo del dominio no supone ningún problema: la pantalla de inicio de sesión detecta el estado y solicita directamente la nueva contraseña.

Con RDP, este paso no existe cuando está activa la autenticación de nivel de red (NLA). NLA autentica al usuario mediante CredSSP antes de que se cree una sesión con pantalla de inicio de sesión en el sistema de destino. La autenticación con una contraseña que debe cambiarse falla y no está previsto mostrar en ese punto un diálogo para cambiarla. El cliente desconecta la conexión y solo muestra el mensaje. NLA es la configuración predeterminada en las versiones actuales de Windows y se impone mediante directiva de grupo en entornos reforzados.

## Opciones para el usuario

Hay cuatro formas de cambiar la contraseña sin usar la conexión RDP afectada. Cuál de ellas es viable depende de qué sea accesible desde el propio puesto de trabajo.

| Opción | Requisito |
|---|---|
| 1. Portal de contraseñas o mesa de servicio | proporcionado por el operador |
| 2. Sesión existente de Windows en el dominio | acceso con otra cuenta o a otro sistema |
| 3. Diálogo de contraseña en el propio PC | controlador de dominio accesible, por ejemplo mediante VPN |
| 4. PowerShell con `NetUserChangePassword` | TCP 445 accesible hacia el controlador de dominio |

### Opción 1: Portal de contraseñas o mesa de servicio

En entornos grandes suele haber una vía prevista para colaboradores externos: un portal de autoservicio, una función de contraseña en el portal VPN o Citrix, o un sistema PAM. Si no existe nada de ello, la mesa de servicio puede eliminar la marca o establecer una contraseña sin obligación de cambio (véase la sección para administradores).

### Opción 2: Sesión existente de Windows en el dominio

Si ya existe una sesión en un sistema del dominio, por ejemplo en un jumphost o con una segunda cuenta, la contraseña puede cambiarse allí. En una sesión RDP, `Ctrl+Alt+End` abre la pantalla de seguridad del sistema remoto (`Ctrl+Alt+Del` se procesaría en el PC local). En **Cambiar una contraseña**, el campo del nombre de usuario es editable: introduzca allí la cuenta afectada con el formato `DOMAIN\benutzer`, debajo la contraseña inicial y dos veces la nueva contraseña.

### Opción 3: Diálogo de contraseña en el propio PC

El mismo diálogo también funciona en un PC que no sea miembro del dominio, siempre que pueda alcanzar un controlador de dominio, por ejemplo mediante una conexión VPN. Pulse `Ctrl+Alt+Del`, seleccione **Cambiar una contraseña** y sustituya el nombre de usuario local preseleccionado por `DOMAIN\benutzer`. El PC debe poder resolver el nombre de dominio mediante DNS; sin los servidores DNS del entorno de destino, Windows no encontrará ningún controlador de dominio.

### Opción 4: PowerShell con NetUserChangePassword

La API de Windows `NetUserChangePassword` cambia una contraseña directamente en un controlador de dominio. Durante el cambio comprueba la contraseña anterior y no requiere un inicio de sesión previo. Por ello, también funciona cuando la contraseña ha caducado o debe cambiarse, e incluso desde un PC fuera del dominio. El script integra la función mediante P/Invoke y solicita ambas contraseñas de forma oculta:

```powershell
$api = Add-Type -Namespace Win32 -Name NetApi32 -PassThru -MemberDefinition @'
[DllImport("netapi32.dll", CharSet = CharSet.Unicode)]
public static extern uint NetUserChangePassword(
    string domainname, string username,
    string oldpassword, string newpassword);
'@

$dc   = 'dc01.example.com'
$user = 'm.muster'
$alt  = Read-Host -AsSecureString 'Bisheriges Kennwort'
$neu  = Read-Host -AsSecureString 'Neues Kennwort'

$klartext = { param($s) [System.Net.NetworkCredential]::new('', $s).Password }
$rc = $api::NetUserChangePassword(
    $dc, $user, (& $klartext $alt), (& $klartext $neu))

if ($rc -eq 0) { 'Kennwort geändert' } else { "Fehler $rc" }
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Argumento | Efecto |
|---|---|
| `domainname` | Nombre DNS o NetBIOS de un controlador de dominio o del dominio. Desde un PC fuera del dominio, indique directamente el controlador de dominio (nombre o dirección IP). |
| `username` | Nombre de inicio de sesión de la cuenta (`sAMAccountName`), sin la parte del dominio. |
| `oldpassword` | Contraseña anterior; aquí, la contraseña inicial. Debe coincidir exactamente. |
| `newpassword` | Contraseña nueva. Debe cumplir la directiva de contraseñas del dominio. |

</details>

El valor de retorno 0 indica éxito. Los códigos de error más frecuentes son:

| Código | Significado |
|---|---|
| 5 | Acceso denegado, por ejemplo porque el controlador de dominio no permite la llamada sin inicio de sesión |
| 86 | La contraseña anterior es incorrecta |
| 2221 | No se encontró el nombre de usuario |
| 2245 | La nueva contraseña no cumple la directiva de contraseñas (longitud, complejidad, historial, antigüedad mínima) |
| 2351 | El nombre de servidor indicado no es válido |

La llamada se realiza mediante SMB y requiere el puerto TCP 445 hacia el controlador de dominio. En redes muy segmentadas, este puerto suele estar bloqueado desde las redes de clientes o desde la VPN; en ese caso, queda la opción 1.

`Set-ADAccountPassword` del módulo Active Directory no ayuda al usuario afectado en esta situación: el cmdlet requiere una conexión autenticada a los Active Directory Web Services, y precisamente ese inicio de sesión no es posible con la cuenta afectada. Es una herramienta para el administrador.

## Desactivar NLA en el cliente: solo sin NLA impuesta

A menudo se recomienda desactivar CredSSP en el archivo .rdp:

```text
enablecredsspsupport:i:0
```

El cliente se conecta entonces sin NLA y recibe la pantalla de inicio de sesión clásica del sistema de destino, donde es posible cambiar la contraseña. Esto solo funciona si el sistema de destino acepta conexiones sin NLA. Si allí está activa la opción **Permitir conexiones solo desde equipos que ejecuten Escritorio remoto con autenticación de nivel de red**, rechazará la conexión. Esta es la configuración predeterminada en las versiones actuales de Windows Server.

## Opciones para administradores

Desde el lado de la administración, el problema puede evitarse o resolverse:

- **Eliminar la marca:** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` elimina la obligación de cambio. El usuario puede iniciar sesión con la contraseña inicial y cambiarla en la sesión mediante `Ctrl+Alt+End`. La contraseña inicial seguirá siendo válida hasta entonces y, por tanto, debe transmitirse por un canal seguro.
- **Restablecer la contraseña:** `Set-ADAccountPassword -Identity m.muster -Reset` establece una contraseña nueva. Compruebe después si la marca sigue activada.
- **RD Web Access:** Si el rol está disponible, puede ofrecer el cambio de contraseña para contraseñas caducadas. En el Administrador de IIS, en **Sitios → Default Web Site → RDWeb → Pages**, establezca en **Application Settings** el valor `PasswordChangeEnabled` en `true`. La página para cambiar la contraseña se encuentra en `/RDWeb/Pages/en-US/password.aspx`.
- **Cuentas para acceso exclusivamente por RDP:** Para usuarios que trabajan exclusivamente mediante RDP (proveedores externos, administradores en jumphosts), la obligación de cambio en el primer inicio de sesión no puede cumplirse sin una de las vías indicadas anteriormente. Por ello, el procedimiento para el primer cambio de contraseña debería formar parte de la incorporación.

Desactivar NLA en el sistema de destino también resuelve el síntoma, pero elimina una capa de seguridad: sin NLA, cualquier cliente de la red llega a la pantalla de inicio de sesión y, con ello, al servicio RDP antes de autenticarse. Microsoft no lo recomienda.

## Fuentes

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): lista de códigos de desconexión del cliente RDP, incluidos 4615 (la contraseña debe cambiarse antes del primer inicio de sesión) y 3847 (contraseña caducada), con el texto de los mensajes.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): referencia de la API con parámetros, permisos y valores de retorno.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): informe práctico de Evotec sobre el cambio de contraseña mediante NetUserChangePassword desde un PC fuera del dominio, incluido el aviso sobre TCP 445 y sobre la entrada enablecredsspsupport ineficaz cuando NLA está impuesta.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): referencia de las propiedades .rdp, incluida enablecredsspsupport con valores y valor predeterminado.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): sección sobre el cambio de contraseña para contraseñas caducadas mediante RD Web Access y la configuración PasswordChangeEnabled.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): referencia del cmdlet con los parámetros Reset, OldPassword, NewPassword, Credential y Server.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): indicaciones de Microsoft sobre errores de inicio de sesión con NLA; desactivar NLA se marca allí como no recomendado.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): artículo de blog sobre el campo de usuario editable del diálogo «Cambiar una contraseña», que permite cambiar la contraseña de una cuenta de otro dominio.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): guía de Windows OS Hub sobre la configuración PasswordChangeEnabled y la ruta de la página password.aspx.
