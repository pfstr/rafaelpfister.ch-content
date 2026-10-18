---
title: "Cifrado de correo electrónico: métodos, claves y modelos operativos"
blatt: "verschluesselung"
description: "Cifrado de correo electrónico para administradores de mensajería: SMTP-TLS, MTA-STS, DANE y REQUIRETLS, S/MIME y OpenPGP, modelos de cliente, gateway y portal, ciclo de vida de claves y certificados, metadatos, monitorización, recuperación y diagnóstico."
fakten:
  - label: Pregunta fundamental
    wert: ¿Qué datos están protegidos entre qué límites de confianza?
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Transporte
    wert: SMTP STARTTLS protege exactamente un salto
    href: https://datatracker.ietf.org/doc/html/rfc3207
  - label: Política de transporte
    wert: DANE · MTA-STS · REQUIRETLS
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf
  - label: S/MIME
    wert: MIME + CMS + X.509/PKIX
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: OpenPGP
    wert: Formato de paquetes, claves, firmas y cifrado
    href: https://datatracker.ietf.org/doc/html/rfc9580
  - label: Firma
    wert: Autenticidad e integridad; no confidencialidad
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Modelo de cliente
    wert: claves privadas en el extremo; el servidor de correo almacena texto cifrado
    href: https://www.nccoe.nist.gov/publication/1800-6/VolC/
  - label: Modelo de gateway
    wert: texto claro dentro de la zona controlada y en el gateway
    href: https://www.seppmail.com/products/
  - label: Modelo de portal
    wert: notificación por correo; contenido en almacenamiento web con identidad propia
    href: https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm
  - label: Gestión de claves
    wert: inventario · protección · distribución · rotación · revocación · recuperación
    href: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
  - label: Metadatos visibles
    wert: el sobre SMTP y las cabeceras externas permanecen visibles según el modelo
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Regla de recuperación
    wert: proteger las claves de cifrado cuando corresponda; no replicar las claves de firma
    href: https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - seppmail
translationSourceHash: 915d86b5c19b97fc5564faf33053144b692f76fd3d6c732fd955f466d25c4983
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T09:07:35.826Z
translationReview: required
---

# Cifrado de correo electrónico: métodos, claves y modelos operativos

