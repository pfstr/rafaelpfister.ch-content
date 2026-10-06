---
title: "Renovar el certificado de TotemoMail: clave de 4096 bits, importación PKCS#12 y reinicio por nodo"
navTitle: "Renovar certificado"
description: "El diálogo de solicitud de TotemoMail solo genera claves de 2048 bits sin nombres alternativos. Sin embargo, muchas CA internas ya solo firman con 4096 bits. Por ello, la clave y la solicitud se crean con openssl; después se realiza el pedido a la entidad PKI, la importación PKCS#12, la vinculación a los conectores y el reinicio por nodo."
date: "2026-10-06"
kategorie: "TotemoMail"
timeToRead: "12 min de lectura"
themen:
  - totemomail
  - e-mail-verschluesselung
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "renovar-el-certificado-de-totemomail-clave-de-4096-bits-importacion-pkcs-12-y-reinicio-por-nodo"
translationId: "article-1e59c4ee01e408a3"
translationOf: totemomail-zertifikat-erneuern
url: https://rafaelpfister.ch/es/blog/renovar-el-certificado-de-totemomail-clave-de-4096-bits-importacion-pkcs-12-y-reinicio-por-nodo
translationSourceHash: 3ba1992d93abcd33fda47f86cb3b1ea4c8884c36fcfa41fa5c098f4aff9dff32
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:33:35.321Z
translationReview: automatic
---

# Renovar el certificado de TotemoMail: clave de 4096 bits, importación PKCS#12 y reinicio por nodo

Renovar el certificado de servidor de un clúster TotemoMail (hoy Kiteworks Email Protection Gateway) parece una tarea rutinaria: generar la solicitud en la interfaz de administración, hacer que se firme e importar la respuesta. En la práctica, este procedimiento suele fallar ya en el primer paso. El cuadro de diálogo «New PKCS#10» genera obligatoriamente una clave de 2048 bits y no ofrece ningún campo para nombres alternativos (Subject Alternative Names). Muchas autoridades de certificación internas ya solo firman con 4096 bits, y los extremos TLS actuales ni siquiera reconocen como válido para un nombre un certificado sin nombres alternativos.

El siguiente procedimiento ha demostrado su eficacia en una sustitución realizada en otoño de 2026 en preproducción y producción: clave y solicitud con openssl en el gateway, pedido a la entidad PKI, importación como PKCS#12, vinculación a los conectores y reinicio nodo por nodo. Durante la sustitución aparecieron varias particularidades, entre ellas un error de la interfaz que se produce al desvincular el certificado antiguo.

## Dos certificados con finalidades distintas

Un clúster TotemoMail detrás de Exchange Online suele necesitar dos tipos de certificados.

| Tipo | Finalidad | Emisor | Validez |
|---|---|---|---|
| Interno | Interfaz web, administración, conexiones SMTP internas. Incluye los nombres internos de los nodos. | PKI interna, por ejemplo Active Directory Certificate Services | elegible libremente, normalmente de 12 a 13 meses |
| Público | Trayecto entre Exchange Online y el gateway, cuando Exchange Online debe comprobar el certificado | autoridad de certificación pública | desde el 15.03.2026, como máximo 200 días; desde el 15.03.2027, como máximo 100 días |

Un único certificado no funciona para ambas finalidades. Las autoridades de certificación públicas no emiten nombres internos ni nombres cortos sin dominio, y todo certificado público aparece en los registros de Certificate Transparency. A la inversa, Exchange Online no confía en ninguna cadena interna. El artículo [Bucle de correo con gateway de cifrado detrás de EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online) describe cómo Exchange Online trata el trayecto hacia un gateway de cifrado.

Los pasos siguientes se aplican al certificado interno. Para el público, el proceso es idéntico hasta la importación; las diferencias se explican en la última sección.

## Resumen del procedimiento

1. Generar la clave y la solicitud con openssl en un nodo.
2. Solicitar el certificado a la entidad PKI.
3. Comprobar el certificado entregado.
4. Combinar certificado, clave e intermedia en un archivo PKCS#12 y llevarlo al equipo con el navegador.
5. Importarlo en la interfaz de TotemoMail y vincularlo a los conectores.
6. Reiniciar cada nodo individualmente y comprobarlo.
7. Probar el flujo de correo y, después, limpiar.

