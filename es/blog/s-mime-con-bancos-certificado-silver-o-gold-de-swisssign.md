---
title: "S/MIME para bancos: ¿certificado Silver o Gold de SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "Un banco que firma y cifra correos electrónicos de clientes con S/MIME necesita certificados Gold para buzones personales y certificados con validación de organización para buzones funcionales. Silver cifra con la misma seguridad, pero no identifica al banco como remitente. Qué comprueba SwissSign, qué figura en el certificado y qué requisitos se aplican en Suiza."
date: "2026-11-16"
kategorie: "Cifrado de correo electrónico"
timeToRead: "9 min de lectura"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-con-bancos-certificado-silver-o-gold-de-swisssign"
translationId: "article-5c2fd8d42f088dcb"
translationOf: smime-silver-gold-schweiz
translationSourceHash: f95529dd7c72d4d783714affbcf18d96e37adf66a52379b94212034dda610b74
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:48:57.431Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/s-mime-con-bancos-certificado-silver-o-gold-de-swisssign
---

# S/MIME para bancos: ¿certificado Silver o Gold de SwissSign?

Conclusión breve: Un banco que firma y cifra con S/MIME correos electrónicos dirigidos a clientes, clientas y socios comerciales debería equipar los buzones personales con certificados Gold y los buzones funcionales con certificados con validación de organización de SwissSign. Silver cifra con la misma seguridad, pero en el certificado solo figura la dirección de correo electrónico, no el nombre del banco. Ni la ley ni la FINMA imponen un nivel de certificado.

La cuestión de qué nivel de certificado se necesita para el envío de correo surge regularmente en mis proyectos con entidades financieras. Suele plantearse al inicio de un proyecto de gateway o cuando clientes empresariales exigen que los documentos confidenciales se envíen únicamente cifrados. Para responderla, hay que distinguir dos funciones de S/MIME: el cifrado, que protege el contenido, y la firma, que acredita ante el destinatario quién ha enviado el correo.

**Enfoque en Suiza:** Este artículo aborda la situación jurídica en Suiza. Cuando las normas de la UE difieren, se incluye una indicación identificada.

## Qué comprueba SwissSign en Silver y Gold

Desde el 1 de septiembre de 2023, para los certificados S/MIME de confianza pública se aplican los `S/MIME Baseline Requirements` del CA/Browser Forum. Estos distinguen cuatro tipos de validación, según lo que la entidad certificadora compruebe antes de la emisión. SwissSign asigna sus productos a estos tipos en los perfiles de certificados publicados (CPR S/MIME):

| Producto | Tipo según CA/B Forum | Qué se comprueba | Qué figura en el certificado |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control del buzón | solo la dirección de correo electrónico |
| E-Mail ID Gold | `Sponsor-validated` | Organización y persona | nombre y apellidos (o seudónimo), organización, identificador de la organización, país, dirección de correo electrónico |
| Organization Validated | `Organization-validated` | Organización | nombre de la organización, sin persona |

En Silver, SwissSign envía un código de confirmación a la dirección solicitada. Quien pueda recibir el correo obtiene el certificado. No se comprueba la identidad de la persona ni la organización, y la dirección también figura en el certificado como `Common Name`.

Gold figura en SwissSign exclusivamente como `Sponsor-validated`. SwissSign comprueba la organización y la persona está vinculada a ella. La identidad de esta persona la comprueba SwissSign o la propia organización como entidad de registro, normalmente en el marco de una PKI gestionada. Para buzones funcionales como `kundendienst@` o `zahlungsverkehr@`, está previsto el perfil `Organization-validated`; en él figura el banco como organización en el certificado, pero no una persona.

Criptográficamente, los niveles no se diferencian. Todos los perfiles utilizan claves RSA de al menos 2048 bits, el uso de clave `digitalSignature` y `keyEncipherment` y el uso ampliado `id-kp-emailProtection`.

## Qué significa la diferencia para un banco como remitente

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Cifrar y descifrar correos electrónicos | sí | sí | sí |
| La firma muestra en el programa de correo | dirección de correo electrónico | nombre y banco | banco |
| Lo que acredita la firma | el remitente tiene acceso al buzón | persona verificada de este banco | buzón de este banco |
| Uso en un banco | no apto para la comunicación con clientes | buzones personales | buzones funcionales |

