---
title: "compauth en Microsoft 365: autenticación compuesta y todos los códigos de motivo"
navTitle: "códigos compauth"
description: "Microsoft 365 complementa SPF, DKIM y DMARC con su propia evaluación: compauth. Qué comprueba Composite Authentication, qué significan pass, softpass, fail y none, y cuál es la causa detrás de cada código de motivo, del 000 al 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lectura"
themen:
  - microsoft-365-exchange
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-hybrid-header-intern-extern
  - dns-records-e-mail-stolpersteine
slug: "compauth-en-microsoft-365-autenticacion-compuesta-y-todos-los-codigos-de-motivo"
translationId: "article-a9dceac9ee095bbd"
translationOf: microsoft-365-compauth-reason-codes
url: https://rafaelpfister.ch/es/blog/compauth-en-microsoft-365-autenticacion-compuesta-y-todos-los-codigos-de-motivo
translationSourceHash: a37557eaef3ea6605e72281d81c56154d6062ae726ef646baa906c2d7d9927a4
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:21:22.398Z
translationReview: automatic
---

# compauth en Microsoft 365: autenticación compuesta y todos los códigos de motivo

En la línea de encabezado `Authentication-Results` de un correo recibido en Microsoft 365 aparece, junto a los resultados estándar de SPF, DKIM y DMARC, un campo propio de Microsoft:

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` significa Composite Authentication: Microsoft 365 combina los resultados de SPF, DKIM y DMARC con otras señales del mensaje para realizar una evaluación global de si la dirección From visible es creíble. La base de evaluación es el dominio From, es decir, la dirección que ven los destinatarios en el cliente de correo. De este modo, Microsoft cubre la brecha que surge cuando un dominio de remitente no ha publicado registros de autenticación o los ha publicado de forma incompleta: incluso sin una política DMARC, se comprueba implícitamente si el correo coincide con el dominio declarado.

## Los cuatro resultados

- `compauth=pass`: El mensaje ha superado la autenticación explícita (DMARC) o implícita.
- `compauth=softpass`: La comprobación implícita se ha superado con menor seguridad.
- `compauth=fail`: El mensaje no ha superado la comprobación explícita o implícita.
- `compauth=none`: No se realizó ninguna comprobación compuesta o se omitió.

Un `compauth=fail` no provoca automáticamente la cuarentena ni el traslado a la carpeta de correo no deseado. Es una señal de entrada para la decisión del filtro; para el tratamiento real son determinantes `CAT` y otros campos de `X-Forefront-Antispam-Report`. A la inversa: quien quiera saber por qué compauth tomó esa decisión necesita el código `reason` directamente después del resultado.

## Los códigos de motivo de un vistazo

El código de tres dígitos indica la regla que condujo al resultado. El primer dígito agrupa: 0xx y 6xx son fallos, 1xx y 7xx son comprobaciones superadas, 2xx es softpass, y 3xx, 4xx y 9xx significan que no se realizó o se omitió la comprobación.

| Código | Significado |
|---|---|
| `000` | Fallo explícito: error de DMARC con una política `p=quarantine` o `p=reject`. |
| `001` | Fallo implícito: el dominio no publica registros de autenticación o solo publica registros débiles (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | La organización ha prohibido explícitamente que este par remitente/dominio envíe correos suplantados (entrada mantenida manualmente). |
| `010` | Error de DMARC con `p=reject`/`p=quarantine`, y el dominio emisor es un Accepted Domain propio (suplantación de la propia organización). |
| `100` | SPF o DKIM superados; los dominios MAIL FROM y From están alineados. |
| `101` | El mensaje está firmado con DKIM por el dominio From. |
| `102` | Los dominios MAIL FROM y From están alineados; SPF superado. |
| `103` / `104` | El dominio From coincide con el registro PTR (búsqueda inversa) de la dirección IP de entrega. |
| `108` | Fallo de DKIM debido a una modificación del cuerpo en estaciones legítimas anteriores, por ejemplo en el propio entorno OnPrem. |
| `109` | El dominio no tiene registro DMARC, pero la comprobación se habría superado. |
| `111` | A pesar de un error temporal o permanente de DMARC, el dominio SPF o DKIM está alineado con el dominio From. |
| `112` | Un tiempo de espera de DNS impidió recuperar el registro DMARC. |
| `115` | El correo procede de una organización de Microsoft 365 en la que el dominio From está configurado como Accepted Domain. |
| `116` | El registro MX del dominio From coincide con el registro PTR de la IP de entrega. |
| `130` | Un ARC-Sealer configurado como de confianza anuló el fallo de DMARC. |
| `201` / `202` | Softpass: el dominio From coincide con el registro PTR o con su subred. |
| `3xx` / `4xx` / `9xx` | No se realizó la comprobación compuesta o se omitió. |
| `501` / `502` | DMARC no se aplicó porque se trata de un NDR válido. |
| `601` | Fallo implícito: el dominio emisor es un Accepted Domain propio (autosuplantación, frecuente con Direct Send). |
| `701`–`704` | DMARC no se aplicó porque la organización recibe correos legítimos demostrablemente desde esta infraestructura. |
| `905` | DMARC no se aplicó debido a un enrutamiento complejo, por ejemplo correos de Internet a través de Exchange OnPrem o un servicio de terceros antes de Microsoft 365. |

## Los casos más frecuentes en la práctica

**`compauth=fail reason=001`** es el caso habitual en dominios sin autenticación o con autenticación débil. La solución corresponde al remitente: publicar SPF con `-all`, firma DKIM y una política DMARC. Mientras falten los registros, la entregabilidad depende de señales de reputación.

**`compauth=fail reason=601`** aparece cuando llegan desde el exterior correos con el propio dominio como remitente, típicamente con Direct Send: dispositivos multifunción, aplicaciones o proveedores entregan directamente al MX sin un conector autenticado. La solución pasa por un Inbound Connector correctamente configurado o por incluir el origen en el SPF propio.

**`compauth=fail reason=000` o `010`** significa que DMARC se aplicó con normalidad. Si junto a ello aparece `action=oreject`, Microsoft 365 ha traducido la política de rechazo del remitente en una entrega en cuarentena. No hay nada que reparar, salvo que el remitente sea legítimo y su autenticación esté defectuosa.

**`reason=108`** y **`reason=130`** afectan a escenarios de reenvío y gateway: una estación intermedia modificó el correo o un ARC-Sealer de confianza conservó los resultados de comprobación originales. Quien opere un gateway delante de Microsoft 365 debería registrar su ARC-Sealing como de confianza en la configuración antispam; de lo contrario, los correos legítimos seguirán bloqueados por DMARC.

## Leer compauth en el encabezado

En la práctica, `compauth` rara vez aparece solo: solo la interacción con los resultados individuales de SPF, DKIM y DMARC, la alineación de los dominios implicados y la cadena `Received` proporciona la imagen completa. El [analizador de encabezados de correo](/tools/header-analyzer) de este sitio web decodifica `compauth` junto con el código de motivo directamente en el navegador y muestra los dominios correspondientes (From, Envelope-From, `d=`) en paralelo para evaluar la alineación; el encabezado pegado no sale del navegador.

## Fuentes

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Referencia oficial de los campos Authentication-Results y de la tabla completa de códigos de motivo de compauth.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Procedimiento ante fallos de autenticación desde la perspectiva de SecOps.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Configuración de ARC-Sealers de confianza para escenarios de gateway y reenvío (código de motivo 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Diferenciación entre la señal compauth y la decisión real del filtro.