Planifique al menos tres semanas antes de la caducidad. Mientras el certificado antiguo siga siendo válido, hay vuelta atrás. Si dispone de preproducción en su entorno, realice primero el cambio allí y utilice esa ejecución como referencia.

## Paso 1: clave y solicitud con openssl

Trabaje en un nodo del clúster con su usuario personal, no con la cuenta de servicio `totemo`. Así podrá recuperar posteriormente el archivo PKCS#12 directamente mediante `scp`. No se necesitan permisos de root para ninguno de los pasos.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

La configuración contiene el titular, los usos y todos los nombres alternativos. Los nombres del ejemplo son marcadores de posición: tres nodos y los nombres de servicio con los que se accede internamente al clúster.

```bash
cat > intern.cnf <<'EOF'
[ req ]
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
C  = CH
O  = Beispiel AG
CN = SecureMail

[ ext ]
basicConstraints = critical, CA:FALSE
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt

[ alt ]
DNS.1 = gw01.intern.example.ch
DNS.2 = gw02.intern.example.ch
DNS.3 = gw03.intern.example.ch
DNS.4 = securemail.intern.example.ch
EOF
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Entrada | Efecto |
|---|---|
| `default_md = sha256` | Algoritmo hash para la firma de la solicitud |
| `prompt = no` | Adoptar los valores del archivo en lugar de solicitarlos interactivamente |
| `req_extensions = ext` | Escribir en la solicitud las extensiones de la sección `[ ext ]` |
| `basicConstraints = critical, CA:FALSE` | Certificado final, no una autoridad de certificación |
| `keyUsage` | Clave para firma e intercambio de claves, como es habitual para servidores TLS |
| `extendedKeyUsage = serverAuth, clientAuth` | Autenticación de servidor y cliente. El gateway actúa como servidor en algunos trayectos y como cliente en otros. |
| `subjectAltName = @alt` | Nombres alternativos de la sección `[ alt ]` |

</details>

Incluya únicamente nombres completos. Muchas entidades de registro rechazan los nombres cortos sin dominio, y no debería confiar en que acaben en el certificado.

Genere la clave sin frase de contraseña y protéjala mediante permisos de archivo. Solo permanecerá en el nodo hasta la importación y después se eliminará. Una frase de contraseña en una clave que existe durante pocos días aporta poca protección y crea un riesgo nuevo: si se pierde, la clave queda inutilizable y el certificado debe emitirse de nuevo.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `genpkey -algorithm RSA` | Generar una nueva clave privada de tipo RSA |
| `-pkeyopt rsa_keygen_bits:4096` | Longitud de clave de 4096 bits |
| `-out intern.key` | Archivo para la clave, sin frase de contraseña |
| `req -new` | Generar una nueva solicitud de certificado (CSR) |
| `-key intern.key` | Usar una clave existente |
| `-config intern.cnf` | Titular, extensiones y nombres de la configuración |
| `-out intern.csr` | Archivo para la solicitud |

</details>

Compruebe la solicitud antes de que salga de la máquina:

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Se esperan `verify OK`, el titular correcto, `4096 bit` y el número de sus nombres. Anote también la huella digital de la clave pública. Con ella podrá asignar inequívocamente el certificado entregado a esta clave más adelante, incluso si dos solicitudes tienen el mismo titular:

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Paso 2: pedido a la entidad PKI

La entidad que emite el certificado necesita estos datos para cada solicitud:

- la CSR como texto
- la huella digital SHA-256 de la clave pública
- el tipo: interno o público y, si hay varios entornos, cuál
- la lista de nombres para copiar, un nombre por línea
- longitud de clave 4096, uso ampliado Server Authentication y Client Authentication
- la fecha en la que debe estar disponible el certificado, como mínimo dos semanas antes de la caducidad

En una sustitución con varios entornos y tipos de certificado, conviene incluir una visión general al principio del correo: qué certificados existen, cuáles se emiten internamente y cuáles de forma pública, y en qué orden se necesitan. No hace falta ningún cambio en el sistema para la emisión. Por tanto, los certificados de todos los entornos pueden emitirse simultáneamente, aunque se instalen uno tras otro.

Durante la sustitución de otoño de 2026, se observaron varios puntos en la entidad de registro (RA) que probablemente se repitan de forma similar en muchos entornos:

- **La RA establece el titular por sí misma.** En el certificado solo figuraban el país, la organización y el Common Name, aunque la solicitud contenía unidad organizativa, localidad y cantón.
- **Sin nombres cortos.** Los nombres sin dominio faltaban en el certificado emitido.
- **Nombres copiados manualmente.** La RA no tomó los nombres alternativos de la CSR, sino que se introdujeron en el formulario. Un nombre llegó truncado. Por ello, siempre hay que comprobar la lista de nombres entregada.
- **Perfil incorrecto.** Una RA que gestiona tanto la CA interna como una CA pública emitió inicialmente solicitudes de certificados públicos a través del perfil interno. La CA interna figuraba como emisor. Tales certificados no sirven para el trayecto hacia Exchange Online.
- **Un solo nombre en certificados públicos individuales.** Un producto Single-Domain falla con una solicitud de dos nombres, por ejemplo con «Only one Subject Alternative Name is allowed». Sin embargo, un nombre es suficiente; véase la sección sobre el certificado público.

La entidad PKI debería revocar posteriormente los certificados generados por emisiones erróneas. Contienen una clave válida y, de lo contrario, siguen vigentes durante un año.

## Paso 3: comprobar la entrega

Guarde el certificado entregado en el mismo directorio que la clave, por ejemplo con `cat > intern.crt`, pegue el contenido y use `Strg+D`. A continuación, compruebe:

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Compruebe el emisor, la lista completa de nombres sin erratas ni duplicados y los dos usos. La comparación de las huellas digitales muestra si el certificado y la clave pertenecen juntos. Ambos valores deben coincidir:

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

En un certificado público se añaden dos comprobaciones. La OID de política `2.23.140.1.2.2` corresponde a un certificado validado por organización conforme a las reglas del CA/Browser Forum, y el certificado debe incluir pruebas de Certificate Transparency. Pocos minutos después de la emisión aparece con su nombre en crt.sh. Si falta alguna de las dos cosas, no es un certificado público, independientemente de cómo esté etiquetada la entrega.

## Paso 4: crear el archivo PKCS#12

TotemoMail importa el certificado y la clave juntos como PKCS#12. Para ello, obtenga el certificado de la intermedia emisora. La dirección figura en el certificado bajo `Authority Information Access`:

```bash
openssl x509 -in intern.crt -noout -ext authorityInfoAccess
curl -sS -o issuing.crt http://pki.example.ch/crt/Issuing-CA.crt
file issuing.crt
```

Si `file` no indica `PEM certificate`, el archivo está codificado en DER y debe convertirse:

```bash
openssl x509 -inform DER -in issuing.crt -out issuing.pem
mv issuing.pem issuing.crt
```

El Subject Key Identifier de la intermedia debe coincidir con el Authority Key Identifier del certificado. A continuación, cree el archivo. El comando solicita una contraseña de exportación que necesitará durante la importación:

```bash
openssl pkcs12 -export \
  -inkey intern.key \
  -in intern.crt \
  -certfile issuing.crt \
  -name "SecureMail" \
  -out intern.p12
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-export` | Crear archivo PKCS#12 |
| `-inkey intern.key` | Clave privada |
| `-in intern.crt` | Certificado emitido |
| `-certfile issuing.crt` | Certificados adicionales de la cadena, aquí la intermedia |
| `-name "SecureMail"` | Nombre visible de la entrada en el archivo |
| `-out intern.p12` | Archivo de destino, protegido con la contraseña de exportación |

</details>

Comprobación: deben aparecer dos entradas, el certificado y la intermedia:

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

La interfaz de TotemoMail se ejecuta en el navegador, normalmente en un jumphost. El archivo debe llevarse allí. En Windows está disponible el cliente OpenSSH con `scp`:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

Si generó la clave como `totemo`, estará en `/opt/totemomail`, y su usuario no podrá leerla. En ese caso, copie brevemente el archivo PKCS#12 a `/tmp`, recupérelo desde allí y elimínelo inmediatamente. Aunque el rodeo por el portapapeles con Base64 funciona, una línea de unas 10'000 caracteres es propensa a errores.

## Paso 5: importar y vincular en TotemoMail

En `Key Management`:

1. **`Issuer Certificates`**: importe la intermedia `issuing.crt`.
2. **`Own Server Certificates`**, botón **`Import`**: el diálogo ofrece dos vías. A la izquierda, «Import certificate» es para un certificado con clave, es decir, el archivo PKCS#12. A la derecha, «Import a PKCS#10 certificate reply» es solo para respuestas a solicitudes generadas por TotemoMail. Elija la opción de la izquierda e introduzca la contraseña de exportación en el segundo paso.
3. Abra el nuevo certificado (icono de lápiz). En `Connector` deberían estar marcados todos los conectores; en la instalación descrita, `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT` y `8444=AdminAPI`. `Host` está configurado en `*`.
4. Abra el certificado antiguo y desmarque todos los conectores salvo uno.

El punto 4 incluye el error que se produjo durante la sustitución: si desmarca **todos** los conectores del certificado antiguo, la interfaz muestra «Could not edit selected server certificate». El registro indica lo siguiente:

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

TotemoMail no admite un certificado de servidor sin conector. Por tanto, deje un conector, por ejemplo `8444=AdminAPI`, y elimine el certificado antiguo tras el período de observación. Eliminarlo en lugar de desmarcarlo funciona; el certificado termina en `Deleted Certificates`.

Para comprender la vinculación, resultan útiles tres observaciones:

- **La lista de conectores solo se aplica a los servicios web.** El puerto 25 no aparece ahí. Si no existe ningún certificado de tipo SMTPS, SMTP utiliza el certificado HTTPS. Por eso, tras la sustitución el puerto 25 muestra el certificado nuevo, aunque no haya ninguna marca para SMTPS en la lista.
- **Una asignación explícita tiene prioridad sobre el asterisco.** El conector que permanece en el certificado antiguo sigue mostrando el anterior, incluso aunque el nuevo esté registrado para todos con `*`.
- **La lista es común para el clúster.** Se ve igual en cada nodo, independientemente de si el nodo ya ha aplicado el cambio. Solo una consulta en los puertos muestra lo que un nodo presenta realmente.

## Paso 6: reiniciar nodo por nodo

TotemoMail carga los certificados al iniciarse. Tras la importación, todos los nodos siguen mostrando el certificado antiguo hasta que se reinician. Reinicie cada nodo individualmente, nunca todos a la vez, para que siempre queden nodos activos detrás del balanceador de carga. Empiece por los nodos en los que no haya iniciado sesión y deje para el final el nodo con la interfaz abierta.

En el nodo como `totemo`:

```bash
totemomail stop
totemomail start
```

Encontrará más indicaciones sobre la detención controlada en el artículo [Controles más importantes para administradores de TotemoMail](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

Los servicios web vuelven a estar disponibles tras aproximadamente un minuto; el servicio SMTP en el puerto 25, solo después de varios minutos. Por tanto, una respuesta vacía en el puerto 25 poco después del inicio aún no indica un error. Solo cuando un nodo haya cambiado en todos los puertos debe continuar con el siguiente.

La comprobación en todos los nodos y puertos:

```bash
for h in gw01 gw02 gw03; do
  for p in 25 443 8443; do
    if [ "$p" = 25 ]; then s="-starttls smtp"; else s=""; fi
    c=$(echo | openssl s_client -connect "$h:$p" $s 2>/dev/null |
        openssl x509 -noout -serial -enddate 2>/dev/null | tr '\n' ' ')
    printf "%-6s %-5s %s\n" "$h" "$p" "$c"
  done
