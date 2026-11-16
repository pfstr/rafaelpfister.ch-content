---
title: "Registrar y consultar responsables de buzones compartidos en Active Directory"
navTitle: "Responsables de buzones compartidos"
description: "Las soluciones DLP deben saber a quién informar cuando bloquean un mensaje procedente de un buzón compartido. Qué atributos de AD son adecuados, cuáles solo reflejan permisos de acceso, por qué los permisos de Exchange Online no llegan al AD local y cómo un proveedor consulta automáticamente la asignación mediante LDAP o Microsoft Graph."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 min de lectura"
themen:
  - active-directory-entra
  - microsoft-365-exchange
produkte:
  - "entra"
  - "exchange-online"
  - "exchange"
protokolle:
  - "ldap"
  - "powershell"
slug: "registrar-y-consultar-responsables-de-buzones-compartidos-en-active-directory"
translationId: "article-ed41bd7fefc08361"
translationOf: shared-mailbox-verantwortliche-ad
url: https://rafaelpfister.ch/es/blog/registrar-y-consultar-responsables-de-buzones-compartidos-en-active-directory
translationSourceHash: 54b0d22f6d1fd74974494b96e5566af25ea846062f8f9ce6a255607e40f937e6
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T08:52:55.166Z
translationReview: automatic
---

# Registrar y consultar responsables de buzones compartidos en Active Directory

En un proyecto de prevención de pérdida de datos surgió la siguiente consulta: en cuanto se bloquee un mensaje procedente de un buzón compartido (Shared Mailbox), se debe notificar a las personas responsables para que puedan actuar. El proveedor de la solución DLP debe leer esta asignación automáticamente desde Active Directory. La pregunta es: ¿dónde indica AD quién es responsable de un buzón compartido?

Conclusión breve: AD solo conoce permisos de acceso, no responsabilidades. Algunos atributos indican quién puede acceder a un buzón o enviar desde él, e incluso estos son incompletos cuando el buzón se encuentra en Exchange Online. Para una automatización fiable, la persona responsable debe mantenerse explícitamente en un atributo. El atributo estándar `manager` es adecuado para ello, Entra Connect lo sincroniza y se puede consultar mediante LDAP y Microsoft Graph.

## El acceso no es responsabilidad

Un buzón compartido es, en AD, un objeto de usuario deshabilitado con atributos de Exchange. Quién lo utiliza se determina mediante permisos: acceso total (FullAccess), enviar como (Send As) y enviar en nombre de (Send on Behalf). Estas listas suelen contener diez o veinte personas, además de suplentes, aprendices y cuentas de servicio. Una alerta DLP dirigida a todas las personas autorizadas llega a muchas personas, ninguna de las cuales se siente responsable.

La cuestión de la responsabilidad es organizativa: una persona, o un grupo reducido, decide si un mensaje bloqueado era legítimo y mantiene los permisos. Esta información solo existe en AD si alguien la registra.

## Qué atributos de AD pueden considerarse

| Atributo | Contenido | Idoneidad para responsables |
|---|---|---|
| `manager` | DN de una persona, valor único, parte de las clases de personas y, por tanto, disponible en el objeto de usuario del buzón | Adecuado. Entra Connect lo sincroniza con Entra ID y se puede leer mediante Graph |
| `managedBy` | DN de un administrador, valor único | No está disponible en el objeto de usuario. Según el esquema, está definido, entre otros, para grupos, contactos, equipos y UO, no para la clase User |
| `msExchDelegateListLink` | DN de las personas que han recibido acceso total con Automapping, multivalor; referencia inversa `msExchDelegateListLinkBL` en el usuario | Solo acceso y, además, incompleto: faltan el acceso total sin Automapping, los permisos mediante grupos y los permisos otorgados en Exchange Online |
| `publicDelegates` | DN de las personas con permiso de enviar en nombre de, multivalor | Solo acceso. En entornos híbridos, Entra Connect lo escribe de vuelta |
| Lista de permisos (`nTSecurityDescriptor`) | Derecho extendido «Send As» en el objeto de usuario | Solo el permiso de enviar como otorgado localmente. Los permisos otorgados en Exchange Online no aparecen ahí |
| `msExchMailboxSecurityDescriptor` | Permisos de buzones locales, binario | Solo acceso, solo para buzones en servidores Exchange locales y difícil de evaluar para los proveedores |
| `extensionAttribute1` hasta `extensionAttribute15` | Campos de texto libre | Posible, pero solo como texto; sin referencia a un objeto ni limpieza automática al cambiar de nombre o al salir de la organización |

En la práctica, la indicación de que la asignación está «directamente en AD» suele referirse a `msExchDelegateListLink`: en ADUC o ADSI Edit se pueden ver personas allí. Sin embargo, el atributo solo se crea como efecto secundario de Outlook Automapping y no sirve como fuente de datos para responsabilidades.

