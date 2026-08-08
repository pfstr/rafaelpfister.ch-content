---
title: "Guía para administradores de DNS: MX, SPF, DKIM, DMARC y los errores habituales"
navTitle: "Registros DNS de correo electrónico"
description: "Quien administra una zona suele recibir los registros de correo ya preparados y solo tiene que publicarlos. Lo que falla habitualmente: el límite de 255 bytes en DKIM, registros SPF duplicados, el límite de consultas, un MX en un CNAME, el sufijo de zona añadido automáticamente y políticas que nadie aplica."
date: "2026-08-04"
kategorie: "SMTP y flujo de correo"
timeToRead: "15 min de lectura"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
produkte:
  - "uebergreifend"
protokolle:
  - "dns"
  - "smtp"
  - "tls"
  - "verschluesselung"
  - "mail-auth"
hauptthema: "smtp-mailflow"
related:
  - smtp-verbindung-testen-linux
  - ghost-sender-exchange-online-nebeneingang
slug: "guia-para-administradores-de-dns-mx-spf-dkim-dmarc-y-los-errores-habituales"
translationId: "article-e4699ad7fcea2e20"
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
translationOf: dns-records-e-mail-stolpersteine
url: https://rafaelpfister.ch/es/blog/guia-para-administradores-de-dns-mx-spf-dkim-dmarc-y-los-errores-habituales
translationSourceHash: dc806bed491a47ecc1118249566d9303b0201f4bdb5153a966385a7c9373b31f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:05:22.712Z
translationReview: automatic
---

# Guía para administradores de DNS: MX, SPF, DKIM, DMARC y los errores habituales

Quien administra una zona DNS rara vez recibe los registros de correo escritos por sí mismo. El equipo de correo, un proveedor o una agencia de marketing envía una línea indicando que «solo hay que publicarla». De ahí surgen precisamente la mayoría de los errores, pues los registros de correo son el tipo de registro en el que una errata puede tener dos consecuencias completamente distintas. O bien la entrega falla inmediatamente y alguien avisa en cuestión de minutos, o bien sigue funcionando sin cambios y solo falla silenciosamente la comprobación del remitente. El segundo caso suele pasar inadvertido durante meses, hasta que un gran destinatario pone el dominio en cuarentena.

Desde que Google y Yahoo endurecieron sus requisitos para remitentes masivos en febrero de 2024 y Microsoft los siguió en mayo de 2025, la tolerancia hacia dominios configurados a medias se ha reducido. SPF, DKIM y un registro DMARC ya no son opcionales para remitentes a partir de cierto volumen, sino un requisito de entrega.

Todos los ejemplos de este artículo utilizan `example.com` y selectores genéricos. Los valores mostrados están abreviados para que sigan siendo legibles.

## Reglas aplicables a todos los registros de correo

### El límite de 255 bytes en los registros TXT

Según RFC 1035, un registro TXT consta de una o varias `character-strings`, y cada una de estas cadenas admite un máximo de 255 bytes. El registro completo puede ser más largo, pero entonces debe dividirse en varias cadenas. Los sistemas que lo evalúan vuelven a concatenar estas partes sin separadores.

Esto resulta relevante en la práctica exactamente en un caso: las claves DKIM de 2048 bits. Su valor Base64 tiene unos 400 caracteres y no cabe en una sola cadena.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La mayoría de los sistemas de gestión DNS realizan esta división automáticamente si el valor se introduce mediante el campo de entrada habitual. Quien añada comillas manualmente debe respetar el límite con exactitud. Un valor dividido con un espacio en el punto de unión produce una clave que existe sintácticamente pero que ya no es válida criptográficamente.

Es importante comprobarlo después, porque una clave mal compuesta parece completamente normal en la interfaz gráfica:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

### Un registro por finalidad

SPF y DMARC están definidos de tal forma que solo puede haber un registro correspondiente en un mismo nombre. En SPF, dos registros `v=spf1` provocan un `permerror`, por lo que la comprobación se considera fallida, no superada. En DMARC, los destinatarios ignoran completamente el dominio si varios registros empiezan por `v=DMARC1`: en vez de aplicarse una política estricta, no se aplica ninguna.

