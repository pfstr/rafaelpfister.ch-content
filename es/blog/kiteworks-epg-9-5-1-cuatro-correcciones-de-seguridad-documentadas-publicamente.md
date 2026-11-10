---
title: "Kiteworks EPG 9.5.1: Cuatro correcciones de seguridad documentadas públicamente"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 corrige cuatro vulnerabilidades documentadas públicamente, incluida una toma de control de cuentas con CVSS 9.4. Las instalaciones anteriores a 9.5.1 deberían actualizarse a esta versión o a una posterior."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 min de lectura"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-9-5-1-cuatro-correcciones-de-seguridad-documentadas-publicamente"
translationId: article-d4b9d984c72a54ea
translationOf: kiteworks-epg-releases-9-5-1
url: https://rafaelpfister.ch/es/blog/kiteworks-epg-9-5-1-cuatro-correcciones-de-seguridad-documentadas-publicamente
translationSourceHash: 7d961f479006e9eb0da1a74925d77e25e7205a64a1ed783dccb55b6eed8c2c5c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:19:44.535Z
translationReview: automatic
---

# Kiteworks EPG 9.5.1: Cuatro correcciones de seguridad documentadas públicamente

Kiteworks continúa la antigua tecnología de totemo como Email Protection Gateway (EPG). La versión 9.5.1 era la versión actual de Kiteworks a más tardar el 25 de septiembre de 2026; el fabricante no indica una fecha de publicación pública. Los avisos del fabricante publicados el 30 de septiembre documentan cuatro correcciones para EPG 9.5.1: una toma de control de cuentas, dos fallos no autenticados para escribir archivos o provocar una denegación de servicio, así como un fallo que permitía a los administradores escribir archivos arbitrarios. La clasificación de la versión homónima de totemomail se encuentra en el artículo sobre [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 9.5.1 | confirmada públicamente a más tardar el 25.09.2026 | Actualización de seguridad | Cuatro correcciones de EPG documentadas públicamente, incluida CVE-2026-102149 con CVSS 9.4 |

## Correcciones de seguridad en 9.5.1

Según Kiteworks, los cuatro avisos afectan a todas las versiones de EPG anteriores a 9.5.1 e indican la 9.5.1 como versión corregida. Los avisos se publicaron el 30 de septiembre de 2026. Describen retrospectivamente el estado de seguridad de la versión, no su fecha exacta de publicación.

- **CVE-2026-102149, CVSS 9.4 (crítica):** Una debilidad en el control de acceso podía permitir a un atacante acceder a cuentas de usuario a través de la red. La evaluación CVSS no requiere privilegios ni interacción del usuario. Kiteworks atribuye la causa a CWE-306 (falta de autenticación para funciones críticas) y CWE-639 (omisión de autorización mediante una clave controlada por el usuario).
- **CVE-2026-102143, CVSS 7.5 (alta):** Un atacante no autenticado podía escribir archivos en el dispositivo. Kiteworks menciona un problema de autenticación, así como CWE-306 y CWE-434 (carga sin restricciones de tipos de archivos peligrosos).
- **CVE-2026-102146, CVSS 6.5 (media):** Un administrador podía escribir archivos arbitrarios. Kiteworks describe un error de permisos; la métrica CVSS requiere privilegios elevados. El fabricante menciona como debilidades CWE-73 (control externo del nombre o la ruta de un archivo) y CWE-1336 (neutralización insuficiente en un motor de plantillas).
- **CVE-2026-102144, CVSS 5.3 (media):** Un atacante en la red podía provocar una denegación de servicio. El aviso menciona falta de autenticación para funciones críticas y consumo incontrolado de recursos (CWE-306, CWE-400). Esta evaluación tampoco requiere privilegios ni interacción del usuario.

Para los operadores, la toma de control de cuentas es la corrección prioritaria, ya que, según el aviso, es posible sin una cuenta existente. Los otros dos fallos sin inicio de sesión también afectan a todas las versiones anteriores a 9.5.1. Los avisos del fabricante no publican detalles técnicos sobre puntos finales, configuraciones o soluciones alternativas.

## Procedimiento recomendado

Actualice EPG como mínimo a la versión 9.5.1 si utiliza una versión anterior. Kiteworks indica expresamente para los cuatro avisos la actualización a la versión 9.5.1 o posterior como medida correctiva. Si ya se ha publicado una versión posterior del fabricante, también cumple este requisito mínimo. Kiteworks no ha publicado en las Fuentes consultadas una ruta de actualización pública, problemas conocidos ni plazos de soporte para EPG 9.5.1.

## Fuentes

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Comunicación del fabricante del 25 de septiembre de 2026 que menciona la 9.5.1 como versión actual.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Resumen oficial de los avisos publicados el 30 de septiembre de 2026, incluidas las entradas de EPG para la versión 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Debilidad crítica de control de acceso, CVSS 9.4, corregida en EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Escritura de archivos no autenticada, CVSS 7.5, corregida en EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Escritura de archivos arbitrarios con privilegios de administrador, CVSS 6.5, corregida en EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Denegación de servicio no autenticada, CVSS 5.3, corregida en EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Información del fabricante sobre la continuación y el cambio de nombre de la tecnología totemo como Email Protection Gateway.
