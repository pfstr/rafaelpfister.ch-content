---
title: "El proveedor de servicios de cajas de pensiones atacado ni siquiera tiene un registro DMARC"
navTitle: "DMARC cajas de pensiones"
description: "PK Softech, el proveedor de servicios para cajas de pensiones atacado a finales de septiembre de 2026, sigue sin tener un registro DMARC. Una revisión pasiva de 66 instituciones de previsión y 30 proveedores de servicios muestra que no es un caso aislado: los fabricantes de software de administración, en particular, están desprotegidos."
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
translationOf: dmarc-pensionskassen-schweiz
translationSourceHash: 57506056bfd8dcc96818df43dc71d22a158978bcd7432c91c56eb92698e52fe1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:42:42.527Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/el-proveedor-de-servicios-de-cajas-de-pensiones-atacado-no-tiene-ningun-registro-dmarc
---

A finales de septiembre de 2026 fue atacada PK Softech AG, un fabricante de software cuya solución de previsión utilizan varias cajas de pensiones suizas. Por esta vía se filtraron datos. Según las informaciones publicadas, entre las entidades afectadas se encuentran la caja de pensiones de la Confederación Publica, la caja de pensiones de Migros y la caja de pensiones de la ciudad de Zúrich, así como otras instituciones. Según la caja, se trata de nombres, números AHV, estado civil, direcciones e IBAN, además de datos salariales y de previsión.

Esta filtración es grave de por sí. Afecta a datos de identificación y financieros de una gran parte de la población trabajadora y, a través de Publica, de grandes sectores de la Administración federal. Datos de esta cantidad y calidad no son solo un problema para cada persona. Permiten el robo de identidad, el engaño selectivo y la extorsión a gran escala, y constituyen por ello un peligro para el orden público y la seguridad nacional. Los fondos de previsión en sí no se vieron afectados, pero la plataforma informativa Cybercrimepolice.ch ya advirtió sobre llamadas fraudulentas con los datos sustraídos.

A ello se suma un hallazgo difícil de explicar en un proveedor de servicios de esta categoría: **PK Softech AG sigue sin tener un registro DMARC.** Su dominio cuenta con una entrada SPF estricta, pero no con una política DMARC (comprobado el 18.10.2026). Esto permite falsificar la dirección del remitente del proveedor de servicios atacado sin que una plataforma de correo receptora lo rechace de forma fiable. Tras una filtración de datos, es la peor situación imaginable. Quien conoce el nombre, la dirección y los datos bancarios de una persona asegurada y además puede falsificar un correo en nombre del proveedor de servicios o de la caja puede redactar un mensaje casi indistinguible de uno auténtico, por ejemplo, solicitando actualizar el IBAN para el próximo pago de pensión. DMARC protege precisamente contra esta falsificación.

PK Softech no es un caso aislado, sino el ejemplo más evidente de un patrón. He comprobado hasta qué punto las instituciones de previsión suizas y sus proveedores de servicios habituales están preparadas en materia de autenticación de correo electrónico.

## Cómo se realizó la comprobación

El análisis abarca 66 instituciones de previsión del segundo pilar, desde las grandes cajas de derecho público y las fundaciones colectivas de las aseguradoras hasta entidades comunitarias más pequeñas. Se comprobaron exclusivamente registros DNS públicos: SPF, DKIM, DMARC, los registros MX, así como DNSSEC y DANE (registros TLSA de los servidores de correo, comprobados el 19.10.2026). Son los mismos datos que todo sistema de correo receptor consulta con cada mensaje entrante.

La comprobación fue puramente pasiva. Solo se consultó DNS público; no se estableció ninguna conexión con los sistemas de las instituciones, no se intentó iniciar sesión ni se recopilaron datos personales. El análisis solo refleja lo que, de todos modos, cualquier persona puede consultar en fuentes públicas. Las fundaciones colectivas de grupos empresariales se agrupan bajo el dominio principal de la entidad responsable; algunas fundaciones afiliadas pueden diferir.

## El resultado

| Característica | Proporción de las 66 instituciones |
|---|---|
| SPF presente | 88 % |
| SPF estricto (`-all`) | 59 % |
| DMARC presente | 59 % |
| DMARC `p=reject` (protección total) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (solo supervisión) | 14 % |
| sin DMARC | 41 % |
| DKIM detectable | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Destacan tres puntos.

