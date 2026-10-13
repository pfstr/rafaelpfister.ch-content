---
title: "SEPPmail: pasarela de correo, claves y entrega GINA"
blatt: "seppmail"
description: "SEPPmail para administradores de mensajería: arquitectura SMTP en línea, canalización de Postfix y Ruleset, S/MIME, OpenPGP y GINA, estado de claves y certificados, integración de directorios, clúster, monitorización, copia de seguridad y recuperación."
fakten:
  - label: Producto
    wert: SEPPmail Secure E-Mail Gateway
    href: https://www.seppmail.com/products/
  - label: Función del sistema
    wert: Pasarela SMTP en la ruta de los mensajes
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Núcleo de transporte
    wert: Postfix MTA
    href: https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html
  - label: Base del sistema
    wert: Pila de appliance basada en OpenBSD
    href: https://docs.seppmail.com/ch/20_revision-history.html
  - label: Criptografía de mensajes
    wert: S/MIME · OpenPGP · claves de dominio
    href: https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html
  - label: Sin clave del destinatario
    wert: Entrega web GINA mediante HTTPS
    href: https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html
  - label: Modelo de reglas
    wert: Ruleset generado · políticas · Custom Commands
    href: https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html
  - label: Directorio
    wert: LDAP/LDAPS para consultas de claves, identidad y reglas
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Interfaces web
    wert: GINA 443 · Admin 8443 · REST 8445
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Monitorización
    wert: Postmaster · SNMP · NRPE · Syslog
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Transporte de clúster
    wert: SSH entre appliances · NTP obligatorio
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Origen
    wert: Desarrollo de producto suizo desde 2001
    href: https://www.seppmail.com/de/die-seppmail-story/
werbung:
  - tools
  - newsletter
ctaThemen:
  - seppmail
translationSourceHash: f01b161a8f003ca453398dd3d393ca671b7346ad63164cd311dabb11055526fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:49:35.238Z
translationReview: automatic
---

# SEPPmail: pasarela de correo, claves y entrega GINA

