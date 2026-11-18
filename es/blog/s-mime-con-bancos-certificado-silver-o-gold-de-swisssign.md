---
title: "S/MIME en los bancos: marco jurídico en Suiza y la elección entre Silver y Gold"
navTitle: "S/MIME Silver/Gold"
description: "Qué requisitos legales se aplican cuando un banco suizo firma y cifra correos electrónicos dirigidos a clientes: secreto bancario, Circular FINMA 2023/1, LPD, ZertES, derecho probatorio y conservación. También se explica qué nivel de certificado SwissSign se deriva de ello."
date: "2026-11-16"
kategorie: "Cifrado de correo electrónico"
timeToRead: "12 min de lectura"
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
translationSourceHash: f4b52127b6883bb31850668cc18a27ebba43f75200bfdc605d199f427d602ddf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:27:03.651Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/s-mime-con-bancos-certificado-silver-o-gold-de-swisssign
---

# S/MIME en los bancos: marco jurídico en Suiza y la elección entre Silver y Gold

Conclusión breve: ninguna norma suiza exige a un banco utilizar S/MIME ni un nivel de certificado concreto. Sin embargo, el secreto bancario, la Circular FINMA 2023/1 y la Ley de Protección de Datos exigen que los datos de los clientes estén adecuadamente protegidos durante el envío, y el banco debe poder justificar y documentar su solución. Una firma S/MIME no sustituye una firma manuscrita, tampoco con un certificado Gold. Como prueba y como acreditación del origen, Gold es claramente superior al certificado Silver, ya que solo Gold contiene el nombre verificado del banco.

La cuestión de qué certificado SwissSign necesita un banco para el envío de correos surge regularmente en mis proyectos con entidades financieras. Detrás de la cuestión técnica suele haber una cuestión jurídica: qué debe hacer el banco para cumplir sus obligaciones al enviar datos de clientes por correo electrónico y qué efecto tiene un correo firmado.

**Enfoque suizo:** Este artículo trata la situación jurídica en Suiza. Cuando las normas de la UE difieren, se incluye una indicación identificada como tal. No sustituye el asesoramiento jurídico para cada caso concreto.

## Las bases jurídicas de un vistazo

| Normativa | Contenido | Importancia para el envío de correos |
|---|---|---|
| Ley de Bancos, art. 47 | Secreto bancario, sancionable penalmente también por negligencia | Los datos de clientes no pueden divulgarse sin autorización |
| Circular FINMA 2023/1, marg. 73 a 82 | Protección de datos críticos durante todo su ciclo de vida | Protección también durante el envío, tecnológicamente neutral |
| LPD art. 8 y 24, OPD art. 1 a 6 | seguridad de los datos, notificación de vulneraciones | medidas adecuadas, notificación al PFPDT en caso de alto riesgo |
| ZertES y CO art. 14 párr. 2bis | Tipos de firmas electrónicas, validez formal | Las firmas S/MIME no sustituyen una firma manuscrita |
| CPC art. 157 y 177 | Medios de prueba y libre apreciación de la prueba | Los correos firmados son documentos cuya fuerza probatoria se aprecia libremente |
| CO art. 958f, Olico | Conservación de documentos comerciales | Los correos cifrados deben seguir siendo legibles |

Ninguna de estas normas menciona S/MIME ni un nivel de certificado. Establecen objetivos de protección y consecuencias jurídicas; el banco decide por sí mismo cómo alcanzarlos.

## Secreto bancario

El art. 47 de la Ley de Bancos castiga a quien, como órgano, empleado o mandatario de un banco, revele un secreto que le haya sido confiado en tal calidad. También es punible la comisión por negligencia. La norma se dirige a personas físicas; sin embargo, se basa en la relación contractual entre la clienta o el cliente, como titular del secreto, y el banco.

Para el envío de correos, esto significa que un correo no cifrado con datos de clientes, enviado a una dirección incorrecta o leído durante el transporte, puede constituir una revelación negligente. Por ello, el banco debe garantizar mediante medidas organizativas y técnicas que el personal solo envíe datos de clientes de forma protegida.

El cliente, como titular del secreto, puede consentir la revelación. Por ello, muchos bancos hacen que el cliente confirme en las condiciones generales o en un acuerdo separado que conoce y desea los riesgos de la comunicación no cifrada por correo electrónico. Tal consentimiento solo cubre la comunicación con ese cliente y únicamente en el alcance acordado. No sustituye el cifrado para documentos que afecten a terceros ni para la comunicación con socios comerciales.

