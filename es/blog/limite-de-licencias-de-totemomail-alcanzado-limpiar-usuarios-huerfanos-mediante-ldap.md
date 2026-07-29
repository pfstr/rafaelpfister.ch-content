---
title: "Límite de licencias de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP"
navTitle: "Límite de licencias alcanzado"
description: "Las cuentas de AD desactivadas permanecen en totemomail y siguen ocupando licencias. Con un acceso LDAPS verificado y el agente de limpieza, Active Directory se convierte en la fuente principal."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min de lectura"
themen:
  - "totemomail"
slug: "limite-de-licencias-de-totemomail-alcanzado-limpiar-usuarios-huerfanos-mediante-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
url: "https://rafaelpfister.ch/es/blog/limite-de-licencias-de-totemomail-alcanzado-limpiar-usuarios-huerfanos-mediante-ldap"
---

# Límite de licencias de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP

El mensaje *«The licensed user limit has been reached»* no significa que el flujo de correo se detenga inmediatamente. Indica una infralicencia. En entornos operativos desde hace tiempo, la causa no suele ser un crecimiento repentino, sino antiguos empleados: la cuenta de AD se desactivó, pero el usuario interno en totemomail permaneció y sigue ocupando una licencia.

La solución sostenible es una sincronización LDAP periódica con Active Directory. Los siguientes pasos configuran la conexión y el agente de limpieza, y verifican toda la ruta antes de la primera ejecución en producción. Los nombres de host, DN y cuentas de servicio con `example.com` son marcadores de posición y deben adaptarse a su propio entorno.

## Qué usuarios ocupan una licencia

Totemomail distingue dos clases de usuarios. Solo los usuarios internos cuentan para el límite de licencias.

| Tipo de usuario | Descripción | Relevante para la licencia |
| --- | --- | --- |
| Internal Users | Usuarios de la propia organización que envían y reciben cifrados | Sí |
| External Users | Interlocutores externos (WebMail, PDF, S/MIME, PGP) | No |


Un usuario interno se crea en cuanto se comunica por primera vez a través del gateway. Esto sucede automáticamente. Sin embargo, no se elimina automáticamente: cuando un empleado abandona la organización, normalmente se desactiva la cuenta de AD. No obstante, la entrada de totemomail permanece. Con los años se acumulan así cuentas huérfanas que siguen ocupando licencias.

### Indicador de estado

Encontrará el estado actual en **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users está en* `*-17*`*. Los 4017 usuarios internos disponen de un número menor de licencias.*

Las líneas importantes:

-   **Internal users** (`4017`): usuarios internos creados
    
-   **Internal blocked users** (`14`): bloqueados, pero siguen siendo relevantes para la licencia
    
-   **Available Users** (`-17`): licencias disponibles; un valor negativo indica infralicencia
    

En cuanto *Available Users* baja de cero, verá la advertencia en la campana:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*«The licensed user limit has been reached.» El flujo de correo sigue funcionando, pero el mensaje permanece visible de forma permanente.*

Importante: la infralicencia no bloquea el flujo de correo. Es una situación legal de licencias, no técnica. Por tanto, dispone de tiempo para una solución adecuada, pero no debería ignorar esta situación permanentemente.

## De la medida inmediata a la solución permanente

### Eliminación manual

Puede buscar y eliminar usuarios internos individualmente en **Internal Users**. Esto soluciona la situación urgente, pero el problema volverá al cabo de unos meses. Con varios miles de cuentas, este método no resulta práctico.

### Integración LDAP con agente de limpieza

La vía sostenible es la integración con Active Directory mediante LDAP. Un agente compara periódicamente los usuarios internos con el directorio y elimina o desactiva las cuentas que ya no existen en AD. De este modo, AD se convierte en la fuente principal y su proceso de baja en AD se encarga también de la higiene de licencias.

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


En un entorno de dominio único, basta con el puerto 636 contra un controlador de dominio. Si opera un bosque con varios dominios, solo el Global Catalog (puerto 3269) proporciona resultados de todo el bosque. Un DC en el puerto 636 conoce exclusivamente los objetos de su propio dominio y responde a búsquedas fuera de su partición con un referral, un detalle que suele pasarse por alto en entornos multidominio.

### userAccountControl

