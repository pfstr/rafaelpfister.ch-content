---
title: "totemomail 9.5.1: corregida la toma de cuentas en el portal web y SSRF a través de correos entrantes"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 corrige ocho vulnerabilidades de seguridad y sitúa totemomail al nivel al que se refieren los avisos de Kiteworks sobre Email Protection Gateway publicados a finales de septiembre. Dos vulnerabilidades pueden explotarse sin iniciar sesión: un fallo en el restablecimiento de contraseña del portal web que permite tomar cuentas de administrador y solicitudes del lado del servidor a URL desde mensajes entrantes. También incluye correcciones de path traversal en la API de administración y la API de correo, así como bibliotecas actualizadas de Tomcat y Batik."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 min de lectura"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-corregida-la-toma-de-cuentas-en-el-portal-web-y-ssrf-a-traves-de-correos"
translationId: "article-2bd15f63632ab728"
draft: false
translationOf: totemomail-9-5-1-sicherheitsupdate
url: https://rafaelpfister.ch/es/blog/totemomail-9-5-1-corregida-la-toma-de-cuentas-en-el-portal-web-y-ssrf-a-traves-de-correos
translationSourceHash: 5d441bdc7ec39d7fcef423e826bf14772f11bca2aa3dbdf4e1e3c83297131635
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T19:46:57.091Z
translationReview: automatic
---

# totemomail 9.5.1: corregida la toma de cuentas en el portal web y SSRF a través de correos entrantes

totemo ha publicado totemomail 9.5.1 (Build 101). El parche incluye ocho correcciones de seguridad, una mejora y tres correcciones de errores. Dos de las vulnerabilidades pueden explotarse sin iniciar sesión: un fallo en el restablecimiento de contraseña del portal web, que también permite tomar cuentas de administrador, y solicitudes del lado del servidor a URL incluidas en mensajes entrantes. Ambas afectan a componentes accesibles desde Internet en la mayoría de las instalaciones. Por ello, planifique la actualización cuanto antes.

## De 9.2.1 directamente a 9.5.1

Las notas de la versión enumeran 9.5.1 inmediatamente después de 9.2.1; las versiones 9.3 y 9.4 no aparecen en el historial de totemomail. No obstante, una entrada de 9.5.1 hace referencia a un «security hardening introduced in 9.3.1». Las versiones intermedias proceden de Kiteworks Email Protection Gateway (EPG), basado en la misma base de código: los avisos de seguridad de Kiteworks indican como versiones corregidas EPG 9.3.1, 9.4.0, 9.4.1, 9.5.0 y 9.5.1. Para instalaciones de totemomail con 9.2.1, 9.5.1 es la siguiente actualización; las notas de la versión no incluyen ningún otro parche intermedio.

## Vulnerabilidades sin inicio de sesión

**Server-Side Request Forgery mediante el contenido del mensaje.** totemomail obtenía del lado del servidor URL presentes en mensajes procesados, sin restringir suficientemente el destino. Un remitente sin iniciar sesión podía diseñar un mensaje de modo que el gateway enviara solicitudes a servicios internos o a puntos de conexión de metadatos de instancias Cloud y devolviera las respuestas. Basta con un correo que procese el gateway; en un gateway situado en el flujo de correo entrante, cualquier remitente externo puede hacerlo. Esto resulta especialmente delicado en instalaciones en Azure, AWS u otras nubes, donde el punto de conexión de metadatos `169.254.169.254` proporciona datos de configuración y, según la plataforma, tokens de acceso de la instancia. Las notas de la versión no indican ningún número CVE para esta vulnerabilidad. Kiteworks enumera para EPG cinco avisos sobre acceso a recursos de red internos (CVE-2026-102095, CVE-2026-102102 a CVE-2026-102105, todos con CVSS 9.1, corregidos en EPG 9.5.0); los escuetos textos de los avisos no permiten determinar cuál de ellos corresponde a la entrada de totemomail.