## Circular FINMA 2023/1: datos críticos

La Circular FINMA 2023/1 «Riesgos operacionales y resiliencia: bancos» está en vigor desde el 1 de enero de 2024 y sustituyó a la Circular 2008/21. La FINMA ha ampliado el anterior enfoque sobre la confidencialidad de los datos de identificación de clientes (CID) a los datos críticos en general. Cada banco determina por sí mismo, en función del riesgo, qué datos son críticos; por regla general, los datos de clientes se incluyen en esta categoría respecto de la confidencialidad.

Para el envío de correos, son especialmente relevantes los siguientes números marginales:

| Marg. | Contenido | Relación con el envío de correos |
|---|---|---|
| 73 | identificar los datos críticos y categorizarlos según su criticidad | determinar qué datos solo pueden enviarse de forma protegida |
| 74, 75 | garantizar la confidencialidad, integridad y disponibilidad durante todo el ciclo de vida | el envío forma parte del ciclo de vida |
| 76 | proteger los datos críticos contra el acceso de personas no autorizadas | protección del correo durante el transporte y en el destinatario |
| 79 | limitar los riesgos elevados del almacenamiento en el extranjero o del acceso desde el extranjero | afecta a gateways y plataformas de correo ubicados en el extranjero |
| 81 | notificar sin demora a la FINMA los incidentes significativos | por ejemplo, un envío masivo de datos de clientes a destinatarios incorrectos |
| 82 | seleccionar y supervisar cuidadosamente a los proveedores de servicios que tratan datos críticos | afecta a proveedores de gateways de cifrado y autoridades de certificación con PKI gestionada |

La circular es tecnológicamente neutral y no menciona ni S/MIME ni niveles de certificados. En sus explicaciones de la consulta, la FINMA cita el cifrado de datos como ejemplo de las medidas especiales con las que se protegen los datos críticos. El banco debe definir su solución en directivas internas e indicar también qué nivel de certificado utiliza y por qué.

## Protección de datos: LPD y OPD

El art. 8 de la LPD obliga a los responsables a garantizar una seguridad de los datos adecuada al riesgo. La Ordenanza de Protección de Datos lo concreta mediante objetivos de protección como la confidencialidad, la integridad y la trazabilidad, pero no prescribe ningún procedimiento. S/MIME, un gateway con entrega a través de un portal web o una plataforma de intercambio cumplen por igual el requisito si se utilizan de manera adecuada al riesgo.

Si una vulneración de la seguridad de los datos puede previsiblemente conllevar un alto riesgo para la personalidad o los derechos fundamentales de las personas afectadas, el banco debe notificarla al PFPDT lo antes posible conforme al art. 24 de la LPD. Esta obligación de notificación existe junto a la obligación de notificar a la FINMA. Si el correo estaba cifrado, el riesgo en caso de envío erróneo suele ser bajo y no se requiere notificación.

## Efecto jurídico de una firma S/MIME

### No es una firma en el sentido del CO

La ZertES distingue entre firma electrónica, firma electrónica avanzada, firma electrónica regulada y firma electrónica cualificada, así como, para empresas, sello electrónico regulado. Conforme al art. 14 párr. 2bis del CO, solo la firma electrónica cualificada con sello de tiempo cualificado equivale a la firma manuscrita.

Los certificados S/MIME de SwissSign se emiten conforme a las reglas del CA/Browser Forum y las políticas ETSI LCP (Silver) y NCP (Gold), respectivamente. No son certificados cualificados conforme a la ZertES. Por lo tanto, un banco no puede realizar formalmente por correo firmado con S/MIME declaraciones para las que la ley o un contrato exijan forma escrita, tampoco con Gold.

### Firma personal y firma de gateway

Una firma avanzada exige, entre otras cosas, que la clave de firma esté bajo el control exclusivo de la persona firmante. Si un gateway firma centralmente todos los correos salientes con claves almacenadas en el gateway, no se cumple este requisito. La firma acredita entonces que el correo procede del sistema del banco y no ha sido alterado, pero no que una persona determinada lo haya autorizado. Por tanto, su contenido se asemeja más a un sello de la organización, sin cumplir los requisitos de un sello regulado conforme a la ZertES.

### Valor probatorio

