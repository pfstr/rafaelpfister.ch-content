---
title: "El proveedor de servicios de cajas de pensiones atacado ni siquiera tiene un registro DMARC"
navTitle: "DMARC cajas de pensiones"
description: "El proveedor de servicios de cajas de pensiones PK Softech, atacado a finales de septiembre de 2026, sigue sin tener un registro DMARC. Una revisión pasiva de 66 instituciones de previsión y 30 proveedores de servicios muestra que no es un caso aislado: los fabricantes de software de administración, en particular, están desprotegidos."
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
translationSourceHash: 56251be16ac00d877c2536fc6384a564114736e38301637d9006f8e21da77d9a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T08:33:36.071Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/el-proveedor-de-servicios-de-cajas-de-pensiones-atacado-no-tiene-ningun-registro-dmarc
---

A finales de septiembre de 2026 fue atacada PK Softech AG, un fabricante de software cuya solución de previsión utilizan varias cajas de pensiones suizas. A través de esta vía se filtraron datos. Según la cobertura informativa, entre las entidades afectadas figuran la caja de pensiones federal Publica, la caja de pensiones de Migros y la caja de pensiones de la ciudad de Zúrich, así como otras instituciones. Según la caja, se trata de nombre, número AHV, estado civil, dirección e IBAN, además de datos salariales y de previsión.

Esta filtración es grave por sí misma. Afecta a datos de identificación y financieros de una gran parte de la población trabajadora y, a través de Publica, de amplios sectores de la Administración federal. Datos de esta cantidad y calidad no son solo un problema para cada persona afectada. Permiten el robo de identidad, el engaño dirigido y la extorsión a gran escala, por lo que representan un peligro para el orden público y la seguridad nacional. Los activos de previsión propiamente dichos no se vieron afectados, pero la plataforma informativa Cybercrimepolice.ch ya advirtió de llamadas fraudulentas utilizando los datos robados.

A ello se suma un hallazgo difícil de explicar para un proveedor de servicios de esta categoría: **PK Softech AG sigue sin tener un registro DMARC.** Su dominio cuenta con un registro SPF estricto, pero no con una política DMARC (comprobado el 18.10.2026). Esto permite falsificar la dirección del remitente del proveedor atacado sin que una plataforma de correo receptora lo rechace de forma fiable. Tras una filtración de datos, esta es la peor situación imaginable. Quien conoce el nombre, la dirección y los datos bancarios de una persona asegurada y además puede falsificar un correo en nombre del proveedor de servicios o de la caja puede redactar un mensaje casi indistinguible de uno auténtico, por ejemplo solicitando actualizar el IBAN para el próximo pago de pensión. Precisamente contra esta falsificación protege DMARC.

PK Softech no es un caso aislado, sino el ejemplo más evidente de un patrón. He comprobado hasta qué punto las instituciones de previsión suizas y sus proveedores de servicios habituales están preparadas en materia de autenticación de correo electrónico.

## Cómo se realizó la comprobación

El análisis abarca 66 instituciones de previsión del segundo pilar, desde las grandes cajas de derecho público y las fundaciones colectivas de las aseguradoras hasta instituciones colectivas más pequeñas. Se examinaron exclusivamente registros DNS públicos: SPF, DKIM, DMARC, los registros MX, así como DNSSEC y DANE (registros TLSA de los servidores de correo, comprobados el 19.10.2026). Son los mismos datos que consulta cualquier sistema de correo receptor para cada mensaje entrante.

La comprobación fue puramente pasiva. Solo se consultó DNS público; no se estableció ninguna conexión con los sistemas de las instituciones, no se intentó iniciar sesión ni se recopilaron datos personales. El análisis únicamente refleja lo que cualquier persona puede consultar de todos modos en fuentes públicas. Las fundaciones colectivas de grupos empresariales se han agrupado bajo el dominio principal de la entidad responsable; las fundaciones afiliadas individuales pueden diferir.

## El resultado

| Característica | Proporción de las 66 instituciones |
|---|---|
| SPF disponible | 88 % |
| SPF estricto (`-all`) | 59 % |
| DMARC disponible | 59 % |
| DMARC `p=reject` (protección total) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (solo supervisión) | 14 % |
| sin DMARC | 41 % |
| DKIM detectable | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Destacan tres puntos.