Este es, con diferencia, el error más frecuente en zonas que han crecido con el tiempo. Se incorpora un nuevo proveedor, alguien añade «su» registro SPF en lugar de ampliar el existente y, desde ese momento, falla la comprobación para todos los remitentes. Por tanto, antes de crear un registro nuevo es imprescindible comprobar qué existe ya:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

Para DKIM ocurre lo contrario: se prevé un registro por selector, y varios selectores en paralelo son lo habitual porque cada servicio de envío aporta su propia clave.

### El sufijo de zona en las interfaces web

En Infoblox, en DNS de Windows y en prácticamente todas las interfaces de hosting, el nombre de zona se añade automáticamente al nombre introducido. Quien escriba el nombre completo en el campo «Nombre» obtiene un registro dos veces más largo de lo previsto:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

En el archivo de zona, el equivalente es la ausencia del punto final. `mail.example.com` sin punto al final es un nombre relativo y se completa con el nombre de zona; `mail.example.com.` con punto es absoluto. En destinos MX y CNAME, ese único punto determina si el dominio es accesible.

### Copiar y pegar es la fuente de errores más frecuente

Los valores de los registros de correo casi nunca se teclean; se copian desde un PDF, un ticket, una celda de Excel o un chat. Así se introducen errores que permanecen invisibles en el campo de entrada:

- Un `p=` duplicado al principio de la clave DKIM, porque el prefijo se añadió dos veces al ensamblarlo. El valor `v=DKIM1;k=rsa;p=p=MIIBIjAN...` es un clásico real y genera una clave inutilizable.
- Comillas tipográficas de Word en lugar de comillas rectas.
- Espacios de no separación procedentes de diseños PDF que parecen normales.
- Saltos de línea en medio del bloque Base64 si el valor ocupaba varias líneas en el PDF.

Base64 solo admite los caracteres de A a Z, de a a z, del 0 al 9, `+`, `/` y `=` como caracteres de relleno. Cualquier otro carácter en la parte `p=` es un error. Un filtro breve antes de introducirlo evita tener que buscar fallos después:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

Si el resultado no es `0`, la clave contiene caracteres ajenos.

### Reducir el TTL antes de los cambios

Antes de cualquier cambio planificado en un registro MX, SPF o DKIM, el TTL debe reducirse durante unas horas a un valor bajo, normalmente 300 segundos. De lo contrario, el valor anterior puede permanecer un día o más en resolutores ajenos, según la zona, y una reversión tardará lo mismo. Tras el cambio y un período de observación, el TTL se vuelve a establecer en su valor habitual.

## MX

El registro MX determina qué host acepta correo para el dominio. Hay dos reglas que se incumplen con frecuencia.

**El destino debe ser un nombre de host con un registro A o AAAA.** No se permite ni una dirección IP ni un CNAME. RFC 2181 establece expresamente que el destino de un registro MX no puede ser un alias. En la práctica sigue funcionando con muchos destinatarios, pero no con otros, lo que produce errores que aparentemente solo afectan a determinados remitentes.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**El número es una preferencia, no una ponderación.** Se intenta primero el valor menor. Un segundo MX con un número alto solo tiene sentido si ese sistema conoce el mismo filtro de destinatarios. Las entradas MX de respaldo en sistemas sin comprobación de destinatarios son un objetivo habitual del spam, porque los atacantes se dirigen deliberadamente a la entrada más débil.

Los dominios que solo envían correo o que no tienen nada que ver con el correo reciben un MX nulo conforme a RFC 7505. Indica que el dominio no acepta correo y garantiza un rechazo inmediato e inequívoco en lugar de tiempos de espera:

```text
example.com.  IN  MX  0 .
```

Sin embargo, el MX nulo no sustituye a un registro SPF y DMARC. No recibir correo no significa que nadie pueda enviar en su nombre. Los subdominios aparcados, en particular, se utilizan para spoofing porque rara vez alguien los vigila.

## A, AAAA, PTR y el nombre HELO

