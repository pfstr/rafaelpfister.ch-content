---
title: "Kerberos: tickets, claves y servicios de confianza"
blatt: "kerberos"
description: "Kerberos para administradores: KDC, flujo AS/TGS/AP, principals y SPN, tickets, claves de sesión, keytabs y KVNO, GSS-API, Active Directory y PAC, trusts, delegación, cifrado, DNS, tiempo, operación y diagnóstico."
fakten:
  - label: Función
    wert: Autenticación de red mediante tickets
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.1
  - label: Versión del protocolo
    wert: Kerberos V5 · RFC 4120
    href: https://datatracker.ietf.org/doc/html/rfc4120
  - label: Entidad de confianza
    wert: KDC con Authentication Service y TGS
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.2
  - label: Intercambio
    wert: AS · TGS · Client/Server AP
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3
  - label: Puerto del KDC
    wert: 88 sobre UDP y TCP
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos
  - label: Servicio de contraseña
    wert: kpasswd · Port 464
    href: https://datatracker.ietf.org/doc/html/rfc3244
  - label: Nombre de servicio
    wert: service/host@REALM
    href: https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html
  - label: Clave de servicio
    wert: Principal · KVNO · Enctype · Key
    href: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
  - label: Ventana temporal
    wert: Clock Skew es una política del realm y del cliente
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3
  - label: API de aplicación
    wert: GSS-API / SSPI · frecuentemente SPNEGO
    href: https://datatracker.ietf.org/doc/html/rfc4121
  - label: Active Directory
    wert: Los controladores de dominio integran el KDC
    href: https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview
  - label: Autorización de AD
    wert: PAC como ticket-Authorization-Data
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
translationSourceHash: b5eccfbd21dfaa9b065424961e28940ca63617bba9dbb9f5ab68c1db128df31f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:21:24.165Z
translationReview: required
---

# Kerberos: tickets, claves y servicios de confianza

