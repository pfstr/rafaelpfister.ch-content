---
title: "S/MIME en los bancos: situación jurídica en Suiza y elección entre Silver y Gold"
navTitle: "S/MIME Silver/Gold"
description: "Qué requisitos legales se aplican cuando un banco suizo firma y cifra correos electrónicos dirigidos a clientes: secreto bancario, Circular 2023/1 de la FINMA, LPD, ZertES, derecho probatorio y conservación. Además, qué nivel de certificado SwissSign se deriva de ello."
date: "2026-11-16"
kategorie: "Cifrado de correo electrónico"
timeToRead: "11 min de lectura"
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
translationSourceHash: 60de430124ebe20a3001bd48fc55affcb921141039e7c85303da9c7216b01e89
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:21:41.094Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/s-mime-con-bancos-certificado-silver-o-gold-de-swisssign
---

# S/MIME en los bancos: situación jurídica en Suiza y elección entre Silver y Gold

Conclusión breve: Ninguna norma suiza obliga a un banco a utilizar S/MIME ni un nivel de certificado concreto. Sin embargo, el secreto bancario, la Circular 2023/1 de la FINMA y la legislación sobre protección de datos exigen que los datos de clientes estén adecuadamente protegidos durante su envío, y el banco debe poder justificar y documentar su solución. Una firma S/MIME no sustituye una firma manuscrita, tampoco con un certificado Gold. Como prueba y acreditación del origen, Gold es claramente superior al certificado Silver, ya que solo Gold contiene el nombre verificado del banco.

La cuestión de qué certificado SwissSign necesita un banco para enviar correos electrónicos surge regularmente en mis proyectos con entidades financieras. Detrás de la cuestión técnica suele haber una cuestión jurídica: qué debe hacer el banco para cumplir sus obligaciones al enviar datos de clientes por correo electrónico y qué efecto tiene un correo firmado.

**Enfoque en Suiza:** Este artículo trata la situación jurídica en Suiza. Cuando las normas de la UE difieren, se incluye una indicación identificada como tal. No sustituye el asesoramiento jurídico para casos concretos.

## Resumen de las bases jurídicas

| Normativa | Contenido | Importancia para el envío de correo electrónico |
|---|---|---|
| Ley de Bancos, art. 47 | Secreto bancario, sancionable penalmente también por negligencia | Los datos de clientes no pueden divulgarse sin autorización |
| Circular FINMA 2023/1, nm. 73 a 82 | Protección de datos críticos durante todo su ciclo de vida | Protección también durante el envío, con neutralidad tecnológica |
| LPD, art. 8 y 24; OPD, art. 1 a 6 | Seguridad de los datos, notificación de vulneraciones | Medidas adecuadas, notificación al PFPDT en caso de alto riesgo |
| ZertES y CO, art. 14, párr. 2bis | Tipos de firmas electrónicas, validez formal | Las firmas S/MIME no sustituyen una firma manuscrita |
| CPC, art. 157 y 177 | Medios de prueba y libre apreciación de la prueba | Los correos firmados son documentos cuyo valor probatorio se aprecia libremente |
| CO, art. 958f; Olico | Conservación de documentos comerciales | Los correos cifrados deben seguir siendo legibles |

Ninguna de estas normas menciona S/MIME ni un nivel de certificado. Establecen objetivos de protección y consecuencias jurídicas; el banco decide por sí mismo cómo alcanzarlos.

## Secreto bancario

El art. 47 de la Ley de Bancos sanciona a quien, como órgano, empleado o mandatario de un banco, revele un secreto que le haya sido confiado en esa calidad. También es punible la comisión por negligencia. La norma se dirige a personas físicas; sin embargo, se basa en la relación contractual entre la clienta o el cliente, como titular del secreto, y el banco.

Para el envío de correo electrónico, esto significa que un correo no cifrado con datos de clientes que llegue a una dirección incorrecta o sea leído durante el tránsito puede constituir una revelación negligente. Por tanto, el banco debe garantizar mediante medidas organizativas y técnicas que el personal envíe los datos de clientes únicamente de forma protegida.

El cliente, como titular del secreto, puede consentir la revelación. Por ello, muchos bancos hacen que el cliente confirme en las condiciones generales o en un acuerdo separado que conoce y desea los riesgos de la comunicación no cifrada por correo electrónico. Dicho consentimiento solo cubre la comunicación con ese cliente y únicamente en el alcance acordado. No sustituye el cifrado para documentos que afecten a terceros ni para la comunicación con socios comerciales.

## Circular FINMA 2023/1: datos críticos

La Circular FINMA 2023/1 «Riesgos operacionales y resiliencia: bancos» está en vigor desde el 1 de enero de 2024 y sustituyó a la Circular 2008/21. La FINMA ha ampliado el anterior enfoque en la confidencialidad de los datos de identificación de clientes (CID) a los datos críticos en general. Cada banco determina por sí mismo, con base en el riesgo, qué datos son críticos; por regla general, los datos de clientes lo son en lo relativo a la confidencialidad.

Para el envío de correo electrónico, son especialmente relevantes los siguientes números marginales:

| Nm. | Contenido | Relación con el envío de correo electrónico |
|---|---|---|
| 73 | Identificar datos críticos y categorizarlos según su criticidad | Determinar qué datos solo pueden enviarse de forma protegida |
| 74, 75 | Garantizar la confidencialidad, integridad y disponibilidad durante todo el ciclo de vida | El envío forma parte del ciclo de vida |
| 76 | Proteger los datos críticos contra el acceso no autorizado | Protección del correo durante el transporte y en el destinatario |
| 79 | Limitar los riesgos elevados al almacenar datos en el extranjero o acceder a ellos desde el extranjero | Afecta a gateways y plataformas de correo ubicados en el extranjero |
| 81 | Notificar sin demora a la FINMA los incidentes importantes | Por ejemplo, un envío masivo de datos de clientes a destinatarios incorrectos |
| 82 | Seleccionar y supervisar cuidadosamente a los proveedores de servicios que tratan datos críticos | Afecta a proveedores de gateways de cifrado y autoridades de certificación con PKI gestionada |

La circular es tecnológicamente neutral y no menciona ni S/MIME ni niveles de certificado. En sus explicaciones sobre la consulta, la FINMA cita el cifrado de datos como ejemplo de las medidas especiales con las que se protegen los datos críticos. El banco debe establecer su solución en sus propias directrices e indicar también qué nivel de certificado utiliza y por qué.

## Protección de datos: LPD y OPD

El art. 8 de la LPD obliga a los responsables del tratamiento a garantizar una seguridad de los datos adecuada al riesgo. La Ordenanza sobre protección de datos lo concreta mediante objetivos de protección como la confidencialidad, la integridad y la trazabilidad, pero no prescribe un procedimiento. S/MIME, un gateway con entrega mediante portal web o una plataforma de intercambio cumplen por igual el requisito si se utilizan de forma adecuada al riesgo.

Si una vulneración de la seguridad de los datos puede previsiblemente conllevar un alto riesgo para la personalidad o los derechos fundamentales de las personas afectadas, el banco debe notificarla al PFPDT lo antes posible conforme al art. 24 de la LPD. Esta obligación de notificación existe además de la obligación de notificar a la FINMA. Si el correo estaba cifrado, por regla general el riesgo en caso de envío erróneo es bajo y no procede notificación.

## Efectos jurídicos de una firma S/MIME

### No es una firma en el sentido del CO

La ZertES distingue entre firma electrónica, firma electrónica avanzada, firma electrónica regulada y firma electrónica cualificada, así como, para empresas, sello electrónico regulado. Conforme al art. 14, párr. 2bis del CO, únicamente la firma electrónica cualificada con sello de tiempo cualificado equivale a la firma manuscrita.

Los certificados S/MIME de SwissSign se emiten conforme a las normas del CA/Browser Forum y las políticas ETSI LCP (Silver) o NCP (Gold). No son certificados cualificados conforme a la ZertES. Por ello, un banco no puede emitir válidamente por correo firmado con S/MIME, ni siquiera con Gold, declaraciones para las que la ley o un contrato exige forma escrita.

### Firma personal y firma de gateway

Una firma avanzada exige, entre otras cosas, que la clave de firma esté bajo el control exclusivo de la persona firmante. Si un gateway firma centralmente todos los correos salientes con claves alojadas en el gateway, este requisito no se cumple. La firma acredita entonces que el correo procede del sistema del banco y no ha sido modificado, pero no que una persona concreta lo haya autorizado. Por tanto, su contenido se asemeja más a un sello de la organización, sin cumplir los requisitos de un sello regulado conforme a la ZertES.

### Valor probatorio

Un correo electrónico es un documento conforme al art. 177 del CPC, y el tribunal aprecia libremente su fuerza probatoria conforme al art. 157 del CPC. Una firma S/MIME válida demuestra que el correo no se ha modificado desde su firma y que fue firmado con la clave de un certificado determinado. Cuánto dice eso sobre el remitente depende de lo que la autoridad de certificación haya verificado antes de emitirlo. Aquí reside la diferencia jurídicamente relevante entre Silver y Gold.

## Silver o Gold desde el punto de vista jurídico

Desde el 1 de septiembre de 2023, se aplican a los certificados S/MIME de confianza pública los `S/MIME Baseline Requirements` del CA/Browser Forum. SwissSign asigna sus productos a estos tipos de validación en los perfiles de certificado (CPR S/MIME):

| Producto | Tipo según el CA/B Forum | Qué verifica SwissSign | Qué figura en el certificado | Lo que acredita la firma |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control sobre el buzón | Solo la dirección de correo electrónico | Alguien con acceso a este buzón |
| E-Mail ID Gold | `Sponsor-validated` | Organización y persona | Nombre, organización, identificador de organización, país, dirección de correo electrónico | Una persona verificada de este banco |
| Organization Validated | `Organization-validated` | Organización | Nombre de la organización, ninguna persona | Un buzón de este banco |