**Solo el 20 por ciento está protegido eficazmente con DMARC `p=reject`.** Solo esta política indica a una plataforma de correo receptora que rechace realmente los mensajes falsificados. En todos los demás casos, una falsificación terminará, en caso de duda, en la bandeja de entrada de la persona asegurada.

**El 41 por ciento no tiene DMARC.** Sus dominios de remitente pueden utilizarse sin más para correos fraudulentos, y la institución ni siquiera se enterará porque, sin DMARC, también falta la función de informes. Llama la atención que entre ellas haya cajas que gestionan su correo con Microsoft 365, donde DMARC puede configurarse con poco esfuerzo.

**DMARC `p=none` no es protección.** El 14 por ciento de este grupo ha establecido una política DMARC, pero la ha dejado en el nivel de supervisión. Esto proporciona informes, pero no rechaza ni un solo mensaje falsificado. En la práctica, es un paso intermedio que a menudo permanece durante meses o años.

La situación es similar en el cifrado de transporte: solo 6 de las 66 instituciones (9 por ciento) protegen sus servidores de correo con DANE. Así, un servidor de correo remitente verifica mediante una entrada firmada con DNSSEC que está conectado de forma cifrada al destinatario correcto. El requisito es DNSSEC, que ya ha sido activado por el 38 por ciento de las instituciones. Para ellas, DANE es el siguiente paso lógico.

## Todos los registros para explorar

La siguiente infografía muestra el estado comprobado de las 66 instituciones de previsión y los 30 proveedores de servicios. Los filtros permiten acotar por estado de DMARC, categoría y proveedor de correo, y cada entrada puede desplegarse para ver los hallazgos DNS completos.

<div data-dmarc-infographic></div>

## Comparación: las mayores empresas suizas

El [Índice de seguridad del correo de Suiza](/open-data/mail-security-index) muestra cómo se sitúan las instituciones de previsión frente al resto de la economía. Con el mismo método pasivo, examina diariamente más de 1000 de las mayores empresas suizas, así como la Confederación, los cantones y los municipios, con un mapa interactivo por cantones, comparaciones por sector y proveedor de correo, y una página detallada para cada organización. Todas las instituciones de previsión y proveedores de servicios analizados aquí figuran allí con su propio perfil, agrupados en el sector de [instituciones de previsión](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) y actualizados diariamente. En la infografía superior, el enlace de la entrada desplegada lleva directamente al perfil correspondiente.

La diferencia es clara: de las 518 empresas del índice, el 18.10.2026 el 38 por ciento utilizaba DMARC con `p=reject`, frente al 20 por ciento de las instituciones de previsión. El 5 por ciento de las empresas no tiene DMARC en absoluto, frente al 41 por ciento de las instituciones de previsión.

## Los proveedores son el problema mayor

El incidente de finales de septiembre no afectó a una caja, sino a un proveedor de servicios común. Por ello, con el mismo método pasivo, comprobé 30 proveedores habituales: fabricantes de software de administración, empresas de gestión externalizada, actuariado y consultoría, bancos depositarios y proveedores de servicios informáticos.

En promedio, los proveedores de servicios están algo mejor posicionados que las cajas: el 40 por ciento utiliza DMARC con `p=reject`, frente al 20 por ciento de las instituciones. Sin embargo, lo decisivo es la distribución entre las categorías.

| Categoría | con DMARC `p=reject` |
|---|---|
| Actuariado y consultoría | bien |
| Bancos depositarios | bien |
| Proveedores de servicios informáticos | mixto |
| Gestión externalizada | débil |
| Software de administración | el más débil |

Los fabricantes de software de administración son el grupo más débil. Ninguno de los proveedores de software analizados utiliza DMARC con plena eficacia, y PK Softech, el proveedor atacado, ni siquiera tiene un registro DMARC. Precisamente esta es la categoría que procesa datos de previsión y está profundamente conectada con las cajas mediante interfaces, y es la categoría del incidente actual.

De ello se desprende el punto central en materia de diligencia: una caja con `p=reject` está bien protegida mientras el atacante quiera falsificar su dominio. Pero si puede falsificar el dominio de un proveedor de servicios mal protegido, a través del cual ya se comunica con las personas aseguradas, el correo será igual de creíble. Por eso, proteger el propio dominio es solo el primer paso. El segundo es obligar a los proveedores de servicios más importantes a alcanzar el mismo nivel, preferiblemente por contrato.

