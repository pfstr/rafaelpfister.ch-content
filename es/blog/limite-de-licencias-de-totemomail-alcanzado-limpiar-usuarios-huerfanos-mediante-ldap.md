---
title: "Límite de licencias de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP"
navTitle: "Límite de licencias alcanzado"
description: "Las cuentas de AD desactivadas permanecen en totemomail y siguen ocupando licencias. Con un acceso LDAPS comprobado y el agente de limpieza, Active Directory pasa a ser la fuente autorizada."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min de lectura"
themen:
  - totemomail
slug: "limite-de-licencias-de-totemomail-alcanzado-limpiar-usuarios-huerfanos-mediante-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: e3ae37a51d159128640441aba3bc6993b47b8c5d10968228c05281452940756d
translatedAt: 2026-09-05T08:00:55.598Z
url: https://rafaelpfister.ch/es/blog/limite-de-licencias-de-totemomail-alcanzado-limpiar-usuarios-huerfanos-mediante-ldap
translationModel: gpt-5.6-terra
---

# Límite de licencias de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP

El mensaje *«The licensed user limit has been reached»* no significa que el flujo de correo se detenga de inmediato. Indica una falta de licencias. En entornos operados durante mucho tiempo, la causa no suele ser un crecimiento repentino, sino antiguos empleados: se desactivó la cuenta de AD, pero el usuario interno en totemomail permaneció y sigue ocupando una licencia.

La solución sostenible es una sincronización LDAP periódica con Active Directory. Los siguientes pasos configuran la conexión y el agente de limpieza, y verifican toda la ruta antes de la primera ejecución en producción. Los nombres de host, DN y cuentas de servicio con `example.com` son marcadores de posición y deben adaptarse a su propio entorno.

## Qué usuarios ocupan una licencia

Totemomail distingue dos clases de usuarios. Solo los usuarios internos cuentan para el límite de licencias.

| Tipo de usuario | Descripción | Relevante para licencias |
| --- | --- | --- |
| Internal Users | Usuarios de la propia organización que envían y reciben cifrados | Sí |
| External Users | Interlocutores externos (WebMail, PDF, S/MIME, PGP) | No |


Un usuario interno se crea en cuanto se comunica por primera vez a través del gateway. Esto sucede automáticamente. Sin embargo, no se elimina automáticamente: cuando un empleado abandona la organización, normalmente se desactiva su cuenta de AD. No obstante, la entrada de totemomail permanece. Con los años se acumulan así cuentas huérfanas que siguen ocupando licencias.

### Indicador de estado

Encontrará el estado actual en **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users muestra* `*-17*`*. Los 4017 usuarios internos superan el número de plazas con licencia.*

Las líneas importantes:

-   **Internal users** (`4017`): usuarios internos creados
    
-   **Internal blocked users** (`14`): bloqueados, pero aún relevantes para licencias
    
-   **Available Users** (`-17`): licencias disponibles; un valor negativo indica falta de licencias
    

En cuanto *Available Users* baja de cero, verá la advertencia en la campana:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*«The licensed user limit has been reached.» El flujo de correo continúa, pero el mensaje permanece visible de forma permanente.*

Importante: la falta de licencias no bloquea el flujo de correo. Es una situación de licenciamiento, no técnica. Por tanto, tiene tiempo para aplicar una solución limpia, pero no debería ignorar esta situación de forma permanente.

## De la medida inmediata a la solución permanente

### Eliminación manual

Puede buscar y eliminar usuarios internos individualmente en **Internal Users**. Esto resuelve la situación urgente, pero el problema vuelve a aparecer tras unos meses. Con varios miles de cuentas, no es práctico.

### Integración LDAP con agente de limpieza

La vía sostenible es la integración con Active Directory mediante LDAP. Un agente compara periódicamente los usuarios internos con el directorio y elimina o desactiva las cuentas que ya no existen en AD. De este modo, AD se convierte en la fuente autorizada, y el proceso de offboarding en AD se encarga también de la higiene de licencias.

## Fundamentos de LDAP

| Término | Significado |
| --- | --- |
| DN (Distinguished Name) | Ruta única a un objeto, p. ej. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Raíz de la búsqueda, p. ej. `DC=corp,DC=example,DC=com` |
| Bind DN | Cuenta con la que totemomail se autentica en AD |
| Filter | Expresión de búsqueda LDAP, p. ej. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Puertos

