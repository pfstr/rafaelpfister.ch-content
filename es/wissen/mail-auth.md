---
title: "Autenticación de correo: SPF, DKIM y DMARC"
blatt: "mail-auth"
description: "SPF, DKIM y DMARC en su contexto técnico: identidades, evaluación de DNS, firmas, alineación, políticas, informes, reenvíos, límites de confianza y operaciones para administradores de mensajería."
fakten:
  - label: Propósito de SPF
    wert: La IP autoriza RFC5321.MailFrom o HELO
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-2
  - label: Publicación de SPF
    wert: un registro TXT con v=spf1
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-3
  - label: Presupuesto DNS de SPF
    wert: como máximo 10 términos que desencadenan búsquedas
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4
  - label: Propósito de DKIM
    wert: Firma de dominio sobre encabezados y cuerpo seleccionados
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3
  - label: Clave DKIM
    wert: selector._domainkey.signing-domain
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3.6.2.1
  - label: Criptografía DKIM
    wert: RSA-SHA256 o Ed25519-SHA256
    href: https://datatracker.ietf.org/doc/html/rfc8463#section-3
  - label: Norma DMARC
    wert: RFC 9989; informes en RFC 9990 y 9991
    href: https://datatracker.ietf.org/doc/html/rfc9989
  - label: Identidad DMARC
    wert: una Author Domain de exactamente un campo From
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.2
  - label: Aprobación DMARC
    wert: SPF o DKIM aprueba y está alineado
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Alineación
    wert: relajada de forma predeterminada; estricta opcional
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Políticas
    wert: none · quarantine · reject
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.7
  - label: Prueba operativa
    wert: Authentication-Results más Aggregate Reports
    href: https://datatracker.ietf.org/doc/html/rfc8601
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 1247c1771c81b476bf23da2eeee6feb35a3d16c7146e2c61d6c0fa5625c55bbd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:30:54.139Z
translationReview: automatic
---

# Autenticación de correo: SPF, DKIM y DMARC

