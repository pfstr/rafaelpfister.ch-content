---
title: "S/MIME con bancos: ¿certificado Silver o Gold de SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "Para los correos cifrados con un banco, un certificado Silver de SwissSign es técnicamente suficiente. Gold también confirma el nombre y la organización del remitente. Qué comprueba cada nivel, qué figura en el certificado y qué requisitos legales se aplican en Suiza."
date: "2026-11-16"
kategorie: "Cifrado de correo electrónico"
timeToRead: "8 min de lectura"
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
url: https://rafaelpfister.ch/es/blog/s-mime-con-bancos-certificado-silver-o-gold-de-swisssign
translationSourceHash: cbcb890de323f2b9db40a5682192296524250e7bc8c23a67c6f643bcba605a14
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:04:37.148Z
translationReview: automatic
---

# S/MIME con bancos: ¿certificado Silver o Gold de SwissSign?

Conclusión breve: ni una ley suiza ni la FINMA prescribe un nivel de certificado específico para S/MIME. Para cifrar, un certificado Silver es técnicamente plenamente suficiente. Gold también confirma el nombre y la organización del remitente, y es la elección adecuada si el banco no establece ningún requisito.

La cuestión de si se necesita un certificado Silver o Gold de SwissSign para el tráfico de correo con un banco surge regularmente en mis proyectos. Por lo general, el banco no dispone de una hoja informativa al respecto, y la asesora o el asesor de clientes no puede responder a la pregunta. La respuesta depende de para qué se utilice el certificado: para el cifrado o como prueba de quién envió un correo.

**Enfoque en Suiza:** Este artículo trata la situación jurídica en Suiza. Cuando las normas de la UE difieren, se indica expresamente.

## Qué comprueba SwissSign en Silver y Gold

Desde el 1 de septiembre de 2023, los certificados S/MIME de confianza pública están sujetos a los `S/MIME Baseline Requirements` del CA/Browser Forum. Estos distinguen cuatro tipos de validación, según lo que la autoridad de certificación compruebe antes de la emisión. SwissSign asigna sus productos a estos tipos en los perfiles de certificados publicados (CPR S/MIME):

| Producto | Tipo según el CA/B Forum | Qué se comprueba | Qué figura en el certificado |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control sobre el buzón de correo | solo la dirección de correo electrónico |
| E-Mail ID Gold | `Sponsor-validated` | Organización y persona | nombre y apellidos (o seudónimo), organización, identificador de la organización, país, dirección de correo electrónico |
| Organization Validated | `Organization-validated` | Organización | nombre de la organización, sin persona |

Para Silver, SwissSign envía un código de confirmación a la dirección solicitada. Quien pueda recibir el correo obtiene el certificado. No se comprueba la identidad de la persona, y la dirección también figura en el certificado como `Common Name`.

Gold se gestiona en SwissSign exclusivamente como `Sponsor-validated`. Se comprueba la organización y la persona queda vinculada a ella: SwissSign o la propia organización, como entidad de registro, comprueba su identidad, por ejemplo, en el marco de una PKI gestionada. Los perfiles de certificados no incluyen un perfil para particulares sin organización (`Individual-validated`).

Para buzones funcionales como `support@` o `buchhaltung@` está previsto el perfil `Organization-validated`. Las solicitudes Gold para estos buzones pueden fallar porque SwissSign rechaza términos genéricos como «Support» o «Helpdesk» en el campo `GivenName`.

Criptográficamente, los niveles no se diferencian. Todos los perfiles utilizan claves RSA de al menos 2048 bits, el uso de clave `digitalSignature` y `keyEncipherment` y el uso ampliado `id-kp-emailProtection`. Un correo cifrado con Silver está tan bien protegido como uno cifrado con Gold.

## Qué significa la diferencia en la práctica

| | Silver | Gold |
|---|---|---|
| Cifrar y descifrar correos | sí | sí |
| Visualización de la firma en Outlook | Dirección de correo electrónico | Nombre y organización |
| Qué acredita la firma | El remitente tiene acceso al buzón de correo | El remitente es una persona verificada de esta organización |
| Esfuerzo al solicitarlo | Confirmación por correo | Prueba de la organización y verificación de identidad |

La diferencia se aprecia cuando se compromete una cuenta. Quien controla un buzón puede solicitar un certificado Silver para esa dirección y enviar con él correos válidamente firmados. En cambio, un certificado Gold exige que se comprueben la organización y la persona. Por ello, para un banco que recibe por correo, por ejemplo, órdenes de pago o correspondencia contractual, una firma Gold tiene mayor valor probatorio.

Para el cifrado, esto no importa. El banco solo necesita la clave pública del destinatario, y es equivalente en ambos niveles.

## Requisitos legales en Suiza

Ninguna ley suiza prescribe S/MIME, y ninguna distingue entre niveles de certificado. Existen obligaciones generales de proteger los datos, cuya implementación técnica queda abierta.

