---
title: "El proveedor de servicios de cajas de pensiones atacado ni siquiera tiene un registro DMARC"
navTitle: "DMARC cajas de pensiones"
description: "El proveedor de servicios de cajas de pensiones PK Softech, atacado a finales de septiembre de 2026, sigue sin tener un registro DMARC. Un análisis pasivo de 66 entidades de previsión y 30 proveedores de servicios muestra que no es un caso aislado: los fabricantes de software de administración son especialmente vulnerables."
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
translationSourceHash: a673909df1426a76c58ced5aaa46d839e480ae867b2ffad468c903d65919e0c3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:15:53.091Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/el-proveedor-de-servicios-de-cajas-de-pensiones-atacado-no-tiene-ningun-registro-dmarc
---

A finales de septiembre de 2026 fue atacada PK Softech AG, un fabricante de software cuya solución de previsión utilizan varias cajas de pensiones suizas. Por esta vía se filtraron datos. Según las informaciones publicadas, entre los afectados se encuentran la caja de pensiones federal Publica, la caja de pensiones de Migros y la caja de pensiones de la ciudad de Zúrich, así como otras entidades. Dependiendo de la caja, se trata de nombre, número AHV, estado civil, dirección e IBAN, además de datos salariales y de previsión.

Esta filtración es grave por sí misma. Afecta a datos de identificación y financieros de gran parte de la población trabajadora y, a través de Publica, de amplios sectores de la Administración federal. Datos de esta cantidad y calidad no son solo un problema para cada persona afectada. Permiten el robo de identidad, el engaño selectivo y la extorsión a gran escala, por lo que representan un peligro para el orden público y la seguridad nacional. Los activos de previsión en sí no se vieron afectados, pero la plataforma informativa Cybercrimepolice.ch ya advirtió sobre llamadas fraudulentas utilizando los datos robados.

A ello se suma un hallazgo difícil de explicar para un proveedor de esta categoría: **PK Softech AG sigue sin tener un registro DMARC.** Su dominio tiene una entrada SPF estricta, pero ninguna política DMARC (verificado el 18.10.2026). Esto permite falsificar la dirección del remitente del proveedor atacado sin que una plataforma de correo receptora lo rechace de forma fiable. Tras una filtración de datos, esta es la peor situación imaginable. Quien conoce el nombre, la dirección y los datos bancarios de una persona asegurada y además puede falsificar un correo en nombre del proveedor o de la caja puede redactar un mensaje apenas distinguible de uno auténtico, por ejemplo solicitando actualizar el IBAN para el próximo pago de pensión. DMARC protege precisamente contra esta falsificación.

PK Softech no es un caso aislado, sino el ejemplo más claro de un patrón. He comprobado hasta qué punto las entidades de previsión suizas y sus proveedores de servicios habituales están preparadas en materia de autenticación de correo electrónico.

## Cómo se realizó la comprobación

El análisis abarca 66 entidades de previsión del segundo pilar, desde las grandes cajas de derecho público y las fundaciones colectivas de las aseguradoras hasta entidades comunitarias más pequeñas. Solo se comprobaron entradas DNS públicas: SPF, DKIM, DMARC, los registros MX, así como DNSSEC y DANE (registros TLSA de los servidores de correo, comprobados el 19.10.2026). Son los mismos datos que todo sistema de correo receptor consulta para cada mensaje entrante.

La comprobación fue exclusivamente pasiva. Solo se consultó DNS público; no se estableció ninguna conexión con los sistemas de las entidades, no se intentó iniciar sesión ni se recopilaron datos personales. El análisis solo refleja lo que, de todos modos, cualquiera puede consultar en fuentes públicas. Las fundaciones colectivas de grupos empresariales se agrupan bajo el dominio principal de la entidad responsable; las fundaciones afiliadas individuales pueden diferir.

## El resultado

| Característica | Proporción de las 66 entidades |
|---|---|
| SPF disponible | 88 % |
| SPF estricto (`-all`) | 59 % |
| DMARC disponible | 59 % |
| DMARC `p=reject` (protección completa) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (solo supervisión) | 14 % |
| sin DMARC | 41 % |
| DKIM identificable | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tres puntos destacan.

