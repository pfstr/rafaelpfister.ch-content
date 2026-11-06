---
title: "Límite de licencias de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP"
navTitle: "Límite de licencias alcanzado"
description: "Las cuentas de AD desactivadas permanecen en totemomail y siguen ocupando licencias. Con un acceso LDAPS verificado y el agente de limpieza, Active Directory se convierte en la fuente principal."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min de lectura"
themen:
  - totemomail
slug: "limite-de-licencias-de-totemomail-alcanzado-limpiar-usuarios-huerfanos-mediante-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: 3d7892da089472f95cc44e06e8ecb8654f53c40099d6ff970060102346823159
translatedAt: 2026-11-06T03:26:59.973Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/limite-de-licencias-de-totemomail-alcanzado-limpiar-usuarios-huerfanos-mediante-ldap
---

# Límite de licencias de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP

El mensaje *«The licensed user limit has been reached»* no significa que el flujo de correo se detenga inmediatamente. Indica una falta de licencias. En entornos con muchos años de funcionamiento, la causa no suele ser un crecimiento repentino, sino antiguos empleados: la cuenta de AD se desactivó, pero el usuario interno en totemomail permaneció y sigue ocupando una licencia.

La solución sostenible es una sincronización LDAP periódica con Active Directory. Los siguientes pasos configuran la conexión y el agente de limpieza, y comprueban toda la ruta antes de la primera ejecución en producción. Los nombres de host, DN y cuentas de servicio con `example.com` son marcadores de posición y deben adaptarse al entorno propio.

## Qué usuarios ocupan una licencia

Totemomail distingue dos clases de usuarios. Solo los usuarios internos cuentan para el límite de licencias.

| Tipo de usuario | Descripción | Relevante para licencias |
| --- | --- | --- |
| Internal Users | Usuarios de la propia organización que envían y reciben correo cifrado | Sí |
| External Users | Interlocutores externos (WebMail, PDF, S/MIME, PGP) | No |


Un usuario interno se crea en cuanto se comunica por primera vez a través del gateway. Esto sucede automáticamente. En cambio, no se elimina automáticamente: cuando un empleado deja la organización, normalmente se desactiva la cuenta de AD. Sin embargo, la entrada de totemomail permanece. Con los años, se acumulan cuentas huérfanas que siguen ocupando licencias.

### Indicador de estado

Encontrará el estado actual en **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users está en* `*-17*`*. Los 4017 usuarios internos superan el número de plazas licenciadas disponibles.*

Las líneas importantes:

-   **Internal users** (`4017`): usuarios internos creados
    
-   **Internal blocked users** (`14`): bloqueados, pero aún relevantes para las licencias
    
-   **Available Users** (`-17`): licencias disponibles; un valor negativo significa falta de licencias
    

En cuanto *Available Users* cae por debajo de cero, verá la advertencia en la campana:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*«The licensed user limit has been reached.» El flujo de correo continúa, pero el mensaje permanece visible de forma permanente.*

Importante: la falta de licencias no bloquea el flujo de correo. Es una situación de licencias, no técnica. Por tanto, hay tiempo para una solución limpia, pero no debería ignorar esta situación de forma permanente.

## De la medida inmediata a la solución permanente

### Eliminación manual

Puede buscar y eliminar usuarios internos individualmente en **Internal Users**. Esto resuelve la situación inmediata, pero el problema vuelve tras unos meses. Con varios miles de cuentas, no es práctico.

### Integración LDAP con agente de limpieza

La vía sostenible es conectar Active Directory mediante LDAP. Un agente compara periódicamente los usuarios internos con el directorio y elimina o desactiva las cuentas que ya no existen en AD. De este modo, AD se convierte en la fuente principal y el proceso de baja en AD se encarga al mismo tiempo de la higiene de licencias.

## Fundamentos de LDAP