El estado de desactivación de una cuenta de AD se encuentra en el campo de bits `userAccountControl`. La marca `ACCOUNTDISABLE` tiene el valor `2`. Mediante la regla de coincidencia LDAP `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) puede evaluar bits individuales:

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
  -AccountPassword (Read-Host -AsSecureString "Contraseña") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

Un usuario de dominio normal ya puede leer AD, por lo que la cuenta no necesita permisos adicionales. Para la contraseña se recomienda un valor largo y aleatorio que guarde en su gestor de contraseñas.

Si su política de seguridad lo contempla, también puede utilizar una gMSA (Group Managed Service Account). Sin embargo, totemomail espera un Bind DN y una contraseña, por lo que en la práctica suele utilizarse una cuenta de servicio clásica con `PasswordNeverExpires`.

## Paso 2: verificar la conexión LDAP en la línea de comandos

Antes de configurar nada en totemomail, debería verificar la conexión LDAP en la línea de comandos. Este es el paso que la mayoría omite. Si `ldapsearch` funciona, la integración en totemomail también funcionará. Si la prueba falla, al menos sabrá dónde está el problema, en vez de adivinar en la interfaz gráfica de totemomail.

### 2.1 Comprobación de puertos

En Linux, por ejemplo desde la appliance de totemomail:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

En Windows con PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

Si no se puede establecer conexión, tiene un problema de firewall o enrutamiento, no de LDAP.

### 2.2 Verificar el certificado TLS

En la práctica, LDAPS falla con mayor frecuencia debido al certificado. Por ello, revise lo que entrega el DC:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

Preste atención a dos aspectos:

-   `**subject=**` **/** `**issuer=**`: el nombre de host del certificado (CN o SAN) debe coincidir con el nombre de host mediante el que se conecta. Si se conecta mediante la dirección IP, la verificación fallará si el certificado solo contiene el FQDN.
    
-   `**Verify return code: 0 (ok)**`: la CA emisora debe ser conocida por totemomail. Si utiliza una CA empresarial interna, debe importar su certificado raíz o emisor en el truststore de totemomail.
    

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

| Marca | Significado |
| --- | --- |
| `-x` | autenticación simple (Bind DN y contraseña) |
| `-H` | URI LDAP incluido el esquema (`ldaps://`) y el puerto |
| `-D` | Bind DN |
| `-W` | solicitar la contraseña de forma interactiva |
| `-b` | base de búsqueda |
| después | filtro y, a continuación, los atributos que se devolverán |


Si la consulta devuelve el objeto con sus atributos, la conexión está establecida. Puede determinar cuántas cuentas están desactivadas en AD mediante el filtro de bits:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

### 2.4 Herramientas en Windows

`**ldp.exe**` es la herramienta gráfica LDAP de Microsoft, disponible en todos los DC y parte de RSAT. Conéctese mediante `Connection → Connect` (host, puerto 636, activar SSL), autentíquese con `Connection → Bind` y navegue por el árbol del directorio mediante `View → Tree` con el Base DN.

Sin RSAT, puede lograrlo en PowerShell mediante el buscador ADSI:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

Con RSAT y el módulo AD es más corto:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

De forma clásica mediante `dsquery`, disponible en todos los DC:

```bash
dsquery user -disabled -limit 0
```

Solo cuando una de estas pruebas se complete correctamente, continúe en totemomail.

## Paso 3: configurar la conexión LDAP en totemomail

Cree el directorio LDAP en la interfaz gráfica de administración, en **Directories / LDAP**. Utilice exactamente los valores que probó antes:

| Campo | Valor de ejemplo |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Contraseña de la cuenta de servicio |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (como alternativa `mail` o `userPrincipalName`) |


Si utiliza LDAPS con una CA interna, debe importar su certificado raíz o emisor en el truststore de totemomail. De lo contrario, el handshake TLS fallará con «certificate verify failed», aunque `ldapsearch` con `-x` haya funcionado previamente: `ldapsearch` no verifica estrictamente el certificado de esta forma.

Después de guardar, ejecute la prueba de conexión integrada. Esta confirma el bind.

## Paso 4: crear el agente de limpieza

En **Maintenance → Agents → Add**, cree un agente de tipo **«Check presence of internal users in directories»**.

### 4.1 Pestaña «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Aquí, el agente se ejecuta mensualmente el día 1 a las 00:30. Mediante «Agent runs on server» se define el nodo ejecutor del clúster.*

| Campo | Recomendación | Motivo |
| --- | --- | --- |
| The agent should run | `monthly`, día `1`, `00:30` | fuera del horario laboral; una ejecución mensual basta para la higiene de licencias |
| Agent enabled | activar solo después de la ejecución de prueba | véase el paso 5 |
| Produced emails are not sent but cached in a queue | activar para la primera ejecución | ejecución de prueba sin envío de correos |
| Agent runs on server | un nodo del clúster | el trabajo debe ejecutarse solo en un nodo |


### 4.2 Pestaña «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Los parámetros controlan qué usuarios internos se eliminan, desactivan o crean.*