SPF, DKIM y DMARC responden a tres preguntas distintas sobre el dominio de remitente utilizado. SPF verifica la IP emisora, DKIM una firma criptográfica y DMARC la relación de ambos resultados con el dominio From visible. Ninguno de estos mecanismos autentica a una persona ni demuestra que un mensaje sea inofensivo. Un aprobado de DMARC solo indica que el uso del Author Domain fue autorizado según las reglas publicadas ([RFC 7208, sección 1](https://datatracker.ietf.org/doc/html/rfc7208#section-1), [RFC 6376, sección 1](https://datatracker.ietf.org/doc/html/rfc6376#section-1), [RFC 9989, sección 1](https://datatracker.ietf.org/doc/html/rfc9989#section-1)).

Para los administradores de mensajería, el sistema es ante todo una cadena de responsabilidades. El servicio saliente debe generar dominios de envelope y firmas DKIM adecuados. El [DNS](/kb/dns) autoritativo debe entregar correctamente y a tiempo la política SPF, las claves DKIM y la política DMARC. El servidor perimetral [SMTP](/kb/smtp) receptor necesita la IP original del cliente, un resolvedor, verificación criptográfica y un límite de confianza definido para los resultados. Un colector de informes debe procesar XML no fiable de forma segura, detectar duplicados y permitir evaluar los datos a lo largo del tiempo. Un `dmarc=pass` es solo el resultado final de esta canalización distribuida.

La explicación sigue las identidades de un correo electrónico: remitente del envelope, dominio From visible y firma DKIM. Primero se explican SPF y DKIM por separado; después, DMARC conecta sus resultados mediante alineación, política e informes.

## Identidades y límites de confianza

Un mensaje contiene varios conceptos de remitente que no son intercambiables. El **RFC5321.MailFrom** es la ruta inversa del envelope SMTP y el objeto SPF principal. Con una ruta inversa vacía, como la prevista para los informes de entrega, SPF deriva la identidad de HELO/EHLO. El **RFC5322.From** está en la cabecera del mensaje, lo muestra el cliente de correo y proporciona a DMARC el Author Domain. Una firma DKIM denomina con `d=` su Signing Domain y con `s=` el selector de la clave pública. SMTP AUTH, por su parte, autentica un cliente frente a un servicio de submission, pero no es SPF, DKIM ni DMARC ([RFC 5321, secciones 3.3 y 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321#section-3.3), [RFC 5322, sección 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2), [RFC 7208, secciones 2.3 y 2.4](https://datatracker.ietf.org/doc/html/rfc7208#section-2.3), [RFC 6376, sección 3.5](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

| Identidad | Fuente | Verificador | Afirmación principal |
|---|---|---|---|
| dirección IP de conexión | conexión TCP en el MTA receptor | SPF | este host está o no está autorizado para el dominio SMTP evaluado |
| dominio HELO/EHLO | diálogo SMTP | SPF | identidad de dominio del cliente SMTP |
| dominio RFC5321.MailFrom | envelope SMTP | SPF y DMARC | dominio de rebote o Return-Path |
| DKIM `d=` y `s=` | `DKIM-Signature` | DKIM y DMARC | Signing Domain y selector de clave |
| dominio RFC5322.From | cabecera visible del mensaje | DMARC | Author Domain con la que se debe establecer la alineación |
| `authserv-id` | `Authentication-Results` | consumidores internos | qué servicio de verificación de confianza generó el resultado |

Esta separación constituye un límite de seguridad. Un atacante puede aprobar por completo SPF y DKIM con un dominio propio y, aun así, mencionar una marca ajena en el nombre visible. DMARC limita el uso no autorizado del dominio en el `From:` visible, no los dominios similares, el fraude mediante nombre visible, los remitentes legítimos comprometidos ni el contenido malicioso ([RFC 7208, sección 11.2](https://datatracker.ietf.org/doc/html/rfc7208#section-11.2), [RFC 9989, secciones 2.2 y 11.4](https://datatracker.ietf.org/doc/html/rfc9989#section-11.4)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-mail-auth.svg?v=20260813" title="Interaktive Infografik: Identitäten, Alignment, DMARC-Entscheidung, Reporting und indirekte Nachrichtenflüsse" loading="lazy">
  <a href="/images/kb-interaktiv-mail-auth.svg?v=20260813">Abrir la infografía sobre SPF, DKIM y DMARC</a>
</iframe>

## SPF: autorización de la IP de conexión

El Sender Policy Framework es una autorización basada en DNS para el dominio en `MAIL FROM` o HELO. El receptor evalúa la IP del cliente, el dominio evaluado, la identidad del remitente y el nombre de host local con la función `check_host()` definida en RFC 7208. El registro se encuentra como recurso TXT directamente en el dominio correspondiente y comienza con `v=spf1`; no se utiliza el tipo de RR DNS SPF obsoleto ([RFC 7208, secciones 3.1 y 4.1](https://datatracker.ietf.org/doc/html/rfc7208#section-3.1)).

Un registro como `v=spf1 ip4:192.0.2.0/24 include:_spf.sender.example -all` se evalúa de izquierda a derecha. Un mecanismo que coincide termina el procesamiento con su calificador. `+` significa `pass` y es el valor predeterminado, `-` significa `fail`, `~` significa `softfail`, `?` significa `neutral`. `all`, `ip4` y `ip6` no requieren búsquedas DNS adicionales durante la evaluación normal; `include`, `a`, `mx`, `ptr`, `exists` y `redirect` sí las requieren ([RFC 7208, secciones 4.6.1 a 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6)).

| Resultado | Significado en el protocolo | Pregunta administrativa |
|---|---|---|
| `pass` | la IP está autorizada para esta identidad SPF | ¿Está exactamente este dominio también alineado con RFC5322.From? |
| `fail` | la política publicada no autoriza la IP | ¿Fuente incorrecta, uso robado del dominio o política obsoleta? |
| `softfail` | afirmación negativa débil del dominio | ¿Sigue sirviendo a una fase de transición controlada o oculta deriva? |
| `neutral` | no hay afirmación de autorización | ¿Falta un mecanismo final o se pretende la neutralidad? |
| `none` | no hay política SPF aplicable | ¿Se evaluó el dominio MailFrom/HELO correcto? |
| `temperror` | error temporal, normalmente DNS | comprobar el resolvedor, el tiempo de espera y la disponibilidad autoritativa |
| `permerror` | registro o evaluación permanentemente no válidos | comprobar sintaxis, varios registros SPF, recursión y presupuesto de búsquedas |

### Presupuesto de búsquedas y dependencias

SPF limita a diez términos la suma de `include`, `a`, `mx`, `ptr`, `exists` y `redirect` en toda la evaluación recursiva. Superarlo debe producir `permerror`. Para `mx` y `ptr` se aplican límites de direcciones adicionales; más de dos respuestas vacías o resultados NXDOMAIN, las denominadas Void Lookups, también deberían conducir a `permerror` ([RFC 7208, sección 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4)).

El presupuesto es un límite de tiempo de ejecución, no una mera comprobación de caracteres. Un solo `include` puede introducir más includes, resoluciones MX y ámbitos de fallo. `include` solo delega la cuestión de si el host actual logra allí un `pass`; `redirect` adopta, después de que fallen los mecanismos, toda la política de otro dominio. RFC 7208 recomienda `include` para cruzar límites administrativos y `redirect` más bien para centralizar dominios administrados de forma conjunta ([RFC 7208, secciones 5.2 y 6.1](https://datatracker.ietf.org/doc/html/rfc7208#section-5.2)). Por tanto, el inventario operativo debe incluir propietario, finalidad, vía de cambio y presupuesto de peor caso medido de cada include externo.

### Reenvío y dominio SPF

Un reenviador clásico se conecta con su propia IP al siguiente receptor, pero mantiene el `MAIL FROM` original. Así, se evalúa una IP frente a la política SPF de un dominio ajeno y SPF puede fallar aunque la entrega original fuera legítima. RFC 7208 describe como contramedida la reescritura de la ruta inversa a un dominio del intermediario; las listas de correo suelen hacerlo de todos modos ([RFC 7208, anexo D.2](https://datatracker.ietf.org/doc/html/rfc7208#appendix-D.2)). Esto soluciona el aprobado SPF para el nuevo dominio de envelope, pero solo produce un aprobado DMARC si el nuevo dominio está alineado con el `From:` visible. Por ello, una firma DKIM conservada y alineada es especialmente importante en rutas indirectas.

## DKIM: firma de un dominio

DomainKeys Identified Mail añade un campo de cabecera `DKIM-Signature`. El firmante canoniza cabeceras seleccionadas y el cuerpo, crea el hash del cuerpo `bh=`, firma los datos establecidos y publica la clave en `s=._domainkey.d=`. El verificador reconstruye los mismos datos, consulta la clave DNS TXT y comprueba la firma y el hash del cuerpo. Un aprobado demuestra que los componentes firmados no se han modificado desde la firma de una manera detectable y que el firmante controlaba la clave privada del Signing Domain. DKIM no acredita a una persona física ni la veracidad del contenido ([RFC 6376, secciones 3.5 a 3.8](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

Las etiquetas importantes son `a=` para el algoritmo, `c=` para la canonización de cabecera/cuerpo, `d=` para el Signing Domain, `s=` para el selector, `h=` para los nombres de cabeceras firmadas, `bh=` para el hash del cuerpo y `b=` para la firma. `t=` y `x=` pueden transportar el momento de firma y de expiración, pero no son una protección fiable contra replay. El `l=` opcional limita la región firmada del cuerpo y puede permitir adjuntar contenido no firmado; RFC 6376 describe explícitamente esta superficie de abuso ([RFC 6376, secciones 3.5 y 8.2](https://datatracker.ietf.org/doc/html/rfc6376#section-8.2)).

La canonización `simple` no tolera prácticamente ningún cambio. `relaxed` normaliza, entre otras cosas, determinadas grafías y espacios en blanco, para que los cambios habituales de transporte no rompan una firma innecesariamente. Las cabeceras y el cuerpo pueden utilizar procedimientos diferentes; sin `c=` se aplica `simple/simple`. La canonización no modifica el mensaje transmitido, sino solo su forma de entrada para la firma o verificación ([RFC 6376, sección 3.4](https://datatracker.ietf.org/doc/html/rfc6376#section-3.4)).

### Claves, algoritmos y rotación

RFC 8301 exige al menos 1024 bits para RSA, recomienda al menos 2048 bits a los firmantes y declara RSA-SHA1 histórico. RFC 8463 añade Ed25519-SHA256 y permite firmas paralelas con distintos selectores para compatibilidad de transición ([RFC 8301, secciones 3.1 y 3.2](https://datatracker.ietf.org/doc/html/rfc8301#section-3), [RFC 8463, secciones 5 y 6](https://datatracker.ietf.org/doc/html/rfc8463#section-5)). El algoritmo utilizado sigue siendo una decisión de interoperabilidad: una obligación estandarizada del lado del verificador no demuestra automáticamente que cada plataforma receptora real la implemente sin errores.

Los selectores separan el cambio de clave del dominio. Para una rotación se publica primero la nueva clave pública, después se firma con la nueva clave privada y la antigua clave DNS se elimina solo cuando los mensajes antiguos ya no necesiten verificarse regularmente. RFC 6376 desaconseja reutilizar un selector con una clave nueva, pues de lo contrario los errores de firmas antiguas no se pueden distinguir de falsificaciones. Un `p=` vacío en el registro de clave revoca la clave ([RFC 6376, secciones 3.1 y 6.1.2](https://datatracker.ietf.org/doc/html/rfc6376#section-3.1)).

La clave privada no pertenece al DNS ni a repositorios generales de configuración. El diseño operativo debe definir propietario de la clave, generación, almacenamiento protegido, acceso del firmante, rotación, revocación de emergencia, decisión de copia de seguridad y trazabilidad de auditoría. RFC 6376 exige cuidado al proteger las claves privadas y menciona el almacenamiento cifrado y el hardware criptográfico como posibles medidas de protección. Varias plataformas de envío deben disponer de selectores separados para que una plataforma comprometida pueda aislarse sin un cambio global de clave. Esta arquitectura sigue la división administrativa del espacio de nombres de selectores prevista por DKIM ([RFC 6376, secciones 3.1 y 8.3 y anexo C](https://datatracker.ietf.org/doc/html/rfc6376#section-8.3)).

SPF puede fallar tras un reenvío aunque el mensaje no haya cambiado. En cambio, DKIM puede sobrevivir a un reenvío, pero romperse por un pie de página modificado. Por ello, DMARC conecta ambos procedimientos mediante la denominada alineación.

## DMARC: alineación, política y evaluación

DMARC requiere un único campo RFC5322.From correctamente formado y extrae de él exactamente un **Author Domain**. Considera el dominio autenticado mediante SPF y todos los Signing Domains DKIM verificados correctamente. Hay un aprobado DMARC cuando al menos un resultado SPF o DKIM es `pass` y su dominio está alineado con el Author Domain. Por tanto, ambos mecanismos no tienen que aprobar al mismo tiempo; operacionalmente se desean ambos porque pueden fallar en distintos flujos indirectos ([RFC 9989, secciones 4.2 a 4.4 y 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-4.2)).

Con **strict alignment**, los dominios deben ser idénticos. Con **relaxed alignment**, deben tener el mismo Organizational Domain. `adkim=s` o `aspf=s` exige strict; sin estas etiquetas se aplica relaxed. La determinación del Organizational Domain se realiza según RFC 9989 mediante un DNS Tree Walk limitado y ya no únicamente mediante una Public Suffix List. El recorrido consulta como máximo ocho niveles de nombres y tiene en cuenta `psd=y` o `psd=n` ([RFC 9989, secciones 4.4 y 4.10](https://datatracker.ietf.org/doc/html/rfc9989#section-4.10)). Este cambio es relevante con verificadores antiguos y nuevos mezclados; el propio RFC 9989 advierte de posibles resultados de alineación diferentes.

### Registro de política y etiquetas

El registro DMARC se encuentra en `_dmarc.<domain>` y utiliza sintaxis etiqueta/valor. `v=DMARC1` identifica el formato. `p=` describe el tratamiento deseado para los mensajes no aprobados del dominio de política: `none`, `quarantine` o `reject`. `sp=` puede tratar de forma diferente los subdominios existentes, `np=` los subdominios inexistentes. `rua=` indica destinos para Aggregate Reports, `ruf=` destinos opcionales para Failure Reports. `t=y` marca el modo de prueba definido en RFC 9989. Las etiquetas desconocidas se ignoran ([RFC 9989, secciones 4.5 a 4.8](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)).

El despliegue porcentual anterior mediante `pct=` ya no forma parte del protocolo según RFC 9989. La experiencia operativa mostró una aplicación inconsistente; `t=` sustituye solo el efecto especial anterior de `pct=0`, no pasos porcentuales arbitrarios. Por lo tanto, la introducción gradual debe planificarse mediante Author Domains separadas, políticas de subdominios, flujos de correo dirigidos y evaluación sólida de informes, no mediante un supuesto control porcentual normalizado ([RFC 9989, anexo A.6](https://datatracker.ietf.org/doc/html/rfc9989#appendix-A.6)).

Una política es una preferencia publicada por el propietario del dominio. El receptor puede desviarse debido a política local, reputación, flujos indirectos u otros hallazgos, y puede documentar las anulaciones en informes. `p=reject` no hace que un mensaje sea automáticamente imposible de entregar en sentido SMTP para cada receptor; crea una afirmación clara y automatizable para el tratamiento de DMARC-Fail ([RFC 9989, secciones 5.3 y 5.4](https://datatracker.ietf.org/doc/html/rfc9989#section-5.3)).

Una política DMARC solo debe endurecerse cuando se conozcan todas las rutas legítimas de envío. Los Aggregate Reports proporcionan los datos para este inventario y muestran qué sistemas envían realmente bajo un dominio.

## Reporting como canalización operativa de datos

RFC 9990 separa el Aggregate Reporting de la especificación central de DMARC. Un informe resume mensajes por IP de origen, política evaluada, disposición, alineación y resultados de autenticación. El formato de datos es XML; el archivo debe comprimirse con GZIP y entonces lleva la extensión `.xml.gz`. Los registros individuales contienen, entre otras cosas, `source_ip`, `count`, `header_from`, resultados SPF y DKIM, así como posibles motivos de anulación de política ([RFC 9990, secciones 3.1 y 3.4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.1)).

Por tanto, un colector es un servicio de ingesta relevante para la seguridad. Recibe correos y adjuntos no solicitados, descomprime datos ajenos, analiza XML, deduplica informes y agrega volúmenes. Los límites de tamaño, el presupuesto de descompresión, el parser XML sin entidades externas, la protección antimalware, la cuarentena de archivos defectuosos, la idempotencia y la retención forman parte de la arquitectura. RFC 9989 advierte de informes deliberadamente malformados y DoS contra destinos de informes; RFC 9990 regula los duplicados y destinos externos ([RFC 9989, sección 11.2](https://datatracker.ietf.org/doc/html/rfc9989#section-11.2), [RFC 9990, secciones 3.5.4 y 4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.5.4)).

Si `rua=` se encuentra fuera del Organizational Domain, el receptor del informe debe autorizar esta relación mediante un registro DNS TXT adicional. De esta manera, un dominio no puede inundar con informes a terceros arbitrarios ([RFC 9990, sección 4](https://datatracker.ietf.org/doc/html/rfc9990#section-4)). Los Failure Reports según RFC 9991 pueden revelar información de mensajes individuales. Debido a los riesgos de protección de datos, muchos operadores los restringen; los Aggregate Reports son el canal de visibilidad recomendado sin contenido de usuarios finales ([RFC 9991, sección 7](https://datatracker.ietf.org/doc/html/rfc9991#section-7)).

Para el análisis administrativo, las series temporales son más importantes que un único valor diario: volumen por IP de origen y Author Domain, SPF alineado, DKIM alineado, DMARC-Fail, `temperror`, `permerror`, selectores desconocidos, anulaciones de política y cobertura de reporteros. Los informes son observaciones de receptores individuales y no una contabilidad completa de envíos; los receptores no están obligados a proporcionar cada evaluación o disposición solicitada ([RFC 9989, secciones 1 y 6](https://datatracker.ietf.org/doc/html/rfc9989#section-6)).

## Confiar correctamente en Authentication-Results

El servicio de verificación receptor puede almacenar los resultados SPF, DKIM y DMARC en el campo de cabecera `Authentication-Results`. `authserv-id` identifica el servicio o el Administrative Management Domain que realizó la comprobación. Este campo solo tiene valor probatorio dentro de un límite de confianza definido. Un remitente externo puede insertar por sí mismo un `Authentication-Results: ... dmarc=pass` convincente ([RFC 8601, secciones 1.5.4 a 1.6](https://datatracker.ietf.org/doc/html/rfc8601#section-1.5.4)).

Por ello, el MTA de borde debe eliminar las instancias externas o permitir únicamente productores explícitamente confiables. Los consumidores internos necesitan una lista de valores `authserv-id` permitidos y deben considerar la posición en la cadena de `Received` confiable o en el modelo interno de procedencia de metadatos. RFC 8601 advierte explícitamente de habilitar el campo de cabecera para decisiones de filtrado sin un MTA de borde verificado y conforme ([RFC 8601, secciones 5 y 7.1](https://datatracker.ietf.org/doc/html/rfc8601#section-5)).

ARC según RFC 8617 puede transportar resultados de autenticación y cambios posteriores a través de intermediarios en una cadena firmada. ARC se publica como protocolo experimental y no proporciona una raíz de confianza global: el receptor final sigue decidiendo en qué ARC-Sealers confía. Por tanto, un estado de cadena ARC válido es contexto para la política local, no un sustituto de DMARC ni una decisión automática de aceptación ([RFC 8617, secciones 1 y 5](https://datatracker.ietf.org/doc/html/rfc8617#section-1)).

Hasta aquí se trató de la entrega directa. Sin embargo, los reenvíos, listas y gateways cambian la dirección IP, el envelope o el contenido, y con ello precisamente las entradas de las tres comprobaciones.

## Flujos indirectos y puntos de ruptura típicos

Los reenvíos, las listas de correo, los gateways de seguridad y los sistemas de tickets modifican distintas partes del mensaje. Un reenvío cambia la IP de conexión y puede romper SPF. Una lista de correo puede modificar Subject, cabeceras de lista, pie de página del cuerpo o estructura MIME, y con ello romper DKIM. Además, puede reescribir el remitente del envelope y el `From:` visible. RFC 7960 describe estos problemas de interoperabilidad y sus respectivos efectos secundarios; no existe una corrección universal que mantenga inalterados al mismo tiempo la identidad, la función de lista y la política existente del receptor ([RFC 7960, secciones 3 y 4](https://datatracker.ietf.org/doc/html/rfc7960#section-3)).

Por ello, para diagnosticar errores los administradores deben comparar el estado **antes y después de cada intermediario**: IP del cliente, HELO, MailFrom, RFC5322.From, firmas DKIM existentes, `Authentication-Results`, nuevos campos `Received` y mutaciones del contenido. Un resultado final `dmarc=fail` por sí solo no muestra qué salto perdió la identidad alineada.

## Arquitectura técnica de una plataforma productiva

La autenticación de correo no es un único appliance, sino un plano de control y de datos distribuido:

| Componente | Stack tecnológico | Estado persistente | Área central de fallo |
|---|---|---|---|
| DNS autoritativo | conjuntos de RR TXT, DNSSEC opcional, cambios basados en zona o API | SPF, claves públicas DKIM, política DMARC | registros obsoletos, Split-Horizon, TTL, delegación defectuosa |
| Firmante saliente | filtro MTA, biblioteca o gateway; RSA/Ed25519 y SHA-256 | claves privadas, configuración del selector, política de firma | acceso a claves, `d=` incorrecto, falta de firma en flujos parciales |
| Verificador entrante | MTA/filtro perimetral, resolvedor recursivo, biblioteca criptográfica, motor de política | caché del resolvedor, límite de confianza, anulaciones locales | IP de cliente incorrecta tras proxy, timeout DNS, Auth-Results manipulados |
| Módulo de política DMARC | alineación, DNS Tree Walk, lógica de dominio/política | caché y versión de política | modelo RFC 7489 antiguo, Organizational Domain incorrecto |
| Generador de informes | telemetría MTA, agregador, XML/GZIP, envío SMTP | ventanas temporales, registros, ID de informe | pérdida de datos, duplicados, cobertura incompleta de reporteros |
| Colector de informes | buzón, descompresor, parser XML, base de datos, panel | informes en bruto, normalización, series temporales | ataque al parser, deduplicación incorrecta, retención no controlada |

El lenguaje de programación y el producto son intercambiables; los objetos de protocolo y los límites de confianza no. Un MTA puede implementar firma y verificación en C, Rust, Java, Go o mediante un proceso de filtro separado. Las preguntas decisivas para las operaciones son las mismas: ¿de dónde procede la IP del cliente después de los balanceadores de carga? ¿Qué resolvedor y caché se utilizan? ¿Dónde está la clave privada? ¿Qué proceso puede firmar? ¿Qué cabeceras se eliminan antes del límite de confianza? ¿Cómo se correlacionan versión de política, respuesta DNS, Message-ID, Queue-ID y resultado? Los RFC definen la semántica de transmisión y evaluación, no el modelo concreto de proceso.

### Proceso controlado de implementación y cambio

RFC 9989 describe una secuencia sólida para propietarios de dominios: publicar SPF alineado, configurar DKIM alineado, crear un buzón de Aggregate Reports, publicar inicialmente DMARC en modo de monitorización con `p=none`, evaluar los informes, subsanar lagunas y solo entonces decidir sobre la aplicación de la política ([RFC 9989, sección 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-5.1)). De ello se deriva un proceso verificable para la gestión de cambios:

1. Inventariar todos los Author Domains, dominios de envelope, nombres HELO, productos de envío, tenants, relays, reenviadores y propietarios de claves.
2. Demostrar al menos una identidad alineada para cada flujo de correo legítimo; SPF y DKIM conjuntamente reducen la dependencia de una única ruta indirecta.
3. Preparar la recepción de informes y su evaluación segura para producción antes de `rua=`.
4. Operar `p=none` como fase de medición, pero sin confundirlo con un efecto de protección.
5. Clasificar las fuentes desconocidas: legítimas y mal configuradas, retiradas, abusivas o distorsionadas por un flujo indirecto.
6. Introducir la aplicación de la política por dominio controlable, documentar excepciones y observarla mediante informes y telemetría de entrega.
7. Practicar periódicamente la rotación de claves, el cambio de proveedor, el rollback DNS, el fallo de informes y el firmante comprometido.

Una aprobación no debe aceptar solo un registro TXT sintácticamente válido. Requiere mensajes de prueba a través de cada ruta de envío, evidencia de cabeceras en el receptor, datos de informes de varios dominios receptores, medición del presupuesto SPF, comprobación de los TTL de selectores y un rollback que no deje el Author Domain desprotegido o imposible de entregar.

De estas dependencias se deriva una secuencia fija de diagnóstico: primero identificar los dominios utilizados, después comprobar DNS y firma y, por último, evaluar alineación y política DMARC.

## Herramientas de diagnóstico

Las siguientes consultas utilizan `example.ch` y el selector de ejemplo `s2026a`. Los nombres productivos y los mensajes almacenados localmente solo pueden investigarse en entornos autorizados.

### Leer SPF, DMARC y DKIM en DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail-Authentifizierungs-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type TXT -DnsOnly
Resolve-DnsName _dmarc.example.ch -Type TXT -DnsOnly
Resolve-DnsName s2026a._domainkey.example.ch -Type TXT -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer example.ch TXT
dig +noall +answer _dmarc.example.ch TXT
dig +noall +answer s2026a._domainkey.example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran los conjuntos de RR TXT, no su evaluación completa del protocolo. Varios Character Strings de un único registro TXT deben concatenarse sin caracteres adicionales; en cambio, varios registros SPF o DMARC en el mismo nombre constituyen un error ([RFC 7208, secciones 3.2 y 4.5](https://datatracker.ietf.org/doc/html/rfc7208#section-3.2), [RFC 9989, sección 4.5](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)). Para cuestiones de Split-Horizon o DNSSEC, las vistas autoritativa y recursiva deben comprobarse por separado.

### Encontrar resultados confiables en la cabecera del mensaje

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Header-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\message.eml |
  Select-String -Pattern '^(Authentication-Results|DKIM-Signature|Received):' -Context 0,8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">grep -E -A8 '^(Authentication-Results|DKIM-Signature|Received):' message.eml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) y [`Select-String`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string), o [`grep`](https://www.gnu.org/software/grep/manual/grep.html), ayudan en la primera revisión. Debido al plegado de cabeceras y a los campos múltiples, la búsqueda de texto no sustituye a un parser conforme a RFC. Son decisivos el `authserv-id` confiable, su posición relativa al límite `Received` propio, los dominios realmente evaluados y la distinción entre resultado bruto y alineación.

### Comprobar estructuralmente un Aggregate Report

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DMARC-XML-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">[xml]$report = Get-Content .\report.xml -Raw
$report.SelectNodes("//*[local-name()='record']").Count
$report.SelectSingleNode("//*[local-name()='org_name']").InnerText</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">xmllint --noout report.xml
xmllint --xpath 'count(//*[local-name()="record"])' report.xml
xmllint --xpath 'string(//*[local-name()="org_name"])' report.xml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) carga aquí un archivo XML local ya descomprimido; [`xmllint`](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) comprueba la estructura y las consultas XPath. Los informes desconocidos no deben abrirse de forma interactiva con herramientas de escritorio privilegiadas. Esta comprobación individual no acredita ni la conformidad con el esquema ni el procesamiento masivo seguro, la detección de duplicados o la agregación correcta.

### Asignar sistemáticamente los errores a un límite

| Observación | Causa probable | Siguiente prueba fiable |
|---|---|---|
| `spf=none` | identidad incorrecta o sin registro SPF | MailFrom/HELO del resultado SMTP o de autenticación y TXT con el nombre exacto |
| `spf=permerror` | sintaxis, varios registros, recursión o presupuesto DNS | evaluar el árbol SPF recursivo completo y los contadores de lookup/Void |
| SPF aprueba, DMARC falla | dominio SPF no alineado | comparar RFC5321.MailFrom con RFC5322.From en el modo de alineación configurado |
| `dkim=fail (body hash did not verify)` | cuerpo modificado después de la firma | comparar salto de firma, normalización MIME, pie de página/disclaimer y canonización |
| `dkim=temperror` | consulta de clave fallida temporalmente | nombre de selector, respuesta del resolvedor, timeout y disponibilidad DNS autoritativa |
| DKIM aprueba, DMARC falla | solo aprueba una firma no alineada | comprobar todos los dominios `d=` individualmente frente al Author Domain |
| Los receptores informan resultados DMARC distintos | rutas, cachés DNS, modelos de verificador o mutaciones diferentes | correlacionar el mismo Message-ID/firma mediante una cabecera completa y el momento DNS de cada caso |
| IP de origen desconocida en Aggregate Reports | nuevo remitente legítimo, reenviador o abuso | determinar volumen, ejemplo de cabecera, asignación inversa/proveedor y propietario interno del servicio |
| `Authentication-Results` se contradicen | varios saltos de verificación o campo externo falsificado | utilizar solo resultados dentro del límite de confianza definido |
| `p=reject`, pero el mensaje se entrega | anulación local en el receptor | comprobar `disposition`, motivo de anulación y otros resultados de filtro |

## Historia técnica

SPF surgió de varias propuestas de autorización de remitentes SMTP y se publicó en 2006 como RFC 4408 experimental. RFC 7208 trasladó SPF en 2014 al Standards Track, eliminó el tipo de RR DNS SPF independiente y precisó, entre otras cosas, los límites DNS y Void ([RFC 4408](https://datatracker.ietf.org/doc/html/rfc4408), [RFC 7208, anexo B](https://datatracker.ietf.org/doc/html/rfc7208#appendix-B)). Su arquitectura permaneció deliberadamente orientada a la conexión SMTP y a la ruta inversa.

DKIM combinó experiencias de DomainKeys e Identified Internet Mail. RFC 4871 estandarizó DKIM en 2007; RFC 6376 lo sustituyó en 2011 y precisó el modelo de firma, clave y verificación. RFC 8301 actualizó en 2018 los algoritmos y las longitudes de claves RSA, y RFC 8463 añadió Ed25519-SHA256 ([RFC 4871](https://datatracker.ietf.org/doc/html/rfc4871), [RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376), [RFC 8301](https://datatracker.ietf.org/doc/html/rfc8301), [RFC 8463](https://datatracker.ietf.org/doc/html/rfc8463)).

DMARC se publicó inicialmente en 2015 con RFC 7489 como documento informativo. RFC 9989 sustituyó en 2026 RFC 7489 y la extensión PSD RFC 9091 como especificación de Standards Track; Aggregate Reporting y Failure Reporting se separaron al mismo tiempo en RFC 9990 y RFC 9991. El cambio introdujo, entre otras cosas, DNS Tree Walk, `np`, `psd` y `t`, y eliminó `pct` ([RFC 9989, anexo C](https://datatracker.ietf.org/doc/html/rfc9989#appendix-C), [RFC 9990](https://datatracker.ietf.org/doc/html/rfc9990), [RFC 9991](https://datatracker.ietf.org/doc/html/rfc9991)).

La transmisión legible por máquina de resultados de verificación evolucionó de RFC 5451, pasando por RFC 7001 y RFC 7601, a RFC 8601. ARC se publicó en 2019 con RFC 8617 como un intento experimental de transmitir de forma firmada los resultados de autenticación de flujos indirectos ([RFC 8601, sección 6](https://datatracker.ietf.org/doc/html/rfc8601#section-6), [RFC 8617](https://datatracker.ietf.org/doc/html/rfc8617)). Esta historia explica por qué las plataformas reales pueden mostrar simultáneamente etiquetas DMARC antiguas, Organizational Domains basados en PSL, distintos algoritmos DKIM y diferentes modelos de confianza de Auth-Results.

## Fuentes

- [RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208) – identidades SPF, evaluación de registros, resultados y límites DNS.
- [RFC 6376 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc6376) – modelo de firma, clave y verificación DKIM.
- [RFC 9989 – Domain-Based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc9989) – protocolo central DMARC, alineación, política, DNS Tree Walk y operaciones.
- [RFC 5321, secciones 3.3 y 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 5322, sección 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322)
- [RFC 8301 – DKIM Cryptographic Algorithm and Key Usage Update](https://datatracker.ietf.org/doc/html/rfc8301) – SHA-256 y longitudes de claves RSA.
- [RFC 8463 – Ed25519-SHA256 for DKIM](https://datatracker.ietf.org/doc/html/rfc8463) – algoritmo adicional de firma y clave.
- [RFC 9990 – DMARC Aggregate Reporting](https://datatracker.ietf.org/doc/html/rfc9990) – modelo de datos XML, transporte, duplicados y destinos de informes externos.
- [RFC 9991 – DMARC Failure Reporting](https://datatracker.ietf.org/doc/html/rfc9991) – Failure Reports por mensaje y protección de datos.
- [RFC 8601 – Authentication-Results](https://datatracker.ietf.org/doc/html/rfc8601) – formato de cabecera, `authserv-id` y límite de confianza.
- [RFC 8617 – Authenticated Received Chain](https://datatracker.ietf.org/doc/html/rfc8617) – cadena ARC experimental para flujos indirectos.
- [RFC 7960, secciones 3 y 4](https://datatracker.ietf.org/doc/html/rfc7960)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – consultas DNS en Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – consultas DNS en sistemas Unix.
- [Microsoft Learn – Get-Content](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) – análisis local de archivos y cabeceras en Windows.
- [Microsoft Learn – Select-String](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) – análisis de patrones en PowerShell.
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) – búsqueda de texto en Linux y Unix.
- [libxml2 – xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) – comprobación XML y XPath.
- [RFC 4408 – Sender Policy Framework, Experimental](https://datatracker.ietf.org/doc/html/rfc4408) – predecesor de RFC 7208.
- [RFC 4871 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc4871) – predecesor de RFC 6376.