| Término | Significado |
| --- | --- |
| DN (Distinguished Name) | Ruta única a un objeto, p. ej., `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Raíz de la búsqueda, p. ej., `DC=corp,DC=example,DC=com` |
| Bind DN | Cuenta con la que totemomail se autentica en AD |
| Filter | Expresión de búsqueda LDAP, p. ej., `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Puertos

| Puerto | Protocolo | Uso |
| --- | --- | --- |
| 389 | LDAP | sin cifrar / STARTTLS |
| 636 | LDAPS | LDAP mediante TLS |
| 3268 | Global Catalog | búsqueda en todo el bosque, sin cifrar |
| 3269 | Global Catalog SSL | búsqueda en todo el bosque mediante TLS |


En un entorno de un solo dominio, basta con el puerto 636 contra un controlador de dominio. Si opera un bosque con varios dominios, solo el Global Catalog (puerto 3269) proporciona resultados de todo el bosque. Un DC en el puerto 636 conoce exclusivamente los objetos de su propio dominio y responde a las búsquedas fuera de su partición con una referencia, un detalle que a menudo se pasa por alto en entornos con varios dominios.

### userAccountControl

Si una cuenta de AD está desactivada se indica en el campo de bits `userAccountControl`. La marca `ACCOUNTDISABLE` tiene el valor `2`. Mediante la regla de coincidencia LDAP `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) puede evaluar bits individuales:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Paso 1: cuenta de servicio en AD

Para la conexión, cree una cuenta dedicada con permisos de solo lectura. No use una cuenta de administrador para ello. El usuario de enlace solo debe poder leer AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Passwort") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Name` | Nombre para mostrar y CN de la nueva cuenta |
| `-SamAccountName` | Nombre de inicio de sesión (nombre de inicio de sesión anterior a Windows 2000) |
| `-UserPrincipalName` | UPN en formato `benutzer@domäne` |
| `-Path` | OU de destino como Distinguished Name |
| `-AccountPassword` | Contraseña como SecureString; `Read-Host -AsSecureString` la solicita de forma oculta en la consola |
| `-PasswordNeverExpires $true` | La contraseña no caduca |
| `-Enabled $true` | Crea la cuenta activada directamente (por defecto estaría desactivada) |

</details>

Un usuario de dominio normal ya puede leer AD, por lo que la cuenta no necesita permisos adicionales. Para la contraseña, se recomienda un valor largo y aleatorio que guarde en su gestor de contraseñas.

Si su política de seguridad lo prevé, también puede utilizar una gMSA (Group Managed Service Account). Sin embargo, totemomail espera Bind DN y contraseña, por lo que en la práctica suele utilizarse una cuenta de servicio clásica con `PasswordNeverExpires`.

## Paso 2: comprobar la conexión LDAP en la línea de comandos

Antes de configurar nada en totemomail, debe verificar la conexión LDAP en la línea de comandos. Este es el paso que la mayoría omite. Si `ldapsearch` funciona, la conexión en totemomail también funcionará. Si la prueba falla, al menos sabrá en qué punto falla, en lugar de adivinar en la interfaz gráfica de totemomail.

### 2.1 Comprobación de puertos

En Linux, por ejemplo desde la appliance de totemomail:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `nc -v` | Salida detallada: informa si la conexión se establece correctamente o falla |
| `nc -z` | Comprueba únicamente el establecimiento de la conexión, sin enviar datos |
| `dc01.corp.example.com 636` | Host y puerto de destino de la comprobación |
| `nmap -p 389,636,3268,3269` | Lista de puertos TCP que se deben comprobar |
| `dc01.corp.example.com` | Host de destino del escaneo de puertos |

</details>