| Parámetro | Recomendación | Efecto |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | activar | Se eliminan los usuarios internos inactivos sin entrada en AD. Este es el núcleo de la limpieza de licencias. |
| Delete blocked users that are not found in a directory? | activar | También se eliminan los usuarios internos bloqueados sin entrada en AD |
| Delete administrators? | dejar vacío | Las cuentas de administrador no deben eliminarse automáticamente |
| Only set users found in the defined groups to inactive | opcional | Los usuarios se marcan como inactivos en lugar de eliminarse. Un `!` delante excluye a los miembros del grupo indicado. Separe los DN con `;`. |
| Additional filter attribute | opcional | atributo adicional para la búsqueda en el directorio, p. ej., `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | dejar vacío | solo se aplica si se ha configurado el parámetro de grupo |
| Create users based on group membership | opcional | crea nuevos usuarios internos según la pertenencia a grupos de AD. Separe varios grupos con `;`. |


La negación en el campo *«Only set users found in the defined groups to inactive»* funciona mediante un `!` delante de un DN de grupo. Los miembros de este grupo quedan excluidos de la acción:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

En este ejemplo, los usuarios del grupo *Empleados* se marcan como inactivos si no están en AD, mientras que los miembros del grupo *Cuentas de servicio* no se modifican.

## Paso 5: ejecución de prueba y validación

No ejecute el agente contra el inventario de producción sin una prueba previa. En su lugar, siga este orden:

1.  **Active el modo de cola**: mediante la opción *«Produced emails are not sent but cached in a queue»*. El agente determina las acciones previstas sin enviar correos.
    
2.  **Ejecútelo manualmente** y evalúe el registro del agente: ¿cuántos usuarios se verían afectados y hay cuentas inesperadas, como buzones funcionales, en la lista?
    
3.  **Compruebe la plausibilidad con** `**ldapsearch**`: el número de usuarios no encontrados en AD debe coincidir con su consulta LDAP manual.
    
4.  Si el resultado es correcto, desactive el modo de cola, active *Agent enabled* y habilite la programación.
    
5.  Tras la primera ejecución en producción, vuelva a comprobar **Settings → Overview → User Information**. *Available Users* debería volver a estar en el rango positivo.
    

## Resolución de problemas

| Síntoma | Causa | Medida |
| --- | --- | --- |
| `Can't contact LDAP server` | Puerto 636 inaccesible / host incorrecto | comprobar con `Test-NetConnection` o `nc -vz`, revisar el firewall |
| `Invalid credentials (49)` | Bind DN o contraseña incorrectos | indicar el Bind DN como DN completo, no como `user@domain` |
| `certificate verify failed` | CA desconocida para el truststore | importar la CA raíz o emisora |
| Discordancia de nombre de host en TLS | conexión mediante IP en lugar de FQDN | utilizar como host el CN/SAN del certificado |
| `Referral (10)` | La búsqueda supera los límites del dominio | utilizar Global Catalog en el puerto 3269 en vez del DC en 636 |
| No se detectan usuarios desactivados | falta el filtro `userAccountControl` | utilizar la regla de coincidencia de bits `:1.2.840.113556.1.4.803:=2` |
| El agente elimina demasiadas cuentas | filtro demasiado amplio / Base DN incorrecto | probar en modo de cola, restringir el Base DN |


Con la marca `-d 1`, `ldapsearch` muestra la salida de depuración del establecimiento de la conexión:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

Así verá si falla el handshake TLS o el bind posterior. La interfaz gráfica de totemomail no muestra esta distinción tras su mensaje de error genérico.

## Seguridad

-   **Cuenta de servicio de solo lectura.** El usuario de bind necesita exclusivamente permisos de lectura.
    
-   **LDAPS en lugar de LDAP.** Utilice el puerto 636 o 3269. LDAP en el puerto 389 transmite la contraseña de bind en texto claro. Además, Active Directory exige cada vez más conexiones seguras mediante LDAP Channel Binding y Signing.
    
-   **Rotación de contraseñas.** `PasswordNeverExpires` es operativo y práctico. Documente la cuenta y rote la contraseña según un plan.
    
-   **Monitorización.** Supervise *Available Users* (idealmente mediante alertas), en lugar de esperar la advertencia de la campana.
    
-   **Primera ejecución en modo de cola.** Un filtro erróneo puede afectar a un gran número de cuentas.
    

## El procedimiento seguro en cuatro pasos

Alcanzar el límite de licencias no es un defecto técnico, sino la consecuencia de un proceso de baja inexistente. La solución sostenible es la sincronización periódica con Active Directory como fuente principal. El orden es decisivo:

1.  Verificar la conexión LDAP en la línea de comandos (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configurar la conexión en totemomail
    
3.  Validar el agente en modo de cola
    
4.  Poner el agente en producción
    

Quien siga este orden resolverá el problema de licencias actual e impedirá que vuelva a producirse.

## Fuentes

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Documentación del producto sobre totemomail (modelo de licencias, integración LDAP, agente de limpieza); la tecnología continúa en Kiteworks como Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Significado de las marcas, entre ellas `ACCOUNTDISABLE` (0x0002) y `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Filtro LDAP bit a bit mediante la OID de regla de coincidencia `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (página de manual)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Opciones de llamada (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) para bind y búsqueda.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): Puertos LDAP 389/636 y puertos Global Catalog 3268/3269.