## La ubicación del buzón determina la fuente de datos

Si los buzones están en Exchange Online y los usuarios se sincronizan desde AD local (híbrido), se aplica lo siguiente: los permisos otorgados en Exchange Online permanecen en Exchange Online. Para Exchange Hybrid, Entra Connect solo escribe de vuelta una pequeña lista de atributos, entre ellos `publicDelegates` y `proxyAddresses`, pero ni `msExchDelegateListLink` ni los permisos Send As. En este caso, un AD local muestra el estado previo a la migración más lo que se haya mantenido localmente desde entonces.

Para Exchange Online, solo Exchange Online proporciona el estado real de los permisos (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). Para ello, el proveedor necesitaría acceso a Exchange Online PowerShell, lo cual es desproporcionado para una simple asignación. Un atributo mantenido en el objeto de usuario es más sencillo y funciona independientemente de dónde se encuentre el buzón.

## Recomendación: `manager` como atributo de responsable

Para cada buzón compartido, se registra la persona responsable en el atributo `manager`. Motivos:

- El atributo forma parte del esquema estándar, por lo que no es necesaria una ampliación del esquema.
- Contiene una referencia (DN) en lugar de texto. Si la persona cambia de nombre o de ubicación, la referencia sigue siendo válida.
- Entra Connect sincroniza `manager` con Entra ID. El proveedor puede leerlo mediante LDAP en el AD local o mediante Microsoft Graph.
- Mediante la referencia inversa `directReports`, cada persona responsable —y cualquier informe— puede ver de qué buzones es responsable.

La limitación: `manager` admite un único valor. Cuando sea imprescindible contar con una suplencia, hay dos posibilidades. Opción 1: un grupo por buzón cuyos miembros sean los responsables; su nombre sigue una convención (por ejemplo, `SM-Verantwortlich-buchhaltung`) o figura en un `extensionAttribute`. Opción 2: el proveedor no deriva la suplencia desde AD, sino desde el `manager` de la persona responsable (escalación a la persona superior). La opción 2 no requiere objetos adicionales y es suficiente en la mayoría de las organizaciones.

El atributo solo se establece en el sistema maestro: en el AD local para objetos sincronizados y en Entra ID para objetos exclusivamente en la nube.

## Inventario: ¿quién tiene acceso actualmente?

Para la carga inicial resulta útil una lista de las personas autorizadas actualmente. La persona responsable suele estar entre ellas; el departamento especializado confirma la selección. Para buzones en Exchange Online:

```powershell
Connect-ExchangeOnline

$postfaecher = Get-EXOMailbox -RecipientTypeDetails SharedMailbox -ResultSize Unlimited

$bestand = foreach ($pf in $postfaecher) {
    $voll = Get-EXOMailboxPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'FullAccess' -and
            -not $_.IsInherited -and
            $_.User -notlike 'NT AUTHORITY\*'
        }
    $sendas = Get-EXORecipientPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'SendAs' -and
            $_.Trustee -notlike 'NT AUTHORITY\*'
        }
    [pscustomobject]@{
        Postfach  = $pf.PrimarySmtpAddress
        Vollzugriff = ($voll.User -join '; ')
        SendenAls = ($sendas.Trustee -join '; ')
    }
}

$bestand | Export-Csv -Path .\shared-mailbox-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Solo buzones compartidos, sin buzones de usuario, sala o equipo |
| `-ResultSize Unlimited` | Todos los buzones en lugar del límite predeterminado de 1000 |
| `-Identity` | Buzón cuyos permisos se consultan |
| `AccessRights -contains 'FullAccess'` | Filtra por acceso total; se excluyen otros permisos de buzón |
| `-not $_.IsInherited` | Elimina entradas heredadas (administradores, cuentas del sistema) |
| `User -notlike 'NT AUTHORITY\*'` | Elimina la entrada `NT AUTHORITY\SELF` del propio buzón |
| `-NoTypeInformation` | Sin línea de tipo al inicio del archivo CSV |
| `-Encoding UTF8` | Se conservan los caracteres especiales en los nombres |

</details>

Si un grupo tiene permisos, su nombre aparece en la lista; sus miembros se pueden resolver con `Get-DistributionGroupMember` o `Get-ADGroupMember`.

## Establecer responsables

La asignación confirmada se prepara como CSV con las columnas `Postfach` y `Verantwortlich` (ambas direcciones de correo electrónico) y se establece en el AD local:

```powershell
Import-Module ActiveDirectory

$zuordnung = Import-Csv -Path .\shared-mailbox-verantwortliche.csv

