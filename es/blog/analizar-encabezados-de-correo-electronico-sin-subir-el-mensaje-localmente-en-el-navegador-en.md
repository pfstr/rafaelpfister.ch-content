---
title: "Analizar encabezados de correo electrónico sin subir el mensaje: localmente en el navegador en lugar de una herramienta web"
navTitle: "Analizar encabezados localmente"
description: "Los encabezados de correo electrónico contienen nombres de host internos, direcciones IP y datos personales. Quien los pega en una herramienta en línea transmite esta información a un servidor ajeno. Por qué el análisis no necesita servidor y qué puede ofrecer una herramienta que se ejecuta localmente en el navegador."
date: "2026-08-26"
kategorie: "SMTP y flujo de correo"
timeToRead: "7 min de lectura"
themen:
  - smtp-mailflow
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "mail-auth"
  - "troubleshooting"
related:
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "analizar-encabezados-de-correo-electronico-sin-subir-el-mensaje-localmente-en-el-navegador-en"
translationId: "article-cad792e705cee24e"
translationOf: e-mail-header-analysieren-ohne-upload
url: https://rafaelpfister.ch/es/blog/analizar-encabezados-de-correo-electronico-sin-subir-el-mensaje-localmente-en-el-navegador-en
translationSourceHash: 11c4e7d120ea34ca557f0136b93120e5e8e9d72dc7350fd2df7880b23ff46649
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:16:45.907Z
translationReview: automatic
---

# Analizar encabezados de correo electrónico sin subir el mensaje: localmente en el navegador en lugar de una herramienta web

La forma habitual de analizar un encabezado de correo electrónico es esta: copiar el encabezado desde el cliente de correo, pegarlo en una herramienta en línea y dejar que lo analice. Es práctico, pero el encabezado completo se envía al servidor del operador de la herramienta. Pocos son conscientes de lo que se transmite exactamente con ello.

## Qué contiene realmente un encabezado

Un encabezado completo de un mensaje procedente de un entorno empresarial contiene normalmente:

- **Nombres de host internos y direcciones IP:** Cada línea `Received` documenta un servidor en la ruta de entrega, incluidos los servidores Exchange internos, gateways y balanceadores de carga con FQDN y, a menudo, una dirección IP privada. En conjunto, forman un esquema de la infraestructura de correo.
- **Datos personales:** Direcciones y nombres visibles del remitente y del destinatario, el asunto, identificadores de mensaje y, según el cliente, la dirección IP del remitente original.
- **Software y versiones:** Las líneas Received y los encabezados específicos de cada producto indican los productos utilizados, en parte con sus versiones.
- **Evaluación interna de la organización:** En Microsoft 365, por ejemplo, la evaluación completa de spam y autenticación, identificadores de tenant y la clasificación interna del mensaje.

Para un atacante, es material útil para prepararse; para la protección de datos, son datos personales: remitente, destinatario y asunto de un mensaje concreto. Tras la revisión de la Ley de Protección de Datos, el tratamiento mediante una herramienta en línea extranjera sigue siendo una comunicación a un tercero, en caso de duda al extranjero. En el caso de un encabezado procedente de un ticket de soporte de un cliente, la cuestión se agrava: introducir sus datos en una herramienta web ajena difícilmente puede justificarse sin base jurídica o consentimiento.

## El análisis no necesita servidor

El punto decisivo es este: un encabezado es texto puro y su evaluación consiste únicamente en analizarlo. Ordenar cronológicamente la cadena Received, calcular las diferencias entre marcas de tiempo, decodificar `Authentication-Results`, comparar dominios: nada de ello requiere un componente de servidor. Todo se ejecuta en JavaScript en el navegador, sin que el encabezado abandone el dispositivo.

Una herramienta construida de este modo se diferencia fundamentalmente, desde el punto de vista de la protección de datos, de un analizador en línea clásico: no hay transmisión, almacenamiento por parte del operador ni archivos de registro con encabezados ajenos. Así, el análisis del encabezado de un cliente se mantiene al mismo nivel que abrir el archivo en un editor local, solo que de forma más legible.

## Qué puede hacer una herramienta local

El [analizador de encabezados de correo](/tools/header-analyzer) de este sitio web está construido según este principio. El encabezado pegado se analiza exclusivamente de forma local en el navegador. Su funcionalidad demuestra que no se pierde nada:

- **Ruta de entrega con tiempos de tránsito:** La cadena `Received` se ordena cronológicamente, se calcula el tiempo de permanencia por estación y se marca el tramo más largo. Así se ve dónde se retrasó realmente una entrega lenta. Se detectan y muestran las diferencias de reloj entre servidores.
- **Cifrado de transporte por salto:** La versión de TLS y el cifrado se leen de las líneas Received cuando el servidor receptor los registra; Microsoft, Postfix y Exim escriben formatos diferentes.
- **Autenticación:** Resultados de SPF, DKIM y DMARC de `Authentication-Results` (RFC 8601), incluidos detalles como `header.d`, `smtp.mailfrom` y `compauth` de Microsoft con código de motivo.
- **Alineación DMARC:** El dominio From, Envelope-From y el dominio DKIM se muestran uno junto a otro y se evalúan según alineación strict y relaxed.
- **Integridad de ARC y DKIM:** Trazas propias en el gráfico de flujo muestran desde dónde hasta dónde permaneció intacto el hash DKIM y a partir de qué estación la cadena ARC conserva los resultados de verificación.
- **Entornos Microsoft:** Se decodifican los campos del filtro de spam (`X-Forefront-Antispam-Report`, SCL, CAT), y se marcan las transiciones de tenant y la clasificación híbrida en la ruta de entrega.

Una limitación se aplica a cualquier herramienta de encabezados, sea local o no: muestra la evaluación documentada del servidor receptor, no una verificación propia. El encabezado no responde si un registro SPF sigue teniendo hoy el mismo aspecto que en el momento de la recepción.

## Clasificación de las demás herramientas

Algunos otros proveedores también realizan ya el análisis del lado del cliente; consultar la política de privacidad y la consola de red del navegador aclara si, al pegarlo, realmente no se realiza ninguna solicitud con el contenido del encabezado. Para los analizadores clásicos del lado del servidor se aplica una regla sencilla: no pegar encabezados de entornos de producción ni de terceros, sino como mucho ejemplos anonimizados.

Por ello, para análisis periódicos de encabezados de incidentes o soporte, una herramienta que se ejecuta localmente es la opción evidente: la pregunta de dónde han terminado los datos ni siquiera se plantea.

## Fuentes

1.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Estándar para la cabecera Authentication-Results, que constituye la base de la evaluación de autenticación.

2.  [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321): Definición de las líneas Received (Trace Information), a partir de las cuales se pueden reconstruir la ruta de entrega y los tiempos de tránsito.

3.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Referencia de los campos de encabezado específicos de Microsoft 365 que decodifica un analizador.