Un correo es un documento conforme al art. 177 del CPC, y el tribunal aprecia libremente su fuerza probatoria conforme al art. 157 del CPC. Una firma S/MIME válida demuestra que el correo no ha sido modificado desde la firma y que fue firmado con la clave de un certificado determinado. Cuánto dice esto sobre el remitente depende de lo que la autoridad de certificación haya verificado antes de su emisión. Aquí reside la diferencia jurídicamente relevante entre Silver y Gold.

## Silver o Gold desde la perspectiva jurídica

Desde el 1 de septiembre de 2023 se aplican a los certificados S/MIME de confianza pública los `S/MIME Baseline Requirements` del CA/Browser Forum. SwissSign asigna sus productos a estos tipos de validación en los perfiles de certificado (CPR S/MIME):

| Producto | Tipo según el CA/B Forum | Qué verifica SwissSign | Qué figura en el certificado | Lo que acredita la firma |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control sobre el buzón | solo la dirección de correo electrónico | alguien con acceso a este buzón |
| E-Mail ID Gold | `Sponsor-validated` | Organización y persona | nombre, organización, identificador de organización, país, dirección de correo electrónico | una persona verificada de este banco |
| Organization Validated | `Organization-validated` | Organización | nombre de la organización, sin persona | un buzón de este banco |

Criptográficamente, los niveles son equivalentes. Por ello, el nivel no tiene importancia para el cifrado ni para las obligaciones de protección derivadas del secreto bancario, la Circular FINMA y la LPD.

Para la firma, la diferencia es considerable. Un certificado Silver lo obtiene cualquiera que pueda recibir correos en una dirección. Quien registre un dominio de nombre similar, por ejemplo `musterbank-ch.com` en lugar de `musterbank.ch`, puede enviar correos firmados con la misma validez que los correos auténticos del banco. Por ello, en una disputa sobre si un correo procede del banco, una firma Silver prueba poco. En cambio, un certificado Gold u Organization Validated contiene el nombre del banco verificado por SwissSign. En Gold se añade la asignación a una persona cuya identidad el banco confirma, como entidad de registro, en el marco de la PKI gestionada.

## Conservación: CO y Olico

Conforme al art. 958f del CO, los libros comerciales y los comprobantes contables deben conservarse durante diez años, y la Ordenanza sobre la conservación de los libros de comercio (Olico) exige que los documentos conservados electrónicamente puedan hacerse legibles durante ese período. Para los bancos se suman obligaciones regulatorias y contractuales de conservación. De ello resultan dos requisitos para la operación de S/MIME:

- Los correos cifrados deben poder descifrarse durante todo el plazo de conservación. Por ello, el banco debe conservar de forma segura las claves privadas de los certificados caducados o descifrar los correos antes de archivarlos.
- Si una firma debe servir como prueba incluso años después, debe seguir siendo posible comprobar que el certificado era válido en el momento de la firma. Para ello, los resultados de verificación o la información de revocación correspondiente al momento de la firma deben archivarse también.

**Nota sobre la UE:** En la UE, conforme al art. 25 párr. 2 de eIDAS, también solo la firma electrónica cualificada equivale a la firma manuscrita. Para las empresas financieras de la UE, desde el 17 de enero de 2025 se aplica además el Reglamento DORA, con requisitos propios para la gestión del riesgo de las TIC.

## Ejemplo: un gateway para todo el personal

Muchos bancos utilizan un gateway de cifrado central, como PrivaSphere, SEPPmail o Totemomail, para que todo el personal pueda cifrar y firmar correos sin tener que gestionar certificados personalmente. Las claves se almacenan en el gateway, que firma y cifra en representación de cada buzón. De las secciones anteriores se desprende la siguiente elección para esta arquitectura:

| Buzón | Nivel | Motivo |
|---|---|---|
| buzones personales | E-Mail ID Gold | identifica al banco como remitente mediante nombre e identificador de organización verificados |
| buzones funcionales (atención al cliente, operaciones de pago) | Organization Validated | identifica al banco sin nombrar a una persona concreta |
| ninguno | E-Mail ID Silver | solo acredita el acceso a un buzón; los correos de dominios de nombre similar están firmados con la misma validez |