done
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `s_client -connect host:port` | Establecer una conexión TLS con el servicio |
| `-starttls smtp` | En el puerto 25, realizar primero el diálogo SMTP y cambiar después a TLS con STARTTLS |
| `x509 -noout` | Leer el certificado sin mostrarlo |
| `-serial -enddate` | Mostrar número de serie y fecha de caducidad |

</details>

Si el puerto 25 continúa vacío después de cinco minutos, `ss -lnt | grep ':25 '` muestra si el servicio está escuchando, y el registro bajo `/opt/totemomail` indica la causa.

La opción `-showcerts` muestra si TotemoMail envía también la intermedia:

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

En la sustitución descrita solo apareció el certificado final, aunque la intermedia estaba en el archivo PKCS#12 y en `Issuer Certificates`. Esto no tiene consecuencias en trayectos internos donde nadie comprueba el certificado. Sin embargo, para la comprobación por Exchange Online, la cadena debe estar completa.

## Paso 7: pruebas, vuelta atrás y limpieza

Tras el último reinicio, pruebe el flujo de correo en ambas direcciones: un mensaje desde el exterior a través del bucle y otro hacia el exterior mediante el gateway. En el seguimiento de mensajes de Exchange Online, ambos tramos deben aparecer como entregados y, en dirección al gateway, no debe quedar nada bloqueado en la cola.