| Normativa | Contenido | Relación con S/MIME |
|---|---|---|
| DSG art. 8 y DSV arts. 1 a 6 | Seguridad de los datos adecuada al riesgo | El cifrado es un medio habitual para datos personales sensibles; no se prescribe un procedimiento concreto |
| BankG art. 47 | Secreto bancario | Obliga al banco, no a su clientela |
| FINMA-RS 2023/1 | Protección de datos críticos en los bancos | Tecnológicamente neutral, sin requisito sobre certificados |
| ZertES y OR art. 14 apdo. 2bis | Solo la firma electrónica cualificada equivale a la firma manuscrita | Silver y Gold no son firmas cualificadas |
| VeÜ-ZSSV | Presentaciones electrónicas ante tribunales y oficinas de ejecución de deudas | Se requiere firma cualificada y una plataforma de entrega reconocida; S/MIME no basta |

### Protección de datos: DSG y DSV

El art. 8 DSG obliga a los responsables y encargados del tratamiento a garantizar una seguridad de los datos adecuada al riesgo. La ordenanza de protección de datos lo concreta con objetivos de protección como la confidencialidad y la integridad, pero no menciona procedimientos específicos. Que se utilice S/MIME, una pasarela con entrega por webmail o una plataforma de intercambio es una ponderación caso por caso.

### Secreto bancario y FINMA

El secreto bancario conforme al art. 47 BankG y la circular 2023/1 de la FINMA «Riesgos operativos y resiliencia: bancos» se dirigen al banco. La circular exige que los bancos protejan sus datos críticos durante todo su ciclo de vida, también durante la transmisión. Los bancos determinan cómo lo implementan en sus propias directrices. Por tanto, que se acepte Silver depende de la política interna de cada banco.

### Firma electrónica: ZertES y OR

La ZertES distingue entre firma electrónica, avanzada, regulada y cualificada. Según el art. 14 apdo. 2bis OR, solo la firma electrónica cualificada con sello de tiempo cualificado equivale a la firma manuscrita. Los certificados S/MIME de SwissSign se emiten según las normas del CA/Browser Forum y las políticas ETSI LCP (Silver) y NCP (Gold), respectivamente, y no son certificados cualificados conforme a la ZertES.

Por tanto, un contrato para el que se prescribe la forma escrita no puede celebrarse válidamente mediante un correo firmado con S/MIME, tampoco con Gold. Sin embargo, en caso de litigio una firma Gold puede servir como prueba; el tribunal valora libremente su fuerza probatoria.

### Presentaciones ante tribunales y autoridades

Para las presentaciones electrónicas en procedimientos civiles, penales, de ejecución de deudas y de quiebra, la VeÜ-ZSSV exige una firma electrónica cualificada y la transmisión a través de una plataforma de entrega reconocida. Un correo S/MIME no cumple estos requisitos.

**Nota sobre la UE:** En la UE, conforme al art. 25 apdo. 2 eIDAS, también solo la firma electrónica cualificada equivale a la firma manuscrita.

## Si el banco no establece requisitos

Si falta una hoja informativa, la situación puede aclararse con tres preguntas al banco:

1.  ¿El banco exige para los correos firmados un certificado con identidad verificada o basta una mera validación de la dirección de correo electrónico?

2.  ¿El banco trabaja con una pasarela de cifrado (p. ej., SEPPmail o Kiteworks/Totemomail) y un certificado de dominio, o con certificados personales de sus empleados?

3.  ¿Cómo pone el banco a disposición sus propias claves públicas?

Una consulta breve para el servicio de atención al cliente, que pueda remitirse al departamento de TI:

> Para el tráfico de correo cifrado con ustedes, voy a adquirir un certificado S/MIME. Les ruego que me indiquen si exigen un certificado con identidad verificada (p. ej., SwissSign E-Mail ID Gold con nombre y organización) o si basta un certificado con mera validación de la dirección de correo electrónico (Silver). Asimismo, les solicito su certificado público o que indiquen si trabajan mediante una pasarela con certificado de dominio.

Muchos bancos envían de todos modos correos confidenciales mediante una pasarela con entrega por webmail. La destinataria o el destinatario inicia sesión en el portal y no necesita un certificado propio. Si ambas partes trabajan con una pasarela, suelen bastar certificados de dominio o TLS forzado entre los servidores de correo, y no se necesitan certificados personales.

Si la cuestión sigue abierta, Gold es la elección adecuada para la correspondencia empresarial. Gold cubre el cifrado y la firma, y un cambio posterior de Silver a Gold implica una nueva solicitud y una nueva distribución de la clave pública a todas las contrapartes. Silver basta si el banco confirma que solo cifra y no evalúa las firmas. Para buzones funcionales está previsto un certificado `Organization-validated`.

## Fuentes

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): perfiles de certificados de SwissSign (a fecha de 18.09.2025), asignación de Silver, Gold y Organization Validated a los tipos de validación, campos Subject y OID de políticas.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): normas vinculantes para certificados S/MIME de confianza pública, definición de los cuatro tipos de validación.

3.  [Ley federal de protección de datos (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sobre la seguridad de los datos.

4.  [Ordenanza de protección de datos (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): arts. 1 a 6 con principios y objetivos de protección de la seguridad de los datos.

5.  [Ley bancaria (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sobre el secreto bancario.

6.  [FINMA: Circular](https://www.finma.ch/de/dokumentation/rundschreiben/): resumen con la circular 2023/1 «Riesgos operativos y resiliencia: bancos», en vigor desde el 1 de enero de 2024.

7.  [Ley federal sobre la firma electrónica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipos de firma electrónica y requisitos para certificados cualificados.

8.  [Código de obligaciones (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 apdo. 2bis sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita.
