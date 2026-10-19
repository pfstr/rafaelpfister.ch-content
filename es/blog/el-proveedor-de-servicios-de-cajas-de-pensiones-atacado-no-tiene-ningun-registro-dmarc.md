---
title: "El proveedor de servicios de cajas de pensiones atacado no tiene ningún registro DMARC"
navTitle: "DMARC cajas de pensiones"
description: "El proveedor de servicios de cajas de pensiones PK Softech, atacado a finales de septiembre de 2026, sigue sin tener un registro DMARC. Una revisión pasiva de 66 entidades de previsión y 30 proveedores de servicios muestra que no es un caso aislado: los fabricantes de software de administración, en particular, no están protegidos."
date: "2026-10-18"
kategorie: "SMTP y flujo de correo"
timeToRead: "9 min de lectura"
themen:
  - smtp-mailflow
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "el-proveedor-de-servicios-de-cajas-de-pensiones-atacado-no-tiene-ningun-registro-dmarc"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
translationOf: dmarc-pensionskassen-schweiz
url: https://rafaelpfister.ch/es/blog/el-proveedor-de-servicios-de-cajas-de-pensiones-atacado-no-tiene-ningun-registro-dmarc
translationSourceHash: 64ec035a358c39b09487102da66cf7aca02df2c567e29e343895906a4490d205
translationModel: gpt-5.6-terra
translatedAt: 2026-10-18T14:54:11.733Z
translationReview: automatic
---

A finales de septiembre de 2026 fue atacada PK Softech AG, un fabricante de software cuya solución de previsión utilizan varias cajas de pensiones suizas. Por esta vía se filtraron datos. Según las informaciones publicadas, entre las entidades afectadas figuran la caja de pensiones federal Publica, la caja de pensiones de Migros y la caja de pensiones de la ciudad de Zúrich, así como otras instituciones. Según la caja, se trata de nombre, número AVS, estado civil, dirección e IBAN, además de datos salariales y de previsión.

Esta filtración es grave por sí sola. Afecta datos de identificación y financieros de una gran parte de la población activa y, a través de Publica, de amplios sectores de la Administración federal. Datos de esta cantidad y calidad no son solo un problema para cada persona afectada. Permiten el robo de identidad, el engaño dirigido y la extorsión a gran escala, y constituyen por tanto un peligro para el orden público y la seguridad nacional. Los activos de previsión propiamente dichos no se vieron afectados, pero la policía de ciberdelincuencia ya advirtió de llamadas fraudulentas con los datos sustraídos.

A ello se suma una constatación difícil de explicar para un proveedor de servicios de esta categoría: **PK Softech AG sigue sin tener ningún registro DMARC.** Su dominio cuenta con un registro SPF estricto, pero no con una política DMARC (verificado el 18.10.2026). Por tanto, la dirección del remitente del proveedor atacado puede falsificarse sin que una plataforma de correo receptora la rechace de forma fiable. Tras una filtración de datos, esta es la situación de partida más desfavorable imaginable. Quien conoce el nombre, la dirección y la cuenta bancaria de una persona asegurada y además puede falsificar un correo en nombre del proveedor o de la caja puede redactar un mensaje casi indistinguible de uno auténtico, por ejemplo solicitando actualizar el IBAN para el próximo pago de la pensión. DMARC protege precisamente contra esta falsificación.

PK Softech no es un caso aislado, sino el ejemplo más evidente de un patrón. He comprobado hasta qué punto están bien preparadas las entidades de previsión suizas y sus proveedores de servicios habituales en materia de autenticación de correo electrónico.

## Cómo se realizó la comprobación

El análisis abarca 66 entidades de previsión del segundo pilar, desde las grandes cajas de derecho público y las fundaciones colectivas de las aseguradoras hasta instituciones comunitarias más pequeñas. Se revisaron exclusivamente registros DNS públicos: SPF, DKIM, DMARC, MTA-STS, TLS-RPT y los registros MX. Son los mismos datos que todo sistema de correo receptor consulta para cada mensaje entrante.

La comprobación fue totalmente pasiva. Solo se consultó DNS público; no se estableció conexión con los sistemas de las entidades, no se intentó iniciar sesión y no se recopilaron datos personales. El análisis refleja únicamente lo que ya es accesible para cualquiera a partir de fuentes públicas. Las fundaciones colectivas de grupos se han agrupado bajo el dominio principal de la entidad responsable; algunas fundaciones afiliadas pueden diferir.

