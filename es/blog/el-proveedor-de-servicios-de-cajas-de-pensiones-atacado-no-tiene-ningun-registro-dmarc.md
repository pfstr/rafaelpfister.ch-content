---
title: "El proveedor de servicios de cajas de pensiones atacado ni siquiera tiene un registro DMARC"
navTitle: "DMARC cajas de pensiones"
description: "El proveedor de servicios de cajas de pensiones PK Softech, atacado a finales de septiembre de 2026, sigue sin tener un registro DMARC. Una revisión pasiva de 66 entidades de previsión y 30 proveedores de servicios muestra que no se trata de un caso aislado: los fabricantes de software de administración, en particular, están desprotegidos."
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
translationSourceHash: 357621c8e85052ed11d792c9e6f911df15e6e32a0ad4cf0c5d92ba1fdbd9f906
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:34:24.907Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/el-proveedor-de-servicios-de-cajas-de-pensiones-atacado-no-tiene-ningun-registro-dmarc
---

A finales de septiembre de 2026 fue atacada PK Softech AG, un fabricante de software cuya solución de previsión utilizan varias cajas de pensiones suizas. Por esta vía se filtraron datos. Según las informaciones publicadas, entre las entidades afectadas se encuentran la caja de pensiones de la Confederación Publica, la caja de pensiones de Migros y la caja de pensiones de la ciudad de Zúrich, así como otras instituciones. Dependiendo de la caja, se trata de nombre, número AHV, estado civil, dirección e IBAN, además de datos salariales y de previsión.

Esta filtración ya es grave por sí misma. Afecta a datos de identificación y financieros de una gran parte de la población trabajadora y, a través de Publica, de amplios sectores de la Administración federal. Datos de este volumen y calidad no son solo un problema para cada persona afectada. Permiten el robo de identidad, el engaño dirigido y la extorsión a gran escala, por lo que constituyen un peligro para el orden público y la seguridad nacional. Los activos de previsión permanecieron intactos, pero la plataforma informativa Cybercrimepolice.ch ya advirtió de llamadas fraudulentas con los datos robados.

A ello se añade una constatación difícil de explicar para un proveedor de esta categoría: **PK Softech AG sigue sin tener un registro DMARC.** Su dominio tiene un registro SPF estricto, pero ninguna política DMARC (comprobado el 18.10.2026). Esto permite falsificar la dirección del remitente del proveedor atacado sin que una plataforma de correo receptora lo rechace de forma fiable. Tras una filtración de datos, es la peor situación imaginable. Quien conoce el nombre, la dirección y los datos bancarios de una persona asegurada y además puede falsificar un correo en nombre del proveedor o de la caja puede redactar un mensaje que apenas se diferencia de uno auténtico, por ejemplo una solicitud para actualizar el IBAN para el próximo pago de pensión. DMARC protege precisamente contra esta falsificación.

PK Softech no es un caso aislado, sino el ejemplo más evidente de un patrón. He comprobado hasta qué punto las entidades de previsión suizas y sus proveedores de servicios habituales están preparadas en materia de autenticación de correo electrónico.

## Cómo se realizó la comprobación

El análisis abarca 66 entidades de previsión del segundo pilar, desde las grandes cajas de derecho público y las fundaciones colectivas de las aseguradoras hasta entidades comunitarias más pequeñas. Se comprobaron exclusivamente registros DNS públicos: SPF, DKIM, DMARC, registros MX, así como DNSSEC y DANE (registros TLSA de los servidores de correo, comprobados el 19.10.2026). Son los mismos datos que cualquier sistema de correo receptor consulta para cada mensaje entrante.

La comprobación fue puramente pasiva. Solo se consultó DNS público; no se estableció ninguna conexión con los sistemas de las entidades, no se intentó iniciar sesión ni se recopilaron datos personales. El análisis refleja únicamente lo que, de todos modos, puede consultarse públicamente. Las fundaciones colectivas de grupos empresariales se agrupan bajo el dominio principal de la entidad responsable; las fundaciones afiliadas individuales pueden diferir.

## El resultado

| Característica | Proporción de las 66 entidades |
|---|---|
| SPF disponible | 88 % |
| SPF estricto (`-all`) | 59 % |
| DMARC disponible | 59 % |
| DMARC `p=reject` (protección completa) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (solo observación) | 14 % |
| sin DMARC | 41 % |
| DKIM detectable | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Destacan tres puntos.

