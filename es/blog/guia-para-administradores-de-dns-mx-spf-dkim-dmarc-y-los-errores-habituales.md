---
title: "Guía para administradores de DNS: MX, SPF, DKIM, DMARC y los errores habituales"
navTitle: "Registros DNS de correo electrónico"
description: "Quien administra una zona suele recibir los registros de correo ya preparados y solo tiene que publicarlos. Lo que suele salir mal: el límite de 255 bytes en DKIM, registros SPF duplicados, el límite de consultas, un MX en un CNAME, el sufijo de zona añadido automáticamente y políticas que ya nadie aplica."
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
translationSourceHash: 63c8a888f2ebd4548bd4222c4273896228649bf02f0406082ec337194af65280
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:07:40.281Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/guia-para-administradores-de-dns-mx-spf-dkim-dmarc-y-los-errores-habituales
---

# Guía para administradores de DNS: MX, SPF, DKIM, DMARC y los errores habituales

Quien administra una zona DNS rara vez recibe los registros de correo escritos por sí mismo. El equipo de correo, un proveedor o una agencia de marketing envía una línea con la indicación de que «solo hay que publicarla». Precisamente de ahí surgen la mayoría de los errores, porque los registros de correo son el tipo de registro en el que una errata puede tener dos consecuencias completamente distintas. O bien la entrega falla de inmediato y alguien avisa en cuestión de minutos, o bien sigue funcionando sin cambios y únicamente la comprobación del remitente falla silenciosamente. El segundo caso suele pasar inadvertido durante meses, hasta que un gran destinatario pone el dominio en cuarentena.

Desde que Google y Yahoo endurecieron sus requisitos para remitentes masivos en febrero de 2024 y Microsoft hizo lo mismo en mayo de 2025, la tolerancia hacia los dominios configurados a medias se ha reducido. SPF, DKIM y un registro DMARC ya no son opcionales para remitentes a partir de cierto volumen, sino un requisito para la entrega.

Todos los ejemplos de este artículo utilizan `example.com` y selectores genéricos. Los valores mostrados están abreviados para que sigan siendo legibles.

## Reglas aplicables a todos los registros de correo

### El límite de 255 bytes en los registros TXT

Según RFC 1035, un registro TXT consta de una o varias `character-strings`, y cada una de estas cadenas puede contener como máximo 255 bytes. El registro completo puede ser más largo, pero debe dividirse entonces en varias cadenas. Los sistemas de evaluación vuelven a unir estas partes sin separadores.

Esto resulta relevante en la práctica precisamente en un punto: las claves DKIM de 2048 bits. Su valor Base64 tiene unos 400 caracteres y no cabe en una sola cadena.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La mayoría de los sistemas de gestión DNS realizan esta división automáticamente cuando se introduce el valor mediante el campo de entrada normal. Quien añada comillas manualmente debe respetar el límite con exactitud. Un valor dividido con un espacio en el punto de unión da como resultado una clave que existe sintácticamente, pero que ya no encaja criptográficamente.

Es importante comprobarlo después, ya que una clave ensamblada incorrectamente parece completamente normal en la interfaz:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `+short` | Muestra solo los valores de los registros, sin cabeceras ni metadatos |
| `TXT selector1._domainkey.example.com` | Tipo de registro y nombre del registro de clave DKIM |
| `tr -d '" '` | Elimina comillas y espacios, uniendo las cadenas parciales tal como las lee un verificador |
| `wc -c` | Cuenta los caracteres del valor ensamblado; la longitud debe coincidir con la plantilla |

</details>

### Un registro por finalidad

SPF y DMARC se definen de tal forma que solo puede haber un registro adecuado por nombre. En SPF, dos registros `v=spf1` provocan un `permerror`, por lo que la comprobación se considera fallida, no superada. En DMARC, los destinatarios ignoran por completo el dominio cuando hay varios registros que empiezan por `v=DMARC1`: en lugar de aplicarse una política estricta, no se aplica ninguna.

Este es, con diferencia, el error más frecuente en zonas que han ido creciendo. Se conecta un nuevo proveedor, alguien añade «su» registro SPF en vez de ampliar el existente y, desde ese momento, la comprobación falla para todos los remitentes. Por tanto, antes de crear cualquier registro nuevo es imprescindible comprobar qué existe ya:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `+short` | Muestra solo los valores de los registros, sin cabeceras ni metadatos |
| `TXT` | Tipo de registro consultado |
| `example.com`, `_dmarc.example.com` | Nombres consultados: el propio dominio para SPF y el nombre `_dmarc` para DMARC |
| `grep -i spf1` | Filtra la línea SPF; `-i` ignora mayúsculas y minúsculas |