El registro PTR de la dirección IP saliente no se encuentra en su zona, sino en la zona `in-addr.arpa` del proveedor propietario del bloque de direcciones. Por tanto, debe solicitarse al proveedor y no configurarse por cuenta propia. Muchos grandes destinatarios exigen que el PTR y el registro directo correspondiente coincidan; es decir, que el nombre del PTR vuelva a resolver a la misma dirección IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

El nombre que su servidor de correo indica en HELO o EHLO debería ser el mismo y también debe poder resolverse. Un gateway que se presenta como `localhost.localdomain` o con un nombre interno recibe una peor valoración de los grandes destinatarios.

Hay que tener cuidado al añadir un registro AAAA. En cuanto el servidor de correo sea accesible y envíe mediante IPv6, se aplican los mismos requisitos que para IPv4, e incluso requisitos más estrictos en algunos aspectos. Google exige un PTR válido para direcciones IPv6 que envían correo. Si falta, el envío se rechaza aunque haya funcionado perfectamente por IPv4. Por tanto, un registro AAAA en el servidor de correo nunca es un mero cambio de DNS.

## SPF

SPF determina qué sistemas pueden enviar en nombre del dominio. El registro se publica como TXT en el propio dominio.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### El límite de consultas

La evaluación de un registro SPF puede activar como máximo diez mecanismos que realizan consultas DNS. Se cuentan `include`, `a`, `mx`, `ptr`, `exists` y `redirect`, de forma recursiva: cada `include` incorpora las consultas del registro incluido. No se cuentan `ip4`, `ip6` ni `all`.

Si se supera el límite, el resultado es un `permerror`. Para DMARC, esto significa que SPF no se ha superado, independientemente de que el servidor remitente estuviera autorizado. Lo engañoso es que el error suele surgir sin intervención propia, porque un proveedor incluido amplía su registro. El registro propio no ha cambiado, pero la entrega se interrumpe igualmente.

Además, solo se permiten dos «void lookups», es decir, consultas sin resultado. Un `include` a un dominio que ya no existe cuenta para este límite. Por ello deben eliminarse las referencias a proveedores retirados y no dejarlas por precaución.

### Lo que no debe incluirse en un registro SPF

- **`ptr`** está especificado, pero se considera obsoleto desde RFC 7208 y no debe utilizarse. Los sistemas que evalúan SPF pueden ignorarlo.
- **`+all`** autoriza a cualquier remitente y, por tanto, es más perjudicial que no tener ningún registro SPF.
- **`?all`** es neutral y, por ello, prácticamente inútil para DMARC.
- **Un registro independiente de tipo SPF (tipo 99)** ya no es necesario. Está obsoleto desde RFC 7208; SPF se publica exclusivamente en TXT.

La elección entre `~all` (softfail) y `-all` (hardfail) depende de cuán completamente estén identificadas las rutas de envío. Mientras haya dudas al respecto, `~all` es la elección correcta. Quien ya aplique DMARC y evalúe los informes puede pasar a `-all`.

### Los subdominios no heredan nada

Un registro SPF en `example.com` no es válido para `newsletter.example.com`. Cada subdominio que envíe correo necesita su propio registro. Para todos los demás, se recomienda una entrada comodín que deje claro que no se envía nada desde ellos:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Atención: un comodín TXT también responde a consultas de nombres como `_dmarc.sub.example.com` si no existe allí un registro explícito. Esto suele ser inofensivo, pero puede confundir la búsqueda de errores, pues toda consulta TXT recibe una respuesta.

### Aplanamiento de SPF

Las herramientas que resuelven todas las referencias `include` y las sustituyen por las direcciones IP subyacentes eliminan el límite de consultas a costa de la mantenibilidad. Si el proveedor cambia sus direcciones, el envío falla y nadie lo nota porque aparentemente todo parece correcto en el registro propio. Por eso, quien elija este camino necesita una comprobación automatizada que contraste la lista regularmente con la fuente. Como tarea manual puntual, este procedimiento falla tarde o temprano.

## DKIM