Kerberos permite que un usuario o proceso se identifique ante un servicio de red sin enviar la contraseña a dicho servicio. Para ello, el cliente y el servidor confían en un Key Distribution Center (KDC), que emite tickets y claves de sesión con una validez limitada en el tiempo. En un inicio de sesión basado en contraseña, no obstante, se deriva una clave de larga duración a partir de la contraseña; por ello, la calidad de la contraseña, la preautenticación y la protección del cliente siguen siendo relevantes para la seguridad ([RFC 4120, secciones 1.1 y 1.2](https://datatracker.ietf.org/doc/html/rfc4120#section-1.1), [RFC 3961, sección 3](https://datatracker.ietf.org/doc/html/rfc3961#section-3)).

Para los administradores de mensajería y directorios, Kerberos suele coexistir con [LDAP](/kb/ldap), no lo sustituye. LDAP transporta operaciones de directorio; Kerberos puede proporcionar la identidad de una sesión LDAP mediante SASL. Las interfaces web utilizan HTTP Negotiate, los servicios de Windows SSPI y las aplicaciones Unix normalmente GSS-API. Por tanto, un error puede originarse en DNS, hora, accesibilidad del KDC, asignación de realm, caché de tickets, SPN, cuenta de servicio, keytab, tipo de cifrado, PAC o política de delegación, aunque la aplicación solo informe «Integrated Authentication failed».

La explicación sigue a un principal desde el primer contacto con el KDC, pasando por el TGT y el ticket de servicio, hasta el servicio de destino. Solo después se profundiza en las extensiones de Active Directory, la delegación, la dependencia temporal, el diagnóstico y la recuperación.

## Enfoque arquitectónico: instancia de confianza de terceros

Kerberos distribuye claves simétricas mediante una instancia de confianza. El KDC tiene acceso a las claves de larga duración de los principals de su realm y reúne lógicamente dos servicios: el Authentication Service, AS, emite un Ticket-Granting Ticket, TGT; el Ticket-Granting Service, TGS, intercambia este TGT por un ticket para un servicio de aplicación concreto. El intercambio cliente/servidor, AP, se realiza después entre el cliente y el servicio. Normalmente, un servicio puede verificar un ticket de servicio con su propia clave sin volver a consultar el KDC en cada inicio de sesión ([RFC 4120, secciones 1.2 y 3](https://datatracker.ietf.org/doc/html/rfc4120#section-1.2)).

Este modelo reduce la transmisión de contraseñas y las comprobaciones centrales en línea, pero crea ámbitos de fallo claros. Sin un KDC accesible, no se pueden emitir tickets nuevos; los tickets existentes pueden seguir funcionando hasta que termine su validez. En cambio, una base de datos de KDC o una clave de realm comprometida pone en riesgo la cadena de confianza de todo el realm. Por tanto, Kerberos no es una plataforma de firmas de tokens sin estado, sino un sistema distribuido compuesto por estado del KDC, cachés de clientes, claves de servicio, relojes y servicios de nombres.

| Capa | Componente técnico | Estado responsable | Evidencia administrativa |
|---|---|---|---|
| Aplicación | HTTP, SMB, LDAP, base de datos, SMTP/IMAP con SASL o servicio propietario | Sesión y autorización de la aplicación | método de autenticación y nombre de destino realmente seleccionados |
| API de integración | GSS-API, Windows SSPI y con frecuencia SPNEGO | Contexto de seguridad, delegación y Channel Binding | mecanismo negociado, iniciador, aceptador y flags |
| Kerberos AP | `KRB_AP_REQ`, opcionalmente `KRB_AP_REP`, en su caso GSS Wrap/MIC | Ticket de servicio, autenticador y clave de sesión | principal de destino, tiempos del ticket, Enctype y autenticación mutua |
| Kerberos KDC | Intercambio AS y TGS | Base de datos de principals, claves de realm, políticas y flags de ticket | resultado del KDC, clave seleccionada, KVNO y evento de auditoría |
| Detección y transporte | DNS SRV, UDP/TCP 88, servicio de contraseña 464 | Caché del resolver, asignación de realm, enrutamiento y tamaño de paquete | KDC realmente seleccionado, transporte, tiempo de respuesta y fallback |
| Persistencia | AD DS o base de datos Kerberos, keytabs, cachés de credenciales y replay | Claves de larga duración, tickets, PAC, replicación y estado de replay | estado de replicación, contenido de caché, permisos de archivos y procedimiento de recuperación |

Kerberos autentica principals y puede proporcionar integridad o confidencialidad para un contexto de seguridad GSS. No cifra automáticamente todo el flujo de datos de una aplicación ni ofrece Perfect Forward Secrecy para las claves de sesión distribuidas en el núcleo de Kerberos. Las aplicaciones pueden utilizar Kerberos para autenticar un canal protegido por separado; [TLS](/kb/tls) sigue siendo, por tanto, una capa independiente con su propia identidad de servidor y verificación de certificados en HTTPS o LDAPS ([RFC 4120, sección 10](https://datatracker.ietf.org/doc/html/rfc4120#section-10), [RFC 4121, secciones 2 y 4](https://datatracker.ietf.org/doc/html/rfc4121#section-2)).

## Principals, realms y material de claves

Un principal Kerberos es un nombre dentro de un realm. Los usuarios se representan con frecuencia como `alice@EXAMPLE.CH`, y los servicios como `HTTP/intranet.example.ch@EXAMPLE.CH`. La parte anterior a `@` puede contener varios componentes; las mayúsculas y minúsculas son relevantes en principio en el modelo de nombres de Kerberos. Un nombre de dominio DNS y un realm Kerberos son espacios de nombres distintos, aunque los realms normalmente parezcan dominios DNS en mayúsculas. Por ello, los clientes necesitan una asignación verificable del nombre de host o dominio DNS al realm ([RFC 4120, secciones 6.1 y 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-6.1), [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html#mapping-hostnames-onto-kerberos-realms)).

Las claves de larga duración pertenecen a los principals. Las claves de sesión se generan para un intercambio limitado. Un ticket contiene, entre otros elementos, cliente, servidor, realm, campos temporales, flags, clave de sesión y datos de autorización; su parte cifrada está protegida con una clave del servicio de destino. El cliente recibe la misma clave de sesión en una parte de respuesta protegida para él. Por tanto, puede transportar el contenido del ticket, pero no modificar por sí mismo la parte cifrada para el servicio ([RFC 4120, secciones 5.3 y 5.4](https://datatracker.ietf.org/doc/html/rfc4120#section-5.3)).

El Key Version Number, KVNO, distingue generaciones de una clave de principal. El Encryption Type, Enctype, define el algoritmo, la longitud de clave, String-to-Key y la suma de comprobación. Un servicio puede conservar varias entradas de keytab para el mismo principal con distintos KVNO o Enctypes para superar una rotación controlada. Si el KVNO y el Enctype del ticket no coinciden con ninguna clave de servicio disponible, el servicio no puede descifrar el ticket. Por tanto, «SPN presente» aún no demuestra que haya una clave adecuada en el sistema de destino ([RFC 4120, sección 5.2.9](https://datatracker.ietf.org/doc/html/rfc4120#section-5.2.9), [MIT Kerberos: Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)).

| Objeto | Ubicación | Confidencialidad | Límite operativo |
|---|---|---|---|
| Clave de principal de larga duración | Base de datos del KDC; adicionalmente, keytab o almacén de claves del sistema operativo en el servicio | extremadamente crítica | rotación, replicación, KVNO y Enctypes permitidos |
| TGT | Caché de credenciales del cliente; parte de ticket cifrada para `krbtgt/REALM` | similar a un bearer más clave de sesión | Lifetime, flags Forwardable/Renewable, aislamiento de caché |
| Ticket de servicio | Caché de credenciales y posteriormente servicio de destino | descifrable solo por el servicio nombrado | SPN, cuenta de servicio, host de destino, PAC y Enctype del ticket |
| Autenticador | Nuevo para cada solicitud AP, protegido con la clave de sesión | protección contra replay | hora del cliente, subclave, secuencia y caché de replay del servidor |
| Keytab | Archivo u otro almacén de keytab en el servicio | proteger como una contraseña no interactiva | permisos de archivo, distribución, inventario, rotación y eliminación segura |
| Caché de replay | Estado local del aceptador | estado de integridad | comprobar por instancia de servicio, host y arquitectura de clúster |

Active Directory almacena SPN en el atributo multivalor `servicePrincipalName` de una cuenta de usuario o equipo. El SPN vincula el nombre de servicio compuesto por el cliente con exactamente la cuenta cuya clave utiliza el KDC para el ticket de servicio. Por ello, un alias, un nombre de balanceador de carga o una cuenta de servicio modificada requieren una asignación SPN consciente. Los SPN duplicados son ambiguos dentro de su ámbito de búsqueda; un SPN en la cuenta incorrecta normalmente hace que el servicio no pueda descifrar el ticket emitido ([Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

Una keytab no es un archivo de exportación con una identidad recuperable arbitrariamente, sino una colección de claves reales de larga duración. Cada entrada contiene principal, KVNO, Enctype y clave. Quien pueda leer el archivo puede hacerse pasar por ese principal. MIT recomienda un almacenamiento local y restrictivo, sin transferencia sin protección. En la interoperabilidad con Active Directory, [`ktpass`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) puede vincular principal, cuenta y keytab; los parámetros seleccionados pueden influir en la contraseña, el salt, el KVNO o el Enctype y deben formar parte de un proceso de rotación probado, no de una hoja de instalación única ([MIT Kerberos: Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-kerberos.svg?v=20260813" title="Interaktive Infografik: Kerberos-Discovery, AS-, TGS- und AP-Fluss, Tickets, Schlüssel, Active-Directory-PAC und Delegationsgrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-kerberos.svg?v=20260813">Abrir la infografía sobre el flujo del protocolo Kerberos y sus límites de confianza</a>
</iframe>

Una vez aclarados principal, realm y claves, el flujo de tickets puede leerse como tres conversaciones consecutivas: AS, TGS y, finalmente, el servicio de aplicación.

## Flujo de protocolo: AS, TGS y AP

El intercambio AS comienza con `KRB_AS_REQ`. Un KDC puede responder con `KDC_ERR_PREAUTH_REQUIRED` e indicar los procedimientos de preautenticación compatibles. Con la marca de tiempo cifrada habitual, el cliente demuestra conocer su clave de larga duración antes de que el KDC emita un TGT. `KRB_AS_REP` contiene el TGT, cifrado para el TGS, y una parte de respuesta para el cliente. PKINIT sustituye esta primera prueba por criptografía de clave pública y certificados; Kerberos FAST puede reforzar la preautenticación dentro de un túnel protegido ([RFC 4120, secciones 3.1 y 5.2.7](https://datatracker.ietf.org/doc/html/rfc4120#section-3.1), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113, sección 5](https://datatracker.ietf.org/doc/html/rfc6113#section-5)).

En el intercambio TGS, el cliente envía `KRB_TGS_REQ` con el TGT, un autenticador y el principal de servicio deseado. El KDC comprueba la política del realm, los flags del ticket, el principal de destino y las claves compatibles, y entrega en `KRB_TGS_REP` un ticket de servicio más una nueva clave de sesión cliente/servicio. La contraseña de usuario no vuelve a ser necesaria. Por tanto, un TGT existente puede utilizarse para muchos servicios hasta que el Lifetime, la política o el estado de caché exijan una nueva autenticación inicial ([RFC 4120, sección 3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3)).

En el intercambio AP, el cliente presenta al servicio `KRB_AP_REQ`: el ticket de servicio y un autenticador nuevo protegido con la clave de sesión. El servicio descifra el ticket con su clave de larga duración, comprueba el principal de destino, los tiempos, flags y el estado de replay, y obtiene de él la clave de sesión. Si el cliente solicita autenticación mutua, el servicio responde con `KRB_AP_REP`. Solo esta respuesta demuestra criptográficamente al cliente que el interlocutor posee la clave de servicio ([RFC 4120, secciones 3.2 y 5.5](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2)).

| Intercambio | Solicitud | Respuesta correcta | Entradas críticas | Límite de error típico |
|---|---|---|---|---|
| AS | `KRB_AS_REQ` | `KRB_AS_REP` con TGT | Principal de cliente, realm, Pre-Auth, Enctypes permitidos | cuenta desconocida, Pre-Auth, hora, falta de clave de cliente |
| TGS | `KRB_TGS_REQ` | `KRB_TGS_REP` con ticket de servicio | TGT, autenticador, SPN, flags, claves de destino | SPN, referral de realm, política, delegación, Enctype |
| AP | `KRB_AP_REQ` | opcionalmente `KRB_AP_REP` | ticket de servicio, autenticador, clave de servicio, caché de replay | cuenta de servicio errónea, keytab/KVNO, hora, replay |
| Error | una de las solicitudes | `KRB_ERROR` | código de error más e-data y hora de servidor opcionales | conservar el código numérico y la fase de protocolo afectada |

La preautenticación no protege frente a todos los ataques offline, pero cambia el requisito previo. Las cuentas sin preautenticación obligatoria pueden proporcionar una respuesta AS cuya parte protegida por contraseña puede verificarse offline. Los tickets de servicio también pueden analizarse offline; las contraseñas débiles de cuentas de servicio y RC4 agravan este riesgo. La protección efectiva consiste en preautenticación, claves de servicio aleatorias y fuertes o gMSA, Enctypes modernos, privilegios limitados y datos de auditoría, no en la mera presencia de Kerberos ([RFC 6113, sección 1](https://datatracker.ietf.org/doc/html/rfc6113#section-1), [RFC 8429, sección 5](https://datatracker.ietf.org/doc/html/rfc8429#section-5), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

## Tickets, tiempos, flags y transporte

Un ticket posee `authtime`, opcionalmente `starttime`, `endtime` y, en los tickets renovables, `renew-till`. El TGT no es un token de acceso universal: va dirigido al TGS y sirve para obtener más tickets. Un ticket de servicio va dirigido exactamente al principal de servidor. Lifetime y renovación son política del KDC y pueden restringirse por realm o cuenta. Por tanto, valores fijos como «diez horas» son valores predeterminados de productos, no una característica del estándar Kerberos ([RFC 4120, secciones 2.3 y 5.3](https://datatracker.ietf.org/doc/html/rfc4120#section-2.3)).

Los flags como `forwardable`, `forwarded`, `proxiable`, `proxy`, `renewable`, `initial`, `pre-authent` y `ok-as-delegate` modifican la usabilidad de un ticket. No son campos de diagnóstico decorativos. Un doble salto puede fallar aunque el primer ticket de servicio sea válido porque falta un flag necesario o una política de KDC. A la inversa, un TGT reenviable amplía el impacto de un servicio comprometido. Por ello, los flags de ticket pertenecen a toda evidencia de delegación e incidente ([RFC 4120, sección 2](https://datatracker.ietf.org/doc/html/rfc4120#section-2)).

Los autenticadores y algunos procedimientos de preautenticación utilizan el tiempo para comprobar la frescura. RFC 4120 deja la desviación admisible a la política local. Active Directory utiliza cinco minutos de forma predeterminada; esta tolerancia no sustituye una sincronización horaria precisa. Son decisivos el cliente, el KDC y el servicio de destino: un cliente puede obtener un ticket y, aun así, fallar en el servicio con `KRB_AP_ERR_SKEW` si su reloj se desvía ([RFC 4120, secciones 3.2.3 y 7.5.1](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance), [Microsoft: Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service)).

Kerberos utiliza el puerto 88 mediante UDP y TCP. RFC 4120 exige compatibilidad con TCP y describe UDP como opcional; las respuestas UDP demasiado grandes pueden activar un reintento TCP con `KRB_ERR_RESPONSE_TOO_BIG`. PAC, pertenencias a grupos y otros datos de autorización aumentan el tamaño de los tickets. Por ello, un firewall que solo permita pruebas UDP pequeñas o bloquee TCP 88 puede generar fallos dependientes del usuario o grupo ([RFC 4120, sección 7.2.1](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.1), [Microsoft: Kerberos KDC configuration keys](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys)).

El ticket de servicio es solo la prueba criptográfica. Para que HTTP, LDAP o SMB lo utilicen, GSS-API, SSPI y SPNEGO incorporan Kerberos en el protocolo de aplicación correspondiente.

## Integración en protocolos de aplicación

GSS-API proporciona a las aplicaciones un contexto de seguridad independiente del mecanismo; RFC 4121 define para ello el mecanismo Kerberos V5. Windows SSPI cumple una función comparable. SPNEGO negocia entre los mecanismos GSS ofrecidos y aparece con frecuencia en HTTP como `Negotiate`. El nombre visible de la cabecera no demuestra automáticamente que se haya seleccionado Kerberos en vez de NTLM. El diagnóstico debe registrar el mecanismo realmente negociado y el Target Name solicitado ([RFC 4121](https://datatracker.ietf.org/doc/html/rfc4121), [RFC 4178](https://datatracker.ietf.org/doc/html/rfc4178), [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)).

SASL GSS-API integra el mismo mecanismo Kerberos en protocolos de aplicación. Esto es posible, por ejemplo, con [LDAP](/kb/ldap), IMAP o SMTP, si el cliente y el servidor ofrecen el mecanismo. Tras un contexto GSS correcto, las capas de seguridad SASL pueden proporcionar integridad o confidencialidad. Que se utilicen realmente y cómo interactúan con TLS depende de la configuración de la aplicación concreta; `GSSAPI` en la lista de capacidades por sí solo no demuestra ni SPN ni Channel Protection ([RFC 4752](https://datatracker.ietf.org/doc/html/rfc4752)).

El cliente forma el nombre de servicio a partir de la clase de servicio y el host de destino. Para HTTP suele ser relevante `HTTP/fqdn`, y para LDAP `ldap/fqdn`. Un alias, CNAME, búsqueda inversa, proxy, nombre de clúster y el nombre de host de la URL pueden cambiar la identidad formada. El cliente debe solicitar el ticket para el mismo principal cuya clave posee el servicio aceptador. Un balanceador de carga no resuelve esta vinculación; todas las instancias de backend requieren una identidad de servicio y una estrategia de claves coherentes ([MIT Kerberos: Application servers – DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html#getting-dns-information-correct), [Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)).

## Active Directory como realm Kerberos

En Active Directory Domain Services, el KDC está integrado en el controlador de dominio. Las cuentas de usuario, equipo y servicio son principals Kerberos; el directorio proporciona claves, SPN, grupos, flags de cuenta y políticas. La cuenta `krbtgt` representa el TGS del dominio. La disponibilidad del KDC y la coherencia de sus datos siguen, por ello, DC Locator, DNS, replicación de AD, sitios y el modelo de recuperación de AD DS, no un protocolo de clúster Kerberos independiente ([Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview), [MS-KILE: Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13), [DC Locator](/kb/ldap#active-directory-als-ldap-serverprofil)).

AD suele añadir a los tickets un Privilege Attribute Certificate, PAC, como datos de autorización. Puede contener, entre otros, SID, pertenencias a grupos, información de perfil y políticas, además de firmas. El KDC genera y firma estos datos; el servicio los utiliza para la autorización de Windows o puede validarlos cuando corresponda. Por tanto, la autenticación central de Kerberos y la autorización de AD son afirmaciones separadas: un principal criptográficamente válido no posee automáticamente el permiso requerido ([MS-PAC](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962), [MS-KILE: PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909)).

El material de claves y los atributos SPN se replican con Active Directory. Después de cambios en cuentas, contraseñas o SPN, distintos DC pueden usar brevemente estados diferentes. Si el cliente utiliza un DC para TGS y el administrador otro DC para la comprobación, el fallo parece intermitente. Una evidencia sólida indica el KDC emisor, el DC de destino de la consulta de directorio, KVNO, Enctype del ticket y estado de replicación. Esto se aplica especialmente a rotaciones manuales de keytab y servicios distribuidos entre sitios.

La selección de Enctype es la intersección entre la oferta del cliente, la política del KDC, las claves de la cuenta de destino y la compatibilidad del servicio. No basta con disponer de software compatible con AES si la cuenta no tiene las claves correspondientes o `msDS-SupportedEncryptionTypes` y la política de dominio las excluyen. RFC 8429 clasifica RC4 y 3DES para Kerberos como obsoletos; Microsoft documenta el inventario mediante los eventos de seguridad 4768 y 4769. Una migración comienza con medición y generación de claves, no con la desactivación global de un bit ([RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

Para los servicios Windows, las Group Managed Service Accounts, gMSA, reducen la administración manual de contraseñas y SPN. Sin embargo, no sustituyen la comprobación de la identidad bajo la que se ejecuta realmente el proceso, los hosts autorizados a leer la contraseña administrada ni los SPN registrados en la cuenta. Las appliances o los servicios Unix suelen seguir necesitando una keytab; su rotación debe sincronizarse con la cuenta de AD ([Microsoft: Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts)).

Dentro de un dominio AD, esta ruta es directa. Al acceder a través de límites de realm o dominio, se añaden tickets Cross-Realm y, si procede, varias estaciones intermedias.

## Trusts y rutas Cross-Realm

Un trust de realm no fusiona bases de datos de principals. La autenticación Cross-Realm utiliza TGT para `krbtgt/TARGET@SOURCE` y, cuando corresponde, varios realms intermedios. El cliente sigue una ruta de tickets hasta alcanzar el realm del servicio de destino. RFC 6806 añade referrals y canonicalización de nombres, tal como se utiliza especialmente en entornos AD. Por ello, la ruta visible en la caché es más significativa que la afirmación general «el trust está en verde» ([RFC 4120, secciones 1.1 y 3.3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3.3), [RFC 6806](https://datatracker.ietf.org/doc/html/rfc6806)).

La dirección del trust, la transitividad, Selective Authentication, el filtrado SID, Name Suffix Routing y los Enctypes disponibles limitan lo que un TGT Cross-Realm permite en la práctica. Los SPN deben poder localizarse en el forest correcto y ser únicos. Una coincidencia local de `setspn -Q` no demuestra unicidad entre forests; [`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) ofrece para ello opciones de forest y dominio. El diagnóstico documenta cada TGT de referral, no solo el último error del ticket de servicio ([Microsoft: Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

Un ticket para el frontend aún no autoriza automáticamente a este a acceder a un backend en nombre del usuario. Aquí comienza precisamente el problema del doble salto.

## Delegación y doble salto

En el intercambio AP normal, un frontend no obtiene una clave de usuario que pueda utilizar libremente. Si debe acceder a un backend en nombre del usuario, necesita un modelo de delegación. La delegación de TGT reenviado o unconstrained delegation entrega al servicio un TGT reutilizable y amplía su alcance mucho más allá de un backend individual. Un frontend comprometido puede utilizarlo para obtener tickets para otros servicios; esta modalidad supone, por tanto, una gran ampliación de confianza ([RFC 4120, secciones 2.5 y 2.6](https://datatracker.ietf.org/doc/html/rfc4120#section-2.5), [MS-SFU: Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a)).

Las extensiones Service-for-User de Microsoft dividen el problema. Con S4U2self, un servicio puede obtener un ticket para sí mismo en nombre de un usuario, por ejemplo tras otra autenticación de frontend. Con S4U2proxy, solicita, según la política del KDC, un ticket para un segundo servicio en nombre de dicho usuario. La constrained delegation clásica almacena los destinos permitidos en la cuenta del frontend; la resource-based constrained delegation define los llamadores permitidos en la cuenta del recurso. En ambos casos, el SPN de destino, los flags de ticket, la configuración de cuenta y la ruta de trust forman parte de la decisión ([MS-SFU: Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf), [MS-SFU: Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d)).

La delegación es autorización para la propagación de identidad, no una mera opción de compatibilidad. El administrador inventaría el principal frontend, los SPN de backend, las clases de usuarios permitidas, Protocol Transition, los límites de trust y el alcance de destino técnicamente mínimo. Un primer salto correcto no demuestra el segundo; por el contrario, una prueba directa del backend bajo el contexto del usuario evita el límite de delegación y puede producir un falso positivo.

## Modelo operativo, monitorización y recuperación

Tras el flujo de tickets y la delegación surge la cuestión operativa: ¿qué componente Kerberos debe estar disponible, qué evidencia muestra su estado y qué puede restaurarse en caso de emergencia? La tabla asigna estas preguntas a los roles implicados.

| Rol | Estado que debe supervisarse | Métrica o evidencia principal | Punto ciego frecuente |
|---|---|---|---|
| Cliente | Asignación de realm, selección de KDC, reloj y caché de credenciales | Latencia AS/TGS, Lifetime de caché, KDC y código de error | La prueba utiliza otro nombre DNS o sesión de inicio de sesión de usuario |
| KDC | Claves de principal, política, replicación, trust y auditoría | 4768/4769/4771, tasa de errores por código, Enctype y DC emisor | Errores globales sin SPN de destino ni oferta de cliente |
| Servicio | Cuenta de servicio, SPN, keytab/almacén de claves, caché de replay y reloj | Éxito AP, KVNO, Enctype de ticket, principal de destino | Puerto abierto, pero el proceso se ejecuta bajo otra identidad |
| Frontend con delegación | Política S4U/forwarding y destinos backend | Primer y segundo salto separados, ruta de delegación y flags de ticket | Prueba directa del backend evita el doble salto |
| Operación de realm/forest | Base de datos del KDC, claves de realm, replicación de AD y recuperación | Estado de claves replicadas, configuración respaldada, restauración probada | La keytab de servicio y la generación del KDC divergen |

Las cachés de tickets son estado operativo. Un proceso de usuario, una cuenta de servicio, un contenedor o una sesión de inicio de sesión de Windows puede ver una caché diferente de la shell administrativa interactiva. Eliminar y volver a obtener un ticket es una prueba dirigida, pero no repara la causa subyacente de SPN, clave o replicación. Antes de purgar, se registran principal, SPN de destino, KDC, KVNO, Enctype, flags y campos temporales; de lo contrario, desaparece la mejor evidencia del error.

Windows registra las solicitudes TGT en 4768, los tickets de servicio en 4769, los errores de preautenticación en 4771 y otros errores AS en 4772, siempre que estén activas las políticas de auditoría avanzada adecuadas. El volumen es alto en los KDC. Por ello, la monitorización requiere agregación por Result Code, cliente, servicio de destino, DC y Enctype, así como líneas de base, en lugar de tratar cada solicitud TGS correcta como una alarma ([Microsoft: Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration)).

La recuperación depende de la implementación del KDC. En AD DS, los datos KDC y `krbtgt` pertenecen al modelo de recuperación de System State y forest; un archivo de base de datos arbitrario o una exportación LDIF no es una copia de seguridad Kerberos válida. Los operadores de realms MIT independientes deben respaldar conjuntamente la base de datos del KDC, el material Stash/Master Key, la configuración, las ACL y la replicación. Las keytabs de servicio deben inventariarse adicionalmente y, tras una restauración, comprobarse su coherencia de KVNO y clave ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [MIT Kerberos: Backups of secure hosts](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html)).

Para la resolución de problemas, se comprueba la ruta en orden inverso: nombre de destino y SPN, ticket de servicio existente, respuesta TGS, TGT, detección de realm, DNS y hora.

## Herramientas de diagnóstico

El diagnóstico comienza desde la misma zona de red, con el mismo nombre de destino, realm, contexto de usuario o servicio y la misma sesión de inicio de sesión que la aplicación. Los datos de prueba utilizan nombres reservados. Las salidas de tickets y keytabs pueden revelar principals e infraestructura; el material de claves nunca debe llegar a tickets, chat ni argumentos de proceso.

### Determinar KDC y servicio de contraseña mediante DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Dienstsuche">
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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran prioridad, peso, puerto y destino. Después deben comprobarse la resolución A/AAAA y la accesibilidad de cada destino proporcionado. RFC 4120 define la detección DNS-SRV, pero permite configuración local de realm; por tanto, una prueba SRV vacía solo demuestra un error cuando el cliente concreto utiliza detección DNS ([RFC 4120, sección 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.3), [RFC 2782](https://datatracker.ietf.org/doc/html/rfc2782)).

### Comparar hora y accesibilidad TCP

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Zeit- und Portprüfung">
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

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) y [`timedatectl`](https://man7.org/linux/man-pages/man1/timedatectl.1.html) muestran la fuente y el estado de sincronización; [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) añade datos de offset y seguimiento para Chrony. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) o [`nc`](https://man.openbsd.org/nc) demuestran únicamente TCP 88. UDP, el protocolo KDC, el realm y la preautenticación requieren una prueba AS/TGS real.

### Volver a obtener y mostrar tickets de forma específica

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Ticketprüfung">
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

El [`klist`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) de Windows funciona en el contexto de la sesión de inicio de sesión seleccionada. En MIT Kerberos, [`kdestroy`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) elimina, [`kinit`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) y [`kvno`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) obtienen, y [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) muestra las credenciales. [`KRB5_TRACE`](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) hace visible la selección de KDC y la ruta de protocolo. Antes de `purge` o `kdestroy` debe documentarse un ticket de error existente.

### Comprobar SPN y claves de servicio entre sí

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-SPN- und Keytabprüfung">
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

[`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) muestra la cuenta de destino de AD y busca duplicados en todo el forest con `-X -F`. [`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) lee SPN y Enctypes declarados; un valor ausente tiene semántica de fallback específica del producto y no debe interpretarse de forma general como «sin AES». El [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) de MIT muestra principal, KVNO y Enctype de la keytab; [`kvno -k`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) solicita un ticket y lo valida con la keytab indicada.

### Asignar errores a una fase de protocolo

| Código o síntoma | Fase y límite frecuente | Siguiente evidencia |
|---|---|---|
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | AS: principal de cliente desconocido en el realm seleccionado | asignación de realm, UPN/principal, KDC emisor y replicación |
| `KDC_ERR_PREAUTH_FAILED` | AS: clave, contraseña, certificado o método Pre-Auth rechazado | hora del cliente, tipo Pre-Auth, clave de cuenta y auditoría KDC 4771 |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` | TGS: SPN de destino no encontrado o no resoluble de forma inequívoca | SPN solicitado exacto, búsqueda en todo el forest y realm de destino |
| `KDC_ERR_ETYPE_NOSUPP` | AS/TGS: no hay intersección común de Enctype/clave | oferta del cliente, política KDC, claves de cuenta, keytab y 4768/4769 |
| `KRB_AP_ERR_MODIFIED` | AP: el ticket no coincide con la clave del servicio que responde | cuenta SPN, identidad de proceso, keytab, KVNO, Enctype y nodo backend |
| `KRB_AP_ERR_SKEW` | AS/AP: hora fuera de tolerancia | hora del cliente, KDC y servicio, así como cada fuente horaria |
| `KRB_AP_ERR_TKT_EXPIRED` | AP: ticket fuera de la ventana de validez | caché, `endtime`, renovación, hora del cliente y reinicialización |
| `KDC_ERR_BADOPTION` | TGS/S4U: flag, delegación o política no permitidos | flag Forwardable, cuenta frontend, SPN backend y configuración de delegación |
| Ticket Kerberos presente, la aplicación utiliza NTLM | Negociación de mecanismo o Target Name | resultado SPNEGO, URL/FQDN, política de zona/cliente y SPN realmente formado |

Los códigos de error están estandarizados en RFC 4120; Windows añade contexto de estado y auditoría. La automatización debe conservar el código numérico, la fase, el KDC, el principal de cliente y el principal de destino. El texto libre por sí solo no es estable ni inequívoco. Para análisis de paquetes, [Wireshark](https://www.wireshark.org/docs/dfref/k/kerberos.html) puede filtrar por `kerberos`; las partes de ticket cifradas permanecen deliberadamente ilegibles sin las claves adecuadas ([RFC 4120, sección 7.5.9](https://datatracker.ietf.org/doc/html/rfc4120#section-7.5.9), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)).

## Historia técnica

Kerberos surgió a principios de la década de 1980 en el MIT Project Athena. El protocolo se basa conceptualmente en trabajos de terceros de confianza de Needham y Schroeder, así como de Denning y Sacco. Las versiones 1 a 4 se desarrollaron en el entorno Athena; la versión 4 fue la primera versión ampliamente utilizada. El nombre hace referencia a Kerberos, el guardián de múltiples cabezas de la mitología griega, y representa el papel central de confianza del KDC ([RFC 4120, sección 1](https://datatracker.ietf.org/doc/html/rfc4120#section-1), [MIT Kerberos Consortium: Documentation](https://kerberos.org/docs/index.html)).

Kerberos V5 eliminó las limitaciones de la versión 4 en nombres, Lifetime de tickets, criptografía, Cross-Realm y extensibilidad. RFC 1510 estandarizó V5 en 1993. RFC 4120 sustituyó esta especificación en 2005 con aclaraciones y una descripción ASN.1 completa. Posteriormente, la familia de protocolos se amplió de forma modular, entre otros mediante PKINIT, el marco de preautenticación y FAST, GSS-API, referrals y nuevos perfiles AES ([RFC 1510](https://datatracker.ietf.org/doc/html/rfc1510), [RFC 4120](https://datatracker.ietf.org/doc/html/rfc4120), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113](https://datatracker.ietf.org/doc/html/rfc6113)).

Microsoft convirtió Kerberos V5 en el protocolo central de autenticación de dominio con Windows 2000 y lo vinculó con principals AD, SPN, PAC, SSPI, trust referrals y extensiones de delegación. Paralelamente, MIT Kerberos, Heimdal y otras implementaciones siguieron siendo interoperables mediante protocolos IETF y GSS-API. La criptografía evolucionó desde DES y posteriormente RC4 hacia perfiles AES; RFC 8429 inició en 2018 la retirada de 3DES y RC4. La historia técnica explica por qué los dispositivos heredados, las cuentas de servicio antiguas y las claves de trust aún muestran límites de Enctype, sin que el artículo fije un estado efímero de versiones de producto ([MS-KILE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/), [RFC 3962](https://datatracker.ietf.org/doc/html/rfc3962), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429)).

## Fuentes

- [IETF RFC 3244 – Microsoft Windows 2000 Kerberos Change Password and Set Password Protocols](https://datatracker.ietf.org/doc/html/rfc3244)
- [MIT Kerberos – Mapping Hostnames onto Kerberos Realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)
- [IANA – Kerberos service names and ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos)
- [RFC 4120 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc4120) – arquitectura, mensajes, tickets, flags, códigos de error, transporte y modelo de seguridad.
- [RFC 3961, sección 3](https://datatracker.ietf.org/doc/html/rfc3961)
- [RFC 4121 – Kerberos V5 GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4121) – contexto GSS, tokens, integridad y confidencialidad.
- [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)
- [MIT Kerberos – Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html) – contenido de keytab, KVNO, Enctype y clave.
- [Microsoft – Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) – unicidad SPN y vinculación con cuentas de servicio.
- [Microsoft – setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) – consulta, registro y búsqueda de duplicados de SPN.
- [Microsoft – ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) – interoperabilidad AD/keytab.
- [MIT Kerberos – Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html) – protección de keytab, rotación, hora y DNS.
- [RFC 4556 – PKINIT](https://datatracker.ietf.org/doc/html/rfc4556) – preautenticación de clave pública.
- [RFC 6113 – Generalized Framework for Kerberos Pre-Authentication](https://datatracker.ietf.org/doc/html/rfc6113) – marco Pre-Auth y FAST.
- [RFC 8429 – Deprecate 3DES and RC4 in Kerberos](https://datatracker.ietf.org/doc/html/rfc8429) – IETF Best Current Practice para Enctypes obsoletos.
- [Microsoft – Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) – inventario de Enctype, claves de cuenta y eventos.
- [Microsoft – Kerberos authentication troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance) – códigos de error de Windows, hora, SPN y doble salto.
- [Microsoft – Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service) – servicio horario y dependencia de Kerberos.
- [Microsoft – Kerberos KDC registry and protocol settings](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys) – tamaño UDP, fallback TCP y diagnóstico KDC.
- [RFC 4178 – SPNEGO](https://datatracker.ietf.org/doc/html/rfc4178) – negociación de mecanismos GSS.
- [Microsoft – Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) – arquitectura Windows, KDC, SSPI y autenticación mutua.
- [RFC 4752 – SASL GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4752) – integración en LDAP, IMAP, SMTP y otros protocolos SASL.
- [MS-KILE – Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13) – intercambios AS, TGS y AP en Windows.
- [MS-PAC – Privilege Attribute Certificate](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962) – grupos, SID, perfiles, políticas y firmas.
- [MS-KILE – PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909) – generación de PAC-Authorization-Data.
- [RFC 8009 – AES with HMAC-SHA2 for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc8009) – Enctypes AES-SHA2.
- [Microsoft – Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts) – gMSA, administración de contraseñas y SPN.
- [RFC 6806 – Kerberos Principal Name Canonicalization and Cross-Realm Referrals](https://datatracker.ietf.org/doc/html/rfc6806) – referrals y canonicalización de nombres.
- [Microsoft – Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts) – trusts, Protocol Transition y constrained delegation.
- [MS-SFU – Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a) – flujos de delegación y riesgo de TGT reenviados.
- [MS-SFU – Service for User Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf) – S4U2self y S4U2proxy.
- [MS-SFU – Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d) – Protocol Transition y constrained delegation.
- [Microsoft – Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration) – eventos 4768, 4769, 4771 y 4772.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – copia de seguridad de AD DS/KDC.
- [MIT Kerberos – kdb5_util](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html) – base de datos KDC y copia de seguridad en MIT Kerberos.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – consultas DNS SRV en Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – consultas DNS SRV en sistemas Unix.
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – prioridad SRV, peso, puerto y destino.
- [Microsoft – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) – fuente horaria de Windows y diagnóstico de offset.
- [Linux man-pages – timedatectl(1)](https://man7.org/linux/man-pages/man1/timedatectl.1.html) – estado de sincronización horaria en Linux.
- [Chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html) – diagnóstico de offset, fuente y seguimiento.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnóstico de conexión TCP en Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – prueba de puerto TCP en sistemas Unix.
- [Microsoft – klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) – cachés de tickets de Windows y prueba de ticket de servicio.
- [MIT Kerberos – kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) – eliminar caché de credenciales.
- [MIT Kerberos – kinit](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) – obtención de TGT y opciones Pre-Auth.
- [MIT Kerberos – kvno](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) – ticket de servicio y validación de keytab.
- [MIT Kerberos – klist](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) – visualización de caché de credenciales y keytab.
- [MIT Kerberos – kerberos environment](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) – `KRB5_TRACE` y variables de caché/keytab.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) – atributos SPN y Enctype de una cuenta de servicio.
- [Wireshark – Kerberos display filter reference](https://www.wireshark.org/docs/dfref/k/kerberos.html) – campos de protocolo y filtros de visualización.
- [MIT Kerberos Consortium – Documentation](https://kerberos.org/docs/index.html) – origen en Project Athena e historia de versiones.
- [RFC 1510 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc1510) – especificación central histórica de V5 de 1993.
- [MS-KILE – Kerberos Protocol Extensions](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/) – Kerberos de Active Directory y extensiones Microsoft.
- [RFC 3962 – AES Encryption for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc3962) – perfiles AES-SHA1.