## El resultado

| Característica | Proporción de las 66 entidades |
|---|---|
| SPF disponible | 88 % |
| SPF estricto (`-all`) | 59 % |
| DMARC disponible | 59 % |
| DMARC `p=reject` (protección total) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (solo supervisión) | 14 % |
| sin DMARC | 41 % |
| DKIM identificable | 55 % |
| MTA-STS | 2 % |
| TLS-RPT | 5 % |

Destacan tres puntos.

**Solo el 20 por ciento está protegido eficazmente con DMARC `p=reject`.** Solo esta política indica a una plataforma de correo receptora que debe rechazar realmente los mensajes falsificados. En todos los demás casos, una falsificación termina, en caso de duda, en el buzón de la persona asegurada.

**El 41 por ciento no tiene DMARC.** Su dominio de remitente puede utilizarse sin más para correos fraudulentos, y la entidad ni siquiera se enterará, porque sin DMARC también falta la función de informes. Llama la atención que entre ellas haya cajas que operan su correo con Microsoft 365, donde DMARC puede configurarse con poco esfuerzo.

**DMARC `p=none` no es protección.** El 14 por ciento de este grupo ha establecido una política DMARC, pero la ha dejado en el nivel de supervisión. Proporciona informes, pero no rechaza ni un solo mensaje falsificado. En la práctica, es un paso intermedio que a menudo permanece durante meses o años.

Además, existe una constatación generalizada: con un 2 por ciento, MTA-STS es prácticamente inexistente. Sin MTA-STS, no se obliga al cifrado de transporte del correo entrante, lo que favorece a un atacante con acceso a la ruta de transmisión.

## Todos los registros para consultar

La siguiente infografía muestra el estado revisado de las 66 entidades de previsión y los 30 proveedores de servicios. Los filtros permiten acotar por estado de DMARC, categoría y proveedor de correo, y cada entrada puede desplegarse para consultar los resultados DNS completos.

<div data-dmarc-infographic></div>

## Comparación: las mayores empresas suizas

El [Índice de seguridad del correo de Suiza](/open-data/mail-security-index) muestra cómo se sitúan las entidades de previsión frente al resto de la economía. Con el mismo procedimiento pasivo, revisa a diario más de 500 de las mayores empresas suizas, con un mapa interactivo por cantones, comparaciones por sector y proveedor de correo, y una página de detalle para cada empresa. Entre ellas se encuentran también los bancos, aseguradoras y proveedores de servicios de TI con los que colaboran las entidades de previsión.

La diferencia es clara: de las 518 empresas del índice, el 18.10.2026 el 38 por ciento utilizaba DMARC con `p=reject`, frente al 20 por ciento de las entidades de previsión. El 5 por ciento de las empresas no tiene DMARC en absoluto, frente al 41 por ciento de las entidades de previsión.

## Los proveedores son el problema mayor

El incidente de finales de septiembre no afectó a una caja, sino a un proveedor compartido. Por ello, con el mismo procedimiento pasivo revisé 30 proveedores típicos: fabricantes de software de administración, empresas de gestión externalizada, actuariado y consultoría, bancos depositarios y proveedores de servicios de TI.

En promedio, los proveedores de servicios están algo mejor posicionados que las cajas: el 40 por ciento utiliza DMARC con `p=reject`, frente al 20 por ciento de las entidades. Sin embargo, lo decisivo es la distribución entre las categorías.

| Categoría | con DMARC `p=reject` |
|---|---|
| Actuariado y consultoría | bueno |
| Bancos depositarios | bueno |
| Proveedores de servicios de TI | mixto |
| Gestión externalizada | débil |
| Software de administración | el más débil |

Los fabricantes de software de administración son el grupo más débil. Ninguno de los proveedores de software revisados utiliza DMARC con plena eficacia, y el proveedor atacado PK Softech no tiene ningún registro DMARC. Precisamente esta categoría procesa datos de previsión y está estrechamente conectada con las cajas mediante interfaces, y es la categoría del incidente actual.