## Qué significa esto para el consejo de fundación

La responsabilidad de la seguridad de la información recae en el órgano supremo de la institución de previsión. Además, una filtración de datos activa obligaciones concretas en virtud de la Ley Federal de Protección de Datos revisada: debe notificarse una violación de la seguridad de los datos al EDÖB si previsiblemente supone un alto riesgo para las personas afectadas, y debe informarse a estas en la medida necesaria para su protección.

En esta situación, la autenticación de correo electrónico es la medida individual más eficaz con el menor esfuerzo. SPF, DKIM y DMARC no requieren licencia ni software nuevo. Consisten en registros DNS y una configuración correcta de la plataforma de correo existente. La protección se aplica de inmediato a cada persona asegurada, sin que tenga que hacer nada por sí misma.

## Qué hay que hacer

Para las instituciones sin protección eficaz, hay un camino claro hacia el objetivo:

1. **Comprobar SPF y configurarlo de forma estricta.** El registro SPF enumera los sistemas de envío autorizados y termina con `-all`, de modo que todos los demás remitentes se consideran no autorizados.
2. **Activar DKIM.** Cada mensaje saliente se firma criptográficamente. En Microsoft 365 y en las pasarelas de correo habituales, es una configuración, no un proyecto.
3. **Introducir DMARC y endurecerlo gradualmente.** Empezar con `p=none` y con informes activos para registrar el tráfico de correo legítimo. Después, `p=quarantine` y, por último, `p=reject`. Según el entorno, todo el proceso dura algunas semanas, no meses.
4. **Evaluar los informes.** Los informes DMARC muestran quién envía correo en nombre del dominio y revelan tanto sistemas legítimos olvidados como usos indebidos.
5. **Al menos comprobar DANE y revisar el cifrado del correo.** DANE protege el cifrado de transporte entre servidores de correo mediante DNSSEC. Puede determinarse con poco esfuerzo si la plataforma de correo propia y el proveedor de DNS lo admiten. Para los mensajes con datos salariales y de previsión, también debe regularse cómo se envían cifrados de extremo a extremo, por ejemplo mediante una pasarela de correo con S/MIME o una entrega segura a través de un portal web.
6. **Exigir responsabilidades a los proveedores de servicios.** Los proveedores de software, la gestión externalizada y los socios de TI deben demostrar el mismo nivel en SPF, DKIM y DMARC. Esto debe incluirse en los contratos y en la revisión periódica de proveedores.

## Conclusión

Los ataques más recientes se dirigieron contra un proveedor de servicios, no contra las cajas mismas. Sin embargo, los daños posteriores se producen allí donde un dominio de remitente está desprotegido, tanto en la caja como entre sus proveedores. Cuatro de cada cinco instituciones de previsión examinadas no están completamente protegidas contra remitentes falsificados, dos de cada cinco no tienen DMARC en absoluto, y precisamente los fabricantes de software de administración obtienen los peores resultados. La brecha es conocida, barata de cerrar y su efecto es inmediato. Tras una filtración de datos que afecta a nombres, direcciones y datos bancarios, no es un detalle técnico menor, sino parte del deber de diligencia del órgano supremo, a lo largo de toda la cadena de suministro.

## Fuentes

1.  [SRF: Caja de pensiones de la Confederación, ¿cómo pudieron acabar datos sensibles en una empresa externa de software?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contexto del incidente, tipos de datos afectados y función del proveedor de software.

2.  [cash.ch (AWP): Ataque al proveedor de Publica, podrían haberse visto afectados datos salariales](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica ante AWP a PK Softech AG como la empresa atacada.

3.  [Netzwoche: La caja de pensiones de la Confederación Publica informa de una filtración de datos](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Información sobre el ataque al proveedor de software y la filtración de datos.

4.  [Mimikama: La policía suiza advierte tras la filtración de datos sobre posibles llamadas de inversión](https://www.mimikama.org/?p=436628): Contextualización de la advertencia sobre llamadas fraudulentas con los datos filtrados.

5.  [EDÖB: Notificación de violaciones de la seguridad de los datos](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Obligación de notificar conforme a la revDSG y procedimiento en caso de violaciones de la seguridad de los datos.
