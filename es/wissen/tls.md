---
title: "TLS: protocolo, validación de certificados y operación"
blatt: "tls"
description: "Transport Layer Security desde la perspectiva de la administración: Handshake y Record Layer, plan de claves de TLS 1.3, validación de rutas X.509 y PKIX, SAN y SNI, almacenes de confianza, integración con SMTP y LDAP, gestión de claves y diagnóstico por capas."
fakten:
  - label: Nombre completo
    wert: Transport Layer Security
    href: https://datatracker.ietf.org/doc/html/rfc9846
  - label: Objetivos de protección
    wert: Autenticación · confidencialidad · integridad
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Partes del protocolo
    wert: Handshake Protocol · Record Protocol · Alerts
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Base
    wert: flujo de bytes fiable y ordenado
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Autenticación
    wert: servidor siempre · cliente opcional
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Intercambio de claves
    wert: (EC)DHE · PSK · combinado
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2
  - label: Protección de registros
    wert: AEAD con Traffic Secrets separados
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-5
  - label: Perfil de certificado
    wert: X.509 v3 y validación de rutas PKIX
    href: https://datatracker.ietf.org/doc/html/rfc5280
  - label: Identidad del servicio
    wert: ID de referencia frente a subjectAltName
    href: https://datatracker.ietf.org/doc/html/rfc9525
  - label: Modelos de inicio
    wert: TLS implícito o STARTTLS específico del protocolo
    href: https://datatracker.ietf.org/doc/html/rfc8314
  - label: Reanudación
    wert: reanudación de sesión basada en PSK
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2.2
  - label: Recomendación operativa
    wert: BCP 195
    href: https://www.rfc-editor.org/info/bcp195
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - smtp-mailflow
translationSourceHash: 50ea371b919edcc4811a303d26ec6c575ef8dc334ce6e681fdc3f74de9ca2058
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:44:23.639Z
translationReview: required
---

# TLS: protocolo, validación de certificados y operación