**Solo el 20 por ciento está protegido de forma efectiva con DMARC `p=reject`.** Solo esta política indica a una plataforma de correo receptora que debe rechazar realmente los mensajes falsificados. En todos los demás casos, una falsificación acabará, en caso de duda, en la bandeja de entrada de la persona asegurada.

**El 41 por ciento no tiene DMARC.** Su dominio remitente puede utilizarse sin más para correos fraudulentos, y la entidad ni siquiera se enterará, porque sin DMARC también falta la función de informes. Llama la atención que entre ellas haya cajas que operan su correo con Microsoft 365, donde DMARC puede configurarse con poco esfuerzo.

**DMARC `p=none` no es protección.** El 14 por ciento de este grupo ha establecido una política DMARC, pero la ha dejado en el nivel de supervisión. Esto proporciona informes, pero no rechaza ni un solo mensaje falsificado. En la práctica, es un paso intermedio que a menudo permanece durante meses o años.

La situación es similar en el cifrado de transporte: solo 6 de las 66 entidades (9 por ciento) protegen sus servidores de correo con DANE. Así, un servidor de correo remitente verifica mediante una entrada firmada con DNSSEC que está conectado de forma cifrada con el destinatario correcto. El requisito es DNSSEC, que el 38 por ciento de las entidades ya ha activado. Para ellas, DANE es el siguiente paso lógico.

## Todos los conjuntos de datos para explorar

La siguiente infografía muestra el estado comprobado de las 66 entidades de previsión y los 30 proveedores de servicios. Los filtros permiten delimitar por estado DMARC, categoría y proveedor de correo, y cada entrada puede desplegarse para consultar los resultados DNS completos.

<div data-dmarc-infographic></div>

## En comparación: las mayores empresas suizas

El [Índice de seguridad del correo de Suiza](/open-data/mail-security-index) muestra cómo se sitúan las entidades de previsión frente al resto de la economía. Con el mismo procedimiento pasivo, comprueba diariamente más de 1000 de las mayores empresas suizas, así como la Confederación, los cantones y los municipios, con un mapa interactivo por cantones, comparaciones por sector y proveedor de correo, y una página detallada para cada organización. Todas las entidades de previsión y proveedores de servicios comprobados aquí figuran allí con su propio perfil, agrupados en el sector [Entidades de previsión](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) y actualizados diariamente. En la infografía anterior, el enlace de la entrada desplegada lleva directamente al perfil correspondiente.

La diferencia es clara: de las 518 empresas del índice, el 18.10.2026 el 38 por ciento utilizaba DMARC con `p=reject`, frente al 20 por ciento de las entidades de previsión. El 5 por ciento de las empresas no tiene DMARC en absoluto, frente al 41 por ciento de las entidades de previsión.

## Los proveedores son el problema mayor

El incidente de finales de septiembre no afectó a una caja, sino a un proveedor de servicios común. Por ello, con el mismo procedimiento pasivo, comprobé 30 proveedores típicos: fabricantes de software de administración, empresas de gestión externalizada, actuaría y consultoría, bancos depositarios y proveedores de servicios TI.

En promedio, los proveedores de servicios están algo mejor posicionados que las cajas: el 40 por ciento utiliza DMARC con `p=reject`, frente al 20 por ciento de las entidades. Sin embargo, lo decisivo es la distribución entre las categorías.

| Categoría | con DMARC `p=reject` |
|---|---|
| Actuaría y consultoría | bien |
| Bancos depositarios | bien |
| Proveedores de servicios TI | mixto |
| Gestión externalizada | débil |
| Software de administración | el más débil |

Los fabricantes de software de administración son el grupo más débil. Ninguno de los proveedores de software comprobados utiliza DMARC con plena eficacia, y el proveedor atacado PK Softech ni siquiera tiene un registro DMARC. Es precisamente la categoría que procesa datos de previsión y está estrechamente conectada con las cajas mediante interfaces, y es la categoría del incidente actual.

De ello se desprende el punto fundamental para la diligencia debida: una caja con `p=reject` está bien protegida mientras el atacante quiera falsificar su dominio. Pero si puede falsificar en su lugar el dominio de un proveedor de servicios mal protegido, a través del cual ya se comunica con las personas aseguradas, el correo será igual de creíble. Por ello, proteger el propio dominio es solo el primer paso. El segundo es obligar a los proveedores de servicios más importantes a alcanzar el mismo nivel, preferiblemente por contrato.