| Puerto | Protocolo | Uso |
| --- | --- | --- |
| 389 | LDAP | sin cifrar / STARTTLS |
| 636 | LDAPS | LDAP sobre TLS |
| 3268 | Global Catalog | búsqueda en todo el forest, sin cifrar |
| 3269 | Global Catalog SSL | búsqueda en todo el forest mediante TLS |


En un entorno de dominio único, basta con el puerto 636 contra un controlador de dominio. Si opera un forest con varios dominios, solo el Global Catalog (puerto 3269) proporciona resultados para todo el forest. Un DC en el puerto 636 conoce exclusivamente los objetos de su propio dominio y responde a búsquedas fuera de su partición con un referral, un detalle que a menudo se pasa por alto en entornos multidominio.

### userAccountControl

Si una cuenta de AD está desactivada se indica en el campo de bits `userAccountControl`. La marca `ACCOUNTDISABLE` tiene el valor `2`. Con la regla de coincidencia LDAP `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) puede evaluar bits individuales:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Paso 1: cuenta de servicio en AD

Para la integración, cree una cuenta dedicada con permisos de solo lectura. No utilice una cuenta de administrador. El usuario de bind solo debe poder leer AD.

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
| `-Name` | Nombre visible y CN de la nueva cuenta |
| `-SamAccountName` | Nombre de inicio de sesión (nombre de inicio de sesión anterior a Windows 2000) |
| `-UserPrincipalName` | UPN en formato `benutzer@domäne` |
| `-Path` | OU de destino como Distinguished Name |
| `-AccountPassword` | Contraseña como SecureString; `Read-Host -AsSecureString` la solicita de forma oculta en la consola |
| `-PasswordNeverExpires $true` | La contraseña no caduca |
| `-Enabled $true` | Crear la cuenta directamente activada (por defecto estaría desactivada) |

</details>

Un usuario de dominio normal ya puede leer AD, por lo que la cuenta no necesita permisos adicionales. Para la contraseña, se recomienda un valor largo y aleatorio que guarde en su gestor de contraseñas.

Si su política de seguridad lo contempla, también puede utilizar una gMSA (Group Managed Service Account). Sin embargo, totemomail espera un Bind DN y una contraseña, por lo que en la práctica se suele usar una cuenta de servicio clásica con `PasswordNeverExpires`.

## Paso 2: comprobar la conexión LDAP en la línea de comandos

Antes de configurar nada en totemomail, debe verificar la conexión LDAP desde la línea de comandos. Este es el paso que la mayoría omite. Si `ldapsearch` funciona, la integración en totemomail también funcionará. Si la prueba falla, al menos sabrá dónde falla, en lugar de adivinar en la GUI de totemomail.

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
| `nc -v` | Salida detallada: informa si el establecimiento de conexión se realiza correctamente o falla |
| `nc -z` | Comprueba solo el establecimiento de conexión, sin enviar datos |
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
| `-Port` | Puerto TCP que se debe comprobar; aquí, 636 para LDAPS |

</details>

Si aquí no se establece ninguna conexión, tiene un problema de firewall o enrutamiento, no un problema de LDAP.

### 2.2 Comprobar el certificado TLS

En la práctica, LDAPS falla con mayor frecuencia debido al certificado. Por eso, revise lo que entrega el DC:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `s_client` | Cliente de prueba TLS de OpenSSL: establece la conexión y muestra los detalles del handshake |
| `-connect dc01.corp.example.com:636` | Host y puerto de destino del handshake TLS |
| `-showcerts` | Muestra toda la cadena de certificados entregada por el servidor, no solo el certificado de servidor |
| `</dev/null` | Cierra la entrada estándar para que `s_client` termine tras el handshake en lugar de esperar entradas |

</details>

Preste atención a dos aspectos:

-   `**subject=**` **/** `**issuer=**`: el nombre de host del certificado (CN o SAN) debe coincidir con el nombre de host mediante el que se conecta. Si se conecta mediante la dirección IP, la validación falla si el certificado solo contiene el FQDN.
    
-   `**Verify return code: 0 (ok)**`: la CA emisora debe ser conocida por totemomail. Si utiliza una CA empresarial interna, debe importar su certificado raíz o emisor en el almacén de confianza de totemomail.
    

### 2.3 Bind y búsqueda con ldapsearch

`ldapsearch` pertenece a `ldap-utils` (Debian/Ubuntu) o a `openldap-clients` (RHEL):

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
| `-H` | URI LDAP, incluido el esquema (`ldaps://`) y el puerto |
| `-D` | Bind DN |
| `-W` | Solicitar la contraseña de forma interactiva |
| `-b` | Search Base |
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
| `-x`, `-H`, `-D`, `-W`, `-b` | como en la consulta anterior: Simple Bind mediante LDAPS con Search Base |
| `sAMAccountName` | único atributo solicitado; mantiene la salida en una línea de atributo por resultado |
| `grep -c sAMAccountName` | cuenta las líneas con este atributo y, por tanto, las cuentas encontradas |