</details>

Para DKIM ocurre lo contrario: se prevé un registro por selector, y varios selectores en paralelo son lo habitual, ya que cada servicio de envío aporta su propia clave.

### El sufijo de zona en las interfaces web

En Infoblox, en DNS de Windows y en prácticamente todas las interfaces de hosting, el nombre de la zona se añade automáticamente al nombre introducido. Quien introduzca el nombre completo en el campo «Nombre» obtendrá un registro que será el doble de largo de lo previsto:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

En el archivo de zona, el equivalente es el punto final que falta. `mail.example.com` sin punto al final es un nombre relativo y se completa con el nombre de la zona; `mail.example.com.` con punto es absoluto. En destinos MX y CNAME, ese único punto determina si el dominio es accesible.

### Copiar y pegar es la fuente de errores más frecuente

Los valores de los registros de correo casi nunca se escriben a mano, sino que se copian de un PDF, un ticket, una celda de Excel o un chat. Esto provoca daños que permanecen invisibles en el campo de entrada:

- Un `p=` duplicado al principio de la clave DKIM porque el prefijo se añadió dos veces al ensamblar. El valor `v=DKIM1;k=rsa;p=p=MIIBIjAN...` aparece habitualmente en la práctica y genera una clave inutilizable.
- Comillas tipográficas de Word en lugar de comillas rectas.
- Espacios no separables de diseños PDF que parecen normales.
- Saltos de línea en medio del bloque Base64 cuando el valor ocupaba varias líneas en el PDF.

Base64 solo conoce los caracteres A a Z, a a z, 0 a 9, `+`, `/` y `=` como caracteres de relleno. Cualquier otra cosa en la parte `p=` es un error. Un filtro breve antes de introducirlo ahorra búsquedas de errores posteriores:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `'%s' "$KEY"` | Muestra el valor de la clave sin modificar y sin salto de línea final añadido |
| `tr -d 'A-Za-z0-9+/='` | Elimina todos los caracteres válidos para Base64; solo quedan caracteres ajenos |
| `wc -c` | Cuenta los caracteres restantes |

</details>

Si el resultado no es `0`, la clave contiene caracteres ajenos.

### Reducir el TTL antes de los cambios

Antes de cualquier cambio planificado de un registro MX, SPF o DKIM, el TTL debe reducirse durante unas horas a un valor bajo, normalmente 300 segundos. De lo contrario, el valor antiguo puede permanecer un día o más en resolutores ajenos, según la zona, y una reversión tardará lo mismo. Tras el cambio y un periodo de observación, el TTL vuelve a establecerse en su valor normal.

## MX

El registro MX determina qué host acepta correo para el dominio. Hay dos reglas que se incumplen con frecuencia.

**El destino debe ser un nombre de host con un registro A o AAAA.** No se permite ni una dirección IP ni un CNAME. RFC 2181 establece expresamente que el destino de un registro MX no puede ser un alias. En la práctica funciona de todos modos con muchos destinatarios, pero no con otros, lo que provoca problemas que aparentemente solo afectan a remitentes concretos.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**El número es una preferencia, no una ponderación.** Primero se intenta el valor más bajo. Un segundo MX con un número alto solo tiene sentido si ese sistema conoce el mismo filtro de destinatarios. Las entradas MX de respaldo en sistemas sin comprobación de destinatarios son un objetivo habitual de spam, porque los atacantes dirigen deliberadamente el tráfico a la entrada más débil.

Los dominios que solo envían o que no tienen relación alguna con el correo reciben un MX nulo según RFC 7505. Indica que el dominio no acepta correo y garantiza un rechazo inmediato e inequívoco en lugar de tiempos de espera:

```text
example.com.  IN  MX  0 .
```

Sin embargo, el MX nulo no sustituye un registro SPF ni DMARC. No recibir correo no significa que nadie envíe en su nombre. Los subdominios aparcados, en particular, se utilizan para suplantación porque rara vez alguien los vigila.

## A, AAAA, PTR y el nombre HELO