Desde el punto de vista criptográfico, los niveles son equivalentes. Por ello, el nivel no desempeña ningún papel para el cifrado ni para las obligaciones de protección derivadas del secreto bancario, la Circular FINMA y la LPD.

Para la firma, la diferencia es considerable. Un certificado Silver lo obtiene cualquiera que pueda recibir correos en una dirección. Quien registre un dominio de nombre similar, por ejemplo `musterbank-ch.com` en lugar de `musterbank.ch`, puede enviar correos firmados con la misma validez que los correos auténticos del banco. En una disputa sobre si un correo procede del banco, una firma Silver acredita poco. En cambio, un certificado Gold u Organization Validated contiene el nombre del banco verificado por SwissSign. Con Gold se añade la asignación a una persona cuya identidad el banco confirma como entidad de registro en el marco de la PKI gestionada.

## Conservación: CO y Olico

Conforme al art. 958f del CO, los libros comerciales y los justificantes contables deben conservarse durante diez años, y la Ordenanza sobre la conservación de los libros exige que los documentos conservados electrónicamente puedan hacerse legibles durante ese período. Para los bancos se añaden obligaciones de conservación en materia de supervisión y contratos. De ello se derivan dos requisitos para la operación S/MIME:

- Los correos cifrados deben poder descifrarse durante todo el plazo de conservación. Por ello, el banco debe conservar de forma segura las claves privadas de certificados caducados o descifrar los correos antes de archivarlos.
- Si una firma debe seguir sirviendo como prueba años después, debe poder determinarse que el certificado era válido en el momento de la firma. Para ello, deben archivarse también los resultados de las comprobaciones o la información de revocación correspondiente al momento de la firma.

**Nota sobre la UE:** En la UE, conforme al art. 25, párr. 2 de eIDAS, también únicamente la firma electrónica cualificada equivale a la firma manuscrita. Para las empresas financieras de la UE se aplica adicionalmente, desde el 17 de enero de 2025, el Reglamento DORA con sus propios requisitos de gestión de riesgos de TIC.

## Recomendación

- **Establecer una directriz:** El banco define en una directriz interna qué datos solo pueden enviarse cifrados, qué procedimientos están autorizados para ello y qué niveles de certificado utiliza. Esta es la base para demostrar el cumplimiento de los requisitos de la Circular FINMA y de la LPD.
- **Gold para buzones personales, Organization Validated para buzones funcionales:** Solo estos niveles identifican al banco como remitente en el certificado y, por tanto, poseen un valor probatorio que va más allá del acceso a un buzón. Silver no es adecuado para la comunicación con clientes.
- **Aclarar los consentimientos:** Quien se comunique con clientes por correo electrónico no cifrado necesita su consentimiento documentado. No sustituye el cifrado frente a terceros y socios comerciales.
- **No enviar por correo declaraciones sujetas a requisitos formales:** Para declaraciones que exigen forma escrita se necesita una firma electrónica cualificada o una firma manuscrita.
- **Revisar a los proveedores de servicios:** Los proveedores de gateways y la PKI gestionada de la autoridad de certificación están sujetos al nm. 82 de la Circular FINMA y, si están ubicados en el extranjero, adicionalmente al nm. 79.
- **Archivar claves y datos de verificación:** Las claves privadas para el descifrado y los datos para verificar la firma deben seguir disponibles durante el plazo de conservación.
- **Vincular los procesos de notificación:** Un envío erróneo de datos de clientes puede desencadenar simultáneamente una notificación a la FINMA (nm. 81) y al PFPDT (art. 24 LPD). Ambas obligaciones pueden integrarse en un proceso común de gestión de incidentes.

## Fuentes

1.  [Circular FINMA 2023/1 «Riesgos operacionales y resiliencia: bancos»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): nm. 73 a 82 sobre datos críticos, en vigor desde el 1 de enero de 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 con explicaciones](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): texto revisado con las explicaciones de la FINMA procedentes de la consulta, entre otras sobre el cifrado de datos como medida de protección.

3.  [Ley de Bancos (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sobre el secreto bancario.

4.  [Swissbanking: Secreto bancario y nube, dictamen de 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): dictamen jurídico sobre la interpretación del art. 47 de la Ley de Bancos, entre otros aspectos sobre el consentimiento del cliente.

5.  [Ley Federal de Protección de Datos (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sobre la seguridad de los datos, art. 24 sobre la notificación de vulneraciones de la seguridad de los datos.

6.  [Ordenanza sobre protección de datos (OPD)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 a 6 con principios y objetivos de protección de la seguridad de los datos.

7.  [Ley Federal sobre la Firma Electrónica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipos de firmas y sellos electrónicos, requisitos para certificados cualificados.

8.  [Código de Obligaciones (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14, párr. 2bis sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita, art. 958f sobre la conservación.

9.  [Código Procesal Civil (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sobre la libre apreciación de la prueba, art. 177 sobre el concepto de documento.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): perfiles de certificado de SwissSign (a fecha de 18.09.2025), asignación de Silver, Gold y Organization Validated a los tipos de validación y campos Subject.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): normas vinculantes para certificados S/MIME de confianza pública, definición de los tipos de validación.