La pregunta «¿Está cifrado este correo electrónico?» es técnicamente incompleta. Un mensaje puede ser legible en el cliente, almacenarse como texto cifrado en el servidor de correo, descifrarse en un gateway, protegerse entre dos MTA mediante [TLS](/kb/tls) y volver a guardarse sin cifrar en el destinatario. Por ello, una respuesta fiable debe indicar el **objeto protegido, los extremos, los titulares de las claves y los puntos con texto claro**. NIST aborda los mecanismos de transporte, de dominio y de contenido precisamente por este motivo como controles que interactúan, pero separados ([NIST SP 800-177 Rev. 1](https://www.nist.gov/publications/trustworthy-email-0)).

El **cifrado de transporte** protege una conexión. SMTP STARTTLS se negocia de nuevo para cada salto y termina en el MTA correspondiente. El **cifrado de contenido** protege una entidad MIME con S/MIME u OpenPGP y puede sobrevivir a varios saltos de transporte y almacenamiento. Que realmente funcione de extremo a extremo no depende solo del formato: si un gateway empresarial descifra en representación del usuario, los extremos criptográficos están en los gateways y no en los clientes de usuario ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Una **firma digital** es una prestación de seguridad distinta. Vincula datos a una clave privada de firma y permite comprobar la integridad y la identidad; no hace secreto el contenido. A la inversa, un mensaje descifrado correctamente no dice todavía quién lo creó. En S/MIME se distinguen para ello CMS `SignedData` y `EnvelopedData` o `AuthEnvelopedData`. OpenPGP también separa los paquetes de firma, clave de sesión y datos cifrados ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 8551, S/MIME Content Types](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580, Packet Syntax](https://datatracker.ietf.org/doc/html/rfc9580)).

La explicación sigue un mensaje y pregunta en cada salto dónde existe texto claro y quién controla las claves. A partir de ello se comparan el cifrado de transporte, los métodos de extremo a extremo, los gateways, los portales, los metadatos, el diagnóstico y la recuperación.

El cifrado de correo electrónico puede proteger el salto de transporte, el contenido del mensaje o el acceso mediante un portal. Estos modelos tienen distintos titulares de claves y distintos puntos en los que existe texto claro. Por ello, el artículo sigue primero la ruta de protección y solo después los productos y procedimientos individuales.

## Enfoque de arquitectura: modelos de protección y límites del texto claro

El cifrado de correo electrónico se diferencia sobre todo en qué dos puntos cifran y descifran. Por ello, la tabla comienza con la ruta normal del mensaje y muestra después dónde permanece técnicamente visible el texto claro.

| Modelo | Extremos criptográficos | El texto claro está presente como mínimo en | Estado operativo principal |
|---|---|---|---|
| SMTP-TLS oportunista | dos MTA adyacentes | cada MTA participante | certificados, capacidades TLS, cola y reintentos |
| TLS de transporte forzado | dos MTA más una política publicada o local | cada MTA participante | MTA-STS/DANE/REQUIRETLS, DNS/HTTPS e informes |
| S/MIME/OpenPGP de cliente a cliente | MUA o módulo criptográfico de los usuarios | antes del cifrado y después del descifrado en el extremo | claves privadas, certificados/claves, compatibilidad del cliente |
| de gateway a gateway | gateways de correo de las organizaciones | sistema de correo interno, procesamiento del gateway, zona de destino | política, directorio, almacén de claves del gateway, cola y asignación |
| portal web | remitente/gateway y aplicación web del portal | gateway/portal, navegador tras iniciar sesión | objeto de mensaje, cuenta del destinatario, OTP, enlace, conservación |

Ninguno de estos modelos es «más fuerte» de forma general. El modelo de cliente a cliente reduce los puntos de texto claro en la infraestructura, pero dificulta el análisis de malware, el journaling, eDiscovery y la recuperación. Por ello, el ejemplo práctico del NCCoE ejecuta la comprobación S/MIME en el cliente de usuario con sus Trust Anchors, tras finalizar los controles de transporte y basados en dominio en el MTA ([NCCoE: DNS-Based Email Security](https://www.nccoe.nist.gov/publication/1800-6/VolC/)). Los gateways centralizan la política y las claves, pero se convierten ellos mismos en puntos de texto claro y claves con privilegios elevados. Los portales llegan a destinatarios sin criptografía de correo, pero crean un procedimiento adicional de web, identidad y almacenamiento. La elección depende de la clasificación de datos, el modelo de amenazas, la conservación, los requisitos legales y las capacidades de ambas partes de la comunicación, no del nombre del producto.

<iframe class="kb-infographic kb-infographic--compact" style="aspect-ratio: 1400 / 1076; max-width: 720px; margin-inline: auto" src="/images/kb-interaktiv-verschluesselung.svg?v=20260813" title="Interaktive Infografik: E-Mail-Verschlüsselung mit Transport-, Client-, Gateway- und Portalmodell, Klartextgrenzen, Schlüsselzustand, Metadaten und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-verschluesselung.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## SMTP-TLS: protección por salto de transporte

[SMTP](/kb/smtp) se diseñó originalmente sin confidencialidad obligatoria. STARTTLS amplía una sesión SMTP existente: tras completar correctamente el handshake TLS, las capacidades se negocian de nuevo mediante EHLO. En modo oportunista, el MTA remitente puede volver al texto claro si falta STARTTLS o se produce un error de certificado. Esto mejora la protección contra la escucha pasiva, pero sin una política adicional no ofrece una garantía fiable frente a un downgrade o un atacante activo ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

**MTA-STS** publica primero una indicación DNS-TXT y obtiene la política propiamente dicha mediante HTTPS. En modo `enforce`, los nombres MX indicados deben validarse mediante PKIX; los errores provocan una entrega retrasada en lugar de texto claro. **DANE para SMTP** vincula información de certificados o claves en registros TLSA validados por DNSSEC. **REQUIRETLS** es una indicación SMTP y de mensaje con la que un remitente exige TLS para la transmisión posterior. Estos procedimientos tienen dependencias y alcances distintos y no deben resumirse en una sola casilla «TLS requerido» ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

SMTP TLS Reporting, **TLS-RPT**, proporciona informes agregados sobre comprobaciones de políticas TLS correctas y fallidas. No demuestra una entrega individual ni contiene un mensaje; ayuda a detectar errores de certificados, DNS y políticas antes de que las colas crezcan durante mucho tiempo. NIST recomienda que los dominios receptores señalen DANE, MTA-STS o ambos, y describe las diferentes dependencias de DNSSEC, HTTPS y PKIX ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [NIST SP 800-177 Rev. 1, sección 5.2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)).

SMTP-TLS termina en el siguiente servidor de correo. Si el contenido del mensaje debe permanecer protegido a través de varios saltos, se utilizan métodos relacionados con el mensaje como S/MIME u OpenPGP.

## S/MIME: CMS, X.509 y MIME

S/MIME no es una única biblioteca de cifrado, sino una combinación de varios estándares. MIME estructura las partes del mensaje; CMS define contenedores firmados y cifrados; X.509/PKIX vincula las claves públicas a identidades y anclas de confianza mediante certificados. Para cada destinatario, el remitente cifra la clave simétrica de contenido con su clave pública o un procedimiento acordado de gestión de claves. Para leerla se requiere la clave privada correspondiente ([RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

El administrador debe separar dos funciones de clave. Una **clave de firma** debe estar disponible solo para el firmante; una copia central permitiría técnicamente a terceros generar nuevas firmas bajo su identidad. Una **clave de descifrado** puede protegerse o depositarse en escrow según la política de la organización, para que los mensajes empresariales antiguos sigan siendo legibles tras la pérdida de un dispositivo, un cambio de función o la renovación de un certificado. RFC 8551 señala expresamente que una persona puede poseer varios pares de claves activos para distintos fines; NIST diferencia la protección y la recuperación según el tipo de clave ([RFC 8551, Multiple Key Pairs](https://datatracker.ietf.org/doc/html/rfc8551), [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Antes de enviar, el remitente necesita un certificado de cifrado adecuado y válido para **cada** destinatario. La distribución puede realizarse mediante un directorio, mensajes firmados, una plataforma de certificados o manualmente. Los grupos de distribución son especialmente exigentes porque la expansión, la visibilidad y la disponibilidad de certificados de todos los miembros deben encajar. Microsoft documenta para Exchange los atributos `userSMIMECertificate` y `userCertificate` así como la colección de certificados para la comprobación de confianza ([Microsoft: S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime), [Microsoft: S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)).

## OpenPGP: el formato no es un modelo de confianza

RFC 9580 define paquetes OpenPGP, claves, fingerprints, firmas, claves de sesión y datos cifrados. Según la norma, un anillo de claves es únicamente una colección de claves; la forma en que una organización encuentra claves, verifica a los propietarios y decide la confianza queda fuera de este formato de datos. Por tanto, el **Web of Trust**, históricamente extendido, es una posible modalidad operativa, pero no una definición sinónima de OpenPGP. Otros entornos utilizan fingerprints verificados manualmente, claves firmadas por la organización, WKD, servicios de directorio o gateways gestionados centralmente ([RFC 9580, Keyrings y Fingerprints](https://datatracker.ietf.org/doc/html/rfc9580)).

Para el correo electrónico, PGP/MIME encapsula firmas y texto cifrado en estructuras MIME. En cambio, Inline-PGP protege solo partes de texto seleccionadas y genera fácilmente problemas de interoperabilidad con HTML, adjuntos, juegos de caracteres y canonicalización de líneas. RFC 3156 estandariza `multipart/encrypted` y `multipart/signed` para OpenPGP. Debe probarse con los interlocutores reales si un cliente admite completamente claves modernas RFC 9580, AEAD, las curvas utilizadas y PGP/MIME ([RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Un Key ID corto no constituye una prueba de identidad suficiente. RFC 9580 advierte que los Key ID no deben considerarse únicos; para las verificaciones se utiliza el fingerprint completo. Los certificados de revocación, la caducidad, las subclaves, los nuevos fingerprints y la distribución segura de una rotación deben formar parte del mismo ciclo de vida que la emisión inicial de la clave.

S/MIME y OpenPGP pueden aplicarse en el dispositivo final o centralmente en el gateway. El gateway simplifica la política y la gestión de claves, pero a cambio obtiene acceso al texto claro.

## Modelo de gateway: política central y poder central sobre el texto claro

Un gateway de correo acepta mensajes internos normales, clasifica remitentes, destinatarios y datos, busca claves S/MIME u OpenPGP adecuadas y elige cifrado, firma, TLS o entrega mediante portal. En entrada, puede descifrar antes de entregar al sistema de correo interno. Productos como [SEPPmail](/kb/seppmail), [Totemomail o Kiteworks EPG](/kb/totemomail) y [HIN](/kb/hin) utilizan plataformas diferentes, pero el mismo desplazamiento fundamental de confianza: la organización delega la criptografía del extremo de usuario a un componente de infraestructura ([SEPPmail: Productos](https://www.seppmail.com/products/), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Esto no resuelve por sí solo la distribución de claves. El gateway sigue necesitando una asignación demostrada entre la identidad del destinatario y la clave pública, una prioridad de varios procedimientos, reglas para claves caducadas o revocadas y un fallback definido. El intercambio automático de claves entre gateways puede simplificar estos pasos, pero es en sí mismo un protocolo de confianza con identidad de pares, caché, caducidad y estado de error.

El orden de las políticas determina qué controles ven el texto claro. Si un gateway de entrada descifra, las soluciones antimalware, DLP y archivado pueden operar después. Si un gateway de salida cifra demasiado pronto, los escáneres posteriores quedan ciegos. Si un gateway modifica una entidad MIME ya firmada, la firma de contenido o de [DKIM](/kb/mail-auth) puede dejar de ser válida. RFC 6376 define las cabeceras canonicalizadas y el cuerpo como objeto de firma; por ello, la firma, el cifrado, los avisos legales, la reescritura y DKIM deben ordenarse en una canalización documentada ([RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)).

Si no hay una clave adecuada disponible en el destinatario externo, un portal puede almacenar el mensaje y controlar el acceso mediante un inicio de sesión web. Así, el problema pasa de la criptografía de correo a la identidad web y a la operación de la plataforma.

## Entrega mediante portal: aplicación web en lugar de clave del destinatario

En la entrega mediante portal, el destinatario recibe un correo de notificación normal y accede al contenido protegido mediante una aplicación web. El extremo criptográfico es el portal, no el cliente de correo del destinatario. Con ello, la tarea técnica pasa de la distribución de claves públicas a la identidad web, la gestión de sesiones, el almacenamiento de mensajes y adjuntos, el enlace de notificación, la conservación y el canal de respuesta seguro.

HIN documenta como ejemplo concreto el registro, el número de móvil y el código SMS, así como la opción de recordar un dispositivo para accesos posteriores. Este ejemplo demuestra el modelo operativo, no una implementación de portal de validez general. Otros productos utilizan contraseña, código de un solo uso a través de un canal separado, identidad federada o una cuenta de cliente existente ([HIN: Mail a no miembros](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Para los administradores, un portal requiere al menos las siguientes evidencias:

- ¿Qué datos aparecen en la notificación no protegida y en el token URL?
- ¿Cómo se protegen el registro inicial, la vinculación de identidad, los intentos fallidos, el recuerdo de dispositivos y la recuperación de cuentas?
- ¿Dónde se almacenan los mensajes, adjuntos, claves y datos de auditoría, y cuándo se eliminan?
- ¿Un mensaje sigue siendo recuperable tras su caducidad, la eliminación de un usuario o un cambio de tenant?
- ¿Cómo se registran la respuesta segura, el reenvío, la descarga y la confirmación de lectura?
- ¿Qué sucede si el portal no es accesible, falla el servicio SMS o se compromete la cuenta del destinatario?

Un estado HTTP `200` solo demuestra que responde una interfaz web. Una prueba sintética del portal debe cubrir la notificación, el registro o el inicio de sesión, el segundo factor, la apertura, el adjunto, la respuesta y el registro de auditoría esperado.

## Metadatos, cabeceras y archivado

El cifrado de contenido no protege automáticamente el sobre SMTP. Los MTA que realizan la entrega necesitan al menos las direcciones de destinatario; los sistemas generan líneas `Received` y procesan información de tamaño y enrutamiento. Incluso con S/MIME, las cabeceras externas suelen permanecer visibles. RFC 8551 permite encapsular un mensaje completo como `message/rfc822` dentro de la entidad MIME protegida, pero exige una representación distinguible de las cabeceras internas y externas. Que el asunto, la visualización del remitente y la lista de destinatarios estén realmente protegidos es una función del cliente y del gateway, no una consecuencia segura del símbolo de candado ([RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322), [RFC 8551, Header Protection](https://datatracker.ietf.org/doc/html/rfc8551)).

El archivado debe establecer **qué representación** se conserva: texto claro original, entidad original firmada, original cifrado, copia de trabajo descifrada u objeto de portal. Solo el texto claro simplifica eDiscovery, pero pierde la prueba criptográfica original. Solo el texto cifrado puede quedar ilegible tras la pérdida de una clave. La validación de una firma en un momento posterior requiere los bytes originales, la cadena de certificados, la información de confianza y revocación de aquel momento y una base temporal; un certificado caducado hoy no hace automáticamente inútil una firma válida entonces.

## Ciclo de vida de claves y certificados

Un servicio criptográfico en producción necesita un inventario que vaya más allá de las fechas de caducidad. NIST SP 800-57 trata la generación, activación, protección, inventario, distribución, uso, rotación, revocación, archivado, recuperación y destrucción como tareas de gestión interrelacionadas ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

| Fase | Pregunta del administrador | Evidencia |
|---|---|---|
| Identidad | ¿Qué persona o función se vincula a la clave? | proceso de enrollment verificado, Subject/SAN/User ID, aprobación |
| Generación | ¿Dónde se genera la clave privada? | cliente, HSM, gateway o CA; algoritmo y exportabilidad |
| Distribución | ¿Cómo obtiene el remitente la clave pública correcta? | directorio, correo firmado, WKD, servicio de certificados, verificación de fingerprint |
| Uso | ¿Puede la clave firmar, descifrar o ambas cosas? | Key Usage/EKU, flags de OpenPGP, política y cliente real |
| Rotación | ¿Cómo se solapan la clave antigua y la nueva? | publicación antes del cambio, prueba en todas las plataformas de destino |
| Revocación | ¿Cómo se enteran verificadores y remitentes de un compromiso? | CRL/OCSP o revocación OpenPGP, política de caché y errores |
| Recuperación | ¿Quién puede descifrar contenido histórico? | procedimiento de escrow, aprobación de cuatro ojos, auditoría, prueba de restauración |
| Destrucción | ¿Cuándo termina deliberadamente la capacidad de descifrado? | evidencia de conservación y eliminación de claves y copias de seguridad |

La validación de rutas S/MIME incluye anclas de confianza, certificados intermedios, periodo de validez, nombre o identidad de correo, Key Usage/Extended Key Usage, algoritmo y estado de revocación. OCSP y las CRL son a su vez dependencias de red y caché. En un soft-fail, una comprobación sigue siendo posible pese a que la información de revocación no sea accesible; en un hard-fail, un problema de infraestructura puede bloquear todos los mensajes firmados o cifrados. Esta decisión debe formar parte de la política y de la monitorización ([RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

El diagnóstico comienza con el modelo de protección elegido: salto TLS, firma y cifrado del contenido o acceso por portal. Solo después se comprueban el certificado, la clave, la política y la ruta de entrega.

## Monitorización y análisis de incidencias

La monitorización debe separarse por modelo de protección. Para TLS de transporte cuentan los protocolos ofrecidos y negociados, los errores de política, la validación de certificados, TLS-RPT y la antigüedad de las colas. Para S/MIME/OpenPGP cuentan las claves de destinatario ausentes, rutas o firmas no válidas, fallbacks, rotación y errores de descifrado. Para los gateways se añaden la decisión de política, Message-ID, fingerprint de la clave, procesamiento de texto claro y siguiente salto. Para los portales, el registro, MFA/OTP, almacenamiento, caducidad de enlaces, inicio de sesión, apertura de mensajes y ruta de respuesta son señales propias.

Un protocolo de incidentes útil responde:

1. ¿Qué mensaje y qué ruta de destinatario están afectados?
2. ¿Qué modelo de protección y qué política se eligieron?
3. ¿Qué clave pública o certificado se utilizó, con qué fingerprint y estado?
4. ¿Dónde fue el mensaje legible, cifrado, almacenado o puesto en cola por última vez?
5. ¿El resultado fue entrega, almacenamiento en portal, cuarentena, fallback, reintento o error permanente?
6. ¿Puede reproducirse el resultado con una identidad de prueba controlada sin revelar datos de producción?

## Copia de seguridad y recuperación

Los datos cifrados y las claves deben planificarse conjuntamente, pero con medidas de protección separadas. Una copia de seguridad de mensajes cifrados sin las claves privadas de descifrado necesarias a largo plazo puede estar correctamente protegida y, aun así, no ser recuperable desde el punto de vista operativo. A la inversa, una copia de seguridad de escrow de claves protegida de forma insuficiente convierte muchos secretos individuales en un punto de ataque central. NIST trata el uso permitido de claves privadas recuperadas en función del tipo de clave y del periodo criptográfico ([NIST SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)).

El alcance mínimo de recuperación incluye:

- mensajes originales cifrados y, cuando corresponda, entidades MIME firmadas con valor probatorio;
- claves privadas de **descifrado** con metadatos, frases de contraseña, funciones y procedimientos de autorización;
- claves públicas, cadenas de certificados, Trust Anchors, datos de revocación y validación histórica;
- políticas de gateway y portal, enrutamiento, asignación de claves, datos de usuario y objetos de mensaje;
- fuentes de auditoría y tiempo para que las decisiones anteriores sigan siendo trazables;
- una prueba de restauración con claves antiguas y nuevas, varios destinatarios, adjuntos y comprobación de firmas.

Las claves privadas de firma no deben copiarse como mecanismo general de recuperación. Si una organización debe firmar de forma automatizada o en representación de alguien, resulta mejor una identidad explícita de organización con su propia política que la copia inadvertida de una clave personal. De lo contrario, la capacidad de recuperación y el no repudio entran en conflicto.

## Herramientas de diagnóstico

La resolución de problemas sigue el modelo de protección elegido. En el cifrado de transporte se comprueban DNS y TLS; en los métodos de extremo a extremo siguen los certificados, las claves y el procesamiento del cliente; en los gateways se añade su estado de política y cola.

### MTA-STS, TLS-RPT y DANE en DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für E-Mail-Transportpolicy-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type TXT _mta-sts.example.ch
Resolve-DnsName -Type TXT _smtp._tls.example.ch
Resolve-DnsName -Type TLSA _25._tcp.mx.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short TXT _mta-sts.example.ch
dig +short TXT _smtp._tls.example.ch
dig +dnssec TLSA _25._tcp.mx.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran las políticas de transporte publicadas. Para DANE también debe demostrarse el estado de validación DNSSEC; un texto TLSA meramente visible todavía no es un registro autenticado.

### STARTTLS y cadena de certificados

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban la ruta TCP. [`curl`](https://curl.se/docs/manpage.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestran STARTTLS, handshake y cadena. Que un MTA de producción recurra a un fallback o ponga en cola ante errores solo se desprende de su política y sus registros.

### Certificado S/MIME y finalidad de uso

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für S-MIME-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
certutil.exe -dump .\recipient-smime.cer
certutil.exe -verify -urlfetch .\recipient-smime.cer
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl x509 -in recipient-smime.pem -noout \
  -subject -issuer -serial -fingerprint -dates -purpose -ext keyUsage -ext extendedKeyUsage
openssl verify -show_chain -CAfile trust-chain.pem recipient-smime.pem
```

  </div>
</div>

[`certutil`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil) y [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) muestran identidad, fingerprint, validez y finalidad de uso; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) comprueba un almacén de confianza indicado. La prueba debe repetirse con los mismos Trust Anchors y reglas de revocación que el cliente o gateway de producción.

### Fingerprint OpenPGP y estructura de claves

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OpenPGP-Schlüsselprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
gpg.exe --show-keys --with-fingerprint --with-subkey-fingerprint .\recipient.asc
gpg.exe --list-packets .\message.pgp
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
gpg --show-keys --with-fingerprint --with-subkey-fingerprint recipient.asc
gpg --list-packets message.pgp
```

  </div>
</div>

[`gpg`](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html) muestra fingerprints completos, subclaves, capacidades y estructura de paquetes. `--list-packets` analiza solo el formato; no confirma ni el propietario ni la confianza. Los fingerprints deben comprobarse mediante un segundo canal autenticado o un procedimiento organizativo definido.

### Ruta de portal y captura de paquetes

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Portal- und Paketpfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
pktmon filter remove
pktmon filter add SecureMail -p 25
pktmon start --capture --pkt-size 0 --file-name securemail.etl
pktmon stop
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
tcpdump -ni any -s 0 -w securemail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) y [`curl`](https://curl.se/docs/manpage.html) comprueban únicamente la accesibilidad HTTP/TLS. [`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) capturan solo en el punto de medición elegido. Los datos de paquetes pueden contener texto claro SMTP, direcciones y metadatos de portal, y deben protegerse en consecuencia.

## Historia técnica

PEM, Privacy Enhanced Mail, estandarizó a principios de la década de 1990 el correo de Internet protegido criptográficamente, pero no logró una interoperabilidad amplia. Posteriormente, S/MIME combinó la estructura CMS derivada de PKCS con MIME y X.509. Las normas S/MIME evolucionaron a través de varias generaciones hasta la versión 4.0 publicada en 2019; CMS está estandarizado en RFC 5652 ([RFC 1421](https://datatracker.ietf.org/doc/html/rfc1421), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652)).

OpenPGP surgió del formato de paquetes PGP y se estandarizó primero en RFC 2440 y después en RFC 4880. RFC 9580 sustituyó RFC 4880 en 2024 y modernizó, entre otros aspectos, las versiones de claves, los fingerprints y los procedimientos de cifrado autenticado. PGP/MIME según RFC 3156 sigue siendo la encapsulación para correo electrónico; sin embargo, la evolución del formato no elimina la tarea organizativa de regular la identidad de las claves y la recuperación ([RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580), [RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156)).

SMTP STARTTLS se estandarizó en 1999 en RFC 2487 y se actualizó en 2002 mediante RFC 3207. Posteriormente, DANE para SMTP, MTA-STS, TLS-RPT y REQUIRETLS añadieron capas de política e informes contra el downgrade y las configuraciones erróneas. Paralelamente, las empresas establecieron gateways y portales web porque los destinatarios externos no disponen de manera fiable de claves S/MIME u OpenPGP compatibles. Como resultado, el cifrado de correo electrónico pasó de ser una única función de cliente a convertirse en un sistema distribuido de SMTP, DNS, HTTPS, PKI, directorio, política, identidad web y recuperación ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

## Fuentes

- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [NCCoE – DNS-Based Email Security Practice Guide](https://www.nccoe.nist.gov/publication/1800-6/VolC/)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8689 – SMTP REQUIRETLS](https://datatracker.ietf.org/doc/html/rfc8689)
- [IETF RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460)
- [NIST – SP 800-177 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime)
- [Microsoft Learn – S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)
- [IETF RFC 3156 – MIME Security with OpenPGP](https://datatracker.ietf.org/doc/html/rfc3156)
- [SEPPmail – Productos](https://www.seppmail.com/products/)
- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [IETF RFC 6376 – DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [HIN Support – Mail a no miembros](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [IETF RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – página de manual de dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – página de manual de nc](https://man.openbsd.org/nc)
- [curl – página de manual de línea de comandos](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – certutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/)
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/)
- [GnuPG – manual de gpg](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – página de manual](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [IETF RFC 1421 – Privacy Enhancement for Internet Electronic Mail](https://datatracker.ietf.org/doc/html/rfc1421)