El registro PTR de la dirección IP saliente no está en su zona, sino en la zona `in-addr.arpa` del proveedor al que pertenece el bloque de direcciones. Por eso se solicita al proveedor y no se configura directamente. Muchos grandes destinatarios exigen que el PTR y el registro directo correspondiente coincidan, es decir, que el nombre del PTR vuelva a resolver a la misma dirección IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `+short` | Muestra solo los valores de los registros, sin cabeceras ni metadatos |
| `-x 192.0.2.10` | Consulta inversa: dig genera por sí mismo el nombre PTR en la zona `in-addr.arpa` |
| `A mail1.example.com` | Consulta directa del nombre del PTR para comprobar que el recorrido vuelve a la misma dirección IP |

</details>

El nombre que su servidor de correo indica en HELO o EHLO debería ser el mismo y también debe poder resolverse. Un gateway que se presenta como `localhost.localdomain` o con un nombre interno recibe una valoración peor por parte de los grandes destinatarios.

Hay que tener cuidado al añadir un registro AAAA. En cuanto el servidor de correo sea accesible y envíe a través de IPv6, se aplican los mismos requisitos que para IPv4, e incluso algunos más estrictos. Google exige un PTR válido para direcciones IPv6 emisoras. Si falta, el envío se rechaza, aunque funcionara perfectamente por IPv4. Por ello, un registro AAAA en el servidor de correo nunca es un simple cambio de DNS.

## SPF

SPF determina qué sistemas pueden enviar en nombre del dominio. El registro se publica como TXT en el propio dominio.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### El límite de consultas

La evaluación de un registro SPF puede desencadenar como máximo diez mecanismos que consulten DNS. Se cuentan `include`, `a`, `mx`, `ptr`, `exists` y `redirect`, de forma recursiva: cada `include` añade las consultas del registro incluido. No se cuentan `ip4`, `ip6` ni `all`.

Si se supera el límite, el resultado es un `permerror`. Para DMARC, esto significa que SPF no ha superado la comprobación, independientemente de que el servidor remitente estuviera realmente autorizado. Lo complicado es que el error suele surgir sin intervención propia, porque un proveedor incluido amplía su registro. El registro propio no ha cambiado, pero la entrega se resiente igualmente.

Además, solo se permiten dos «void lookups», es decir, consultas sin resultado. Un `include` a un dominio que ya no existe cuenta para este límite. Por tanto, deben eliminarse las referencias a proveedores retirados, no conservarse por precaución.

### Lo que no debe incluirse en un registro SPF

- **`ptr`** está especificado, pero se considera obsoleto desde RFC 7208 y no debe utilizarse. Los sistemas de evaluación pueden ignorarlo.
- **`+all`** autoriza a cualquier remitente y, por tanto, es más perjudicial que no tener ningún registro SPF.
- **`?all`** es neutral y, por tanto, prácticamente no tiene valor para DMARC.
- **Un registro independiente de tipo SPF (tipo 99)** ya no es necesario. Está eliminado desde RFC 7208; SPF se publica exclusivamente en TXT.

Entre `~all` (softfail) y `-all` (hardfail), la decisión depende de cuán completos estén identificados los canales de envío. Mientras haya dudas al respecto, `~all` es la elección correcta. Quien ya aplique DMARC y evalúe los informes puede pasar a `-all`.

### Los subdominios no heredan nada

Un registro SPF en `example.com` no se aplica a `newsletter.example.com`. Cada subdominio que envíe necesita su propio registro. Para todos los demás, se recomienda una entrada comodín que deje claro que desde allí no se envía nada:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Atención: un comodín TXT también responde a consultas de nombres como `_dmarc.sub.example.com`, siempre que no exista allí un registro explícito. Por lo general no supone un problema, pero puede complicar la búsqueda de errores porque cada consulta TXT obtiene una respuesta.

### SPF flattening

Las herramientas que resuelven todas las referencias `include` y las sustituyen por las direcciones IP subyacentes solucionan el límite de consultas a costa de la mantenibilidad. Si el proveedor cambia sus direcciones, el envío falla y nadie lo nota porque aparentemente todo sigue correcto en el registro propio. Por ello, quien elija esta vía necesita una comparación automatizada que verifique periódicamente la lista con la fuente. Como trabajo manual puntual, el procedimiento acabará fallando tarde o temprano.

## DKIM