Dado que firma el gateway y no cada persona, las claves no están bajo el control exclusivo del personal. Jurídicamente, la firma funciona como un sello del banco y no como una firma personal. Por ello, lo decisivo es que la organización figure de forma verificada en el certificado, y esto solo lo proporcionan Gold y Organization Validated. Para el cifrado en sí, el nivel no tiene importancia.

Antes de la implantación, deben aclararse los siguientes puntos con el proveedor del gateway y la autoridad de certificación:

- **Emisión y renovación:** Si el gateway obtiene y renueva automáticamente certificados Gold a través de la PKI gestionada de la autoridad de certificación. Con Gold, el banco confirma como entidad de registro la identidad de cada persona; para ello se necesita un proceso para altas y bajas, cambios de nombre y revocaciones.
- **Evaluación de proveedores:** El proveedor del gateway y la PKI gestionada tratan datos críticos en el sentido del marg. 82 de la Circular FINMA 2023/1. Debe documentarse la ubicación de las claves y los datos; si se almacenan en el extranjero o se accede a ellos desde allí, también se aplica el marg. 79.
- **Archivado:** Las claves privadas de certificados caducados y los datos de verificación correspondientes al momento de la firma deben permanecer disponibles durante todo el plazo de conservación.
- **Directiva:** Los niveles elegidos y los procedimientos autorizados deben figurar en la directiva interna, que sirve como prueba ante la FINMA y la auditoría.

## Recomendación

- **Definir una directiva:** El banco establece en una directiva interna qué datos solo pueden enviarse cifrados, qué procedimientos están autorizados para ello y qué niveles de certificado utiliza. Esta es la base para demostrar el cumplimiento de los requisitos de la Circular FINMA y la LPD.
- **Gold para buzones personales, Organization Validated para buzones funcionales:** Solo estos niveles identifican al banco como remitente en el certificado y, por tanto, tienen un valor probatorio que va más allá del acceso a un buzón. Silver no es adecuado para la comunicación con clientes.
- **Aclarar los consentimientos:** Quien se comunique con clientes por correo no cifrado necesita su consentimiento documentado. No sustituye el cifrado frente a terceros y socios comerciales.
- **No enviar por correo declaraciones sujetas a forma:** Para las declaraciones que requieren forma escrita se necesita una firma electrónica cualificada o una firma manuscrita.
- **Evaluar proveedores:** Los proveedores de gateways y la PKI gestionada de la autoridad de certificación están sujetos al marg. 82 de la Circular FINMA y, en caso de ubicaciones en el extranjero, también al marg. 79.
- **Archivar claves y datos de verificación:** Las claves privadas para el descifrado y los datos para la verificación de firmas deben permanecer disponibles durante el plazo de conservación.
- **Vincular los procesos de notificación:** Un envío erróneo de datos de clientes puede desencadenar simultáneamente una notificación a la FINMA (marg. 81) y al PFPDT (art. 24 LPD). Ambas obligaciones pueden integrarse en un proceso común de gestión de incidentes.

## Fuentes

1.  [Circular FINMA 2023/1 «Riesgos operacionales y resiliencia: bancos»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): marg. 73 a 82 sobre datos críticos, en vigor desde el 1 de enero de 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 con explicaciones](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): texto revisado con las explicaciones de la FINMA durante la consulta, entre otras sobre el cifrado de datos como medida de protección.

3.  [Ley de Bancos (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sobre el secreto bancario.

4.  [Swissbanking: secreto bancario y nube, dictamen de 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): dictamen jurídico sobre la interpretación del art. 47 de la Ley de Bancos, entre otros aspectos sobre el consentimiento del cliente.

5.  [Ley Federal de Protección de Datos (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sobre seguridad de los datos, art. 24 sobre la notificación de vulneraciones de la seguridad de los datos.

6.  [Ordenanza de Protección de Datos (OPD)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 a 6 con principios y objetivos de protección de la seguridad de los datos.

7.  [Ley Federal sobre la Firma Electrónica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipos de firmas y sellos electrónicos, requisitos para certificados cualificados.

8.  [Código de Obligaciones (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 párr. 2bis sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita, art. 958f sobre conservación.

9.  [Código Procesal Civil (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sobre la libre apreciación de la prueba, art. 177 sobre el concepto de documento.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): perfiles de certificado de SwissSign (a fecha de 18.09.2025), asignación de Silver, Gold y Organization Validated a los tipos de validación y campos Subject.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): normas obligatorias para certificados S/MIME de confianza pública, definición de los tipos de validación.