foreach ($eintrag in $zuordnung) {
    $postfach = Get-ADUser -Filter "mail -eq '$($eintrag.Postfach)'"
    $person = Get-ADUser -Filter "mail -eq '$($eintrag.Verantwortlich)'"
    Set-ADUser -Identity $postfach -Manager $person
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Filter "mail -eq '…'"` | Busca el objeto mediante la dirección de correo electrónico principal |
| `-Identity` | Objeto de usuario del buzón compartido |
| `-Manager` | Escribe el DN de la persona responsable en el atributo `manager` |

</details>

Para objetos exclusivamente en la nube sin AD local, `Set-MgUserManagerByRef` del SDK de PowerShell de Microsoft Graph establece el mismo valor en Entra ID.

## Consulta por el proveedor: LDAP

Los buzones compartidos se pueden identificar en AD mediante `msExchRecipientTypeDetails`: `4` representa un buzón compartido en un servidor Exchange local y `34359738368` un buzón compartido en Exchange Online (RemoteSharedMailbox). La consulta que el proveedor puede incorporar a su solución:

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

El valor de `manager` es un DN. La dirección de correo electrónico de la persona responsable se obtiene mediante una segunda consulta a este DN con el atributo `mail`. Basta una cuenta de lectura sin permisos adicionales, ya que `manager` y `mail` son legibles para usuarios autenticados. La conexión debe realizarse mediante LDAPS (puerto 636).

Para comprobar la asignación en PowerShell:

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

$verantwortlich = @{
    Name       = 'Verantwortlich'
    Expression = {
        if ($_.manager) {
            (Get-ADUser -Identity $_.manager -Properties mail).mail
        }
    }
}

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Select-Object mail, $verantwortlich |
    Sort-Object mail
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-LDAPFilter` | Filtro LDAP para los dos tipos de destinatario de buzones compartidos |
| `-Properties mail, manager` | Carga los dos atributos que `Get-ADUser` no proporciona de forma predeterminada |
| `Expression = { … }` | Resuelve el DN de `manager` en la dirección de correo electrónico de la persona responsable |

</details>

Para asignar los mensajes, el proveedor debe considerar todas las direcciones de `proxyAddresses`, no solo `mail`: un mensaje puede enviarse con una dirección secundaria del buzón.

## Consulta por el proveedor: Microsoft Graph

Si el proveedor no tiene acceso LDAP a la red interna, Microsoft Graph es una buena alternativa. Los buzones compartidos existen en Entra ID como objetos de usuario deshabilitados y `manager` está disponible allí mediante la sincronización:

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

Para ello, el registro de aplicaciones del proveedor necesita el permiso de aplicación `User.Read.All`. Graph no proporciona directamente una lista de todos los buzones compartidos, ya que Entra ID no conoce el tipo de buzón. Por tanto, el proveedor consulta específicamente la dirección del remitente del mensaje bloqueado, lo cual es suficiente para el caso de uso DLP.

## Integración en la notificación DLP

La forma en que se activa técnicamente la notificación depende de la solución DLP. Con Microsoft Purview DLP debe tenerse en cuenta lo siguiente: las notificaciones pueden enviarse adicionalmente a personas introducidas de forma fija, pero solo a destinatarios individuales, no a grupos ni listas de distribución. Por lo general, no se contempla una asignación dinámica «buzón a persona responsable». Con pocos buzones, puede plantearse una regla por buzón; con muchos buzones, se requiere un proceso posterior que recupere los incidentes, resuelva la dirección del remitente mediante LDAP o Graph e informe a la persona responsable.

Para una solución externa se aplica la misma lógica: determinar la dirección del remitente del mensaje bloqueado, encontrar el buzón mediante `mail` o `proxyAddresses`, resolver `manager` y notificar. Si falta `manager`, la solución debe informar a una dirección colectiva definida, como el servicio de soporte o protección de datos, en lugar de descartar la alerta.

## Mantenimiento operativo

Para mantener actualizada la asignación, en la práctica bastan dos reglas:

- Los nuevos buzones compartidos solo se crean con una persona responsable registrada; la solicitud incluye este campo.
- Un informe mensual enumera los buzones sin `manager` y aquellos cuya persona responsable está deshabilitada.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

Cuando una persona deja la organización, `directReports` en su objeto indica qué buzones necesitan una nueva persona responsable. Esta comprobación se puede incorporar al proceso de salida.

## Fuentes

1.  [Atributo Manager (esquema de Active Directory)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): atributo DN de valor único de las clases de personas, referencia inversa `directReports`.

2.  [Atributo Managed-By (esquema de Active Directory)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): lista de las clases para las que está definido `managedBy`; la clase User no está incluida.

3.  [Microsoft Entra Connect Sync: atributos sincronizados con Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): atributos sincronizados, incluido `manager`, así como la lista de atributos de escritura de vuelta de Exchange Hybrid.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): consulta de permisos de buzón en Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): consulta de permisos Send As en Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): artículo de soporte de Microsoft sobre la relación entre acceso total y `msExchDelegateListLink` en Exchange 2016 y 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): punto de conexión de Graph y permisos necesarios para consultar el administrador.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): destinatarios de notificaciones DLP en Microsoft Purview; solo destinatarios individuales, no grupos.