DKIM firma los mensajes salientes. La clave pública se encuentra en `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

El selector se puede elegir libremente y lo determina el sistema de envío. Un nombre descriptivo con fecha facilita mucho más la rotación posterior que `s1` y `s2`.

### Delegación mediante CNAME

Cuando el servicio de envío lo ofrece, debe preferirse la variante CNAME frente a la entrada directa:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

El proveedor puede entonces rotar su clave de forma autónoma, sin que nadie tenga que intervenir en su zona. De otro modo, esta rotación suele quedar pendiente porque requiere coordinación entre dos equipos. No obstante, un CNAME excluye cualquier otro registro con el mismo nombre; es una regla básica del DNS, no una particularidad de DKIM.

### Rotación sin interrupciones

Al cambiar la clave, primero se publica el nuevo selector, después el servidor remitente cambia a él y solo entonces se elimina el registro anterior. Quien borre inmediatamente la clave antigua invalida las firmas de todos los mensajes que aún estén en tránsito o en colas y hace imposibles las comprobaciones posteriores. Es adecuado dejar unos días entre el cambio y la eliminación.

Por cierto, un registro con `p=` vacío no es una entrada defectuosa, sino la forma especificada de marcar una clave como retirada.

### Longitud de la clave

1024 bits se consideran obsoletos; 2048 bits son el estándar. Las claves RSA más grandes no aportan prácticamente ninguna ventaja adicional y solo aumentan la probabilidad de que un sistema intermedio no procese correctamente el registro.

## DMARC

DMARC combina SPF y DKIM con una instrucción sobre qué hacer cuando una comprobación no se supera y devuelve informes. El registro se encuentra en `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Desde mayo de 2026, con RFC 9989 y las especificaciones de informes RFC 9990 y RFC 9991, está vigente la versión revisada, que sustituye a RFC 7489. Para la práctica, son importantes tres cambios:

- **`pct` se ha eliminado.** Ya no existe la introducción gradual mediante un porcentaje. En su lugar se utiliza `t=y`, que identifica el dominio como en fase de prueba: los informes siguen funcionando, pero la política no debe aplicarse.
- **`np` es nuevo.** Establece la política para subdominios inexistentes y cierra así una brecha que los atacantes aprovechan con frecuencia, ya que los subdominios inventados hasta ahora solo quedaban cubiertos por `sp`. Si no se especifica expresamente, `np` adopta el valor de `sp`.
- **La Public Suffix List ha sido sustituida por un `Tree Walk`.** El dominio organizativo ya no se determina a partir de una lista mantenida externamente, sino mediante consultas DNS escalonadas a lo largo del árbol de nombres. Esto modifica notablemente la evaluación en espacios de nombres grandes con muchos niveles.

### El alignment es el verdadero núcleo

DMARC no se supera porque SPF o DKIM se hayan superado técnicamente, sino solo si al menos uno de ambos coincide además con el dominio visible del remitente en la cabecera `From`. SPF se comprueba frente al dominio del remitente del sobre, que suele diferir en reenvíos, servicios de newsletters y sistemas de tickets. Precisamente por eso, los mensajes con SPF válido a veces no superan la comprobación DMARC.

Con `adkim=r` y `aspf=r` (relajado, el estándar), basta con que coincidan en el nivel del dominio organizativo. `s` exige igualdad exacta, incluido el subdominio, y en la práctica casi siempre falla en alguno de los canales de envío.

### Las direcciones de informes externas necesitan autorización

Si los informes deben enviarse a una dirección fuera del propio dominio, por ejemplo a un servicio de análisis DMARC, el dominio receptor debe autorizarlo. Sin este registro, muchos destinatarios simplemente no envían nada y el análisis permanece vacío aunque todo parezca correcto en el registro propio:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Esta entrada la crea el operador de la zona de destino, no usted. En servicios comerciales sucede automáticamente, pero no en un buzón de recopilación autogestionado en otro dominio propio.

### Errores de sintaxis habituales

Los nombres de etiqueta y los valores de política deben escribirse en minúsculas; `p=Reject` no es válido. Entre las etiquetas debe haber un punto y coma; si falta un separador, el resto de la línea queda sin efecto. Además, `p` debe ser la primera etiqueta después de `v`. Un registro que solo consta de `v=DMARC1; rua=...` no contiene ninguna política y está incompleto.

### El despliegue

