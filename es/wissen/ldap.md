---
title: "LDAP: protocolo, modelo de datos y funcionamiento de directorios"
blatt: "ldap"
description: "LDAP para administradores: pila de protocolos y formato BER en el cable, DIT, Distinguished Names, esquema, Bind y SASL, búsqueda y controles, TLS, Active Directory, Global Catalog, límites de replicación, escalabilidad y diagnóstico."
fakten:
  - label: Nombre
    wert: Lightweight Directory Access Protocol
    href: https://datatracker.ietf.org/doc/html/rfc4510
  - label: Versión del protocolo
    wert: LDAPv3 · RFC 4510 a 4519
    href: https://datatracker.ietf.org/doc/html/rfc4510#section-1
  - label: Formato en el cable
    wert: Estructuras ASN.1, codificadas en BER
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5.1
  - label: Transporte
    wert: TCP; opcionalmente TLS y SASL sobre este
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5
  - label: Puertos
    wert: 389 LDAP · 636 LDAP sobre TLS
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap
  - label: Modelo de datos
    wert: DIT de Entries y atributos con nombre
    href: https://datatracker.ietf.org/doc/html/rfc4512#section-2
  - label: Nombres
    wert: DN formado por RDN ordenados
    href: https://datatracker.ietf.org/doc/html/rfc4514
  - label: Filtros
    wert: Sintaxis de prefijo según RFC 4515
    href: https://datatracker.ietf.org/doc/html/rfc4515
  - label: Bind
    wert: anonymous, simple o SASL
    href: https://datatracker.ietf.org/doc/html/rfc4513#section-5
  - label: StartTLS
    wert: Extended Operation en una sesión existente
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-4.14
  - label: Paginación
    wert: Simple Paged Results Control
    href: https://datatracker.ietf.org/doc/html/rfc2696
  - label: AD Global Catalog
    wert: 3268 LDAP · 3269 LDAP sobre TLS
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - seppmail
translationSourceHash: 7b219213aa84d6ce78de262f2cd3a5ba23a220cc1669e134bceb3a325b06de33
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:23:53.648Z
translationReview: required
---

# LDAP: protocolo, modelo de datos y funcionamiento de directorios