En Windows con PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-ComputerName` | Host de destino de la comprobación de conexión |
| `-Port` | Puerto TCP que se debe comprobar, aquí 636 para LDAPS |

</details>

Si no se establece conexión aquí, tiene un problema de firewall o de enrutamiento, no un problema de LDAP.

### 2.2 Comprobar el certificado TLS

En la práctica, LDAPS suele fallar por el certificado. Por ello, revise lo que entrega el DC:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `s_client` | Cliente de prueba TLS de OpenSSL: establece la conexión y muestra los detalles del handshake |
| `-connect dc01.corp.example.com:636` | Host y puerto de destino del handshake TLS |
| `-showcerts` | Muestra toda la cadena de certificados entregada por el servidor, no solo el certificado del servidor |
| `</dev/null` | Cierra la entrada estándar para que `s_client` termine tras el handshake, en lugar de esperar entradas |

</details>

Preste atención a dos aspectos:

-   `**subject=**` **/** `**issuer=**`: El nombre de host en el certificado (CN o SAN) debe coincidir con el nombre de host mediante el que se conecta. Si se conecta mediante la dirección IP, la comprobación fallará si el certificado solo contiene el FQDN.
    
-   `**Verify return code: 0 (ok)**`: La CA emisora debe ser conocida por totemomail. En el caso de una CA empresarial interna, debe importar su certificado raíz o emisor en el almacén de confianza de totemomail.
    
### 2.3 Enlace y búsqueda con ldapsearch

`ldapsearch` pertenece a `ldap-utils` (Debian/Ubuntu) o `openldap-clients` (RHEL):

```bash
ldapsearch -x \
  -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" \
  -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(sAMAccountName=jdoe))" \
  dn sAMAccountName mail userAccountControl
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Indicador | Significado |
| --- | --- |
| `-x` | Autenticación simple (Bind DN y contraseña) |
| `-H` | URI LDAP incluido esquema (`ldaps://`) y puerto |
| `-D` | Bind DN |
| `-W` | Solicita la contraseña de forma interactiva |
| `-b` | Base de búsqueda |
| después | Filtro y, a continuación, los atributos que se devolverán |

</details>

Si la consulta devuelve el objeto con sus atributos, la conexión está establecida. Puede determinar cuántas cuentas están desactivadas en AD mediante el filtro de bits:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-x`, `-H`, `-D`, `-W`, `-b` | como en la consulta anterior: enlace simple mediante LDAPS con base de búsqueda |
| `sAMAccountName` | único atributo solicitado; limita la salida a una línea de atributo por resultado |
| `grep -c sAMAccountName` | cuenta las líneas con este atributo y, por tanto, las cuentas encontradas |

</details>

### 2.4 Herramientas en Windows

`**ldp.exe**` es la herramienta gráfica LDAP de Microsoft, presente en todos los DC y parte de RSAT. Conéctese mediante `Connection → Connect` (host, puerto 636, activar SSL), autentíquese con `Connection → Bind` y navegue por el árbol de directorios mediante `View → Tree` utilizando el Base DN.

Sin RSAT, puede hacerlo en PowerShell mediante el buscador ADSI:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `[adsisearcher]"(…)"` | crea un `DirectorySearcher` con el filtro LDAP indicado |
| `SearchRoot` | punto inicial de la búsqueda como ruta ADSI: servidor más Base DN |
| `FindOne()` | devuelve el primer resultado; `.Properties` muestra sus atributos |

</details>

Con RSAT y el módulo AD es más breve:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server` | Controlador de dominio contra el que se ejecuta la consulta |
| `-SearchBase` | Raíz de la búsqueda como Distinguished Name |
| `-Filter` | Filtro en sintaxis de PowerShell; aquí solo cuentas activadas |
| `Measure-Object` | cuenta los objetos devueltos en lugar de enumerarlos |

</details>

De forma clásica mediante `dsquery`, disponible en todos los DC:

```bash
dsquery user -disabled -limit 0
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `user` | Tipo de objeto de la búsqueda: cuentas de usuario |
| `-disabled` | Solo cuentas desactivadas |
| `-limit 0` | Sin límite de resultados (predeterminado: 100) |

</details>

Solo cuando una de estas pruebas se complete correctamente debe continuar en totemomail.

## Paso 3: configurar la conexión LDAP en totemomail

Cree el directorio LDAP en la interfaz gráfica de administración, en **Directories / LDAP**. Use exactamente los valores que probó previamente:

| Campo | Valor de ejemplo |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Contraseña de la cuenta de servicio |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativamente `mail` o `userPrincipalName`) |


Si utiliza LDAPS con una CA interna, debe importar su certificado raíz o emisor en el almacén de confianza de totemomail. De lo contrario, el handshake TLS fallará con «certificate verify failed», aunque `ldapsearch` con `-x` haya funcionado antes: en esta forma, `ldapsearch` no verifica estrictamente el certificado.

Después de guardar, ejecute la prueba de conexión integrada. Esta confirma el enlace.

## Paso 4: crear el agente de limpieza

En **Maintenance → Agents → Add**, cree un agente de tipo **«Check presence of internal users in directories»**.

### 4.1 Pestaña «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Aquí el agente se ejecuta mensualmente el día 1 a las 00:30. Mediante «Agent runs on server» se define el nodo ejecutor del clúster.*

| Campo | Recomendación | Motivo |
| --- | --- | --- |
| The agent should run | `monthly`, día `1`, `00:30` | fuera del horario laboral; una vez al mes es suficiente para la higiene de licencias |
| Agent enabled | activarlo solo después de la ejecución de prueba | véase el paso 5 |
| Produced emails are not sent but cached in a queue | activarlo para la primera ejecución | ejecución de prueba sin enviar correos |
| Agent runs on server | un nodo del clúster | el trabajo debe ejecutarse solo en un nodo |


### 4.2 Pestaña «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Los parámetros controlan qué usuarios internos se eliminan, desactivan o crean de nuevo.*

| Parámetro | Recomendación | Efecto |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | activar | Se eliminan los usuarios internos inactivos sin entrada en AD. Este es el núcleo de la limpieza de licencias. |
| Delete blocked users that are not found in a directory? | activar | También se eliminan los usuarios internos bloqueados sin entrada en AD |
| Delete administrators? | dejar vacío | Las cuentas de administrador no deben eliminarse automáticamente |
| Only set users found in the defined groups to inactive | opcional | Los usuarios se establecen como inactivos en lugar de eliminarse. Un `!` delante excluye de la acción a los miembros del grupo indicado. Separe los DN con `;`. |
| Additional filter attribute | opcional | atributo adicional para la búsqueda en el directorio, p. ej., `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | dejar vacío | solo se aplica si se ha definido el parámetro de grupo |
| Create users based on group membership | opcional | crea nuevos usuarios internos según la pertenencia a grupos de AD. Separe varios grupos con `;`. |


La negación en el campo *«Only set users found in the defined groups to inactive»* funciona mediante un `!` delante de un DN de grupo. Los miembros de este grupo quedan excluidos de la acción:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

En este ejemplo, los usuarios del grupo *Mitarbeiter* se establecen como inactivos cuando no están presentes en AD, mientras que los miembros del grupo *Dienstkonten* permanecen sin cambios.

## Paso 5: ejecución de prueba y validación

No ejecute el agente contra el entorno de producción sin una ejecución de prueba. Proceda en este orden:

1.  **Active el modo de cola** mediante la opción *«Produced emails are not sent but cached in a queue»*. El agente determina las acciones previstas sin enviar correos.
    
2.  **Ejecútelo manualmente** y evalúe el registro del agente: ¿cuántos usuarios se verían afectados y hay cuentas inesperadas, como buzones funcionales, en la lista?
    
3.  **Compruebe la coherencia con** `**ldapsearch**`: el número de usuarios no encontrados en AD debe coincidir con su consulta LDAP manual.
    
4.  Si el resultado es correcto, desactive el modo de cola, active *Agent enabled* y habilite la programación.
    
5.  Tras la primera ejecución en producción, vuelva a comprobar **Settings → Overview → User Information**. *Available Users* debería volver a estar en valores positivos.
    