`p=none` es un estado de medición, no un objetivo. No cambia cómo tratan los destinatarios sus correos y sirve únicamente para identificar todos los canales de envío legítimos mediante los informes. Quien no pase de `quarantine` a `reject` en unos pocos meses tras la introducción habrá realizado el esfuerzo sin obtener la protección. La vertiente organizativa de este proceso, incluida una propuesta de decisión, es un tema aparte y está descrita en el blueprint de DMARC.

## MTA-STS y TLS-RPT

SMTP cifra de forma oportunista: si el extremo remoto ofrece STARTTLS, se cifra; de lo contrario, no. Un atacante en posición de manipular el tráfico puede eliminar el anuncio de STARTTLS y mantener así la conexión en texto claro. MTA-STS cierra esta brecha para los dominios receptores.

MTA-STS consta de dos partes, y solo una de ellas está en el DNS:

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

Las fuentes de error están casi todas fuera de la zona:

- **El `id` debe cambiar con cada modificación de política.** Es la única indicación para los sistemas remitentes de que deben obtener una política nueva. Quien modifique el archivo y mantenga el `id` seguirá trabajando contra copias en caché hasta que expire `max_age`.
- **La lista MX de la política y los registros MX deben coincidir.** Los remitentes con `mode: enforce` rechazan un MX nuevo que falte en la política. Por ello, en migraciones la política debe adaptarse antes del cambio de MX.
- **Primero `mode: testing`.** En este modo, las infracciones solo se notifican, no se aplican. El cambio a `enforce` se realiza cuando los informes estén limpios.
- **Un registro CAA puede bloquear la emisión del certificado para el host de la política** si se ha indicado allí una autoridad de certificación distinta de la utilizada.

TLS-RPT proporciona los informes correspondientes y es un único registro:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT también es útil sin MTA-STS, porque hace visibles por primera vez los fallos de cifrado del transporte.

## DANE

DANE logra el mismo objetivo que MTA-STS, pero ancla la confianza en DNS en lugar de en la PKI web. Requiere una zona firmada de extremo a extremo con DNSSEC; sin DNSSEC, un registro TLSA no tiene efecto.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Lo decisivo en la operación: con cada cambio de certificado, el registro TLSA debe coincidir previamente. El procedimiento habitual publica el nuevo hash en paralelo al antiguo, cambia después el certificado y elimina posteriormente la entrada antigua. Quien invierta este orden hará que el servidor de correo sea inaccesible para todos los remitentes que comprueban DANE, entre ellos los grandes proveedores de habla alemana. En Suiza, DANE es mucho menos frecuente que MTA-STS, generalmente por la falta de firma DNSSEC de la zona.

## BIMI

BIMI muestra el logotipo de la marca en la bandeja de entrada y es el único mecanismo tratado aquí que aún no es un RFC, sino que sigue siendo un borrador de Internet.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Los requisitos son elevados: una política DMARC aplicada con `quarantine` o `reject`, un logotipo en formato SVG Tiny Portable/Secure y, para la mayoría de los proveedores, un Verified Mark Certificate de pago. Por tanto, BIMI no es un mecanismo de seguridad, sino una cuestión de visibilidad, y debe ir al final del orden, no al principio.

## Otros registros relacionados

**Autodiscover y SRV:** Los entornos Exchange utilizan `autodiscover.example.com` como CNAME o un registro SRV `_autodiscover._tcp.example.com`. Ambos afectan a la configuración de clientes y no al flujo de correo, pero suelen olvidarse en las migraciones y luego provocan que ya no se puedan configurar perfiles.

**CAA:** No tiene relación directa con el correo, pero determina qué autoridad de certificación puede emitir un certificado para `mta-sts.example.com` o el nombre del servidor de correo.

**Zonas split-horizon:** Cuando una zona DNS interna tiene el mismo nombre que la pública, los registros de correo a menudo no existen internamente. Los sistemas internos que realizan una comprobación SPF o DKIM obtienen entonces resultados distintos del exterior. Por tanto, cada cambio en los registros de correo debe incluir la pregunta de si hay que actualizar la zona interna.

## Algunas pruebas rápidas

Realice deliberadamente todas las consultas contra un resolutor público, para que no responda la caché interna ni una zona split-horizon:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `@1.1.1.1` | Envía la consulta a este resolutor en lugar del configurado en `/etc/resolv.conf` |
| `+short` | Muestra solo los valores de los registros, sin cabeceras ni metadatos |
| `MX`, `TXT` | Tipos de registro consultados |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | Los nombres definidos para DMARC, DKIM, MTA-STS y TLS-RPT bajo el dominio |