</details>

### 2.4 Herramientas en Windows

`**ldp.exe**` es la herramienta gráfica LDAP de Microsoft, disponible en todos los DC y como parte de RSAT. Conéctese mediante `Connection → Connect` (host, puerto 636, activar SSL), autentíquese con `Connection → Bind` y navegue por el árbol de directorios mediante `View → Tree` usando el Base DN.

Sin RSAT, puede hacerlo en PowerShell mediante ADSI-Searcher:

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
| `SearchRoot` | Punto de inicio de la búsqueda como ruta ADSI: servidor más Base DN |
| `FindOne()` | devuelve el primer resultado; `.Properties` muestra sus atributos |

</details>

Con RSAT y el módulo de AD es más breve:

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
| `-Filter` | Filtro en sintaxis de PowerShell; aquí, solo cuentas activadas |
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

Solo cuando una de estas pruebas se ejecute correctamente, continúe en totemomail.

## Paso 3: configurar la conexión LDAP en totemomail

Cree el directorio LDAP en la GUI de administración, en **Directories / LDAP**. Aplique exactamente los valores que probó anteriormente:

| Campo | Valor de ejemplo |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Contraseña de la cuenta de servicio |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativamente `mail` o `userPrincipalName`) |


Si utiliza LDAPS con una CA interna, debe importar su certificado raíz o emisor en el almacén de confianza de totemomail. De lo contrario, el handshake TLS fallará con «certificate verify failed», incluso si `ldapsearch` con `-x` funcionó antes, ya que `ldapsearch` no valida estrictamente el certificado de esta forma.

Después de guardar, ejecute la prueba de conexión integrada. Esta confirma el bind.

## Paso 4: crear el agente de limpieza

En **Maintenance → Agents → Add**, cree un agente de tipo **«Check presence of internal users in directories»**.

### 4.1 Pestaña «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Aquí el agente se ejecuta mensualmente el día 1 a las 00:30. Mediante «Agent runs on server» se define el nodo ejecutor en el clúster.*

| Campo | Recomendación | Motivo |
| --- | --- | --- |
| The agent should run | `monthly`, día `1`, `00:30` | fuera del horario laboral; una vez al mes basta para la higiene de licencias |
| Agent enabled | activar solo tras la ejecución de prueba | véase el paso 5 |
| Produced emails are not sent but cached in a queue | activar para la primera ejecución | ejecución de prueba sin envío de correos |
| Agent runs on server | un nodo del clúster | el trabajo debe ejecutarse solo en un nodo |


### 4.2 Pestaña «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Los parámetros controlan qué usuarios internos se eliminan, se desactivan o se crean.*

| Parámetro | Recomendación | Efecto |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | activar | Se eliminan los usuarios internos inactivos sin entrada en AD. Este es el núcleo de la limpieza de licencias. |
| Delete blocked users that are not found in a directory? | activar | También se eliminan los usuarios internos bloqueados sin entrada en AD |
| Delete administrators? | dejar vacío | Las cuentas de administrador no deben eliminarse automáticamente |
| Only set users found in the defined groups to inactive | opcional | Los usuarios se establecen como inactivos en lugar de eliminarlos. Un `!` inicial excluye de la acción a los miembros del grupo indicado. Separe los DN con `;`. |
| Additional filter attribute | opcional | atributo adicional para la búsqueda en el directorio, p. ej. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | dejar vacío | solo se aplica si se ha definido el parámetro de grupo |
| Create users based on group membership | opcional | crea nuevos usuarios internos según la pertenencia a grupos de AD. Separe varios grupos con `;`. |


La negación en el campo *«Only set users found in the defined groups to inactive»* funciona mediante un `!` antes de un DN de grupo. Los miembros de este grupo quedan excluidos de la acción:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

En este ejemplo, establece como inactivos a los usuarios del grupo *Mitarbeiter* cuando no están presentes en AD, mientras que los miembros del grupo *Dienstkonten* no se modifican.