**Solo el 20 por ciento está eficazmente protegido con DMARC `p=reject`.** Solo esta política indica a una plataforma de correo receptora que debe rechazar realmente los mensajes falsificados. En todos los demás casos, una falsificación acaba, en caso de duda, en el buzón de la persona asegurada.

**El 41 por ciento no tiene DMARC.** Sus dominios de remitente pueden utilizarse sin más para correos fraudulentos, y la entidad ni siquiera se entera, porque sin DMARC también falta la función de informes. Llama la atención que entre ellas haya cajas que gestionan su correo con Microsoft 365, donde DMARC puede configurarse con poco esfuerzo.

**DMARC `p=none` no es protección.** El 14 por ciento de este grupo ha configurado una política DMARC, pero la ha dejado en el nivel de observación. Esto proporciona informes, pero no rechaza ni un solo mensaje falsificado. En la práctica, es un paso intermedio que a menudo permanece durante meses o años.

En el cifrado de transporte se observa una situación similar: solo 6 de las 66 entidades (9 por ciento) protegen sus servidores de correo con DANE. Así, un servidor de correo remitente comprueba mediante un registro firmado con DNSSEC si está conectado de forma cifrada con el destinatario correcto. El requisito es DNSSEC, que el 38 por ciento de las entidades ya ha activado. Para ellas, DANE es el siguiente paso lógico.

## Todos los registros para explorar

La siguiente infografía muestra el estado comprobado de las 66 entidades de previsión y 30 proveedores de servicios. Los filtros permiten acotar por estado de DMARC, categoría y proveedor de correo, y cada entrada puede desplegarse para ver los resultados DNS completos.

<div data-dmarc-infographic></div>

## Comparación: las mayores empresas suizas

El [Índice de seguridad del correo de Suiza](/open-data/mail-security-index) muestra cómo se sitúan las entidades de previsión frente al resto de la economía. Con el mismo método pasivo, comprueba diariamente más de 1000 de las mayores empresas suizas, así como la Confederación, los cantones y los municipios, con un mapa interactivo por cantones, comparaciones por sector y proveedor de correo, y una página detallada para cada organización. Todas las entidades de previsión y proveedores de servicios analizados aquí figuran allí con un perfil propio, agrupados en el sector de [entidades de previsión](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) y actualizados diariamente. En la infografía anterior, el enlace de la entrada desplegada lleva directamente al perfil correspondiente.

La diferencia es clara: de las 518 empresas del índice, el 18.10.2026 el 38 por ciento utiliza DMARC con `p=reject`, frente al 20 por ciento de las entidades de previsión. El 5 por ciento de las empresas no tiene DMARC en absoluto, frente al 41 por ciento de las entidades de previsión.

## Los proveedores son el problema mayor

El incidente de finales de septiembre no afectó a una caja, sino a un proveedor de servicios común. Por ello, comprobé con el mismo método pasivo a 30 proveedores habituales: fabricantes de software de administración, empresas de gestión externalizada, actuaría y consultoría, bancos depositarios y proveedores de TI.

En promedio, los proveedores de servicios están algo mejor posicionados que las cajas: el 40 por ciento utiliza DMARC con `p=reject`, frente al 20 por ciento de las entidades. Sin embargo, lo decisivo es la distribución entre las categorías.

| Categoría | con DMARC `p=reject` |
|---|---|
| Actuaría y consultoría | bien |
| Bancos depositarios | bien |
| Proveedores de TI | mixto |
| Gestión externalizada | débil |
| Software de administración | el más débil |

Los fabricantes de software de administración son el grupo más débil. Ninguno de los proveedores de software analizados utiliza DMARC con plena efectividad, y el proveedor atacado PK Softech ni siquiera tiene un registro DMARC. Se trata precisamente de la categoría que procesa datos de previsión y está profundamente conectada con las cajas a través de interfaces, y es la categoría del incidente actual.

De ello se desprende el punto esencial en materia de diligencia: una caja con `p=reject` está bien protegida mientras el atacante quiera falsificar su dominio. Pero si puede falsificar en su lugar el dominio de un proveedor de servicios mal protegido a través del cual ya se comunica con las personas aseguradas, el correo resulta igual de creíble. Por tanto, proteger el propio dominio es solo el primer paso. El segundo consiste en obligar a los proveedores de servicios más importantes a alcanzar el mismo nivel, preferiblemente por contrato.