**Solo el 20 por ciento está protegido eficazmente con DMARC `p=reject`.** Solo esta política indica a una plataforma de correo receptora que debe rechazar realmente los mensajes falsificados. En todos los demás casos, una falsificación acaba, en caso de duda, en la bandeja de entrada de la persona asegurada.

**El 41 por ciento no tiene DMARC.** Su dominio de remitente puede utilizarse fácilmente para correos fraudulentos, y la institución ni siquiera se enterará, porque sin DMARC también falta la función de informes. Llama la atención que entre ellas haya cajas que operan su correo con Microsoft 365, donde DMARC puede configurarse con poco esfuerzo.

**DMARC `p=none` no es protección.** El 14 por ciento de este grupo ha configurado una política DMARC, pero la ha dejado en el nivel de supervisión. Esto proporciona informes, pero no rechaza ni un solo mensaje falsificado. En la práctica, es un paso intermedio que a menudo permanece así durante meses o años.

En el cifrado de transporte se observa una imagen similar: solo 6 de las 66 instituciones (9 por ciento) protegen sus servidores de correo con DANE. Así, un servidor de correo emisor verifica mediante un registro firmado con DNSSEC si está conectado de forma cifrada al destinatario correcto. El requisito previo es DNSSEC, que el 38 por ciento de las instituciones ya ha activado. Para ellas, DANE es el siguiente paso lógico.

## Todos los registros para explorar

La siguiente infografía muestra el estado comprobado de las 66 instituciones de previsión y los 30 proveedores de servicios. Los filtros permiten delimitar por estado de DMARC, categoría y proveedor de correo, y cada entrada se puede desplegar para consultar todos los hallazgos DNS.

<div data-dmarc-infographic></div>

## Comparación: las mayores empresas suizas

El [Índice de seguridad del correo de Suiza](/open-data/mail-security-index) muestra cómo se sitúan las instituciones de previsión frente al resto de la economía. Mediante el mismo procedimiento pasivo, examina diariamente más de 500 de las mayores empresas suizas, con un mapa interactivo por cantones, comparaciones por sector y proveedor de correo, y una página detallada para cada empresa. Entre ellas figuran también los bancos, aseguradoras y proveedores de servicios de TI con los que colaboran las instituciones de previsión.

La diferencia es clara: de las 518 empresas del índice, el 18.10.2026 el 38 por ciento utilizaba DMARC con `p=reject`, frente al 20 por ciento de las instituciones de previsión. El 5 por ciento de las empresas carece por completo de DMARC, frente al 41 por ciento de las instituciones de previsión.

## Los proveedores son el problema mayor

El incidente de finales de septiembre no afectó a una caja, sino a un proveedor de servicios común. Por ello, mediante el mismo procedimiento pasivo, examiné 30 proveedores típicos: fabricantes de software de administración, empresas de gestión externalizada, servicios actuariales y de consultoría, bancos depositarios y proveedores de servicios de TI.

Por término medio, los proveedores de servicios están algo mejor preparados que las cajas: el 40 por ciento utiliza DMARC con `p=reject`, frente al 20 por ciento de las instituciones. Sin embargo, lo decisivo es la distribución entre las categorías.

| Categoría | con DMARC `p=reject` |
|---|---|
| Actuaría y consultoría | bien |
| Bancos depositarios | bien |
| Proveedores de servicios de TI | mixto |
| Gestión externalizada | débil |
| Software de administración | más débil |

Los fabricantes de software de administración son el grupo más débil. Ninguno de los proveedores de software examinados utiliza DMARC con plena eficacia, y el proveedor atacado PK Softech ni siquiera tiene un registro DMARC. Precisamente se trata de la categoría que procesa datos de previsión y está profundamente conectada con las cajas mediante interfaces, y es la categoría del incidente actual.

De ello se deriva el punto central de la diligencia debida: una caja con `p=reject` está bien protegida mientras el atacante quiera falsificar su dominio. Sin embargo, si puede falsificar el dominio de un proveedor de servicios mal protegido a través del cual ya se comunica habitualmente con las personas aseguradas, el correo será igual de creíble. Por tanto, proteger el propio dominio es solo el primer paso. El segundo es obligar a los proveedores de servicios más importantes a alcanzar el mismo nivel, preferiblemente mediante contrato.