## Paso 5: ejecución de prueba y validación

No ejecute el agente contra los datos de producción sin una ejecución de prueba. Proceda en este orden:

1.  **Active el modo de cola** mediante la opción *«Produced emails are not sent but cached in a queue»*. El agente determina las acciones previstas sin enviar correos.
    
2.  **Ejecútelo manualmente** y evalúe el registro del agente: ¿cuántos usuarios se verían afectados y hay cuentas inesperadas, como buzones funcionales, en la lista?
    
3.  **Compruebe la plausibilidad con** `**ldapsearch**`: el número de usuarios no encontrados en AD debe coincidir con su consulta LDAP manual.
    
4.  Si el resultado es correcto, desactive el modo de cola, active *Agent enabled* y habilite la programación.
    
5.  Tras la primera ejecución en producción, vuelva a comprobar **Settings → Overview → User Information**. *Available Users* debería volver a estar en valores positivos.
    

## Solución de problemas

| Síntoma | Causa | Medida |
| --- | --- | --- |
| `Can't contact LDAP server` | Puerto 636 inaccesible / host incorrecto | comprobar con `Test-NetConnection` o `nc -vz`, revisar el firewall |
| `Invalid credentials (49)` | Bind DN o contraseña incorrectos | indicar el Bind DN como DN completo, no como `user@domain` |
| `certificate verify failed` | CA desconocida en el almacén de confianza | importar la CA raíz o emisora |
| Discordancia de nombre de host en TLS | Conexión mediante IP en lugar de FQDN | utilizar como host el CN/SAN del certificado |
| `Referral (10)` | La búsqueda cruza el límite de dominio | utilizar Global Catalog en el puerto 3269 en lugar del DC en 636 |
| No se detectan los usuarios desactivados | Falta el filtro `userAccountControl` | utilizar la regla de coincidencia de bits `:1.2.840.113556.1.4.803:=2` |
| El agente elimina demasiadas cuentas | Filtro demasiado amplio / Base DN incorrecto | probar en modo de cola, restringir el Base DN |


Con la marca `-d 1`, `ldapsearch` entrega la salida de depuración del establecimiento de conexión:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-d 1` | Nivel de depuración 1: registra el proceso de establecimiento de conexión, incluido el handshake TLS, en stderr |
| `-x`, `-H` | Simple Bind y URI LDAP como en las consultas anteriores |

</details>

Así verá si falla el handshake TLS o si falla el bind posteriormente. La GUI de totemomail no muestra esta distinción bajo su mensaje de error genérico.

## seguridad

-   **Cuenta de servicio de solo lectura.** El usuario de bind necesita exclusivamente permisos de lectura.
    
-   **LDAPS en lugar de LDAP.** Utilice el puerto 636 o 3269. LDAP en el puerto 389 transmite la contraseña de bind en texto plano. Además, Active Directory exige cada vez más conexiones protegidas mediante LDAP Channel Binding y Signing.
    
-   **Rotación de contraseñas.** `PasswordNeverExpires` es operativo y práctico. Documente la cuenta y rote la contraseña según el plan.
    
-   **Monitorización.** Supervise *Available Users* (idealmente mediante alertas), en lugar de esperar la advertencia de la campana.
    
-   **Primera ejecución en modo de cola.** Un filtro incorrecto puede afectar a un gran número de cuentas.
    

## El procedimiento seguro en cuatro pasos

Alcanzar el límite de licencias no es un fallo técnico, sino la consecuencia de un proceso de offboarding inexistente. La solución sostenible es la sincronización periódica con Active Directory como fuente autorizada. El orden es decisivo:

1.  Verificar la conexión LDAP en la línea de comandos (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configurar la conexión en totemomail
    
3.  Validar el agente en modo de cola
    
4.  Poner el agente en producción
    

Quien siga este orden resolverá el problema de licencias actual y evitará que vuelva a producirse.

## Fuentes

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): documentación del producto sobre totemomail (modelo de licencias, integración LDAP y agente de limpieza); Kiteworks sigue desarrollando la tecnología como Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): significado de las marcas, entre ellas `ACCOUNTDISABLE` (0x0002) y `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): filtro LDAP bit a bit mediante la OID de regla de coincidencia `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (página de manual)](https://www.openldap.org/software/man.cgi?query=ldapsearch): opciones de invocación (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) para bind y búsqueda.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): puertos LDAP 389/636 y puertos Global Catalog 3268/3269.