## Solución de problemas

| Síntoma | Causa | Medida |
| --- | --- | --- |
| `Can't contact LDAP server` | Puerto 636 inaccesible / host incorrecto | comprobar con `Test-NetConnection` o `nc -vz`, revisar el firewall |
| `Invalid credentials (49)` | Bind DN o contraseña incorrectos | indicar el Bind DN como DN completo, no como `user@domain` |
| `certificate verify failed` | CA desconocida para el almacén de confianza | importar la CA raíz o emisora |
| Discordancia de nombre de host en TLS | Conexión mediante IP en lugar de FQDN | usar como host el CN/SAN del certificado |
| `Referral (10)` | La búsqueda supera el límite de dominio | usar Global Catalog en el puerto 3269 en lugar de DC en 636 |
| No se detectan usuarios desactivados | falta el filtro `userAccountControl` | usar la regla de coincidencia de bits `:1.2.840.113556.1.4.803:=2` |
| El agente elimina demasiadas cuentas | Filtro demasiado amplio / Base DN incorrecto | probar en modo de cola, restringir Base DN |


Con la marca `-d 1`, `ldapsearch` proporciona la salida de depuración del establecimiento de conexión:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-d 1` | Nivel de depuración 1: registra el proceso de establecimiento de conexión, incluido el handshake TLS, en stderr |
| `-x`, `-H` | Enlace simple y URI LDAP como en las consultas anteriores |

</details>

Así podrá ver si falla el handshake TLS o solo el enlace. La interfaz gráfica de totemomail no le muestra esta distinción detrás de su mensaje de error genérico.

## Seguridad

-   **Cuenta de servicio de solo lectura.** El usuario de enlace necesita exclusivamente permisos de lectura.
    
-   **LDAPS en lugar de LDAP.** Use el puerto 636 o 3269. LDAP en el puerto 389 transmite la contraseña de enlace en texto plano. Además, Active Directory exige cada vez más conexiones seguras mediante LDAP Channel Binding y Signing.
    
-   **Rotación de contraseñas.** `PasswordNeverExpires` es viable operativamente. Documente la cuenta y rote la contraseña según un plan.
    
-   **Monitorización.** Supervise *Available Users* (idealmente mediante alertas), en lugar de esperar a la advertencia de la campana.
    
-   **Primera ejecución en modo de cola.** Un filtro erróneo puede afectar a un gran número de cuentas.
    
## El proceso seguro en cuatro pasos

Alcanzar el límite de licencias no es un defecto técnico, sino la consecuencia de un proceso de baja inexistente. La solución sostenible es la comparación periódica con Active Directory como fuente principal. El orden es decisivo:

1.  Verificar la conexión LDAP en la línea de comandos (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configurar la conexión en totemomail
    
3.  Validar el agente en modo de cola
    
4.  Poner el agente en producción
    

Quien siga este orden resolverá el problema inmediato de licencias e impedirá que vuelva a producirse.

## Historial de cambios

- **25 de octubre de 2026:** Fuente sustituida: la anterior página de documentación en totemo.com redirige a kiteworks.com; ahora se hace referencia a la página de Kiteworks sobre la continuación de totemo como Email Protection Gateway.

## Fuentes

1.  [Kiteworks – «Unlocking the Power of Kiteworks Email Protection Gateway (EPG) with totemo Integration»](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): adquisición de totemo por Kiteworks y continuación de la tecnología como Email Protection Gateway (EPG); la anterior página de descargas y documentación en totemo.com redirige ahora a la página principal de kiteworks.com.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): significado de las marcas, entre otras `ACCOUNTDISABLE` (0x0002) y `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): filtro LDAP bit a bit mediante la OID de regla de coincidencia `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (página de manual)](https://www.openldap.org/software/man.cgi?query=ldapsearch): opciones de llamada (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) para enlace y búsqueda.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): puertos LDAP 389/636 y puertos de Global Catalog 3268/3269.