## Lo que esto significa para el consejo de fundación

La responsabilidad de la seguridad de la información recae en el órgano supremo de la institución de previsión. Además, una filtración de datos activa obligaciones concretas en virtud de la Ley de Protección de Datos revisada: una violación de la seguridad de los datos debe notificarse al EDÖB si previsiblemente conlleva un alto riesgo para las personas afectadas, y estas deben ser informadas cuando sea necesario para su protección.

En esta situación, la autenticación de correo electrónico es la medida individual más eficaz con el menor esfuerzo. SPF, DKIM y DMARC no requieren licencia ni software nuevo. Consisten en registros DNS y una configuración correcta de la plataforma de correo existente. La protección actúa inmediatamente para cada persona asegurada, sin que tenga que hacer nada por sí misma.

## Qué hay que hacer

Para las instituciones sin protección eficaz, hay un camino claro hacia el objetivo:

1. **Comprobar SPF y configurarlo de forma estricta.** El registro SPF enumera los sistemas de envío autorizados y termina con `-all`, de modo que todos los demás remitentes se consideren no autorizados.
2. **Activar DKIM.** Cada mensaje saliente se firma criptográficamente. En Microsoft 365 y en las pasarelas de correo habituales, es una configuración, no un proyecto.
3. **Introducir DMARC y endurecerlo gradualmente.** Comenzar con `p=none` e informes activos para identificar el tráfico de correo legítimo. Después `p=quarantine` y, por último, `p=reject`. Todo el proceso tarda unas semanas según el entorno, no meses.
4. **Evaluar los informes.** Los informes DMARC muestran quién envía correo en nombre del dominio y revelan tanto sistemas legítimos olvidados como abusos.
5. **Al menos comprobar DANE y revisar el cifrado del correo.** DANE protege el cifrado de transporte entre servidores de correo mediante DNSSEC. Puede determinarse con poco esfuerzo si la propia plataforma de correo y el proveedor de DNS lo admiten. Además, para los mensajes con datos salariales y de previsión debe regularse cómo se envían cifrados de extremo a extremo, por ejemplo mediante una pasarela de correo con S/MIME o entrega segura a través de un portal web.
6. **Exigir responsabilidades a los proveedores de servicios.** Los proveedores de software, la gestión externalizada y los socios de TI deben demostrar el mismo nivel en SPF, DKIM y DMARC. Esto debe incluirse en los contratos y en la evaluación periódica de proveedores.

## Conclusión

Los ataques recientes se dirigieron contra un proveedor de servicios, no contra las cajas en sí mismas. Sin embargo, los daños derivados se producen allí donde un dominio de remitente está desprotegido, tanto en la caja como en sus proveedores. Cuatro de cada cinco instituciones de previsión examinadas no están completamente protegidas contra remitentes falsificados, dos de cada cinco no tienen DMARC en absoluto y precisamente los fabricantes de software de administración obtienen los peores resultados. La brecha es conocida, económica de cerrar y surte efecto de inmediato. Tras una filtración de datos que afecta al nombre, la dirección y los datos bancarios, no es un detalle técnico secundario, sino parte de la obligación de diligencia del órgano supremo, a lo largo de toda la cadena de suministro.

## Fuentes

1.  [SRF: Caja de pensiones federal: ¿cómo pudieron acabar datos sensibles en una empresa de software externa?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contexto del incidente, tipos de datos afectados y papel del proveedor de software.

2.  [cash.ch (AWP): Ataque al proveedor de Publica; podrían haberse visto afectados datos salariales](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica ante AWP a PK Softech AG como la empresa atacada.

3.  [Netzwoche: La caja de pensiones federal Publica notifica una filtración de datos](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Información sobre el ataque al proveedor de software y la filtración de datos.

4.  [Mimikama: La policía suiza advierte de posibles llamadas de inversión tras una filtración de datos](https://www.mimikama.org/?p=436628): Contextualización de la advertencia sobre llamadas fraudulentas con los datos filtrados.

5.  [EDÖB: Notificación de violaciones de la seguridad de los datos](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Obligación de notificación conforme a la revDSG y procedimiento ante violaciones de la seguridad de los datos.