</details>

Contra el servidor autoritativo, para evitar por completo la caché:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `NS example.com` | Determina los servidores de nombres autoritativos de la zona |
| `@ns1.example.com` | Envía la consulta posterior directamente a uno de estos servidores autoritativos |
| `+norecurse` | No activa el bit Recursion Desired; el servidor responde solo con sus propios datos de zona, no desde una caché |

</details>

En Windows sin `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-type=TXT` | Tipo de registro que se consulta |
| `_dmarc.example.com` | Nombre consultado |
| `1.1.1.1` | Resolutor que se debe utilizar en lugar del configurado para todo el sistema |

</details>

Para la evaluación completa, incluido el recuento de consultas SPF, la búsqueda de selectores DKIM y la comprobación de alignment, esta página ofrece el [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), que comprueba un dominio de una sola vez frente a todos los registros descritos aquí.

Sin embargo, la prueba más reveladora sigue siendo un mensaje real. Envíe un correo a un buzón de un gran proveedor y examine la línea `Authentication-Results` de la cabecera. Muestra en una sola línea el resultado real de SPF, DKIM y DMARC, y sustituye cualquier teoría sobre el archivo de zona.

## Orden en una migración

Al cambiar de proveedor de correo, esta secuencia ha dado buenos resultados:

1. Reducir el TTL de todos los registros afectados a 300 segundos, como mínimo un día antes.
2. Publicar los selectores DKIM del nuevo proveedor mientras los antiguos sigan presentes.
3. Ampliar SPF con el nuevo proveedor sin eliminar el antiguo y recalcular el límite de consultas.
4. En MTA-STS, adaptar la política a los nuevos nombres MX y aumentar el `id` antes de cambiar los registros MX.
5. Cambiar los MX y observar la entrega.
6. Solo después de varios días sin incidencias, eliminar los includes SPF y selectores DKIM antiguos.
7. Restablecer el TTL.

El problema más frecuente en esta secuencia es adelantar demasiado el paso 6: se eliminan las entradas antiguas junto con el cambio, y todo lo que aún circula por la ruta anterior falla en la comprobación del remitente.

## Conclusión

Los registros de correo se diferencian de todas las demás entradas DNS en que un error no tiene por qué llamar necesariamente la atención. Un registro A incorrecto genera un ticket en cuestión de minutos; en cambio, un registro SPF duplicado o una clave DKIM con un carácter de más provoca una tasa de entrega que disminuye lentamente durante semanas.

Tres reglas evitan la mayoría de estos casos. Primera: antes de añadir un registro nuevo, comprobar qué existe ya en lugar de colocar un segundo al lado. Segunda: tras cada cambio, verificar contra un resolutor público y comparar el valor carácter por carácter con la plantilla, no solo visualmente. Tercera: en los cambios, publicar siempre primero lo nuevo, luego cambiar y después eliminar lo antiguo. Quien respete este orden tendrá siempre una vía de retorno con los registros de correo.

## Fuentes

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Define, entre otras cosas, el límite de 255 bytes de una `character-string` individual en registros TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Establece en la sección 10.3 que el destino de un registro MX no puede ser un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Límite de diez mecanismos de consulta, límite de void lookups, eliminación del tipo RR SPF y desaconseja el mecanismo `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Estructura del registro de clave bajo `_domainkey`, significado del selector y del `p=` vacío.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Especificación DMARC actual de mayo de 2026, sustituye RFC 7489; eliminación de `pct`, nueva etiqueta `np`, Tree Walk en lugar de Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Formato y entrega de los informes agregados, incluida la autorización de dominios destinatarios externos.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identificación de dominios que no aceptan correo.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Registro DNS, archivo de política, significado de `id` y de los modos `testing` y `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Estructura del registro `_smtp._tls` y de los informes.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Registros TLSA para SMTP y el requisito de una zona firmada con DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Estado actual de la especificación BIMI, que sigue sin ser un RFC.

12.  [Google: Directrices para remitentes de correo electrónico](https://support.google.com/a/answer/81126): Requisitos para remitentes, entre ellos la obligación de PTR para direcciones IPv6 emisoras y las normas aplicables a remitentes masivos desde febrero de 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Requisitos para remitentes a partir de 5000 mensajes al día, vigentes desde mayo de 2025.