DKIM firma los mensajes salientes. La clave pública se publica en `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

El selector puede elegirse libremente y lo define el sistema remitente. Un nombre descriptivo con fecha facilita mucho más la rotación posterior que `s1` y `s2`.

### Delegación mediante CNAME

Cuando el servicio de envío lo ofrece, debe preferirse la variante CNAME a la publicación directa:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Así, el proveedor puede rotar su clave de forma autónoma sin que nadie tenga que intervenir en su zona. De otro modo, esta rotación suele quedar pendiente porque requiere coordinación entre dos equipos. No obstante, un CNAME excluye cualquier otro registro con el mismo nombre; esta es una regla fundamental de DNS y no una particularidad de DKIM.

### Rotación sin interrupciones

Al cambiar de clave, primero se publica el nuevo selector, después se cambia el servidor remitente a él y solo entonces se elimina el registro antiguo. Quien borre inmediatamente la clave antigua invalida las firmas de todos los mensajes que aún están en tránsito o en colas y hace imposibles las comprobaciones posteriores. Es adecuado dejar pasar varios días entre el cambio y la eliminación.

Por cierto, un registro con `p=` vacío no es una entrada defectuosa, sino la forma especificada de indicar que una clave ha sido retirada.

### Longitud de clave

1024 bits se consideran obsoletos; 2048 bits son el estándar. Las claves RSA mayores no aportan prácticamente ninguna ventaja adicional y solo aumentan la probabilidad de que un sistema intermedio no procese correctamente el registro.

## DMARC

DMARC combina SPF y DKIM con una instrucción sobre qué hacer cuando una comprobación falla y devuelve informes. El registro se publica en `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Desde mayo de 2026, con RFC 9989 y las especificaciones de informes RFC 9990 y RFC 9991, está vigente la versión revisada que sustituye a RFC 7489. En la práctica, hay tres cambios importantes:

- **`pct` ha desaparecido.** Ya no existe la introducción gradual mediante un porcentaje. En su lugar está `t=y`, que identifica el dominio como en fase de pruebas: los informes siguen llegando, pero la política no debe aplicarse.
- **`np` es nuevo.** Define la política para subdominios inexistentes y así cierra una brecha que los atacantes aprovechan con frecuencia, pues los subdominios inventados hasta ahora solo quedaban cubiertos por `sp`. Si no se indica explícitamente, `np` adopta el valor de `sp`.
- **La Public Suffix List se ha sustituido por un `Tree Walk`.** El dominio organizativo ya no se determina mediante una lista mantenida externamente, sino mediante consultas DNS escalonadas a lo largo del árbol de nombres. Esto modifica perceptiblemente la evaluación para espacios de nombres grandes con muchos niveles.

### El alignment es el verdadero núcleo

DMARC no se supera porque SPF o DKIM hayan superado técnicamente sus comprobaciones, sino solo si al menos uno de los dos también coincide con el dominio visible del remitente en la cabecera `From`. SPF se comprueba frente al dominio del remitente del sobre, que difiere con frecuencia en reenvíos, servicios de boletines y sistemas de tickets. Por eso los mensajes con SPF válido a veces no superan la comprobación DMARC.

Con `adkim=r` y `aspf=r` (relaxed, el valor predeterminado), basta con que coincidan a nivel de dominio organizativo. `s` exige igualdad exacta, incluido el subdominio, y en la práctica casi siempre falla en alguna de las rutas de envío.

### Las direcciones externas de informes necesitan autorización

Si los informes deben enviarse a una dirección fuera del dominio propio, por ejemplo a un servicio de análisis DMARC, el dominio receptor debe autorizarlo. Sin este registro, muchos destinatarios simplemente no envían nada y el análisis permanece vacío aunque el registro propio parezca correcto:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Esta entrada la crea el operador de la zona de destino, no usted. En servicios comerciales sucede automáticamente, pero no en un buzón de recopilación autogestionado situado en otro dominio propio.

### Errores de sintaxis habituales

Los nombres de las etiquetas y los valores de política deben escribirse en minúsculas; `p=Reject` no es válido. Las etiquetas se separan mediante punto y coma; si falta un separador, el resto de la línea queda sin efecto. Además, `p` debe ser la primera etiqueta después de `v`. Un registro compuesto solo por `v=DMARC1; rua=...` no contiene ninguna política y está incompleto.