La **vuelta atrás** consiste en volver a marcar los conectores en el certificado antiguo, desmarcarlos en el nuevo salvo uno y reiniciar de nuevo los nodos individualmente. Esto solo es posible mientras el certificado antiguo siga siendo válido.

Después de realizar correctamente las pruebas:

- eliminar el archivo PKCS#12 del jumphost
- eliminar el directorio de trabajo del nodo; la clave ya se encuentra en el almacén de claves de TotemoMail
- al cabo de unos días, eliminar el certificado antiguo y reiniciar los nodos individualmente una vez más para que también cambie el último conector
- solicitar a la entidad PKI que revoque las emisiones erróneas
- registrar la nueva fecha de caducidad en el recordatorio

Avise a los administradores de la sustitución si el nuevo certificado ya no contiene nombres cortos. Quien hasta ahora haya abierto la interfaz con `https://gw01:8443` verá después una advertencia de certificado. Esto también se aplica a supervisiones y scripts que utilizan nombres cortos.

## Particularidades de un vistazo

| Observación | Consecuencia | Procedimiento |
|---|---|---|
| «New PKCS#10» genera 2048 bits sin nombres alternativos | Solicitud inutilizable para CA de 4096 bits | Generar clave y solicitud con openssl |
| Los cambios solo surten efecto tras el reinicio | Los nodos siguen mostrando el certificado antiguo | Reiniciar cada nodo individualmente |
| El puerto 25 aparece varios minutos después de los servicios web | Respuesta vacía poco después del inicio | Esperar antes de continuar con el siguiente nodo |
| Un certificado sin conector provoca una NullPointerException | No se puede desvincular por completo el certificado antiguo | Dejar un conector y eliminarlo más tarde |
| Una asignación explícita tiene prioridad sobre `*` | Un conector sigue mostrando el certificado antiguo | Eliminar el certificado antiguo tras la observación |
| Solo se envía el certificado final | Los extremos que comprueban no pueden construir la cadena | Aclararlo antes de una comprobación por Exchange Online |
| La RA incorpora los nombres manualmente | Posibles erratas y nombres ausentes | Comprobar la lista de nombres de la entrega |