## Qué significa esto para el consejo de fundación

La responsabilidad de la seguridad de la información recae en el órgano supremo de la entidad de previsión. Una filtración de datos también activa obligaciones concretas conforme a la Ley de Protección de Datos revisada: debe notificarse una vulneración de la seguridad de los datos al EDÖB si previsiblemente conlleva un riesgo elevado para las personas afectadas, y estas deben ser informadas cuando sea necesario para su protección.

En esta situación, la autenticación del correo electrónico es la medida individual más eficaz con el menor esfuerzo. SPF, DKIM y DMARC no requieren licencia ni software nuevo. Consisten en registros DNS y una configuración adecuada de la plataforma de correo existente. La protección actúa de inmediato para cada persona asegurada, sin que esta tenga que hacer nada.

## Qué debe hacerse

Para las entidades sin protección eficaz, un camino claro conduce al objetivo:

1. **Comprobar SPF y configurarlo de forma estricta.** El registro SPF enumera los sistemas de envío autorizados y termina con `-all`, de modo que todos los demás remitentes se consideran no autorizados.
2. **Activar DKIM.** Cada mensaje saliente se firma criptográficamente. En Microsoft 365 y en las pasarelas de correo habituales, es una configuración, no un proyecto.
3. **Introducir DMARC y endurecerlo gradualmente.** Empezar con `p=none` y la generación activa de informes para registrar el tráfico de correo legítimo. Después `p=quarantine` y, por último, `p=reject`. Todo el proceso dura unas semanas según el entorno, no meses.
4. **Evaluar los informes.** Los informes DMARC muestran quién envía correo en nombre del dominio y revelan tanto sistemas legítimos olvidados como usos indebidos.
5. **Al menos comprobar DANE y revisar el cifrado del correo.** DANE protege mediante DNSSEC el cifrado de transporte entre servidores de correo. Puede determinarse con poco esfuerzo si la propia plataforma de correo y el proveedor de DNS lo admiten. Además, para los mensajes con datos salariales y de previsión debe definirse cómo se envían cifrados de extremo a extremo, por ejemplo mediante una pasarela de correo con S/MIME o una entrega segura a través de un portal web.
6. **Exigir responsabilidades a los proveedores de servicios.** Los proveedores de software, las empresas de gestión externalizada y los socios de TI deben demostrar el mismo nivel en SPF, DKIM y DMARC. Esto debe incluirse en los contratos y en la evaluación periódica de proveedores.

## Conclusión

Los ataques recientes tuvieron como objetivo a un proveedor de servicios, no a las propias cajas. Sin embargo, los daños posteriores se producen allí donde un dominio de remitente está desprotegido, tanto en la caja como en sus proveedores. Cuatro de cada cinco entidades de previsión analizadas no están completamente protegidas contra remitentes falsificados, dos de cada cinco no tienen DMARC en absoluto y, precisamente, los fabricantes de software de administración obtienen los peores resultados. La brecha es conocida, puede cerrarse a bajo coste y surte efecto de inmediato. Tras una filtración de datos que afecta al nombre, la dirección y los datos bancarios, no es una cuestión técnica secundaria, sino parte del deber de diligencia del órgano supremo, a lo largo de toda la cadena de suministro.

## Fuentes

1.  [SRF: Caja de pensiones de la Confederación, ¿cómo pudieron acabar datos sensibles en una empresa externa de software?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contexto del incidente, tipos de datos afectados y papel del proveedor de software.

2.  [cash.ch (AWP): Ataque al proveedor de Publica, los datos salariales podrían verse afectados](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica ante AWP a PK Softech AG como la empresa atacada.

3.  [Netzwoche: La caja de pensiones de la Confederación Publica informa de una filtración de datos](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Noticia sobre el ataque al proveedor de software y la filtración de datos.

4.  [Mimikama: La policía suiza advierte de posibles llamadas de inversión tras una filtración de datos](https://www.mimikama.org/?p=436628): Contexto de la advertencia sobre llamadas fraudulentas con los datos filtrados.

5.  [EDÖB: DataBreach](https://www.edoeb.admin.ch/de/databreach): Obligación de notificación conforme al art. 24 de la LPD, portal de notificación y guía sobre el procedimiento ante vulneraciones de la seguridad de los datos.