LDAP es el lenguaje común con el que las aplicaciones acceden a los servicios de directorio. Un cliente puede buscar entradas con nombre, leer o modificar atributos y autenticarse ante el directorio. El protocolo define mensajes, operaciones y códigos de error. En cambio, cómo un servidor almacena sus datos, los replica o los protege frente a fallos depende de cada implementación. Por ello, Active Directory Domain Services, OpenLDAP y 389 Directory Server hablan LDAP sin ser internamente la misma plataforma ([RFC 4510, secciones 1 y 2](https://datatracker.ietf.org/doc/html/rfc4510#section-1), [RFC 4511, sección 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3)).

Esta separación es decisiva en la operación de mensajería. Antes de aceptar SMTP, una pasarela puede comprobar si existe un destinatario, resolver grupos para una política o iniciar sesión como administrador. Si esta consulta falla o devuelve datos obsoletos, no se trata simplemente de que «LDAP está averiado»: según la integración, se rechazan mensajes, se aplican reglas incorrectamente o se bloquean inicios de sesión. Por ello, el administrador debe saber en qué punto se interrumpe el recorrido desde el nombre DNS hasta el atributo leído.

La explicación sigue este recorrido. Primero, el cliente encuentra un servidor y establece una sesión protegida. Después se autentica, realiza una búsqueda e interpreta las respuestas. Solo cuando este flujo normal está claro se pueden situar correctamente el esquema, las particularidades de Active Directory, la replicación, la escalabilidad y la recuperación.

## Pila de protocolos y modelo de sesión

Antes de que una aplicación pueda buscar, necesita un destino de servicio concreto. En entornos de Active Directory, los registros DNS SRV proporcionan posibles Domain Controllers o Global Catalogs; otros productos utilizan FQDN estáticos, su propio descubrimiento de servicios o un equilibrador de carga. Esta selección no determina solo la dirección IP, sino también la ubicación, la función del servidor y el nombre contra el que se comprueba el certificado. Por tanto, una prueba de puerto contra cualquier servidor accesible aún no responde si la aplicación alcanza su destino previsto.

En el destino elegido, LDAP establece una conexión [TCP](/kb/tcp). El puerto 389 comienza como LDAP y puede pasar a una sesión protegida con StartTLS Extended Operation. El puerto 636 está registrado en IANA como `ldaps` y Active Directory, entre otros, lo utiliza para TLS que comienza de inmediato. En ambos casos, el cliente debe verificar la cadena de certificados y el nombre del servidor; «cifrado» y «conectado al servidor correcto» son dos pruebas distintas ([RFC 4511, secciones 4.14 y 5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14), [IANA Service Name Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap), [MS-ADTS, Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81)).

Dentro de esta conexión, LDAP no transmite líneas de comandos legibles como [SMTP](/kb/smtp). Los mensajes se describen como estructuras ASN.1 y se codifican con Basic Encoding Rules, BER. Cada `LDAPMessage` lleva un `messageID`, exactamente una operación y, opcionalmente, controles. Mediante el ID de mensaje, una conexión persistente puede distinguir varias operaciones en curso; sus respuestas no tienen que llegar en el orden de las solicitudes. Por tanto, un handshake TCP correcto no dice nada sobre la decodificación BER, el Bind o una búsqueda completamente finalizada ([RFC 4511, secciones 3.1, 4.1.1 y 5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.1)).

| Capa | Contenido estandarizado | Observación relevante para administración |
|---|---|---|
| Aplicación | Bind, Search, Compare, Modify, Add, Delete, ModifyDN, Extended Operations y Controls | Result Code, `diagnosticMessage`, Entries, References y Controls |
| Codificación | Tipos de datos ASN.1 en BER | Errores de decodificador, tamaño máximo de solicitud, ID de mensaje y OID |
| Seguridad | TLS y mecanismos SASL con su Security Layer | Nombre del certificado, cadena de confianza, método Bind, Signing, Channel Binding |
| Transporte | Conexión TCP persistente | Destino DNS, puerto, latencia de conexión, resets, Idle Timeout y estado del pool |
| Interna del servidor | DIT, esquema, ACL, índice, almacenamiento y replicación | no estandarizado por LDAP; específico de producto y topología |

Para los cambios existe un límite importante: una sola operación LDAP es atómica dentro de su alcance, pero varias entradas no forman una transacción conjunta del protocolo central. RFC 5805 describe una extensión experimental de transacciones, cuyo soporte el cliente debe detectar en el Root DSE. Incluso entonces hay que comprobar cómo ven el cambio las réplicas. Por ello, los procesos de aprovisionamiento necesitan sus propias reglas para reintentos, errores parciales y conciliación, en lugar de asumir silenciosamente una transacción de base de datos ([RFC 4511, sección 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3), [RFC 5805, secciones 1 y 3](https://datatracker.ietf.org/doc/html/rfc5805#section-1)).

## Modelo de datos: DIT, Entry, atributo y esquema

Tras establecer la sesión, el cliente debe poder indicar dónde y qué busca. Para ello, LDAP organiza los datos de directorio como Directory Information Tree, abreviado DIT. Cada Entry tiene un Distinguished Name único y atributos. El esquema describe qué atributos existen, cómo se comparan sus valores y qué Object Classes exigen o permiten. Sin este modelo, la base de búsqueda, los filtros y los resultados son meramente cadenas sin significado fiable ([RFC 4512, secciones 2 y 3](https://datatracker.ietf.org/doc/html/rfc4512#section-2), [RFC 4511, sección 4.1.7](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.7)).

El Distinguished Name forma la ruta de una entrada en el árbol. En `cn=Mail Gateway,ou=Services,dc=example,dc=ch`, `cn=Mail Gateway` designa el Relative Distinguished Name local; los RDN siguientes conducen por el contenedor hasta la raíz de nombres. Como los RDN pueden tener varios valores y caracteres como coma, signo más o barra invertida se escapan, el software no debe procesar un DN dividiéndolo simplemente por comas. Necesita un analizador conforme a RFC 4514 ([RFC 4512, sección 2.3](https://datatracker.ietf.org/doc/html/rfc4512#section-2.3), [RFC 4514, secciones 2 y 3](https://datatracker.ietf.org/doc/html/rfc4514#section-2)).

```text
dn: cn=Mail Gateway,ou=Services,dc=example,dc=ch
objectClass: top
objectClass: person
objectClass: organizationalPerson
cn: Mail Gateway
sn: Gateway
mail: mail-gateway@example.ch
```

Para exportaciones e importaciones, LDIF ofrece una representación de texto estandarizada. LDIF representa Entries o registros de cambios, pero no es el formato en el cable de la sesión LDAP en curso. El plegado de líneas, los valores Base64 y los Change Records siguen sus propias reglas. Sobre todo, una exportación contiene únicamente lo que el servidor y los permisos hacen visible; pueden faltar atributos operativos, ACL o el estado del backend. Por tanto, un volcado LDIF es una extracción de datos, pero no automáticamente una copia de seguridad recuperable del servidor ([RFC 2849, secciones 2 y 4](https://datatracker.ietf.org/doc/html/rfc2849#section-2)).

El significado de un valor de atributo procede del esquema. Una Matching Rule como `caseIgnoreMatch`, `integerMatch` o una comparación DN decide si dos valores son iguales y qué filtros funcionan sobre ellos. En el cable, los valores aparecen inicialmente como Octet Strings; la sintaxis y el tipo de atributo aportan su interpretación. Por ello, los elementos de esquema propios necesitan OID permanentemente únicos, sintaxis y Matching Rules definidas, así como un despliegue que tenga en cuenta conjuntamente los servidores y todos los clientes dependientes ([RFC 4512, sección 4](https://datatracker.ietf.org/doc/html/rfc4512#section-4), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517), [RFC 4520](https://datatracker.ietf.org/doc/html/rfc4520)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ldap.svg?v=20260813" title="Interaktive Infografik: LDAP-Protokollstack, Nachrichtenschicht, DIT, Serverarchitektur und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-ldap.svg?v=20260813">Abrir infografía sobre el protocolo LDAP y la arquitectura de directorios</a>
</iframe>

## Operaciones y cambios de estado

Con el transporte, los nombres y el esquema, la estructura básica está lista; ahora comienza el diálogo de protocolo propiamente dicho. El primer cambio de estado decisivo suele ser `Bind`. Determina bajo qué identidad y con qué derechos derivados se ejecutan las operaciones siguientes. Un Bind renovado sustituye este estado. Mientras el servidor procesa un Bind, el cliente no puede iniciar otras operaciones en la misma conexión ([RFC 4511, secciones 3.1 y 4.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2)).

Tras un Bind correcto, el cliente puede leer o escribir. Una búsqueda no devuelve una única respuesta grande, sino cero o varios mensajes `SearchResultEntry`, posiblemente References y, al final, exactamente un `SearchResultDone`. Solo este resultado final indica si la secuencia estaba completa. `Modify`, `Add`, `Delete` y `ModifyDN` modifican Entries; `Compare` comprueba un valor de atributo según su Matching Rule sin devolver un resultado de búsqueda normal ([RFC 4511, secciones 4.5 a 4.9](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5)).

El final de la sesión también tiene una semántica clara. `Unbind` es una solicitud unilateral de cierre y no tiene respuesta. `Abandon` pide al servidor que cancele una operación determinada, pero no garantiza dicha cancelación. Si en cambio se interrumpe TCP, desaparecen todas las operaciones en curso. En una operación de escritura, el cliente ya no puede saber con seguridad si el cambio surtió efecto antes o después de perder la conexión; por eso, un reintento requiere primero una conciliación del estado en lugar de repetir a ciegas ([RFC 4511, secciones 4.3 y 4.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.3)).

| Operación | Uso típico | Límite que debe tratar un cliente |
|---|---|---|
| Bind | Cuenta de servicio, comprobación de usuario o autenticación SASL | una conexión TCP/TLS correcta aún no es un Bind correcto |
| Search | Destinatarios, grupos, direcciones, políticas y lectura de Root DSE | varios Entries, References, límites, Controls y resultado final |
| Compare | comprobar en el servidor un valor de atributo conocido | el resultado es `compareTrue` o `compareFalse`, no un resultado Search |
| Modify/Add/Delete/ModifyDN | Aprovisionamiento y ciclo de vida | atómico por operación, pero sin transacción del protocolo central entre varias Entries |
| Extended Operation | StartTLS, Password Modify o funciones específicas del fabricante | comprobar OID y soporte en el servidor de destino |
| Controls | Paginación, ordenación, Assertion, Sync o función del fabricante | un Control crítico desconocido debe provocar un error |

Controls y Extended Operations complementan este flujo sin introducir una nueva versión de LDAP. Cada Control tiene un OID, una Criticality y, opcionalmente, un valor codificado en BER. Si un cliente marca como crítico un Control desconocido o no ejecutable, la operación debe fallar con `unavailableCriticalExtension`; de lo contrario, el servidor puede ignorarlo. Por ello, antes de usar paginación, Sync o una función de fabricante, un cliente correcto lee en el Root DSE, entre otros, `supportedControl`, `supportedExtension`, `supportedFeatures`, `supportedLDAPVersion` y `supportedSASLMechanisms` ([RFC 4511, sección 4.1.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.11), [RFC 4512, sección 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1)).

## Bind, SASL y límites de confianza TLS

El Bind decide a quién atribuye el servidor la siguiente búsqueda. En Simple Bind deben distinguirse tres casos: un DN vacío y una contraseña vacía dan como resultado acceso anónimo. Un DN no vacío con contraseña vacía es un *unauthenticated Bind* y no confirma expresamente la identidad indicada, aunque el servidor pueda devolver `success`. Solo un DN no vacío con una contraseña no vacía constituye la autenticación normal de nombre/contraseña. Por ello, los clientes deben rechazar las contraseñas vacías antes de la solicitud; los servidores no deben habilitar por accidente los unauthenticated Binds ([RFC 4513, secciones 5.1.1 a 5.1.3 y 6.3.1](https://datatracker.ietf.org/doc/html/rfc4513#section-5.1)).

En esta autenticación por contraseña, el servidor conoce el secreto presentado. Por ello, el transporte no solo debe estar cifrado, sino también autenticado. Esto incluye una cadena de certificados válida y comprobar que el nombre DNS configurado figure en el certificado. Quien acepta cualquier certificado o utiliza una dirección IP puede establecer un canal cifrado con la contraparte equivocada. La misma comprobación se aplica a StartTLS y a LDAP con TLS inmediato ([RFC 4513, secciones 3.1 y 5.1.3](https://datatracker.ietf.org/doc/html/rfc4513#section-3.1), [RFC 9525, secciones 2 y 4](https://datatracker.ietf.org/doc/html/rfc9525#section-4), [TLS](/kb/tls)).

SASL permite utilizar distintos mecanismos de autenticación en lugar de un simple Bind de contraseña y, además, puede negociar protección para los mensajes LDAP posteriores. En Active Directory se encuentran en particular Negotiate, Kerberos y NTLM. LDAP Signing protege allí la integridad de determinadas sesiones SASL; Channel Binding vincula la autenticación con la conexión TLS subyacente. Por tanto, TLS, Signing y Channel Binding resuelven problemas relacionados, pero no idénticos. Una prueba debe reproducir el tipo de Bind real del producto ([RFC 4513, sección 5.2](https://datatracker.ietf.org/doc/html/rfc4513#section-5.2), [Microsoft: LDAP signing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [MS-ADTS, Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0)).

Para introducir políticas AD más estrictas, no basta con mirar la versión de Windows. Las nuevas implementaciones de AD DS en Windows Server 2025 exigen LDAP Signing de forma predeterminada, mientras que las actualizaciones adoptan la configuración existente. Microsoft indica los eventos de Directory Service 2886 a 2889 para Signing y 3039 a 3041 para Channel Binding. Estos datos de auditoría muestran qué clientes, puertos y métodos Bind se verían realmente afectados; solo después puede planificarse la aplicación de las políticas sobre una base de datos sólida ([Microsoft: LDAP signing, Default Security Behavior und Event Monitoring](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [Microsoft: LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023)).

## Search: base, Scope, filtro y proyección de atributos

Tras un Bind seguro llega la operación de la que dependen la mayoría de las integraciones: Search. La solicitud indica un Base DN, el Scope, el tratamiento de alias, sus propios límites de tamaño y tiempo, un filtro y los atributos solicitados. `baseObject` lee solo la entrada base, `singleLevel` sus hijos directos y `wholeSubtree` todo el subárbol incluido la base. El servidor puede imponer límites más estrictos. Cero resultados con `success` son una respuesta válida; en cambio, `noSuchObject` significa que falta la base de búsqueda o que no es visible para esta identidad ([RFC 4511, secciones 4.5.1 y 4.5.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1)).

El filtro no describe lógica SQL libre, sino un árbol en notación prefija. Por ejemplo, `(&(objectClass=person)(mail=*@example.ch))` combina una expresión Equality con una Substring. `|` representa OR, `!` NOT, `=*` Presence y `:=` un Extensible Match. La Matching Rule del atributo correspondiente determina si una comparación utiliza distinción entre mayúsculas y minúsculas, ordenación numérica o semántica DN ([RFC 4511, sección 4.5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1), [RFC 4515](https://datatracker.ietf.org/doc/html/rfc4515), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517)).

Así, la creación de filtros se convierte en una tarea de seguridad. Los valores procedentes de entradas de usuario deben codificarse conforme a RFC 4515; en particular, `*`, paréntesis, barra invertida, NUL y octetos UTF-8 no válidos no pueden llegar sin procesar a la expresión. De lo contrario, una concatenación de cadenas puede modificar la estructura del filtro y permitir LDAP injection. El escape de DN según RFC 4514 sigue reglas distintas y no sustituye la codificación de filtros ([RFC 4515, sección 3](https://datatracker.ietf.org/doc/html/rfc4515#section-3), [RFC 4514, sección 3](https://datatracker.ietf.org/doc/html/rfc4514#section-3)).

Además del filtro, la lista de atributos determina cuánto devuelve el servidor. Una lista vacía solicita todos los atributos de usuario normales, `1.1` ningún atributo, `*` todos los atributos de usuario y `+` todos los atributos operativos según RFC 3673. Las ACL pueden seguir ocultando valores. Los clientes de producción deben solicitar solo los atributos necesarios: los valores múltiples grandes cargan la red, el decodificador y la memoria, y pueden activar límites propios del servidor ([RFC 4511, sección 4.5.1.8](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1.8), [RFC 3673](https://datatracker.ietf.org/doc/html/rfc3673)).

### Paginación, ordenación y resultados cambiantes

Los conjuntos de resultados grandes se transmiten normalmente con Simple Paged Results Control. El servidor incluye una cookie opaca en cada página, que el cliente devuelve junto con la misma solicitud. Esta cookie no es un desplazamiento ni un cursor permanente. Si el contenido del directorio cambia durante la secuencia, pueden faltar entradas o aparecer duplicadas. Por tanto, la paginación limita la cantidad de datos por respuesta, pero no crea una instantánea coherente ([RFC 2696, secciones 2 y 3](https://datatracker.ietf.org/doc/html/rfc2696#section-2)).

Active Directory hace visible esta distinción en la práctica cotidiana: la política LDAP `MaxPageSize` limita de forma predeterminada los resultados sin paginar a 1000 objetos. Por ello, una importación que recibe exactamente 1000 entradas no ha demostrado su integridad. El cliente debe procesar correctamente las páginas y las cookies y detectar una interrupción. Otras políticas limitan la duración de las consultas, el búfer de recepción y los conjuntos de resultados mantenidos simultáneamente. Por ello, para la operación deben registrarse el tamaño de página, el número de páginas, el último progreso de la cookie, el timeout y el reinicio ([MS-ADTS, LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99), [Microsoft: Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results)).

## Active Directory como perfil de servidor LDAP

Las reglas anteriores se aplican a LDAP en general. Active Directory Domain Services es una implementación concreta de servidor con funciones y convenciones adicionales. Sus datos se distribuyen entre Naming Contexts; un Domain Controller mantiene como mínimo el Schema, Configuration y su propio Domain Naming Context. El Root DSE tiene el DN vacío e indica, entre otros, `defaultNamingContext`, `configurationNamingContext`, `schemaNamingContext`, todos los `namingContexts`, el nombre del servidor y los mecanismos admitidos. Después de TCP y TLS, esta entrada es la primera prueba que realmente dice algo sobre el servicio de directorio alcanzado ([RFC 4512, sección 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1), [Microsoft RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse), [MS-ADTS, rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db)).

La elección entre Domain Controller y Global Catalog cambia el resultado de la búsqueda. Un DC sirve LDAP en 389 o 636 y conoce el Domain Naming Context completo de su dominio. El Global Catalog utiliza adicionalmente 3268 o 3269 y mantiene una réplica parcial de todos los dominios del bosque. Puede encontrar objetos en todo el bosque, pero para dominios remotos solo devuelve atributos del Partial Attribute Set. Por ello, un resultado correcto aún no demuestra que esté presente el atributo requerido por la aplicación ([MS-ADTS, Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a), [Microsoft: Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents), [Microsoft: Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog)).

Los filtros también pueden ser específicos de AD. Por ejemplo, la Matching Rule `1.2.840.113556.1.4.1941`, `LDAP_MATCHING_RULE_TRANSITIVE_EVAL`, sigue atributos vinculados y puede evaluar grupos anidados. Su compatibilidad no aparece simplemente en `supportedControl`. Además, `memberOf` no contiene el Primary Group. Por ello, una decisión de autorización basada en pertenencias a grupos debe considerar explícitamente el anidamiento de grupos, el Primary Group, el ámbito del grupo, la visibilidad ACL y el estado de replicación ([MS-ADTS, LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5), [Microsoft: Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group)).

Qué servidor responde estas consultas lo decide en clientes Windows el DC Locator junto con los registros [DNS](/kb/dns)-SRV. Los registros relacionados con el sitio y la función proporcionan candidatos con prioridad y peso. Una IP configurada estáticamente evita esta selección y dificulta la comprobación del certificado. Un equilibrador TCP simple distribuye conexiones, pero sin lógica adicional no conoce DC escribibles, Global Catalogs, Naming Contexts ni el estado de la replicación ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft: Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)).

Por último, la accesibilidad LDAP no debe confundirse con una replicación saludable. AD replica cambios de directorio mediante Directory Replication Service Remote Protocol; en cambio, `syncrepl` de OpenLDAP utiliza LDAP Content Synchronization con Provider, Consumer y cookies. Una búsqueda de prueba puede mostrar que un servidor concreto responde. Debe comprobarse con las herramientas de la plataforma correspondiente si todos los servidores poseen los mismos cambios y se ponen al día de nuevo después de un fallo ([MS-DRSR, Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1), [RFC 4533](https://datatracker.ietf.org/doc/html/rfc4533), [OpenLDAP Administrator's Guide: Replication](https://www.openldap.org/doc/admin25/replication.html)).

## Modelos de integración y operación

Para la operación, es menos importante que un producto «admita LDAP» que cómo utiliza LDAP. En una **consulta en tiempo de ejecución**, un mensaje o una sesión espera directamente Search y la respuesta del servidor. En una **comprobación de credenciales**, una cuenta técnica busca primero el DN del usuario y luego realiza un segundo Bind con la contraseña introducida. En cambio, una **importación o caché** lee muchas Entries y trabaja con una copia local hasta la siguiente ejecución. Estos patrones tienen consecuencias distintas para la latencia, el procesamiento de contraseñas, el failover y la antigüedad de los datos; la documentación del producto debe indicar el comportamiento concreto ([RFC 4511, secciones 4.2 y 4.5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2), [RFC 2696](https://datatracker.ietf.org/doc/html/rfc2696)).

| Patrón | Ruta crítica | Prueba operativa inmediata |
|---|---|---|
| Consulta en tiempo de ejecución | DNS, Connect, TLS, pool, Bind, Search y respuesta del servidor por operación | p50/p95/p99 por operación, saturación del pool, Result Codes, destino de respaldo |
| Comprobación de credenciales | búsqueda de usuario más segundo Bind con contraseña de usuario | resolución de DN, bloqueo de contraseña vacía, comprobación de nombre TLS, comportamiento de bloqueo |
| importación periódica | enumeración completa y paginada, y Commit en caché local | progreso de página/cookie, cantidad de objetos, modelo de eliminación, último Commit correcto |
| Change Sync | cursor de sincronización específico del fabricante o LDAP | persistencia de cursor, Replay, Resync y objetos eliminados |

Independientemente del patrón, el cliente necesita timeouts separados de Connect, Bind, operación e inactividad. Un Connection Pool ahorra el establecimiento de TCP, TLS y Bind, pero lleva consigo el estado de autenticación de la conexión. Las sesiones muertas deben detectarse y una conexión no debe cambiar accidentalmente entre usuarios o tenants. El failover necesita una secuencia de destinos comprensible, reintentos limitados y un camino de vuelta al destino preferido. De lo contrario, los reintentos paralelos multiplican la carga precisamente durante una caída del directorio ([RFC 4511, secciones 3.1, 4.2 y 5.3](https://datatracker.ietf.org/doc/html/rfc4511#section-3.1)).

En el lado del servidor, Base DN, Scope, filtro y lista de atributos determinan el trabajo. Una condición de igualdad selectiva sobre un atributo indexado es diferente de un substring inicial o una expresión OR grande. LDAP no publica un plan de ejecución ni prescribe una técnica de indexación. Por ello, el administrador debe correlacionar los filtros reales del producto con el volumen de resultados, la latencia p95/p99 y las métricas del servidor. Una búsqueda rápida de una única cuenta de prueba no demuestra que la comprobación de destinatarios escale bajo carga máxima ([OpenLDAP Administrator's Guide: Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html), [MS-ADTS: LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99)).

La supervisión debe dividir el flujo en las mismas etapas que la resolución de problemas: selección DNS, establecimiento de TCP y TLS, Bind, latencia de Search, Result Code, cantidad de resultados y progreso de paginación. Se añaden la ocupación del pool, la tasa de reintentos y el estado de importación o Sync. Un único Bind sintético puede confirmar la accesibilidad, pero no detecta atributos ausentes, una importación incompleta ni un socio de replicación retrasado.

Para backup y recuperación, el contenido visible del directorio tampoco es suficiente. Deben asegurarse el esquema, las ACL, la configuración del backend y del servidor, claves y certificados, identidades de replicación y el procedimiento para reincorporar un nodo restaurado a la topología. Active Directory utiliza para ello System State y sus propios pasos de Forest Recovery; OpenLDAP depende de su backend. El manual distingue, por ejemplo, una copia de seguridad LMDB de `slapcat` y advierte sobre estados LDIF semánticamente inconsistentes en cambios de varias partes. LDAP por sí mismo no define ningún mecanismo de backup ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [OpenLDAP Administrator's Guide: Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html)).

Una prueba de restauración solo concluye cuando un cliente encuentra el servicio restaurado mediante el nombre DNS previsto, TLS y Bind funcionan correctamente, Root DSE y esquema son correctos, las búsquedas reales devuelven atributos completos y la replicación vuelve a iniciarse de forma controlada. Así, la recuperación lleva de vuelta al comienzo del artículo: cuenta todo el recorrido, no solo una base de datos iniciada.

## Herramientas de diagnóstico

El diagnóstico sigue el mismo recorrido que una consulta de producción. Comienza en la red de la aplicación afectada y utiliza su nombre DNS, truststore, método Bind, Base DN, filtro y lista de atributos. De lo contrario, una prueba desde el portátil del administrador puede tener éxito mientras la pasarela sigue utilizando otro DC, otra CA u otro Scope. Los ejemplos utilizan nombres reservados y solo leen metadatos; las contraseñas Bind no pertenecen ni al historial de shell ni a los argumentos de proceso. `ldapsearch -W` las solicita de forma interactiva.

### Determinar destinos de servicio mediante DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran Targets, puertos, prioridades y pesos. A continuación, deben comprobarse la resolución A/AAAA, la relación con la ubicación y la accesibilidad de cada Target realmente seleccionable. Un único DC accesible no corrige un conjunto SRV defectuoso ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)).

### Comprobar TCP y TLS implícito en el puerto 636

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) inicialmente demuestra solo la conexión TCP. El posterior [.NET `SslStream`](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) o [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) comprueba TLS con el nombre DNS configurado. `s_client -showcerts` solo muestra los certificados enviados por el servidor y, por sí solo, no constituye una prueba satisfactoria de cadena ni de nombre de host. StartTLS en 389 puede comprobarse por separado en Unix con `openssl s_client -starttls ldap` ([OpenSSL `s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [RFC 4511, sección 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

### Leer Root DSE y capacidades

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Root-DSE-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADRootDSE`](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) utiliza aquí el módulo ActiveDirectory y, de forma predeterminada, la identidad Windows con sesión iniciada. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) exige mediante `-ZZ` un StartTLS correcto y lee anónimamente solo los atributos Root DSE liberados por el servidor. La ausencia de un OID demuestra que precisamente este destino no publica la función; no dice nada sobre otros nodos del clúster.

### Reproducir una búsqueda real con Scope, filtro y paginación

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Suchprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) acepta con `-LDAPFilter` la sintaxis de filtro cercana a RFC y realiza paginación mediante `-ResultPageSize`. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) utiliza `-E pr=500/noprompt` para Paged Results Control y `-W` para solicitar una contraseña de forma interactiva. Además del resultado, la prueba debe documentar el resultado final, el número de páginas, los atributos devueltos y el tiempo de ejecución.

### Asignar errores a un límite

| Observación | Significado de protocolo | Siguiente prueba fiable |
|---|---|---|
| Timeout antes de TLS | Resolución de destino, routing, firewall, listener o pool agotado | SRV/A/AAAA, handshake TCP, listener del servidor y latencia de conexión |
| Error de certificado | La cadena, validez, nombre o confianza del cliente no coincide | cadena enviada, Trust Anchor, SAN frente al FQDN configurado exacto |
| `strongAuthRequired` / `confidentialityRequired` | El servidor exige un Bind o método de protección más fuerte | puerto, éxito de StartTLS, mecanismo SASL, política de Signing/CBT |
| `invalidCredentials` | Se rechazó la identidad Bind o las credenciales presentadas | tipo de Bind y DN exactos; no registrar contraseñas |
| `invalidDNSyntax` | El DN no es sintácticamente válido | codificación RFC 4514 y DN real del Search Result |
| `noSuchObject` con `matchedDN` | Falta el Base DN o es invisible a partir de un antecesor | Root DSE, Naming Context, visibilidad ACL y `matchedDN` |
| `sizeLimitExceeded` | Límite de cliente o servidor antes del resultado completo | Control de paginación, Page Cookies, LDAP Policy y recuento total |
| `adminLimitExceeded` / `busy` / `unavailable` | Recurso de servidor o límite administrativo | métricas del servidor, Query Policy, coste de filtro, tasa de reintentos y nodo de destino |
| cero resultados con `success` | Búsqueda válida sin coincidencia visible | comparar Base, Scope, filtro, ACL, nodo de destino y estado de replicación |

Los Result Codes numéricos pertenecen al protocolo LDAP; `diagnosticMessage` y los subcódigos AD adicionales son, en cambio, contexto específico de la implementación. Por ello, la automatización debe evaluar primero el Result Code y registrar el texto como complemento. Para `busy` y `unavailable`, cada cliente necesita un presupuesto de reintentos limitado con backoff. Los reintentos ilimitados convierten un único problema de directorio en un pico de carga en todos los sistemas dependientes ([RFC 4511, sección 4.1.9 y apéndice A](https://datatracker.ietf.org/doc/html/rfc4511#appendix-A)).

## Historia técnica

LDAP no surgió como una base de datos de directorio independiente. X.500 había definido a finales de la década de 1980 un modelo de directorio amplio y el Directory Access Protocol. RFC 1487 describió en 1993 un acceso más ligero a este modelo; RFC 1777 le siguió en 1995 como LDAP Version 2. «Lightweight» se refería al acceso de protocolo simplificado en comparación con DAP, no a directorios pequeños ni a poca importancia operativa. El desarrollo inicial está estrechamente vinculado a Tim Howes y a la University of Michigan ([RFC 1487](https://datatracker.ietf.org/doc/html/rfc1487), [RFC 1777](https://datatracker.ietf.org/doc/html/rfc1777)).

LDAPv3 se publicó en 1997 con RFC 2251 y documentos complementarios. Las operaciones extensibles, Controls, SASL, internacionalización y el modelo de datos revisado lo convirtieron en la base de las implementaciones actuales. El trabajo LDAPbis reorganizó este estado en 2006: RFC 4510 sirve como hoja de ruta, RFC 4511 describe el protocolo, RFC 4512 el modelo de información y RFC 4513 la seguridad; RFC 4514 a 4519 complementan representaciones, URL, sintaxis y esquema ([RFC 2251](https://datatracker.ietf.org/doc/html/rfc2251), [RFC 4510, sección 3](https://datatracker.ietf.org/doc/html/rfc4510#section-3)).

En paralelo se desarrollaron servidores muy diferentes. OpenLDAP surgió en 1998 de la implementación de University of Michigan y continuó `slapd`, bibliotecas y herramientas como proyecto de código abierto. Active Directory llevó con Windows 2000 un perfil LDAPv3 con su propio esquema, Naming Contexts, Controls, Matching Rules y protocolo de replicación separado al uso empresarial generalizado ([OpenLDAP Release Road Map](https://www.openldap.org/software/roadmap.html), [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html), [MS-ADTS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/)).

Esta historia explica la regla operativa más importante: LDAP unifica el acceso, no la arquitectura interna. Por ello, quien traslada un cliente de OpenLDAP a AD DS o entre dos appliances debe comprobar más que host, puerto y Bind DN. El esquema, los Controls, los límites, la resolución de grupos, la replicación y la recuperación siguen siendo propiedades del producto.

## Fuentes

- [RFC 4510, secciones 1 y 2](https://datatracker.ietf.org/doc/html/rfc4510)
- [RFC 4511 – LDAP: The Protocol](https://datatracker.ietf.org/doc/html/rfc4511) – capa de mensajes, operaciones, BER, TCP, StartTLS y Result Codes.
- [IANA – Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap) – `ldap` 389 y `ldaps` 636.
- [MS-ADTS – Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81) – TLS implícito y StartTLS en Active Directory.
- [RFC 5805, secciones 1 y 3](https://datatracker.ietf.org/doc/html/rfc5805)
- [RFC 4512 – LDAP Directory Information Models](https://datatracker.ietf.org/doc/html/rfc4512) – DIT, Entries, atributos, esquema, Root DSE y Subschema.
- [RFC 4514, secciones 2 y 3](https://datatracker.ietf.org/doc/html/rfc4514)
- [RFC 2849, secciones 2 y 4](https://datatracker.ietf.org/doc/html/rfc2849)
- [RFC 4517 – LDAP Syntaxes and Matching Rules](https://datatracker.ietf.org/doc/html/rfc4517) – sintaxis estándar y reglas de comparación.
- [RFC 4520 – IANA Considerations for LDAP](https://datatracker.ietf.org/doc/html/rfc4520) – registro de OID y parámetros de protocolo.
- [RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513) – métodos Bind, SASL, TLS y límites de seguridad.
- [RFC 9525, secciones 2 y 4](https://datatracker.ietf.org/doc/html/rfc9525)
- [Microsoft – LDAP signing for AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing) – Signing, Channel Binding, valores predeterminados y eventos.
- [MS-ADTS – Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0) – LDAP Channel Binding en Active Directory.
- [Microsoft – LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023) – requisitos de seguridad por modelo Bind y TLS.
- [RFC 4515 – String Representation of Search Filters](https://datatracker.ietf.org/doc/html/rfc4515) – gramática de filtros y codificación de valores.
- [RFC 3673 – All Operational Attributes](https://datatracker.ietf.org/doc/html/rfc3673) – `+` como selector de atributos operativos.
- [RFC 2696 – Simple Paged Results Control](https://datatracker.ietf.org/doc/html/rfc2696) – páginas, cookies y límites de consistencia.
- [MS-ADTS – LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99) – límites administrativos de búsqueda y recursos.
- [Microsoft – Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results) – búsqueda paginada en Active Directory.
- [Microsoft – RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse) – Naming Contexts y capacidades del servidor.
- [MS-ADTS – rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db) – atributos Root DSE específicos de AD.
- [MS-ADTS – Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a) – puertos LDAP, LDAPS y Global Catalog.
- [Microsoft – Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents) – búsqueda en todo el bosque y réplica parcial.
- [Microsoft – Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog) – Partial Attribute Set del Global Catalog.
- [MS-ADTS – LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5) – OID Extensible Match específicos de AD.
- [Microsoft – Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group) – delimitación entre `memberOf` y Primary Group.
- [Microsoft – DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator) – selección DNS SRV, LDAP Ping y relación con el sitio.
- [Microsoft – Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created) – registro SRV de los Domain Controllers.
- [MS-DRSR – Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1) – delimitación de la replicación AD respecto de LDAP.
- [RFC 4533 – LDAP Content Synchronization Operation](https://datatracker.ietf.org/doc/html/rfc4533) – Controls LDAP Sync, cookies y modelo de estado.
- [OpenLDAP Administrator's Guide – Replication](https://www.openldap.org/doc/admin25/replication.html) – `syncrepl`, cookies y modelo Provider/Consumer.
- [OpenLDAP Administrator's Guide – Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html) – indexación y costes internos de búsqueda del servidor.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – copia de seguridad System State para AD DS.
- [OpenLDAP Administrator's Guide – Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html) – copia de seguridad LMDB, `slapcat` y límites de consistencia.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – consultas DNS y SRV en Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – consultas DNS y SRV en sistemas Unix.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnóstico de conexiones TCP en Windows.
- [Microsoft Learn – SslStream](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) – handshake TLS y comprobación de certificados con .NET.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – handshake TLS, comprobación de nombre y LDAP StartTLS.
- [Microsoft Learn – Get-ADRootDSE](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) – diagnóstico Root DSE en Windows.
- [OpenLDAP – ldapsearch(1)](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) – opciones de Search, StartTLS, SASL y Controls.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) – LDAPFilter, SearchBase, Scope y paginación.
- [RFC 1487 – X.500 Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1487) – primera especificación LDAP de 1993.
- [RFC 1777 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1777) – LDAPv2 y modelo de protocolo histórico.
- [RFC 2251 – Lightweight Directory Access Protocol v3](https://datatracker.ietf.org/doc/html/rfc2251) – primera especificación central LDAPv3.
- [OpenLDAP – Release Road Map](https://www.openldap.org/software/roadmap.html) – Release 1.0 en agosto de 1998.
- [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html) – LDAP de University of Michigan como base del proyecto.
- [MS-ADTS – Active Directory Technical Specification](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/) – perfil de servidor LDAP de AD DS y AD LDS.
