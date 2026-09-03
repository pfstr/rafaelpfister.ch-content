---
title: "SEPPmail 15.0.6 y 15.0.6.1: correcciones de seguridad y nuevas funciones de administración"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail publicó en julio de 2026 la versión de parche 15.0.6 y el hotfix 15.0.6.1. Además de vulnerabilidades corregidas en la generación de PDF y el procesamiento de PGP, las versiones incorporan un campo MFA independiente, autenticación LDAP para la GUI de administración y correcciones en RuleEngine, Webmail y la API REST."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min de lectura"
themen:
  - seppmail
slug: "seppmail-15-0-6-y-15-0-6-1-correcciones-de-seguridad-y-nuevas-funciones-de-administracion"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
translationSourceHash: 636a7246234584a2b5797f53239fe65129de0f4463b8f773d0a7d9ed06d61f91
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:15:32.316Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/seppmail-15-0-6-y-15-0-6-1-correcciones-de-seguridad-y-nuevas-funciones-de-administracion
---

# SEPPmail 15.0.6 y 15.0.6.1: correcciones de seguridad y nuevas funciones de administración

SEPPmail publicó el 21 de julio de 2026 la versión de parche 15.0.6 y, un día después, el hotfix 15.0.6.1. La versión de parche corrige varias vulnerabilidades, actualiza OpenSSH y OpenSSL e incorpora mejoras perceptibles para la administración. El hotfix corrige dos errores en RuleEngine que se introdujeron o se hicieron visibles con la versión 15.0.6. Los cambios también afectan a las appliances operadas como HIN Mailgateway, ya que se basan en el mismo firmware de SEPPmail.

## Hotfix 15.0.6.1 del 22 de julio de 2026

El hotfix corrige dos aspectos de RuleEngine. En primer lugar, un valor no definido en el objeto Message impedía que las entradas de registro se escribieran en el registro de correo. Por tanto, los mensajes afectados pasaban por el sistema sin quedar registrados. En segundo lugar, RuleEngine reconoce ahora la dirección de los correos electrónicos archivados para que su entrega se gestione correctamente.

Quienes ya hayan instalado la versión 15.0.6 o tengan previsto actualizar deberían pasar directamente a la versión 15.0.6.1.

Al parecer, las appliances de HIN también han recibido el hotfix: un HIN Mailgateway con la versión instalada 15.0.6-RC-42-g278c81f84 informa ahora de 15.0.6-RC-88-g916e513cc como próxima versión en la rama 15.0. Las denominaciones RC del firmware de HIN no pueden asignarse directamente a una versión de SEPPmail, pero el momento en que se ofrece apunta al hotfix.

## Correcciones de seguridad en 15.0.6

La parte más importante de la versión de parche son tres correcciones en la arquitectura de seguridad:

- Se cerró una posible vulnerabilidad de recorrido de rutas en la generación de PDF. Fue descubierta por InfoGuard.
- Todo el contenido descifrado mediante PGP se codifica ahora en Base64 para evitar la inyección de estructuras MIME.
- La función hashencrypt se cambió a AES-256-CBC con PBKDF2.

Además, se actualizaron bibliotecas: OpenSSH 10.4 y OpenSSL 3.0.21 corrigen conjuntamente más de veinte CVE. Solo por estos puntos, la actualización es recomendable para sistemas en producción.

## Nuevas funciones para la administración

Tres cambios en la GUI de administración destacan en el día a día:

- **Campo de entrada MFA independiente:** Ya no es necesario añadir el segundo factor a la contraseña, sino que cuenta con su propio campo. Esto elimina una fuente de errores de larga data al iniciar sesión.
- **Autenticación LDAP para la GUI de administración:** Los administradores ya pueden autenticarse contra un servidor LDAP externo en lugar de mantener cuentas locales en la appliance. La configuración se describe en el artículo sobre la [conexión de la GUI de administración a Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Aún estoy probando si HIN Mailgateway también ha recibido esta función y añadiré la información al artículo posteriormente; dado que HIN utiliza la misma base de firmware, parto de esa premisa.
- **Botón AutoRenew para MPKI:** En la configuración del conector MPKI, la renovación automática de certificados puede iniciarse manualmente mediante «Trigger AutoRenew...» .

Además, la appliance utiliza ahora sistemáticamente zonas horarias válidas (predeterminada: Europe/Zurich), y la System Object ID en System >> Advanced View se valida como una OID válida.

## Procesamiento de correo y Webmail

Se corrigieron cuatro aspectos en RuleEngine. El tratamiento del asunto funciona ahora también con codificaciones desconocidas. Los mensajes se devuelven cuando se solicita explícitamente una firma, pero no puede crearse; hasta ahora, estos mensajes podían continuar sin firma. Las copias de archivo pasan ahora por la función de entrega y reciben así cabeceras ARC. Y en los mensajes PGP sin datos MDC, los errores MDC se ignoran en lugar de interferir con el procesamiento.

En Webmail (GINA) se corrigieron cuatro errores: vuelve a funcionar la eliminación automática de cuentas no registradas tras expirar el período de gracia, la función hashdecrypt devolvía en determinados casos un resultado de descifrado falso positivo, añadir un adjunto vaciaba los campos Para y CC, y la salida de hora en los registros SMS era incorrecta.

## API REST, clúster y copia de seguridad

La API REST recibe correcciones en varios endpoints: /system/ifaliasconfig (gestión de valores null), /system/applySysconfig (configuración de acceso), /crypto/domain/{domainName} (carga de certificados de dominio), así como GET y POST /ssl/csr. El tiempo de espera para las llamadas REST se incrementó de 300 a 900 segundos, lo que hace más fiables las solicitudes de larga duración, como cambios de configuración mayores.

En funcionamiento en clúster, una IP CARP existente bloqueaba hasta ahora la configuración IP de un miembro recién añadido; esto se ha corregido. Antes de crear el snapshot diario, la copia de seguridad comprueba ahora adicionalmente si hay una base de datos corrupta antes de escribir el snapshot.

## Relación con el fallo de inicio de sesión en 15.0.5

Al actualizar un clúster a la versión 15.0.5, el inicio de sesión podía fallar en ambos nodos. El patrón del error y la recuperación se describen en el artículo sobre el [fallo de inicio de sesión tras la actualización a 15.0.5](/blog/hin-update-issue-version-15.0.5). El fabricante ya conocía entonces el problema y anunció una corrección para una versión posterior.

En las notas de la versión 15.0.6 aparece ahora precisamente una entrada que encaja con este patrón de error: «prevent password rehashing when cluster members use different firmware versions». Durante una actualización de clúster, los nodos operan inevitablemente de forma temporal con versiones de firmware diferentes. Si un nodo vuelve a calcular hashes de contraseñas durante esta fase y los replica en el clúster, los hashes dejan de coincidir con la otra versión y el inicio de sesión falla en ambos nodos, exactamente como en el fallo observado entonces. Las notas de la versión no mencionan explícitamente el fallo de inicio de sesión, pero la entrada cubre exactamente la configuración que lo había provocado. Por tanto, la causa queda abordada en la versión 15.0.6; el procedimiento de emergencia con disolución del clúster necesario en la versión 15.0.5 debería dejar de ser necesario en futuras actualizaciones.

## Correcciones menores

En el registro de correo se corrigió la ordenación por fecha, que hasta ahora ordenaba alfabéticamente en lugar de cronológicamente, y el tamaño mostrado de los mensajes LFT vuelve a ser correcto. Los accesos a cabeceras X inexistentes ya no se registran. El conector CertCentral de MPKI gestiona de forma más robusta los errores de entrada y de REST.

## Valoración

Los dos errores de RuleEngine corregidos por el hotfix aconsejan omitir la versión 15.0.6 e implementar directamente la versión 15.0.6.1. En clústeres, cree snapshots de ambos nodos antes de actualizar y respete el orden de actualización indicado en la documentación del fabricante. El fallo de inicio de sesión en la versión 15.0.5 mostró por qué esta preparación no es un mero formalismo.

## Fuentes

1.  [Documentación de SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Notas oficiales de las versiones 15.0.6 y 15.0.6.1 con todos los detalles.

2.  [HIN Mailgateway 15.0.5: solucionar el fallo de inicio de sesión tras la actualización del clúster](/blog/hin-update-issue-version-15.0.5): Por qué los snapshots y el orden correcto de actualización son decisivos en el clúster.