## El certificado público para el trayecto a Exchange Online

Para que Exchange Online compruebe el trayecto hacia el gateway además de cifrarlo, el gateway necesita un certificado público en el puerto 25. Solo entonces se puede cambiar el conector saliente mediante `TlsSettings DomainValidation` a un `TlsDomain` y vincular el conector entrante al certificado mediante `TlsSenderCertificateName`.

Para ello basta un solo nombre. Exchange Online compara en ambas direcciones únicamente el nombre del certificado con el valor registrado; no comprueba la dirección que hay detrás. Un certificado para el nombre bajo el que el gateway es conocido externamente cubre ida y vuelta. Al realizar el pedido, pregunte expresamente por Client Authentication: varias autoridades de certificación públicas eliminaron este uso de los certificados TLS en 2026. Es necesario para el trayecto de vuelta, en el que el gateway se identifica como cliente.

Para usarlo en TotemoMail, las observaciones anteriores apuntan al siguiente procedimiento: importar el certificado público como tipo SMTPS para que el puerto 25 lo presente y los servicios web conserven el interno. Antes deben aclararse dos puntos. La cadena debe enviarse completa, y el puerto 25 presentará después el certificado público a todos los remitentes, también a los internos como un gateway anterior. Si un remitente espera un certificado determinado, este trayecto fallará.

## Fuentes

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): calendario para la validez de certificados TLS públicos: 200 días desde marzo de 2026, 100 días desde marzo de 2027 y 47 días desde marzo de 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector): parámetros `TlsSettings` y `TlsDomain` para comprobar el certificado en el extremo remoto.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector): parámetro `TlsSenderCertificateName` para la asignación mediante el certificado del remitente.

4.  [Documentación de OpenSSL: openssl-req](https://docs.openssl.org/master/man1/openssl-req/): estructura del archivo de configuración y opciones para solicitudes de certificados.

5.  [Documentación de OpenSSL: openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): creación y comprobación de archivos PKCS#12.

6.  [crt.sh](https://crt.sh): búsqueda en los registros de Certificate Transparency para comprobar la emisión de un certificado público.