**Transport Layer Security** (TLS) protege una conexión individual entre dos extremos. Primero, el cliente y el servidor acuerdan los procedimientos criptográficos y las claves; después, SMTP, LDAP, HTTP u otro protocolo de aplicación circulan por este canal protegido. TLS impide la escucha y las modificaciones inadvertidas en esta conexión, y autentica como mínimo al servidor. Sin embargo, no confirma el remitente de un correo electrónico ni garantiza que el siguiente salto SMTP también use TLS ([RFC 9846, sección 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1), [RFC 3207, sección 6](https://datatracker.ietf.org/doc/html/rfc3207#section-6)).

Por tanto, durante la operación deben superarse varias comprobaciones sucesivas: el host y el puerto correctos deben ser accesibles, el cliente y el servidor necesitan parámetros TLS comunes, la cadena de certificados debe conducir a un ancla de confianza y el nombre solicitado debe figurar en el certificado. Solo entonces comienza el protocolo de aplicación propiamente dicho. El artículo sigue exactamente este proceso y muestra en cada punto qué debe observar, configurar y proteger un administrador.

## Posición en la pila de protocolos

En la operación habitual de servidores, TLS se sitúa entre [TCP](/kb/tcp) y el protocolo de aplicación. TCP proporciona el flujo de datos ordenado; TLS lo protege; [SMTP](/kb/smtp), [LDAP](/kb/ldap), IMAP o HTTP determinan cuándo empieza TLS y qué nombre de servidor se espera. Esta integración explica por qué un puerto abierto aún no supone una prueba TLS satisfactoria y un Handshake correcto aún no demuestra que funcione un servicio de correo o de directorio ([RFC 9846, sección 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1)).

| Capa | Objeto visible | Pregunta típica de administración |
|---|---|---|
| Selección de nombre y servicio | Nombre DNS, destino MX o SRV, puerto | ¿Qué extremo se seleccionó realmente? |
| Transporte | Conexión TCP y máquina de estados | ¿Llegan SYN, SYN-ACK y ACK en ambas direcciones? |
| Handshake TLS | Versión, Key Share, firma, certificados, Finished | ¿Existen parámetros comunes y se acepta al par? |
| TLS Record Layer | Registros cifrados con protección AEAD | ¿Se transmiten, descartan o finalizan con Alert los registros? |
| Protocolo de aplicación | Diálogo SMTP, LDAP, IMAP o HTTP | ¿Qué comandos e identidades se aplican tras el Handshake? |

QUIC es una excepción importante a la imagen clásica de «TLS sobre TCP»: utiliza el Handshake de TLS 1.3, pero no el TLS Record Layer; QUIC protege sus propios paquetes con claves derivadas de TLS ([RFC 9001](https://datatracker.ietf.org/doc/html/rfc9001)). Para los protocolos de correo y directorio considerados aquí, el canal TLS basado en TCP sigue siendo el modelo relevante.

## Arquitectura: Handshake, Record Layer y Alerts

TLS realiza dos tareas consecutivas. En el **Handshake Protocol**, los extremos negocian parámetros, se autentican y generan material de claves compartido. Después, el **Record Protocol** divide los datos útiles en registros protegidos. Los **Alerts** notifican errores de protocolo o un cierre ordenado; no obstante, en caso de un fallo abrupto de red o proceso, la conexión TCP también puede terminar sin un Alert aprovechable ([RFC 9846, secciones 4 a 6](https://datatracker.ietf.org/doc/html/rfc9846#section-4)).

Esta separación es decisiva en la operación. Un error de certificado pertenece a la fase de autenticación del Handshake. Un `bad_record_mac` o un `decrypt_error` afecta al procesamiento criptográfico. En cambio, un timeout sin ServerHello suele indicar confusión de protocolos, filtrado, un puerto incorrecto o un servidor que primero espera un diálogo en texto claro y un comando STARTTLS.

## Handshake de TLS 1.3 paso a paso

El Handshake completo de TLS 1.3 puede leerse como una conversación: el cliente hace una oferta, el servidor elige entre ella, acredita su identidad y ambas partes confirman el protocolo realizado hasta ese momento. Ya tras el **ServerHello**, los demás mensajes de Handshake están protegidos. Con los mensajes **Finished** de ambas partes, queda finalmente establecido que ambas poseen la misma transcripción y los mismos secretos ([RFC 9846, sección 2](https://datatracker.ietf.org/doc/html/rfc9846#section-2)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-tls-handshake.svg?v=20260813" title="Interaktive Infografik: TLS-1.3-Handshake, Zertifikatskette, Identitätsprüfung und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tls-handshake.svg?v=20260813">Abrir infografía sobre el Handshake TLS y la validación de certificados</a>
</iframe>

El proceso en detalle:

1. El **ClientHello** ofrece, entre otros elementos, versiones TLS admitidas, Cipher Suites, algoritmos de firma y grupos criptográficos. Un `key_share` normalmente ya contiene la parte pública efímera del cliente. SNI puede transportar el nombre de servidor deseado y ALPN posibles protocolos de aplicación.
2. El **ServerHello** selecciona la versión, la Cipher Suite y el Key Share. A partir de las partes del cliente y del servidor se genera el secreto (EC)DHE compartido. Si no hay un grupo adecuado enviado previamente, el servidor puede solicitar un nuevo ClientHello mediante **HelloRetryRequest**.
3. **EncryptedExtensions** confirma otras extensiones negociadas. Si el servidor solicita un certificado de cliente, también sigue **CertificateRequest**.
4. **Certificate** transporta el certificado Leaf y normalmente los certificados Intermediate necesarios para el cliente. **CertificateVerify** firma la transcripción del Handshake con la clave privada correspondiente al certificado Leaf. De este modo, el servidor demuestra la posesión de la clave; el certificado por sí solo no la demuestra.
5. **Finished** es un MAC sobre la transcripción realizada hasta entonces y confirma los secretos de Handshake derivados. El cliente valida los parámetros, la ruta del certificado y la identidad del servicio, y después envía su Finished; en caso de autenticación mutua solicitada, también Certificate y CertificateVerify.
6. Para los datos de aplicación se derivan Traffic Secrets separados. Los mensajes **NewSessionTicket** posteriores permiten una reanudación basada en PSK sin que el servidor tenga que reenviar el certificado original en cada Handshake posterior.

Por ello, el número observado de Round Trips depende de DNS, TCP, un Handshake TLS completo o reanudado, HelloRetryRequest y la aplicación. Una captura de paquetes debe medir estas fases por separado; «TLS fue lento» no es un diagnóstico sin esta división.

## Plan de claves y Record Layer

A partir del material (EC)DHE o PSK acordado, TLS 1.3 no genera una única clave de sesión, sino varios secretos separados mediante **HKDF**. El Handshake, los datos de aplicación, las exportaciones y la reanudación posterior utilizan material propio; además, el cliente y el servidor poseen claves y vectores de inicialización separados para cada dirección. Esta división es importante para el análisis y la respuesta a incidentes, porque una clave de registro expuesta no descifra automáticamente todas las fases y direcciones ([RFC 9846, sección 7.1](https://datatracker.ietf.org/doc/html/rfc9846#section-7.1)).

El Record Layer utiliza exclusivamente **Authenticated Encryption with Associated Data** (AEAD). En TLS 1.3, la Cipher Suite designa el algoritmo AEAD y el hash para HKDF; el tipo de certificado, el algoritmo de firma y el grupo de intercambio de claves se negocian por separado. Esto difiere de TLS 1.2, donde nombres como `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` agrupan varias decisiones en una suite.

| Propiedad | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Semántica de Cipher Suite | Intercambio de claves, autenticación, cifrado y hash agrupados | Procedimiento AEAD y hash HKDF; intercambio de claves y firma separados |
| Forward Secrecy | depende de la suite elegida | intercambio de claves públicas solo con (EC)DHE efímero |
| Visibilidad del Handshake | grandes partes en texto claro | cifrado después de ServerHello |
| Reanudación | Session ID o ticket según el modelo anterior | modelo PSK uniforme |
| 0-RTT | no forma parte del protocolo | opcional, pero repetible y no adecuado para operaciones no idempotentes |

0-RTT ahorra un Round Trip en una reanudación, pero posee propiedades de replay más débiles. Por tanto, un servidor debe decidir según la aplicación qué datos tempranos acepta. Para operaciones administrativas, de correo o de directorio que modifican el estado, la baja latencia no debe confundirse con seguridad frente a replay ([RFC 9846, secciones 2.3 y 8](https://datatracker.ietf.org/doc/html/rfc9846#section-2.3)).

## Certificado X.509 y clave privada

Un certificado X.509 es la descripción firmada de una clave pública. Indica, entre otros datos, el emisor, el período de validez, los usos permitidos y los nombres para los que puede emplearse la clave. La clave privada correspondiente no forma parte del certificado y debe permanecer protegida por el operador. RFC 5280 define el perfil PKIX utilizado para ello, con campos como Key Usage, Extended Key Usage, Basic Constraints y Subject Alternative Name ([RFC 5280, sección 4](https://datatracker.ietf.org/doc/html/rfc5280#section-4)).

X.509 se codifica en binario con ASN.1 DER. Las líneas visibles habitualmente `BEGIN CERTIFICATE` y `END CERTIFICATE` son una envoltura textual basada en Base64 descrita por RFC 7468. En cambio, un contenedor **PKCS #12** puede transportar conjuntamente una clave privada, una cadena de certificados y otros atributos ([RFC 7468](https://datatracker.ietf.org/doc/html/rfc7468), [RFC 7292](https://datatracker.ietf.org/doc/html/rfc7292)). Las extensiones de archivo como `.cer`, `.crt`, `.pem`, `.p12` o `.pfx` son convenciones, pero no una prueba fiable del contenido o la codificación.

Las extensiones importantes tienen tareas diferentes:

| Extensión | Pregunta de comprobación | Error operativo frecuente |
|---|---|---|
| Subject Alternative Name | ¿Contiene la identidad DNS, IP, URI o SRV esperada por el cliente? | Falta el nombre de gestión o el alias del balanceador de carga |
| Basic Constraints | ¿Puede el certificado actuar como CA y cuál puede ser la profundidad de la ruta? | Intermediate sin `CA=TRUE` o Path Length incorrecto |
| Key Usage | ¿Para qué operación criptográfica básica puede utilizarse la clave? | La CA no puede firmar o el Leaf no se ajusta al uso de firma |
| Extended Key Usage | ¿Está permitida la clave para autenticación de servidor o cliente? | Un certificado S/MIME se usa erróneamente como certificado de servidor TLS |
| Authority / Subject Key Identifier | ¿Qué variantes de claves ayudan a construir la ruta? | Se elige un Intermediate incorrecto con el mismo Subject |
| CRL Distribution Points / AIA | ¿Dónde encuentra el cliente información de revocación o del emisor? | El destino no es accesible desde la red del servidor o falta un proxy |

## Ruta de certificados y límite de confianza

Con su **certificado Leaf**, un servidor normalmente envía los **certificados Intermediate** necesarios. El cliente intenta construir con ellos una ruta hacia una CA raíz en la que ya confía localmente. Precisamente este último punto es decisivo: una CA raíz entregada por el servidor no se vuelve fiable por ello. El Truststore del proceso que realmente establece la conexión determina qué anclas de confianza se aplican ([RFC 5280, sección 6](https://datatracker.ietf.org/doc/html/rfc5280#section-6)).

La construcción y validación de rutas es más que tres simples marcas de verificación. El cliente debe comprobar, entre otros, firmas, intervalos de validez, Basic Constraints, Path Length, Name Constraints, Key Usage, Extended Key Usage, extensiones críticas, políticas y, si procede, revocación. A continuación viene la identidad de servicio específica del protocolo. Distintos clientes pueden construir rutas válidas diferentes a partir del mismo certificado Leaf, porque difieren sus Truststores, descarga AIA, Cross-Signatures y reglas de algoritmos.

Por ello, el término **Truststore** no designa una ubicación universal. Una aplicación Windows puede usar Schannel y el almacén de certificados de Windows; una aplicación Java, JSSE con un archivo propio `cacerts` o de la aplicación; un contenedor, un paquete de CA integrado; y un appliance, un almacén propietario. Incluso en el mismo host, dos procesos pueden confiar de manera diferente en el mismo servidor. Ante cada incidencia, debe identificarse el proceso que realmente valida y su Truststore.

La revocación es una decisión de política adicional. Las CRL distribuyen listas firmadas de certificados revocados; OCSP responde consultas de estado para certificados individuales. Que un cliente compruebe, cómo reaccione ante un timeout y si utiliza OCSP Stapling depende de la biblioteca y de la aplicación ([RFC 5280, sección 5](https://datatracker.ietf.org/doc/html/rfc5280#section-5), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)). «El certificado es válido temporalmente» no significa automáticamente «el certificado será aceptado por este cliente».

## Identidad del servicio: SAN, ID de referencia y SNI

Una vez aceptada la ruta, el cliente comprueba el nombre del servicio. El punto de partida es la **identidad de referencia**, normalmente el nombre de host configurado o la parte de host de una URL. Debe coincidir con una entrada del tipo adecuado en el **Subject Alternative Name** del certificado Leaf. Para nombres DNS se aplica `dNSName`, para direcciones IP configuradas directamente `iPAddress`; el Common Name formulado libremente no es una solución alternativa moderna. Los comodines solo pueden sustituir la etiqueta completa situada más a la izquierda ([RFC 9525, secciones 4 y 6](https://datatracker.ietf.org/doc/html/rfc9525#section-4)).

**Server Name Indication** (SNI) debe distinguirse de ello. El cliente envía el nombre de servidor deseado en el ClientHello para que un listener compartido pueda seleccionar el certificado y la configuración virtual correctos. SNI indica al servidor lo que el cliente quiere alcanzar; no sustituye ni la comparación SAN ni la validación de la ruta ([RFC 6066, sección 3](https://datatracker.ietf.org/doc/html/rfc6066#section-3)). Por ello, una prueba sin SNI puede mostrar el certificado predeterminado, aunque el cliente de producción reciba otro.

**Application-Layer Protocol Negotiation** (ALPN) permite al cliente y al servidor seleccionar un protocolo superior dentro del Handshake TLS. Es especialmente visible para HTTP y QUIC, pero conceptualmente tampoco es una validación de certificados ([RFC 7301](https://datatracker.ietf.org/doc/html/rfc7301)).

## TLS implícito y STARTTLS

El protocolo de aplicación determina cuándo comienza el Handshake. Con **TLS implícito**, el ClientHello sigue inmediatamente al establecimiento de TCP. Con **STARTTLS**, ambas partes primero hablan en texto claro y solo cambian a TLS tras un comando específico del protocolo. Por tanto, quien envía un ClientHello directamente a un puerto STARTTLS utiliza TLS, pero en el punto incorrecto del diálogo.

| Ruta de servicio | Puerto | Modelo de inicio | Regla de protocolo |
|---|---:|---|---|
| Relay SMTP entre MTA | 25 | ESMTP `STARTTLS` | tras un Handshake correcto se reinicia SMTP; el cliente vuelve a enviar EHLO |
| Envío de mensajes | 587 | STARTTLS | Submission puede exigir TLS antes de la autenticación |
| Envío con TLS implícito | 465 | TLS inmediato | descrito por RFC 8314 para Mail Submission |
| IMAP / IMAPS | 143 / 993 | STARTTLS / TLS inmediato | el acceso en texto claro se considera obsoleto para uso moderno |
| POP3 / POP3S | 110 / 995 | STLS / TLS inmediato | también rutas separadas de texto claro y TLS implícito |
| LDAP / LDAPS | 389 / 636 | LDAP StartTLS Extended Operation / TLS inmediato | Bind y validación de identidad siguen la política de seguridad LDAP |
| HTTP / HTTPS | 80 / 443 | esquemas URI separados; TLS inmediato en 443 | Host/SNI, identidad del certificado y versión HTTP son decisiones separadas |

Las asignaciones de puertos proceden de la [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); la semántica deriva del estándar de aplicación correspondiente. Por ello, el puerto 465 no es de forma general «SMTPS para entrega de servidor a servidor», sino la ruta TLS implícita registrada para Submission.

SMTP STARTTLS define un cambio de estado especialmente importante. Antes de la actualización, la respuesta EHLO y la oferta `STARTTLS` son manipulables. Tras el Handshake, ambas partes deben descartar los conocimientos SMTP aprendidos previamente, y el cliente vuelve a enviar EHLO ([RFC 3207, sección 4.2](https://datatracker.ietf.org/doc/html/rfc3207#section-4.2)). Para LDAP, StartTLS es una Extended Operation dentro de la sesión LDAP; no equivale a un nuevo puerto TCP ([RFC 4511, sección 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

## Política TLS en SMTP

En SMTP por el puerto 25, TLS suele ser oportunista: el MTA emisor cifra si el extremo remoto ofrece STARTTLS y la política local acepta el Handshake. Sin una política adicional, un atacante puede eliminar la oferta o interrumpir el Handshake. Entonces, el MTA emisor debe decidir si entrega sin cifrar, vuelve a intentarlo más tarde o aborta definitivamente. RFC 3207 deja así claro que un salto SMTP protegido no es una protección de extremo a extremo del mensaje.

Dos mecanismos pueden vincular más estrechamente el dominio receptor y el requisito TLS:

- **DANE for SMTP** publica datos TLSA bajo DNSSEC y deriva la identidad que debe comprobarse de la ruta MX. De este modo, un resolver validador y una cadena DNSSEC intacta pasan a formar parte de la base de confianza ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).
- **MTA-STS** publica una política mediante DNS y HTTPS. Indica patrones MX permitidos y exige TLS validado mediante PKIX durante la vigencia de la política ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)). **SMTP TLS Reporting** informa patrones agregados de éxito y error de estas políticas ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)).

Estos modelos no son intercambiables. DANE ancla la afirmación en DNSSEC y TLSA; MTA-STS utiliza PKIX web y una política obtenida por HTTPS. Por ello, además del certificado y la Cipher, un administrador debe supervisar [DNS](/kb/dns), la selección MX, la caché de políticas y el canal de informes.

## Autenticación TLS mutua

**Mutual TLS** (mTLS) complementa la prueba normal del servidor con un certificado de cliente. El servidor lo solicita mediante CertificateRequest; el cliente envía su cadena y demuestra mediante CertificateVerify que posee la clave privada. Después, el servidor comprueba la ruta, la validez, Extended Key Usage y su asignación local. El certificado identifica en primer lugar a un titular de clave. La aplicación sigue decidiendo qué derechos de cuenta, LDAP o relay SMTP se derivan de ello.

En entornos heterogéneos aparecen con frecuencia problemas de confianza asimétricos: el cliente confía en la CA del servidor, pero el servidor no confía en la CA del cliente; el servidor solo solicita determinados Issuers; el certificado de cliente posee únicamente `serverAuth`; o un balanceador de carga termina TLS y no transmite una identidad de cliente verificada al backend. Por tanto, un Handshake con autenticación de servidor correcto no prueba que mTLS funcione.

## Operación de claves, certificados y renovaciones

Un cambio de certificado es más que sustituir un archivo. La clave, el certificado, la cadena Intermediate, la vinculación al listener y el estado de despliegue deben coincidir. Un ciclo de vida sólido comprende al menos:

1. **Inventario:** registrar nombre del servicio, puerto, modelo de inicio, proceso que termina TLS, SAN esperados, Issuer, número de serie, fingerprint, `notBefore`/`notAfter`, Key Store, Truststore y automatización responsable.
2. **Generación de claves:** generar un nuevo par de claves en el contexto previsto de software, TPM o HSM. Las claves privadas deben permanecer confidenciales y requieren protección, control de acceso y, según el modelo de recuperación, una estrategia documentada de copia de seguridad o reemisión ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).
3. **Solicitud y emisión:** generar CSR con las identidades y parámetros correctos; cumplir la política de la CA y las pruebas requeridas. ACME estandariza la solicitud y renovación automatizadas, pero aún no garantiza una vinculación correcta al servicio ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)).
4. **Instalación:** llevar el Leaf, el bundle Intermediate correspondiente y la clave privada al formato que realmente lee el proceso. Comprobar permisos de archivo, acceso mediante contraseña y coincidencia del par de claves.
5. **Activación:** recargar el servicio o cambiar la vinculación de forma atómica. En clústeres, balanceadores de carga y múltiples roles de protocolo, tratar cada terminador TLS por separado.
6. **Validación:** comprobar externa e internamente con SNI, puerto y modo STARTTLS de producción; registrar cadena entregada, SAN, validez, algoritmo, política y prueba de aplicación.
7. **Sustitución:** retirar el certificado antiguo solo cuando todos los nodos entreguen el nuevo y no quede ninguna dependencia del fingerprint o Issuer anterior. En caso de compromiso, revocarlo adicionalmente y tratar la clave privada como comprometida.

Una copia de seguridad debe ajustarse a este modelo. Una parte pública de certificado exportada no restaura un servicio si faltan la clave privada, la contraseña, la vinculación de proveedor, la ACL o el objeto HSM. A la inversa, cada copia sin protección de la clave privada aumenta la superficie de ataque. La recuperación de CA raíz y emisora, la recuperación de claves de servidor y la reemisión rápida son procedimientos distintos y no deben agruparse bajo «hacer copia de seguridad de certificados».

La PKI web pública, la PKI empresarial interna y las anclas de confianza autogestionadas tienen reglas distintas de emisión y vigencia. En lugar de utilizar un valor máximo estático como objetivo operativo, el monitoreo debería medir el `notAfter` concreto de cada certificado entregado y prever suficiente tiempo para emisión, despliegue, validación y rollback.

## Pila de implementación y tecnología

La pila TLS que realmente opera depende del proceso. Los servicios Windows pueden utilizar Schannel, las aplicaciones Java JSSE, los servicios Unix OpenSSL o GnuTLS y los appliances una biblioteca propia. Con ello cambian el Truststore, los procedimientos admitidos, las herramientas de diagnóstico y la ubicación de la configuración, aunque todos los extremos hablen «TLS».

| Pila | Tiempo de ejecución y API | Fuente típica de confianza y claves | Relevancia administrativa |
|---|---|---|---|
| [OpenSSL](https://docs.openssl.org/master/) ([Fuentes](https://github.com/openssl/openssl)) | biblioteca nativa y CLI, principalmente C | archivo/directorio de CA, PEM, DER, PKCS #12, Provider | muchos servicios Unix y appliances; `s_client`, `x509`, `verify` |
| [Schannel](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) | Windows Security Support Provider mediante SSPI | almacén de certificados de Windows, claves CNG/CSP | las aplicaciones de sistema y .NET pueden usar la política y los almacenes del SO |
| [JSSE](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) | API de Java `SSLSocket`, `SSLEngine`, `SSLContext` y Provider | KeyStore, TrustStore, `cacerts`, PKCS #12 | una aplicación JVM puede diferir de la confianza del sistema operativo |
| [GnuTLS](https://www.gnutls.org/documentation.html) | biblioteca TLS nativa, API C y CLI | archivos, PKCS #11, confianza del sistema según la integración | biblioteca alternativa en software Unix y GNU |

Por encima se encuentra la configuración de la aplicación. Un servidor de correo puede seleccionar certificado y política por listener o nombre SNI; un proxy inverso termina TLS antes del backend; un producto Java incluye su propio JRE con Truststore; una imagen de contenedor congela un paquete CA en el momento de la compilación. Por ello, «el certificado está instalado» a nivel de sistema operativo no es una afirmación sólida sin probar el proceso y la vinculación.

El soporte de algoritmos también es una intersección entre versión de protocolo, biblioteca, proveedor criptográfico, certificado, oferta del cliente y política local. Un certificado ECDSA no ayuda si el cliente no admite la firma; una cadena Cipher de TLS 1.3 no modifica las suites de TLS 1.2; una configuración Schannel de todo el sistema no controla un tiempo de ejecución OpenSSL o Java incluido por separado. [BCP 195](https://www.rfc-editor.org/info/bcp195) proporciona la guía de seguridad de la IETF, pero la activación concreta sigue dependiendo de la implementación y del ciclo de vida.

## Diagnóstico desde el puerto hasta la aplicación

Por ello, un diagnóstico TLS no empieza por el certificado, sino por la ruta de conexión exacta. Cada prueba debe incluir nombre de destino, dirección resuelta, puerto, SNI, modo STARTTLS, biblioteca cliente, Truststore, momento y política aplicable. Solo las pruebas con los mismos parámetros son comparables entre sí.

### Comprobar por separado la accesibilidad TCP

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP-Erreichbarkeit vor TLS">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) solo comprueban aquí la resolución de nombres y el establecimiento TCP. Una prueba satisfactoria aún no dice nada sobre versión TLS, certificado, SNI o aplicación.

### Probar SMTP STARTTLS con SNI de producción

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) es un cliente de diagnóstico, no una prueba de la política del MTA de producción. `-starttls smtp` realiza el diálogo en texto claro hasta la actualización, `-servername` establece SNI, `-showcerts` muestra la lista de certificados enviada por el servidor y `-verify_return_error` termina ante un error de validación. Para una prueba de identidad completa también debe indicarse `-verify_hostname mail.example.ch` y la ruta CA realmente utilizada. OpenSSL no forma parte integrada del sistema en Windows; `openssl.exe` debe proceder de una instalación controlada.

### Distinguir LDAP StartTLS y TLS implícito

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

La primera llamada espera TLS inmediatamente en el puerto 636; la segunda ejecuta la LDAP StartTLS Extended Operation en el puerto 389. El éxito de la prueba OpenSSL aún no prueba que la biblioteca LDAP de producción utilice el mismo Truststore, la misma política de revocación o la misma identidad de referencia.

### Examinar localmente el archivo de certificado y la ruta

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`certutil`](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) utiliza la lógica de certificados de Windows para `-verify` y puede incluir un nombre de servidor con `-sslpolicy`. [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) muestra campos de un certificado; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) construye y comprueba una ruta frente a los Trust Anchors e Intermediates indicados explícitamente. [`keytool`](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) es la interfaz oficial adecuada de administración e inspección para Key Stores y Truststores de JVM; una prueba con OpenSSL no sustituye su perspectiva.

### Asignar el síntoma de error a una fase de comprobación

| Observación | Capa probable | Siguiente comprobación |
|---|---|---|
| Timeout TCP | DNS, ruta, firewall o listener | dirección resuelta, SYN/SYN-ACK, ruta de retorno, vinculación del listener |
| Banner de texto claro en vez de respuesta TLS | modelo de inicio incorrecto | puerto y `-starttls smtp` o `-starttls ldap` |
| Alert TLS inmediato tras ClientHello | sin parámetros comunes o política | versiones, grupos, algoritmos de firma, Cipher, SNI |
| certificado incorrecto | SNI o vinculación del listener | SNI enviado, VHost predeterminado, configuración del balanceador de carga |
| `unable to get local issuer certificate` | Intermediate o Trust Anchor ausente | lista del servidor, AIA, Truststore del cliente, construcción de ruta |
| `hostname mismatch` | la ID de referencia no coincide con SAN | nombre utilizado realmente, tipo SAN, alias y límite de comodín |
| solo falla la aplicación Java | política JSSE o Truststore propio | JVM, `cacerts`, Truststore de la aplicación, disabledAlgorithms |
| solo algunos nodos del clúster entregan el certificado antiguo | despliegue o vinculación | probar cada extremo y cada IP con el mismo SNI |
| SMTP vuelve ocasionalmente a texto claro | política oportunista o downgrade | registros MTA, oferta STARTTLS, estado MTA-STS/DANE, TLS-RPT |
| el Handshake mTLS tiene éxito, pero el acceso sigue prohibido | autorización posterior a TLS | EKU de cliente, mapeo de certificados, reenvío por proxy, rol de aplicación |

[`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) puede hacer visibles ClientHello, ServerHello, extensiones y Alerts; sin claves de sesión, los datos de aplicación y muchos mensajes de Handshake TLS 1.3 permanecen cifrados. Además, una captura no muestra automáticamente qué Truststore utiliza la aplicación ni por qué su política rechaza una ruta.

## Monitoreo y criterios operativos

Una comprobación de puerto y una alerta antes de la fecha de vencimiento no son suficientes. El monitoreo debería comprobar la misma ruta que un cliente real: seleccionar destino, establecer el Handshake, validar certificado y nombre y, después, alcanzar el protocolo de aplicación. En particular, son útiles las siguientes señales:

- selección de destino y SNI por servicio, listener y nodo de clúster;
- protocolo negociado, Cipher Suite, grupo y algoritmo de firma;
- fingerprint Leaf, número de serie, SAN, Issuer y `notBefore` y `notAfter`;
- cadena Intermediate realmente enviada y validación de ruta desde la perspectiva de los Truststores relevantes;
- accesibilidad OCSP/CRL y comportamiento real de fail-open/fail-closed;
- latencia del Handshake, proporción de reanudación, clases de Alert y tasa de errores;
- oferta STARTTLS, actualizaciones correctas y retornos a texto claro por destino SMTP;
- estado de MTA-STS, DANE y TLS-RPT donde se utilizan estos mecanismos de política;
- uniformidad de todos los terminadores TLS tras la renovación y prueba satisfactoria de la aplicación detrás de la capa TLS.

El monitoreo solo desde dentro no detecta la cadena pública, el DNS externo o los terminadores de borde; una comprobación externa por sí sola no ve los Truststores internos ni mTLS. Por ello, para plataformas de mensajería y directorio se requiere como mínimo una ruta de cliente externa, una ruta de aplicación interna y una comprobación directa de cada terminador.

## Historia técnica

La historia comienza con SSL de Netscape en la década de 1990. SSL 2.0 fue posteriormente prohibido expresamente ([RFC 6176](https://datatracker.ietf.org/doc/html/rfc6176)); SSL 3.0 solo está documentado como protocolo histórico y no está permitido desde RFC 7568. A partir de ahí, la IETF desarrolló en 1999 **TLS 1.0**, en 2006 **TLS 1.1** y en 2008 **TLS 1.2**. RFC 8996 declaró obsoletos TLS 1.0 y 1.1 en 2021. Para los administradores, esta procedencia explica por qué los nombres de productos antiguos, la sintaxis Cipher y los conmutadores de compatibilidad siguen apareciendo en las configuraciones.

**TLS 1.3** apareció en 2018 con RFC 8446. Eliminó el intercambio de claves RSA estático, los procedimientos no AEAD antiguos y otras funciones heredadas, cifró antes el Handshake e introdujo el plan de claves basado en HKDF y el modelo de reanudación PSK. RFC 9846 sustituyó RFC 8446 en julio de 2026 como especificación consolidada de TLS 1.3 e integró correcciones y requisitos documentados hasta entonces por separado. El valor de protocolo `1.3` se mantuvo igual; cambió el documento normativo estándar ([RFC 9846, sección 1.2](https://datatracker.ietf.org/doc/html/rfc9846#section-1.2)).

Paralelamente evolucionaron la identidad y la operación: X.509-PKIX se consolidó en RFC 5280, SNI y otras extensiones en RFC 6066, ALPN en RFC 7301, la emisión automatizada en ACME/RFC 8555 y la validación moderna de identidad de servicio en RFC 9525. Por tanto, la historia de TLS no es solo una secuencia de versiones, sino el desarrollo conjunto de criptografía, rutas de certificados, integración de aplicaciones y automatización operativa.

## Fuentes

- [IETF RFC 8314 – Cleartext Considered Obsolete](https://datatracker.ietf.org/doc/html/rfc8314)
- [RFC 9846 – The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc9846) – Handshake, Record Layer, plan de claves, reanudación y modelo de seguridad.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS, cambio de estado SMTP y política oportunista.
- [RFC 9001 – Using TLS to Secure QUIC](https://datatracker.ietf.org/doc/html/rfc9001) – delimitación entre Handshake TLS y protección de paquetes QUIC.
- [RFC 5280 – Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280) – campos de certificado, extensiones y validación de rutas.
- [RFC 7468 – Textual Encodings of PKIX, PKCS, and CMS Structures](https://datatracker.ietf.org/doc/html/rfc7468) – envoltura textual PEM.
- [RFC 7292 – PKCS #12](https://datatracker.ietf.org/doc/html/rfc7292) – contenedor para claves y certificados.
- [RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960) – comprobación de estado OCSP.
- [RFC 9525 – Service Identity in TLS](https://datatracker.ietf.org/doc/html/rfc9525) – identidades de referencia, subjectAltName, comodines y validación de nombres.
- [RFC 6066, sección 3](https://datatracker.ietf.org/doc/html/rfc6066)
- [RFC 7301 – TLS Application-Layer Protocol Negotiation](https://datatracker.ietf.org/doc/html/rfc7301) – negociación ALPN.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – puertos y nombres de servicio registrados.
- [RFC 4511, sección 4.14](https://datatracker.ietf.org/doc/html/rfc4511)
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – autenticación SMTP basada en DNSSEC/TLSA.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – política MTA-STS y comprobación MX.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – informes agregados de errores TLS.
- [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) – gestión de claves y necesidades de protección.
- [RFC 8555 – Automatic Certificate Management Environment](https://datatracker.ietf.org/doc/html/rfc8555) – emisión y renovación automatizadas.
- [OpenSSL](https://docs.openssl.org/master/)
- [Fuentes](https://github.com/openssl/openssl)
- [Microsoft Learn – Schannel SSP](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) – implementación TLS de Windows y clasificación SSPI.
- [Oracle – JSSE Reference Guide](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) – API TLS de Java, KeyManager y TrustManager.
- [GnuTLS Documentation](https://www.gnutls.org/documentation.html) – biblioteca TLS nativa alternativa y familia de herramientas.
- [BCP 195 – Recommendations for Secure Use of TLS and DTLS](https://www.rfc-editor.org/info/bcp195) – guía de seguridad de la IETF mantenida para configuración TLS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – accesibilidad TCP en Windows.
- [OpenBSD Manual – nc](https://man.openbsd.org/nc) – prueba de conexión TCP en Unix.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – cliente genérico de diagnóstico TLS y STARTTLS.
- [Microsoft Learn – certutil](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) – almacén de certificados de Windows y Chain Engine.
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/) – campos de certificados y opciones de salida.
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/) – construcción y validación explícitas de rutas.
- [Oracle – keytool](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) – administración de Key Stores y Truststores Java.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – análisis de Handshake, Record y Session Key.
- [RFC 6176 – Prohibiting SSL Version 2.0](https://datatracker.ietf.org/doc/html/rfc6176) – retirada formal de SSL 2.0.