## Qué significa esto para el consejo de fundación

La responsabilidad de la seguridad de la información recae en el órgano supremo de la entidad de previsión. Además, una filtración de datos desencadena obligaciones concretas conforme a la Ley de Protección de Datos revisada: una violación de la seguridad de los datos debe notificarse al EDÖB si previsiblemente conlleva un riesgo elevado para las personas afectadas, y estas deben ser informadas cuando sea necesario para su protección.

En esta situación, la autenticación del correo electrónico es la medida individual más eficaz con el menor esfuerzo. SPF, DKIM y DMARC no requieren licencia ni software nuevo. Consisten en entradas DNS y una configuración correcta de la plataforma de correo existente. La protección actúa de inmediato para cada persona asegurada, sin que esta tenga que hacer nada.

## Qué hay que hacer

Para las entidades sin protección efectiva, existe un camino claro hacia el objetivo:

1. **Comprobar SPF y configurarlo de forma estricta.** La entrada SPF enumera los sistemas de envío autorizados y termina con `-all`, para que todos los demás remitentes se consideren no autorizados.
2. **Activar DKIM.** Cada mensaje saliente se firma criptográficamente. En Microsoft 365 y en las pasarelas de correo habituales, es una configuración, no un proyecto.
3. **Introducir DMARC y reforzarlo gradualmente.** Empezar con `p=none` y la generación activa de informes para registrar el tráfico de correo legítimo. Después `p=quarantine` y, por último, `p=reject`. Todo el proceso dura unas semanas según el entorno, no meses.
4. **Evaluar los informes.** Los informes DMARC muestran quién envía correo en nombre del dominio y revelan tanto sistemas legítimos olvidados como abusos.
5. **Al menos comprobar DANE y revisar el cifrado del correo.** DANE protege el cifrado de transporte entre servidores de correo mediante DNSSEC. Puede aclararse con poco esfuerzo si la propia plataforma de correo y el proveedor DNS lo admiten. Para mensajes con datos salariales y de previsión también debe regularse cómo se envían cifrados de extremo a extremo, por ejemplo mediante una pasarela de correo con S/MIME o una entrega segura a través de un portal web.
6. **Exigir responsabilidades a los proveedores de servicios.** Los proveedores de software, la gestión externalizada y los socios TI deben demostrar el mismo nivel en SPF, DKIM y DMARC. Esto debe incluirse en los contratos y en la revisión periódica de proveedores.

## Conclusión

Los ataques recientes se dirigieron contra un proveedor de servicios, no contra las cajas mismas. Sin embargo, los daños posteriores surgen allí donde un dominio remitente está desprotegido, tanto en la caja como en sus proveedores. Cuatro de cada cinco entidades de previsión comprobadas no están completamente protegidas contra remitentes falsificados, dos de cada cinco no tienen DMARC en absoluto, y precisamente los fabricantes de software de administración obtienen los peores resultados. La brecha es conocida, económica de cerrar y surte efecto de inmediato. Tras una filtración de datos que afecta al nombre, la dirección y los datos bancarios, no es una cuestión técnica secundaria, sino parte del deber de diligencia del órgano supremo, a lo largo de toda la cadena de suministro.

## Historial de cambios

- **25 de octubre de 2026:** Actualizado el enlace de fuentes a la nueva dirección del EDÖB.

## Fuentes

1.  [SRF: Caja de pensiones federal: ¿cómo pudieron acabar datos sensibles en una empresa externa de software?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contexto del incidente, tipos de datos afectados y papel del proveedor de software.

2.  [cash.ch (AWP): Ataque al proveedor de Publica; podrían haberse visto afectados datos salariales](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica ante AWP a PK Softech AG como la empresa atacada.

3.  [Netzwoche: La caja de pensiones federal Publica informa de una filtración de datos](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Información sobre el ataque al proveedor de software y la filtración de datos.

4.  [Mimikama: La policía suiza advierte tras una filtración de datos sobre posibles llamadas de inversión](https://www.mimikama.org/?p=436628): Contextualización de la advertencia sobre llamadas fraudulentas utilizando los datos filtrados.

5.  [EDÖB: DataBreach](https://www.edoeb.admin.ch/de/databreach): Obligación de notificación conforme al art. 24 DSG, portal de notificación y guía sobre el procedimiento ante violaciones de la seguridad de los datos.