### El despliegue

`p=none` es un estado de medición, no un objetivo. No cambia el tratamiento que los destinatarios dan a sus correos y sirve únicamente para identificar mediante informes todas las rutas legítimas de envío. Quien, tras la introducción, no pase en unos meses de `quarantine` a `reject`, habrá realizado el esfuerzo sin obtener la protección. El aspecto organizativo de este proceso, incluida la propuesta de decisión, es un tema aparte y se describe en el plano de DMARC.

## MTA-STS y TLS-RPT

SMTP cifra de forma oportunista: si la otra parte ofrece STARTTLS, se cifra; de lo contrario, no. Un atacante que pueda manipular el tráfico puede eliminar el anuncio de STARTTLS y mantener así la conexión en texto claro. MTA-STS cierra esta brecha para los dominios receptores.

MTA-STS consta de dos partes y solo una de ellas está en DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La política propiamente dicha se encuentra como archivo en `https://mta-sts.example.com/.well-known/mta-sts.txt` y debe servirse mediante un certificado válido:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Casi todos los problemas están fuera de la zona:

- **El `id` debe cambiar con cada modificación de política.** Es la única señal para los sistemas remitentes de que deben obtener una nueva política. Quien adapte el archivo y deje el `id` sin cambios trabajará contra copias almacenadas en caché hasta que venza `max_age`.
- **La lista MX de la política y los registros MX deben coincidir.** Los remitentes con `mode: enforce` rechazarán un MX nuevo que falte en la política. Por ello, durante las migraciones, la política debe adaptarse antes del cambio de MX.
- **Primero `mode: testing`.** En este modo, las infracciones solo se notifican, no se aplican. El cambio a `enforce` se realiza cuando los informes están limpios.
- **Un registro CAA puede bloquear la emisión del certificado para el host de la política** si indica una autoridad de certificación distinta de la utilizada.

TLS-RPT proporciona los informes correspondientes y consiste en un único registro:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT también resulta útil sin MTA-STS porque hace visibles los fallos de cifrado del transporte.

## DANE

DANE logra el mismo objetivo que MTA-STS, pero ancla la confianza en DNS en lugar de en la PKI web. Requiere una zona firmada de extremo a extremo con DNSSEC; sin DNSSEC, un registro TLSA no tiene efecto.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Lo decisivo en operación: con cada cambio de certificado, el registro TLSA debe coincidir previamente. El procedimiento habitual publica el nuevo hash en paralelo con el antiguo, cambia después el certificado y elimina luego la entrada antigua. Quien invierta este orden hará que el servidor de correo sea inaccesible para todos los remitentes que verifican DANE, entre ellos los grandes proveedores de habla alemana. En Suiza, DANE es mucho menos frecuente que MTA-STS, principalmente debido a la falta de firma DNSSEC en la zona.

## BIMI

BIMI muestra el logotipo de la marca en la bandeja de entrada y es el único mecanismo tratado aquí que aún no es un RFC, sino que sigue siendo un borrador de Internet.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Los requisitos son elevados: una política DMARC aplicada con `quarantine` o `reject`, un logotipo en formato SVG Tiny Portable/Secure y, para la mayoría de los proveedores, un Verified Mark Certificate de pago. Por tanto, BIMI no es un mecanismo de seguridad, sino una cuestión de visibilidad, y debe ocupar el final del orden, no el principio.

## Otros registros relacionados

**Autodiscover y SRV:** Los entornos Exchange utilizan `autodiscover.example.com` como CNAME o un registro SRV `_autodiscover._tcp.example.com`. Ambos afectan a la configuración de clientes y no al flujo de correo, pero suelen olvidarse durante una migración y entonces provocan perfiles que ya no pueden configurarse.

**CAA:** No está directamente relacionado con el correo, pero determina qué autoridad de certificación puede emitir un certificado para `mta-sts.example.com` o para el nombre del servidor de correo.