La diferencia afecta a la firma. Un banco firma los correos salientes para que los destinatarios puedan comprobar que el mensaje procede realmente de él y que no se ha modificado durante el envío. Con Silver, esto solo es posible de forma limitada:

- Quien registre un dominio de nombre similar, por ejemplo `musterbank-ch.com` en lugar de `musterbank.ch`, obtiene un certificado Silver para direcciones de ese dominio sin más comprobaciones. Sus correos de phishing quedan entonces firmados de forma tan válida como los correos auténticos del banco, y en ambos casos el certificado solo muestra una dirección de correo electrónico.
- Quien se haga con un buzón del banco también puede solicitar un certificado Silver para esa dirección.

Un certificado Gold o con validación de organización contiene el nombre verificado del banco en el campo `Organization`. Solo recibe esta entrada quien supere la validación de organización de SwissSign. Para los destinatarios que consultan la firma, esta es la prueba relevante.

Para el cifrado, el nivel no importa. Un correo cifrado con Silver está tan bien protegido como uno cifrado con Gold.

## El cifrado requiere certificados de los destinatarios

Para cifrar un correo con S/MIME, el banco necesita la clave pública del destinatario. El certificado propio del banco no se utiliza para ello; sirve para la firma y para recibir respuestas cifradas.

La mayoría de los clientes particulares no tienen un certificado S/MIME. Por ello, los bancos suelen entregarles los correos confidenciales mediante un gateway de cifrado, como SEPPmail (GINA) o Kiteworks/Totemomail, que proporciona el mensaje en un portal web o como adjunto cifrado. El cifrado S/MIME se utiliza principalmente con clientes empresariales y socios que operan sus propios certificados o un gateway. Si ambas partes trabajan con un gateway, a menudo bastan certificados de dominio o TLS forzado entre los servidores de correo.

## Certificados en funcionamiento: PKI gestionada y gateway

Cuando hay más que un puñado de buzones, un banco obtiene sus certificados a través de la PKI gestionada de SwissSign. En este caso, el banco actúa como entidad de registro y confirma por sí mismo la identidad de los empleados; esto corresponde al tipo `Sponsor-validated`. Gateways como SEPPmail y Kiteworks/Totemomail pueden solicitar, renovar y asignar automáticamente certificados a los buzones mediante una interfaz con la PKI gestionada.

De los proyectos se conocen tres aspectos que causan problemas durante la operación:

- Las solicitudes Gold para buzones funcionales fallan porque el gateway toma el nombre y los apellidos del directorio, y SwissSign rechaza términos genéricos como «Support» o «Helpdesk» en el campo `GivenName`. Para estos buzones, puede configurarse el perfil `Organization-validated` en el gateway.
- Las renovaciones automáticas pueden fallar durante semanas sin que se detecte, hasta que expiran los primeros certificados y los clientes ya no pueden enviar correos cifrados. Por ello, deben supervisarse las fechas de vencimiento de los certificados emitidos y los mensajes de error de los procesos de renovación.
- El gateway firma sus solicitudes a la PKI gestionada con un certificado propio de la entidad de registro. Si este caduca, se detiene toda la automatización.

## Requisitos legales en Suiza

Ninguna ley suiza prescribe S/MIME y ninguna distingue entre niveles de certificado. Las obligaciones de un banco se refieren a la protección de los datos de los clientes; el banco decide por sí mismo cómo implementarla.

| Normativa | Contenido | Importancia para el envío de correo del banco |
|---|---|---|
| Ley de Bancos (BankG), art. 47 | Secreto bancario | Los datos de clientes no pueden revelarse sin autorización; el envío sin cifrar por Internet es un riesgo que el banco debe evaluar |
| Circular FINMA 2023/1 | Protección de datos críticos | Protección también durante la transmisión, neutral desde el punto de vista tecnológico, sin requisito de certificados |
| DSG, art. 8, y DSV, arts. 1 a 6 | seguridad de los datos adecuada al riesgo | El cifrado es un medio habitual; no se prescribe un procedimiento específico |
| ZertES y CO, art. 14, apdo. 2bis | solo la firma electrónica cualificada equivale a la firma manuscrita | Las firmas S/MIME no sustituyen una firma |