SEPPmail es una pasarela de correo especializada que recibe mensajes en la ruta SMTP, los procesa según reglas centrales y los reenvía a un sistema de correo interno, un Smarthost o el siguiente interlocutor de Internet. Los tipos de procesamiento documentados incluyen firma o cifrado S/MIME y OpenPGP, cifrado de dominios basado en S/MIME, requisitos TLS relacionados con la conexión y la entrega web GINA para destinatarios sin una clave pública utilizable ([SEPPmail: Products](https://www.seppmail.com/products/), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

El procesamiento central simplifica el cliente: los usuarios pueden enviar desde su sistema de correo habitual, mientras la pasarela busca claves, evalúa políticas y ejecuta operaciones criptográficas. Al mismo tiempo, esto convierte la appliance en un límite de confianza y disponibilidad. Ve el contenido del mensaje antes de un cifrado saliente o después de un descifrado entrante, conserva material de clave privada y puede convertirse en el nuevo punto de responsabilidad del mensaje tras una aceptación SMTP correcta. Por tanto, un administrador no solo debe demostrar que el «cifrado está activo», sino poder seguir toda la ruta formada por enrutamiento, regla, selección de clave, procedimiento de entrega, cola, registro y recuperación ([RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321#section-6.1), [RFC 8551: S/MIME](https://datatracker.ietf.org/doc/html/rfc8551), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

La explicación sigue un mensaje a través de la aceptación SMTP, el Ruleset y el método de cifrado elegido hasta la entrega o el portal GINA. Las claves, el clúster, el diagnóstico y la recuperación se explican posteriormente siguiendo esta ruta.

SEPPmail se sitúa como pasarela con estado en la ruta del mensaje. Acepta SMTP, determina la política y la capacidad del destinatario, elige S/MIME, OpenPGP, TLS o GINA y luego reenvía el mensaje. La arquitectura, la operación y la recuperación se orientan a este proceso.

## Enfoque arquitectónico: pasarela en línea con estado

En la configuración más sencilla, el sistema de correo interno envía mensajes salientes mediante [SMTP](/kb/smtp) a SEPPmail; para los mensajes entrantes, el [registro MX](/kb/dns#dns-im-mail--und-identitätsbetrieb) público apunta directamente o a través de un Smarthost previo a la pasarela. Para cada dominio gestionado se define un servidor de reenvío interno. Para los mensajes salientes, la appliance puede entregar por sí misma mediante DNS o utilizar un SMTP Smarthost definido. La matriz de firewall documentada por el fabricante representa estas direcciones por separado e indica TCP 25 entre el servidor de correo, la appliance, Internet o el Smarthost, según corresponda ([SEPPmail: Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

La pasarela no es un dispositivo transparente de filtrado de paquetes. Termina una sesión SMTP y abre otra nueva para el reenvío. Tras `250` al final de la fase DATA, la responsabilidad de entrega corresponde al sistema SMTP que acepta el mensaje; el establecimiento correcto de una conexión o `STARTTLS` por sí solo no demuestra esta transferencia ([RFC 5321, sección 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207: SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)). Desde el punto de vista operativo, surgen al menos cinco objetos de comprobación separados: sesión SMTP entrante, procesamiento local, siguiente salto SMTP, estado de claves y políticas y, cuando corresponda, estado web de GINA.

Para Exchange Online, SEPPmail describe un modelo explícito de conectores y reglas de flujo de correo. En entornos multiinquilino, la guía del fabricante recomienda conectores basados en certificados; además, los hosts de pasarela múltiples deben tenerse en cuenta en los conectores de Microsoft y, en su caso, en la configuración antispam. Microsoft también documenta los conectores como una relación de confianza y enrutamiento definida, no como una desactivación general de todas las demás rutas de entrega ([SEPPmail: Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html), [Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)).

## Pila tecnológica y estados persistentes

El historial de revisiones disponible públicamente documenta una pila de appliance basada en OpenBSD con Postfix como MTA, así como, entre otros, Apache HTTP Server, OpenSSL o LibreSSL, GnuPG, OpenLDAP, Dovecot, Fetchmail, ClamAV, OpenSSH y Perl. Estos componentes cumplen funciones diferentes; las versiones indicadas en una nota de versión concreta no son características atemporales del producto. Para la operación, lo decisivo es qué interfaz admite el fabricante: por ejemplo, los parámetros SMTP y TLS se administran mediante «Mail System», mientras que «Extended Postfix MTA settings» se describe expresamente como arriesgado y no sincronizado en el clúster ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html), [SEPPmail: Mail System – Extended Postfix settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

El flujo de correo visible es solo una parte del estado. Para un inventario fiable, deben inventariarse por separado al menos los siguientes objetos:

| Espacio de estado | Ejemplos | Por qué se comprueba por separado |
|---|---|---|
| SMTP y enrutamiento | Managed Domains, Forwarding Server, Smarthost, redes de relay, destinos TLS | determina la aceptación y el siguiente salto |
| Procesamiento | Ruleset generado, Encryption Policies, Custom Commands, desencadenantes de cabecera y asunto | determina la firma, el cifrado, GINA, el rebote o la ruta de texto sin cifrar |
| Criptografía | claves privadas S/MIME/OpenPGP, claves públicas de destinatarios, confianza de CA, OCSP/CRL, claves de dominio | determina si una operación es posible y verificable, y para quién |
| GINA | dominios, diseño, cuentas, contraseñas, historial de entregas, mensajes y claves almacenados | constituye una ruta adicional de HTTPS, identidad y conservación |
| Plataforma | configuración de la appliance, nombre del sistema, red, certificados, licencias, función de clúster | determina accesibilidad, administración y reinicio |
| Observabilidad | registros de correo, sistema y auditoría, mensajes de Postmaster, SNMP/NRPE/Syslog | proporciona pruebas y alertas, pero no está incluida íntegramente en la copia de seguridad |

La asignación se basa en la estructura documentada de menús y componentes, la administración de cuentas GINA, la descripción del clúster y la Quick Setup Guide. Es un modelo de inventario para administradores, no una afirmación sobre una estructura interna de base de datos no documentada ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html), [SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-seppmail.svg?v=20260813" title="Interaktive Infografik: SEPPmail-SMTP-Pfad, Ruleset, Verschlüsselungsverfahren, GINA, Schlüsselzustand, Cluster, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-seppmail.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Aceptación SMTP, enrutamiento y límite de relay

Un Managed Domain vincula un espacio de nombres SMTP aceptado con el Forwarding Server interno y otros ajustes específicos del dominio. Opcionalmente, la pasarela puede comprobar la dirección del destinatario mediante una consulta SMTP contra este servidor. La ruta saliente se determina mediante el MTA integrado a través de DNS o mediante un servidor SMTP configurado; los ajustes de TLS y de enrutamiento basado en remitente específicos del dominio pueden anular esta ruta predeterminada ([SEPPmail: Mail System – Managed Domains and Outgoing Server](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

«Relaying allowed» define desde qué direcciones IP o redes la appliance acepta mensajes para destinatarios externos. La documentación del fabricante advierte expresamente de no introducir de forma general el servidor saliente, pues ello puede crear un Open Relay. Por ello, la apertura de puertos, el permiso de relay, el dominio remitente gestionado y, si procede, la autenticación SMTP son controles distintos. Una prueba TCP positiva no dice nada sobre la decisión de relay tras `MAIL FROM` y `RCPT TO` ([SEPPmail: Mail System – Relaying](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [RFC 5321: SMTP commands](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1)).

Con varias pasarelas, el equilibrador de carga precedente o el servicio remitente debe poder alcanzar cada host de producción. Al mismo tiempo, los sistemas posteriores deben aceptar todas las identidades de pasarela esperadas. En Exchange Online, esto afecta a certificados de conectores o rangos IP; en un entorno de clúster también a IP virtuales y a la particularidad documentada de que se recibe con un alias IP, pero se envía mediante la dirección física del sistema remitente ([SEPPmail: Firewall and IP aliases](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Exchange Online clustering and multi-host](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)).

Tras la aceptación SMTP, el Ruleset decide qué ruta de protección y entrega se aplica. Esta decisión debe poder rastrearse posteriormente a partir del mensaje, la identidad, los datos del directorio y la versión de la regla.

## Ruleset, políticas y evidencia de decisión

SEPPmail genera reglas de procesamiento ejecutables a partir de los ajustes del Ruleset Generator. Encryption Policies adicionales pueden imponer procedimientos como solo S/MIME, solo OpenPGP, solo GINA o una selección jerárquica para Managed Domains seleccionados y flags opcionales. Según el fabricante, tras un cambio de política debe regenerarse el Ruleset antes de que el cambio esté activo. «Guardado en la GUI» y «efectivo en la ruta activa del mensaje» son, por tanto, dos estados distintos ([SEPPmail: Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Custom Commands amplía esta canalización con condiciones y acciones. El fabricante documenta, entre otros, consultas LDAP, flags, creación de usuarios y claves, operaciones de cabecera, firma, cifrado, entrega y rechazo. Las macros se expanden al generar el Ruleset; un error de sintaxis puede impedir la generación ([SEPPmail: Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Los desencadenantes de cabecera o asunto son entradas en una decisión de seguridad. Solo deben evaluarse desde una dirección fiable o depurarse antes de su evaluación. Los ejemplos de SEPPmail para la firma entre inquilinos advierten expresamente de no asumir sin verificar la identidad del remitente. Por tanto, para cada desencadenante deben documentarse el origen, el conector permitido, la autenticación, la normalización, la prioridad y el comportamiento Fail-Closed ([SEPPmail: Signing emails between clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html), [SEPPmail: Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)).

Una evidencia de decisión reproducible contiene al menos: remitente y destinatario del sobre, la cabecera visible `From:` y, en su caso, la cabecera `Sender:`, dirección y Managed Domain, la política o regla desencadenante, las claves encontradas, el procedimiento elegido, la acción final y el ID de cola. Esto es especialmente importante porque la identidad de firma S/MIME suele comprobarse frente al remitente visible, mientras que el enrutamiento SMTP trabaja inicialmente con el sobre ([SEPPmail: Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html), [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2)).

## S/MIME, OpenPGP y TLS como capas de protección separadas

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) encapsula contenidos MIME firmados o cifrados con Cryptographic Message Syntax. Las identidades y las claves públicas suelen vincularse mediante certificados X.509; por ello, la validación de ruta, el período de validez, el uso de clave y la información de revocación forman parte de la decisión ([RFC 5652: CMS](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280: PKIX certificate validation](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960: OCSP](https://datatracker.ietf.org/doc/html/rfc6960)). SEPPmail puede administrar certificados de usuario y dominio y obtenerlos mediante interfaces MPKI; la documentación de firewall indica HTTPS para ello, así como HTTP o HTTPS para OCSP/CRL ([SEPPmail: Firewall – MPKI and revocation](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) utiliza otro modelo de paquetes, claves y confianza. La existencia de una clave pública no demuestra automáticamente que pertenezca al destinatario previsto o que deba seguir utilizándose. La huella digital, el origen, la User ID, la caducidad, la revocación, el algoritmo y la capacidad de descifrado mediante clave privada deben tratarse como un ciclo de vida. La Encryption Policy distingue entre PGP/MIME y Inline PGP y puede configurar por separado la firma y el cifrado ([SEPPmail: Encryption Policy – OpenPGP options](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html), [RFC 9580: Key structures](https://datatracker.ietf.org/doc/html/rfc9580#section-5.5)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) protege una conexión entre dos saltos. Sin una política vinculante, STARTTLS es oportunista: si no está disponible, el MTA remitente puede entregar sin cifrar según la configuración. SEPPmail puede configurar TLS por destino con niveles, servidor fijo, huella de certificado u otros requisitos. Esto no es ni S/MIME ni OpenPGP y no protege el mensaje frente a los MTA participantes ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672: SMTP DANE](https://datatracker.ietf.org/doc/html/rfc7672), [SEPPmail: Mail System – TLS Settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Para el cifrado mediante pasarela, el texto sin cifrar y las claves privadas del remitente se encuentran dentro del límite de confianza de SEPPmail. Esto permite políticas centrales, pero también significa que las funciones administrativas, las copias de seguridad, las conexiones de clúster, las exportaciones de depuración, los registros y el procesamiento posterior de contenidos deben protegerse como si pudieran afectar a datos de comunicación confidenciales o claves. Debe verificarse mediante la documentación del fabricante y una prueba controlada si una exportación concreta contiene realmente contenido.

Si no hay disponible una clave de destinatario adecuada o una ruta de protección directa, GINA traslada el acceso a un portal web. Con ello, la identidad web, la sesión, el almacenamiento y el correo de notificación se añaden a la ruta SMTP.

## GINA como servicio web con identidad y almacenamiento

GINA es la ruta alternativa documentada cuando un mensaje no se entrega con una clave de destinatario adecuada o cuando una política impone expresamente GINA. La comunicación con el portal se realiza mediante HTTPS en TCP 443. Los GINA Domains controlan, entre otros aspectos, el nombre del portal, el diseño, el comportamiento de cuentas y contraseñas, la transmisión inicial de contraseñas y los parámetros de conservación ([SEPPmail: GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html), [SEPPmail: Firewall – GINA](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Para los administradores, GINA no es un simple formato de presentación, sino un estado de servicio propio. Una cuenta GINA posee estado, ajustes de contraseña y, en su caso, de autenticación externa, registros de usuario, historial de correo y una clave. La documentación del fabricante advierte de que eliminar la cuenta también elimina irrevocablemente su clave y que los mensajes GINA restantes pueden quedar ilegibles ([SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)).

Por tanto, la operación de GINA incluye como mínimo:

- un nombre de host público estable y conforme al certificado, así como su ciclo de vida de [DNS](/kb/dns);
- certificado HTTPS, cadena completa, renovación y accesibilidad externa;
- transmisión o derivación separada del secreto inicial;
- bloqueo y restablecimiento de cuentas, identidad externa y proceso de soporte;
- reglas de conservación y eliminación para mensajes, metadatos, registros y claves;
- una prueba de recuperación que abra efectivamente, tras la restauración, un mensaje GINA creado antes de la copia de seguridad.

La Quick Setup Guide señala que un cambio de nombre de host puede hacer que los mensajes GINA anteriores ya no sean accesibles. Además, documenta que una restauración en un sistema nuevo requiere la contraseña establecida al realizar la copia de seguridad ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Por tanto, el nombre de host, el certificado, el dominio GINA, la cuenta y la clave deben considerarse conjuntamente.

## Directorio y ciclo de vida de claves y certificados

LDAP/LDAPS puede proporcionar claves públicas, evaluar condiciones de usuario o grupo y limitar la creación automática de cuentas o claves. La appliance también puede ofrecer sus propios endpoints LDAP para distribuir claves públicas recopiladas o internas. Las direcciones y los puertos necesarios difieren según la función; la matriz oficial de firewall indica 389/636 para consultas de directorio externas o internas, así como puertos separados para los endpoints de Keyserver integrados ([SEPPmail: Firewall – LDAP and Key server](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Una consulta LDAP forma parte de la ruta del mensaje cuando una regla no puede decidir sin su resultado. Por ello, Search Base, filtro, clase de objeto, identidad de enlace, confianza TLS, timeout, comportamiento de caché y comportamiento ante indisponibilidad deben formar parte del modelo operativo. Un resultado de búsqueda vacío, un timeout y un certificado no fiable no deben tratarse como el mismo resultado. Los requisitos generales de protocolo y TLS se describen en el [artículo de LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen).

Para claves y certificados se necesitan, como mínimo, los siguientes ciclos de vida medibles por separado:

| Objeto | Generación u obtención | Control continuo | Evidencia de recuperación |
|---|---|---|---|
| Clave de usuario S/MIME | local, importación o MPKI | caducidad, ruta de CA, estado de revocación, vinculación de dirección, algoritmo | descifrar un mensaje antiguo con la clave privada histórica |
| Clave OpenPGP | local, importación o Keyserver | huella digital, User ID, caducidad/revocación, capacidades | descifrar un mensaje antiguo con la Secret Key correspondiente |
| Clave de dominio | Managed Domain Service o intercambio manual | estado publicado del interlocutor, rotación, ruta de correo completa | probar un mensaje de dominio antiguo y uno nuevo a través del límite de rotación |
| Certificado SMTP/HTTPS | importación o CA local/pública | SAN, cadena, caducidad, coincidencia de clave privada, política de pares | iniciar el servicio con la clave restaurada y la cadena esperada |
| Confianza de CA | importación de raíz/intermedia | propósito, alcance, eliminación de CA antiguas, OCSP/CRL | probar frente a frente una cadena válida conocida y una revocada |

Las características de comprobación se derivan de las normas respectivas y de las comprobaciones documentadas de importación, políticas y firmas. En caso de una firma S/MIME no válida, SEPPmail menciona en particular la confianza de CA, la cadena completa, la vinculación del remitente y una modificación posterior del contenido como causas ([SEPPmail: Reasons for an invalid signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Puesto que la pasarela, las claves, el directorio, GINA y la cola actúan conjuntamente, la alta disponibilidad debe cubrir más que un listener virtual. La separación entre frontend y backend determina qué estados deben estar disponibles conjuntamente.

## Clúster, separación frontend/backend y dominios de fallo

Según la matriz de firewall, la comunicación general de clúster utiliza SSH entre appliances; la sincronización horaria mediante NTP es obligatoria en el clúster. La matriz también indica IP virtuales y señala que, con alias IP, el envío se realiza mediante la dirección física del sistema activo. Por ello, las listas de permitidos de firewall, Smarthost y conectores deben reflejar la ruta real del remitente, no solo la dirección virtual de recepción ([SEPPmail: Firewall – Cluster and NTP](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Un clúster especializado frontend/backend separa el procesamiento de correo y los datos de configuración persistentes. Según la descripción del fabricante, el frontend no alberga una base de datos de configuración; la configuración y el material de clave necesarios se proporcionan desde el backend para una acción y después se descartan en el frontend. Sin embargo, cuando Large File Transfer está activado, sigue siendo necesario un estado de archivo adicional en el frontend ([SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Esta arquitectura reduce la retención permanente de claves en la DMZ, pero no elimina todas las dependencias. El backend, la ruta SSH, la resolución de nombres, la hora, los certificados y, en su caso, el almacenamiento LFT siguen siendo dominios de fallo. «El frontend acepta SMTP» no demuestra ni que el backend sea accesible ni que pueda ejecutarse una operación criptográfica. A la inversa, un backend funcional no demuestra que funcionen la IP virtual, la ruta externa y el siguiente salto SMTP.

No todos los ajustes abarcan todo el clúster. La documentación de Mail System designa varios parámetros relacionados con la máquina y, especialmente, los ajustes Extended Postfix y determinados destinos TLS como ajustes que deben mantenerse localmente. Por ello, antes de las pruebas de failover resulta útil una matriz objetivo-real por nodo: nombre del sistema, IP, certificado, Outgoing Identity, rutas TLS, Extended Settings, servicios locales, destino de monitorización y replicación de clúster esperada ([SEPPmail: Mail System – cluster notes](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Monitorización, registros, copia de seguridad y recuperación

Según el fabricante, la dirección Postmaster recibe mensajes importantes de Watchdog y Daily Report; sin una dirección configurada y realmente accesible, falta esta vía de alerta. Como complemento, SEPPmail documenta SNMP mediante UDP 161, NRPE mediante TCP 5666 y Syslog mediante UDP 514 o TCP 6514 protegido por TLS. Los valores y eventos que se supervisan deben definirse para cada instalación; una interfaz de monitorización abierta por sí sola no es un health check ([SEPPmail: Mail System – Postmaster](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Firewall – Monitoring and Syslog](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Un dashboard significativo separa al menos la aceptación y los rechazos SMTP, la antigüedad de la cola por Next Hop, los errores de Ruleset, el éxito y los errores criptográficos, la caducidad de certificados, la accesibilidad de OCSP/CRL y MPKI, la latencia LDAP, los errores de inicio de sesión y entrega de GINA, el estado de disco, la comunicación de clúster y la antigüedad de las copias de seguridad. Los umbrales necesitan relacionarse con el rendimiento normal y el RPO/RTO acordado. Una cola elevada puede indicar un interlocutor lento; una cola vacía también puede significar que ya no se acepta ningún mensaje.

La Quick Setup Guide describe una copia de seguridad protegida por contraseña que guarda datos de la appliance, incluidos usuarios y certificados registrados, pero excluye los registros. La contraseña es necesaria para la restauración en un sistema nuevo. La guía también describe una entrega nocturna al «Backup Operator» ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). De ello se derivan tres controles separados:

1. El archivo de copia de seguridad debe llegar fuera de la appliance y debe supervisarse su antigüedad.
2. La contraseña de copia de seguridad debe almacenarse por separado, protegida mediante control de acceso y disponible en caso de emergencia.
3. Los registros necesitan su propia ruta de exportación y conservación si deben mantenerse para fines forenses o de evidencia.

Una restauración solo está probada cuando el sistema de destino se inicia, la red y las licencias son correctas, las reglas están activas, SMTP funciona en ambas direcciones, pueden descifrarse contenidos históricos S/MIME/OpenPGP y GINA, y un caso de prueba documentado ejecuta la política esperada. La replicación de clúster no sustituye esta prueba: aborda la disponibilidad, no una copia independiente frente a errores de configuración, eliminación o claves comprometidas.

El diagnóstico sigue un mensaje desde la aceptación SMTP, pasando por la coincidencia de reglas y la decisión de clave, hasta la salida elegida o el portal GINA.

## Herramientas de diagnóstico

La búsqueda de errores sigue el mensaje desde el nombre público a través de SMTP y TLS hasta el Ruleset y la ruta de cifrado elegida. Así, primero se demuestra el transporte antes de modificar funciones de claves o del portal.

### DNS, MX y nombres públicos

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName securemail.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.securemail.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A securemail.example.ch
dig +short AAAA securemail.example.ch
dig +short TLSA _25._tcp.securemail.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) comprueban MX, dirección de pasarela y, en su caso, DANE-TLSA. Deben probarse por separado las vistas de los resolutores externos e internos. El nombre de host GINA también debe coincidir con el certificado; el MX por sí solo no dice nada sobre el Forwarding Server posterior ni sobre el conector de Exchange Online ([RFC 5321: Address resolution and mail routing](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).

### Accesibilidad TCP, SMTP y STARTTLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-SMTP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection securemail.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection securemail.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://securemail.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz securemail.example.ch 25
nc -vz securemail.example.ch 443
openssl s_client -starttls smtp -connect securemail.example.ch:25 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) solo demuestran accesibilidad TCP. [`curl.exe`](https://curl.se/docs/manpage.html) de Windows puede solicitar STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestra en sistemas Unix las capacidades SMTP, el handshake y la cadena de certificados. Para una transacción SMTP controlada con sobre y DATA es adecuado [`swaks`](https://jetmore.org/john/code/swaks/); solo debe utilizarse contra destinatarios de prueba autorizados. Una prueba de entrega solo está completa con ID de cola, evento de registro y llegada al siguiente salto.

### HTTPS de GINA y nombre de certificado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-GINA-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) y [`curl`](https://curl.se/docs/manpage.html) comprueban la accesibilidad HTTP y el comportamiento de respuesta; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) hace visibles la sesión TLS negociada y la cadena presentada. Un `200`, `301` o `302` todavía no demuestra ni un inicio de sesión GINA correcto ni el acceso a un mensaje histórico. Para ello se necesita un flujo de usuario sintético con registro, inicio de sesión, lectura, respuesta y restablecimiento según el dominio GINA configurado.

### Ruta de paquetes en un punto de medición autorizado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEPPmail-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seppmail.etl
pktmon stop
pktmon pcapng seppmail.etl -o seppmail.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seppmail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) solo capturan tráfico que pasa por el punto de medición elegido. En un cliente administrativo, esto no equivale automáticamente al tráfico de correo entre el sistema de correo y la pasarela; se requieren un host autorizado, un puerto espejo u otro punto de observación adecuado. SMTP antes de STARTTLS puede contener contenido personal; HTTPS y STARTTLS proporcionan al menos metadatos. Por tanto, la captura, el acceso, la conservación y la eliminación deben formar parte del runbook de incidentes.

## Historia técnica

SEPPmail describe el origen de la empresa en 2001: el fundador Stefan Klein desarrolló inicialmente una solución para la comunicación cifrada entre un bufete de abogados de Zúrich y sus clientes. De este requisito surgió posteriormente la empresa de productos ([SEPPmail: Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)).

La historia documentada públicamente del producto muestra la evolución desde una pasarela de cifrado central a una appliance modular: las notas de versión históricas mencionan ya en 2007 una CA/RA interna, cifrado de dominios S/MIME, funciones de webmail, antispam/antivirus y un usuario de copia de seguridad restringido. Generaciones posteriores añadieron, entre otras funciones, políticas multiinquilino, interfaces REST, conectores MPKI, Large File Transfer, DKIM/ARC, métodos S/MIME modernos y funciones de clúster especializadas ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html)).

La base tecnológica también siguió evolucionando. El historial de revisiones menciona a lo largo de los años OpenBSD, Postfix, Apache, Perl, GnuPG, OpenLDAP, OpenSSL/LibreSSL, Dovecot, Fetchmail y ClamAV. Para los administradores, este origen es relevante porque los términos de registro, los parámetros MTA, los formatos de certificado y el comportamiento de red proceden de estos componentes. Sin embargo, las intervenciones directas por debajo de la interfaz del fabricante no están automáticamente admitidas por ello; en particular, la documentación advierte sobre los cambios Extended Postfix y su falta de sincronización de clúster ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Fuentes

- [SEPPmail – Products](https://www.seppmail.com/products/)
- [SEPPmail – Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)
- [SEPPmail – Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)
- [RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [SEPPmail – Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)
- [SEPPmail – Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [SEPPmail – Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [SEPPmail – Revision History](https://docs.seppmail.com/ch/20_revision-history.html)
- [SEPPmail – GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)
- [SEPPmail – Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)
- [SEPPmail – Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html)
- [SEPPmail – LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)
- [SEPPmail – Signing Emails Between Clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html)
- [SEPPmail – Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)
- [SEPPmail – Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html)
- [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [SEPPmail – GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html)
- [SEPPmail – Reasons For An Invalid Signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [SEPPmail – Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)