De ello se desprende el punto esencial en materia de diligencia: una caja con `p=reject` está bien protegida mientras el atacante quiera falsificar su dominio. Pero si en su lugar puede falsificar el dominio de un proveedor mal protegido a través del cual ya se comunica con las personas aseguradas, el correo es igual de creíble. Por ello, proteger el propio dominio es solo el primer paso. El segundo es obligar a los proveedores de servicios más importantes a alcanzar el mismo nivel, idealmente por contrato.

## Qué significa esto para el consejo de fundación

La responsabilidad de la seguridad de la información recae en el órgano supremo de la entidad de previsión. Además, una filtración de datos activa obligaciones concretas en virtud de la Ley de Protección de Datos revisada: una violación de la seguridad de los datos debe notificarse al EDÖB si previsiblemente supone un alto riesgo para las personas afectadas, y estas deben ser informadas en la medida necesaria para su protección.

En esta situación, la autenticación de correo electrónico es la medida individual más eficaz con el menor esfuerzo. SPF, DKIM y DMARC no requieren licencias ni software nuevo. Consisten en registros DNS y una configuración correcta de la plataforma de correo existente. La protección tiene efecto inmediato para cada persona asegurada, sin que esta tenga que hacer nada.

## Qué hacer

Para las entidades sin protección eficaz, existe una ruta clara hacia el objetivo:

1. **Revisar SPF y establecerlo de forma estricta.** El registro SPF enumera los sistemas de envío autorizados y termina con `-all`, de modo que todos los demás remitentes se consideran no autorizados.
2. **Activar DKIM.** Cada mensaje saliente se firma criptográficamente. En Microsoft 365 y en las pasarelas de correo habituales, es una configuración, no un proyecto.
3. **Introducir DMARC y endurecerlo gradualmente.** Empezar con `p=none` y la generación activa de informes para registrar el tráfico de correo legítimo. Después `p=quarantine` y, por último, `p=reject`. Todo el proceso tarda unas semanas según el entorno, no meses.
4. **Evaluar los informes.** Los informes DMARC muestran quién envía correo en nombre del dominio y detectan tanto sistemas legítimos olvidados como usos indebidos.
5. **Añadir MTA-STS y TLS-RPT.** Ambos obligan y supervisan el cifrado de transporte del correo entrante.
6. **Exigir responsabilidades a los proveedores de servicios.** Los proveedores de software, de gestión externalizada y los socios de TI deben demostrar el mismo nivel en SPF, DKIM y DMARC. Esto debe incluirse en los contratos y en la revisión periódica de proveedores.

## Conclusión

Los ataques recientes se dirigieron contra un proveedor de servicios, no contra las cajas mismas. Sin embargo, los daños posteriores se producen allí donde un dominio de remitente no está protegido, tanto en la caja como entre sus proveedores. Cuatro de cada cinco entidades de previsión revisadas no están completamente protegidas contra remitentes falsificados, dos de cada cinco no tienen DMARC en absoluto y, precisamente, los fabricantes de software de administración obtienen los peores resultados. La brecha es conocida, barata de cerrar y tiene efecto inmediato. Tras una filtración de datos que afecta al nombre, la dirección y la cuenta bancaria, no es un detalle técnico secundario, sino parte del deber de diligencia del órgano supremo, a lo largo de toda la cadena de suministro.

## Fuentes

1.  [SRF: Caja de pensiones federal: cómo pudieron acabar datos sensibles en una empresa de software externa](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contexto del incidente, tipos de datos afectados y papel del proveedor de software.

2.  [cash.ch (AWP): Ataque al proveedor de Publica, los datos salariales podrían haberse visto afectados](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica ante AWP a PK Softech AG como la empresa atacada.

3.  [Netzwoche: La caja de pensiones federal Publica comunica una filtración de datos](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Noticia sobre el ataque al proveedor de software y la filtración de datos.

4.  [Mimikama: La policía suiza advierte de posibles llamadas de inversión tras la filtración de datos](https://www.mimikama.org/?p=436628): Contextualización de la advertencia sobre llamadas fraudulentas con los datos filtrados.

5.  [EDÖB: Notificación de violaciones de la seguridad de los datos](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Obligación de notificación conforme a la revDSG y procedimiento ante violaciones de la seguridad de los datos.