### Secreto bancario y FINMA

El secreto bancario conforme al art. 47 de la Ley de Bancos obliga al banco, a sus órganos y a sus empleados a no revelar sin autorización los datos de los clientes. La circular 2023/1 de la FINMA «Riesgos operacionales y resiliencia: bancos» exige que los bancos protejan sus datos críticos durante todo el ciclo de vida, también durante la transmisión. El banco establece cómo hacerlo en sus propias directrices. La elección del nivel de certificado forma, por tanto, parte de la política interna y debería estar documentada en ella.

### Protección de datos: DSG y DSV

El art. 8 de la DSG obliga a los responsables a garantizar una seguridad de los datos adecuada al riesgo. La ordenanza de protección de datos lo concreta mediante objetivos de protección como la confidencialidad y la integridad, pero no menciona procedimientos. S/MIME, un gateway con entrega mediante webmail o una plataforma de intercambio cumplen por igual este requisito, siempre que se utilicen de forma adecuada al riesgo.

### Firma electrónica: ZertES y CO

La ZertES distingue entre firma electrónica, firma electrónica avanzada, firma electrónica regulada y firma electrónica cualificada. Conforme al art. 14, apdo. 2bis, del CO, solo la firma electrónica cualificada con sello de tiempo cualificado equivale a la firma manuscrita. Los certificados S/MIME de SwissSign se emiten conforme a las normas del CA/Browser Forum y a las políticas ETSI LCP (Silver) y NCP (Gold), respectivamente, y no son certificados cualificados según la ZertES. Los contratos o declaraciones para los que se prescribe la forma escrita no pueden presentarse válidamente mediante un correo firmado con S/MIME, ni siquiera con Gold.

**Nota sobre la UE:** En la UE, conforme al art. 25, apdo. 2, de eIDAS, también solo la firma electrónica cualificada equivale a la firma manuscrita.

## Recomendación

- **Buzones personales** de asesores de clientes y otros empleados con contacto con clientes: Gold (`Sponsor-validated`) mediante la PKI gestionada.
- **Buzones funcionales:** `Organization-validated`, para que el banco figure como remitente en el certificado.
- No utilizar **Silver** para la comunicación con clientes, porque la firma no identifica al banco como remitente.
- **Cifrado para clientes particulares** mediante entrega push o por portal del gateway; S/MIME para destinatarios con certificado o gateway propio.
- **La firma y la verificación del remitente** se complementan: la firma S/MIME no sustituye SPF, DKIM ni DMARC, pero muestra al destinatario la organización verificada en el programa de correo.
- **Ficha informativa para clientes:** Describa qué certificados utiliza el banco, cómo pueden los clientes verificar una firma y qué niveles de certificado acepta el banco de clientes que quieran enviar correos cifrados por sí mismos. Si no existe dicha ficha, la pregunta llega al asesoramiento al cliente, que normalmente no puede responderla.

## Fuentes

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): perfiles de certificados de SwissSign (a 18/09/2025), asignación de Silver, Gold y Organization Validated a los tipos de validación, campos Subject y OID de políticas.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): normas vinculantes para certificados S/MIME de confianza pública, definición de los cuatro tipos de validación.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): conexión del gateway SEPPmail a la PKI gestionada de SwissSign.

4.  [Ley de Bancos (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sobre el secreto bancario.

5.  [FINMA: Circular](https://www.finma.ch/de/dokumentation/rundschreiben/): resumen que incluye la circular 2023/1 «Riesgos operacionales y resiliencia: bancos», en vigor desde el 1 de enero de 2024.

6.  [Ley Federal de Protección de Datos (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sobre la seguridad de los datos.

7.  [Ordenanza de Protección de Datos (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): arts. 1 a 6 con principios y objetivos de protección de la seguridad de los datos.

8.  [Ley Federal sobre la Firma Electrónica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipos de firma electrónica y requisitos para certificados cualificados.

9.  [Código de Obligaciones (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14, apdo. 2bis, sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita.