**Zonas split-horizon:** Cuando una zona DNS interna tiene el mismo nombre que la pública, los registros de correo suelen no existir internamente. Los sistemas internos que realizan una comprobación SPF o DKIM llegan entonces a resultados distintos de los del exterior. Por ello, con cada cambio en los registros de correo debe plantearse si la zona interna también debe actualizarse.

## Algunas pruebas breves

Realice todas las consultas deliberadamente contra un resolutor público, para que no responda la caché interna ni una zona split-horizon:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

Contra el servidor autoritativo, para evitar completamente la caché:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

En Windows sin `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

Para la evaluación completa, incluido el recuento de consultas SPF, la búsqueda de selectores DKIM y la comprobación de alignment, esta página ofrece el [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), que comprueba un dominio de una sola vez frente a todos los registros descritos aquí.

Sin embargo, la prueba más significativa sigue siendo un mensaje real. Envíe un correo a un buzón de un gran proveedor y examine la línea `Authentication-Results` de la cabecera. En una sola línea muestra el resultado real de SPF, DKIM y DMARC, y sustituye cualquier teoría sobre el archivo de zona.

## Orden durante una migración

Al cambiar de proveedor de correo, esta secuencia ha demostrado ser eficaz:

1. Reducir el TTL de todos los registros afectados a 300 segundos, al menos un día antes.
2. Publicar los selectores DKIM del nuevo proveedor mientras los antiguos sigan activos.
3. Ampliar SPF con el nuevo proveedor sin eliminar el antiguo y recalcular el límite de consultas.
4. En MTA-STS, adaptar la política a los nuevos nombres MX y aumentar el `id` antes de cambiar los registros MX.
5. Cambiar los MX y observar la entrega.
6. Solo después de varios días sin incidencias, eliminar los includes SPF y selectores DKIM antiguos.
7. Restablecer el TTL.

El problema más frecuente en este proceso es adelantar demasiado el paso 6: se eliminan las entradas antiguas junto con el cambio y todo lo que aún circula por la ruta anterior falla en la comprobación del remitente.

## Conclusión

Los registros de correo se diferencian de todas las demás entradas DNS en que un error no tiene por qué ser evidente. Un registro A incorrecto genera un ticket en cuestión de minutos; en cambio, un registro SPF duplicado o una clave DKIM con un carácter de más produce una tasa de entrega que disminuye lentamente durante semanas.

Tres reglas evitan la mayoría de estos casos. Primera: antes de crear un registro, comprobar qué existe ya, en lugar de poner un segundo al lado. Segunda: después de cada cambio, verificar contra un resolutor público y comparar el valor carácter por carácter con la plantilla, no solo visualmente. Tercera: en los cambios, publicar siempre primero lo nuevo, después conmutar y luego eliminar lo antiguo. Quien respete este orden siempre tendrá una vía de vuelta con los registros de correo.

## Fuentes

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Define, entre otras cosas, el límite de 255 bytes de una sola `character-string` en registros TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Establece en la sección 10.3 que el destino de un registro MX no puede ser un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Límite de diez mecanismos de consulta, límite de consultas vacías, eliminación del tipo RR SPF y recomendación de no usar el mecanismo `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Estructura del registro de clave en `_domainkey`, significado del selector y del `p=` vacío.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Especificación DMARC actual de mayo de 2026, sustituye RFC 7489; eliminación de `pct`, nueva etiqueta `np`, recorrido de árbol en lugar de Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Formato y entrega de informes agregados, incluida la autorización de dominios destinatarios externos.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identificación de dominios que no aceptan correo.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Registro DNS, archivo de política, significado de `id` y de los modos `testing` y `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Estructura del registro `_smtp._tls` y de los informes.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Registros TLSA para SMTP y el requisito de una zona firmada con DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Estado actual de la especificación BIMI, que sigue sin ser un RFC.

12.  [Google: Directrices para remitentes de correo electrónico](https://support.google.com/a/answer/81126): Requisitos para remitentes, entre ellos el PTR obligatorio para direcciones IPv6 que envían correo y las condiciones vigentes desde febrero de 2024 para remitentes masivos.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Requisitos para remitentes a partir de 5000 mensajes al día, vigentes desde mayo de 2025.