**Restablecimiento de contraseña en el portal web (CVE-2026-85066).** Un fallo en el proceso de restablecimiento de contraseña permitía omitir los pasos de validación y establecer una nueva contraseña para una cuenta existente. Las notas de la versión incluyen explícitamente las cuentas de administrador. Kiteworks califica la vulnerabilidad en el aviso asociado GHSA-rwpq-5xfv-54pv con CVSS 9.8 (crítica) e indica EPG 9.5.0 como versión corregida. El portal web está pensado para destinatarios externos de mensajes cifrados y, por tanto, normalmente es accesible públicamente.

## Vulnerabilidades con derechos de administrador

Otras tres correcciones requieren iniciar sesión con derechos de administrador. Sin embargo, en combinación con la toma de cuentas en el portal web, estos requisitos pierden importancia.

- **API de administración, gestión de colas (CVE-2026-34843):** mediante cargas de archivos insuficientemente protegidas, era posible escribir archivos en ubicaciones arbitrarias del sistema de archivos. Según las notas de la versión, esto permitía la ejecución remota de código. El aviso GHSA-mhc9-4pph-2r9x califica la vulnerabilidad con CVSS 7.2; en EPG está corregida desde la versión 9.4.0.
- **Rule Designer:** al importar una configuración de reglas, la canalización de procesamiento podía cargar y ejecutar código fuera de los componentes autorizados de procesamiento de correo. Se requieren permisos para configurar reglas de correo.
- **API de correo:** al descomprimir archivos ZIP cargados existía una vulnerabilidad de path traversal. Ahora los archivos solo se descomprimen en el directorio temporal.

La cuarta corrección afecta a la propiedad `security.login.requestingKeysAvailable`. Hasta ahora, en el portal web solo ocultaba el enlace para solicitar claves públicas; la función en sí seguía siendo accesible mediante una solicitud directa. Desde la versión 9.5.1, la propiedad desactiva la función. Si ha configurado la propiedad para impedir solicitudes de claves, solo funcionará como se espera con esta actualización.

## Lo que lista adicionalmente el registro de Kiteworks

El 30 de septiembre de 2026, Kiteworks publicó de forma conjunta 125 avisos de seguridad en su registro de avisos en GitHub, 28 de ellos sobre Email Protection Gateway. Las notas de la versión de totemomail 9.5.1 mencionan claramente menos puntos. El registro no distingue entre EPG y totemomail; no permite determinar si cada vulnerabilidad de EPG también afecta a totemomail. Las entradas críticas de EPG son:

| CVE | CVSS | Impacto según el aviso | Corregido desde EPG |
|---|---|---|---|
| CVE-2026-54154 | 10.0 | Ejecución de código con privilegios de root sin iniciar sesión | 9.4.1 |
| CVE-2026-85066 | 9.8 | Toma de cuentas de usuario (restablecimiento de contraseña) | 9.5.0 |
| CVE-2026-85065 | 9.8 | Toma de cuentas de usuario | 9.5.0 |
| CVE-2026-102149 | 9.4 | Toma de cuentas de usuario | 9.5.1 |
| CVE-2026-102106 | 9.1 | Modificación de la configuración de administración | 9.5.0 |
| CVE-2026-102095, CVE-2026-102102 a 102105 | 9.1 | Acceso a recursos de red internos | 9.5.0 |

Además, hay 13 entradas clasificadas como altas, la mayoría de ellas de ejecución de código con derechos de administrador (CVSS 7.2). La versión corregida más alta indicada es 9.5.1; Kiteworks describió 9.5.1 a finales de septiembre como la versión actual que tiene en cuenta todas las vulnerabilidades conocidas. Quien actualice totemomail a 9.5.1 tendrá así la versión a la que se refieren todos los avisos de EPG del registro.

## Bibliotecas actualizadas

Se han actualizado las bibliotecas incluidas de Apache Tomcat y Apache XML Graphics Batik. Las notas de la versión mencionan CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956 y CVE-2026-59083, sin asignarlas individualmente a las dos bibliotecas. Además, se actualizó woodstox-core-asl debido a una vulnerabilidad; no se indica ningún número CVE para ella.

## Mejora y correcciones de errores

- **Migración de EPG:** al migrar desde una base de datos Oracle, totemomail ahora admite Oracle Wallet. Si no hay configurada una contraseña de base de datos, la migración se conecta mediante el wallet.
- **RMS:** los mensajes podían quedarse bloqueados durante el procesamiento porque las acciones de RMS esperaban indefinidamente la autenticación. El tiempo de espera está limitado ahora a un minuto. Esto afecta a instalaciones con Microsoft Purview Information Protection (RMS).
- **TRE:** la función «Open Online» para Registered Envelopes vuelve a funcionar. Había dejado de funcionar desde el endurecimiento mencionado de la versión 9.3.1.
- **Informe de licencias:** en las licencias Cloud, el informe vuelve a contar correctamente a los usuarios internos con tipos de función predefinidos. El artículo sobre el [límite de licencia y la limpieza mediante LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup) describe cómo afectan las cuentas huérfanas al recuento de licencias.

## Instalación

El parche puede instalarse en sistemas con totemomail 7 b985 o posterior en Windows, Linux, Solaris y FreeBSD. Inicie sesión en la consola de administración, cargue el archivo TPF incluido en `[Instanzname]` > «Update» e inicie la actualización. Si hay varias instancias, deberá hacerlo individualmente para cada una. Después, totemomail se reinicia automáticamente si es posible; planifique una ventana de mantenimiento y cree antes una copia de seguridad o una instantánea.

Hasta que se instale la actualización, dos medidas reducen el riesgo: hacer que la consola de administración y la API de administración sean accesibles solo desde la red de gestión, y restringir las conexiones salientes del gateway a los destinos realmente necesarios. En instalaciones Cloud, el punto de conexión de metadatos `169.254.169.254` es uno de los destinos a los que el gateway no necesita acceder. Ninguna de las dos medidas sustituye al parche; solo la actualización corrige la vulnerabilidad de restablecimiento de contraseña del portal web.

## Fuentes

1.  Notas de la versión de totemomail 9.5.1 (Build 101), totemo ag: guía de instalación y notas de la versión en PDF, disponibles para clientes a través del soporte del fabricante; base de toda la información de este artículo.

2.  [Kiteworks Security Advisories en GitHub](https://github.com/kiteworks/security-advisories/security/advisories): Registro oficial con valores CVSS y versiones corregidas para Email Protection Gateway, Core, MFT y Advanced Forms.

3.  [GHSA-rwpq-5xfv-54pv: toma de cuentas (CVE-2026-85066)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-rwpq-5xfv-54pv): Aviso sobre el fallo de restablecimiento de contraseña, CVSS 9.8, corregido en EPG 9.5.0.

4.  [GHSA-5xhq-9wq3-rvj6: ejecución arbitraria de código (CVE-2026-54154)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Vulnerabilidad más crítica de EPG con CVSS 10.0, corregida en EPG 9.4.1.

5.  [P.K. Sharma: Kiteworks publicó 125 avisos](https://www.pk-sharma.com/briefing/kiteworks-125-advisories-which-is-the-shutdown-flaw): Análisis de la publicación del 30 de septiembre de 2026, con recuento por producto y nivel de gravedad, y las declaraciones de Kiteworks sobre 9.5.1.

6.  [Entender el enrutamiento de correo entre totemomail y Exchange Online](/blog/totemomail-m365): Cómo procesa totemomail los mensajes entrantes; relevante para determinar qué correos pueden desencadenar la vulnerabilidad SSRF.

7.  [Límite de licencia de Totemomail alcanzado: limpiar usuarios huérfanos mediante LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup): Contexto sobre el recuento de licencias al que afecta la corrección del informe de licencias.
