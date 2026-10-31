---
title: "Probar SMTP en Linux: desde la conexión TCP hasta el correo entregado"
navTitle: "Probar SMTP"
description: "Cuando una appliance deja de entregar correos, una prueba SMTP manual ayuda más que cualquier registro. Cómo comprobar capa por capa con herramientas integradas, qué significan los distintos errores y por qué un balanceador de carga puede falsear el diagnóstico."
date: "2026-07-31"
kategorie: "SMTP y flujo de correo"
timeToRead: "10 min de lectura"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "probar-smtp-en-linux-desde-la-conexion-tcp-hasta-el-correo-entregado"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
translationSourceHash: 17f985224e55b45d0395c6f5f6a100918ad6718f2bd22191bc29f30e204e5e11
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:21:38.776Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/probar-smtp-en-linux-desde-la-conexion-tcp-hasta-el-correo-entregado
---

# Probar SMTP en Linux: desde la conexión TCP hasta el correo entregado

Cuando un gateway de correo deja de entregar de repente, los registros de la appliance suelen mostrar solo el resultado final: falla una entrega, la cola crece y un mensaje de error indica un tiempo de espera agotado. La causa real solo se revela mediante una prueba manual desde la línea de comandos. SMTP es un protocolo de texto plano que puede comunicarse completamente a mano, y eso es precisamente lo que lo convierte en una herramienta de diagnóstico disponible en cualquier lugar sin instalaciones adicionales.

La segunda razón para realizar una prueba manual: en las appliances normalmente no se puede instalar nada. Sin gestor de paquetes, sin permisos de root, sin `swaks`. Por ello, todos los pasos siguientes funcionan con lo que ya está disponible en prácticamente cualquier sistema Linux.

## Separar las capas

Un envío de correo fallido puede fallar en cinco niveles distintos, y cada uno genera un patrón de error diferente:

1. **Resolución de nombres:** El host de destino no se puede traducir a una dirección IP.
2. **Conexión TCP:** No se puede establecer la conexión con el puerto o esta se restablece.
3. **Diálogo SMTP:** La conexión está establecida, pero el servidor rechaza el remitente, el destinatario o el contenido.
4. **Cifrado de transporte:** Falta STARTTLS, el certificado no es válido o la versión de TLS no coincide.
5. **Verificación del remitente:** El correo se acepta y se descarta en el destinatario debido a SPF, DKIM o DMARC.

El diagnóstico mejora enormemente si comprueba estos niveles uno tras otro e individualmente, en lugar de enviar directamente un correo de prueba completo. Un intento global fallido solo le indica que algo no funciona. La comprobación por capas le dice qué es.

## Paso 1: Resolución de nombres

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `hosts` | Base de datos NSS que se consulta; utiliza las mismas Fuentes y el mismo orden que el propio sistema, conforme a `nsswitch.conf` |
| `relay.example.com` | Nombre de host que se resolverá |

</details>

Si la salida queda vacía, no hay ningún servidor de nombres accesible desde este host o no responde a nombres externos. Esto ocurre con frecuencia en la práctica: las appliances en zonas aisladas suelen recibir únicamente un resolvedor interno que solo conoce sus propias zonas.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/etc/resolv.conf` | Archivo emitido por `cat` con los servidores de nombres configurados |
| `hosts:` | Patrón de búsqueda para `grep`: la línea que define el orden de las Fuentes de resolución (archivos, DNS) |
| `/etc/nsswitch.conf` | Archivo con la configuración NSS que busca `grep` |

</details>

Si falta la resolución, pruebe a continuación directamente contra la dirección IP. Para el diagnóstico esto es completamente suficiente y separa limpiamente el problema de DNS del problema de transporte. Para producción, la falta de resolución sigue siendo, por supuesto, un hallazgo independiente que debe corregirse.

## Paso 2: Accesibilidad del puerto

Para la comprobación TCP pura basta con bash. El pseudodispositivo `/dev/tcp` abre una conexión sin necesidad de tener instalados `nc` ni `telnet`:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `timeout 10` | Interrumpe el comando siguiente después de 10 segundos y devuelve entonces el código de salida 124 |
| `bash -c '…'` | Ejecuta la cadena de comandos en una bash; es necesario porque `/dev/tcp` es una función de bash |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Abre el descriptor de archivo 3 para lectura y escritura como conexión TCP a 192.0.2.25, puerto 25 |
| `echo "exit=$?"` | Muestra el código de salida del comando anterior |

</details>

El código de salida es aquí la información relevante:

| salida | Significado |
|---|---|
| `0` | La conexión está establecida, el puerto está abierto |
| `124` | Tiempo de espera agotado: se descartan paquetes, típico de un firewall con regla DROP |
| `1` | Rechazo inmediato (RST) o falta de ruta |

En la práctica, la diferencia entre 124 y 1 es la pista más importante. Un tiempo de espera agotado significa que alguien descarta silenciosamente en el camino, y casi siempre se debe a una regla de firewall. En cambio, un RST inmediato procede de un sistema que responde, pero no ofrece el servicio.

Compruebe de inmediato ambos puertos relevantes y además cualquier otro destino para ver si el host puede establecer conexiones salientes en absoluto:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `set -- $t` | Divide el par de valores en el espacio en los parámetros posicionales `$1` (dirección IP) y `$2` (puerto) |
| `timeout 8` | Interrumpe el intento de conexión después de 8 segundos (código de salida 124) |
| `bash -c "…"` | Ejecuta la apertura de conexión `/dev/tcp` en una bash |
| `2>/dev/null` | Suprime los mensajes de error para que aparezca exactamente una línea de resultado por destino |

</details>

Si también falla la prueba de control, el sistema no tiene salida directa en general y el tráfico debe pasar por un relay o proxy interno. Más abajo encontrará por qué este caso es especialmente engañoso.

Si falta `/dev/tcp`, el shell no es bash. En `sh`, `ash` o `ksh` no existe esta función, algo que a menudo se interpreta erróneamente como un supuesto problema de red:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-p $$` | Limita la salida al proceso con el PID del shell actual (`$$`) |
| `-o comm=` | Muestra solo el nombre del comando; la etiqueta vacía después de `=` suprime la línea de encabezado |
| `${BASH_VERSION:-keine bash}` | Muestra la versión de bash o el texto alternativo si la variable no está definida |

</details>

## Paso 3: Primero escuchar, no enviar

Un servidor SMTP saluda por sí mismo con un banner `220`. Por ello, la prueba individual más reveladora consiste en abrir una conexión y no hacer nada:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Abre el descriptor de archivo 3 como conexión TCP al destino |
| `timeout 15 cat <&3` | Lee durante 15 segundos todo lo que el servidor envía por sí mismo y lo muestra |
| `echo "[ende exit=$?]"` | Muestra el código de salida al finalizar; 124 significa: no llegó nada más durante 15 segundos |

</details>

Estos pocos caracteres separan dos situaciones completamente distintas. Si llega un `220 mail.example.com ESMTP`, el extremo remoto está hablando y todos los demás errores están en el diálogo. Si no llega nada, no se debe a un comando mal formulado por su parte, porque no ha enviado ninguno.

El descriptor de archivo permanece abierto en el shell después. Ciérrelo antes de iniciar la siguiente prueba; de lo contrario, podría seguir trabajando con una conexión antigua que ya no está intacta:

```bash
exec 3<&- 3>&-
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `3<&-` | Cierra el lado de lectura del descriptor de archivo 3 |
| `3>&-` | Cierra el lado de escritura del descriptor de archivo 3 |

</details>

## Paso 4: El diálogo SMTP a mano

Si está presente el banner, realice el diálogo completo. Es importante disponer de un proceso de lectura en ejecución para ver cada respuesta en el momento en que llega. Un script que primero envía todo y después lee no le muestra nada si hay una interrupción a mitad del diálogo:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\n' >&3
printf 'Date: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Abre el descriptor de archivo 3 como conexión TCP al destino |
| `cat <&3 & R=$!` | Inicia un lector en segundo plano para el descriptor de archivo 3 y guarda su PID en `R` |
| `printf '…\r\n' >&3` | Envía un comando SMTP con la terminación de línea CRLF requerida a la conexión |
| `sleep n` | Espera los segundos indicados a la respuesta del servidor antes de continuar con el siguiente comando |
| `date -R` | Devuelve la fecha en formato compatible con RFC para la cabecera `Date:` |
| `date +%s` | Devuelve la hora Unix como base simple y única para el ID del mensaje |
| `kill $R 2>/dev/null` | Finaliza el lector en segundo plano; no muestra el mensaje de error si ya ha terminado |

</details>

Dos detalles determinan el éxito o el fracaso. SMTP exige CRLF como terminación de línea, por eso `printf` con `\r\n` y no `echo`. Y el punto en una línea propia finaliza la parte del mensaje; debe enviarse como `\r\n.\r\n`.

El flujo esperado: `220` al establecer la conexión, `250` para EHLO, `250 2.1.0` para MAIL FROM, `250 2.1.5` para RCPT TO, `354` para DATA y, al final, `250 2.0.0 Ok: queued as <id>`. Anote el ID de cola. Con él se puede rastrear el mensaje con el proveedor operador si nunca llega al destinatario.

El nombre EHLO merece atención: algunos relays lo verifican frente a DNS directo e inverso y, de lo contrario, responden con `501` o `504`. Utilice el FQDN real del sistema emisor, no el nombre corto.

## Paso 5: STARTTLS y certificado

Para la conexión cifrada, `openssl s_client` realiza por sí mismo la negociación STARTTLS y luego entrega el canal a la entrada estándar:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-connect 192.0.2.25:25` | Host y puerto de destino de la conexión |
| `-starttls smtp` | Realiza primero el diálogo SMTP en texto plano y luego cambia a TLS mediante STARTTLS |
| `-tls1_2` | Negocia exclusivamente TLS 1.2 |
| `-brief` | Reduce la salida a un breve resumen de la conexión negociada |
| `</dev/null` | Cierra de inmediato la entrada estándar para que `s_client` no espere de forma interactiva tras el handshake |

</details>

Si se conecta mediante la dirección IP porque falta DNS, la comprobación del nombre de host no funciona. El nombre del certificado no coincide entonces con la dirección numérica. SNI y el nombre de comprobación se pueden establecer explícitamente, sin ninguna consulta DNS:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-servername mail.example.com` | Establece el nombre SNI en ClientHello, independientemente de la dirección de conexión |
| `-verify_hostname mail.example.com` | Comprueba el certificado del servidor contra este nombre en vez de contra la dirección numérica |

</details>

Aquí aparecen regularmente dos patrones de error que suelen interpretarse mal.

**«Didn't find STARTTLS in server response, trying anyway»** significa que el servidor no ofreció STARTTLS en su respuesta EHLO. `openssl` envía aun así un ClientHello TLS, el servidor ve datos de protocolo no válidos y la conexión termina con `wrong version number` o `write:errno=32` (EPIPE). Ambos mensajes son errores posteriores. La información real es: no hay STARTTLS. Consulte con el diálogo en texto plano del paso 4 qué capacidades informa realmente el servidor.

**No hay STARTTLS en un salto interno** suele ser completamente correcto. Si un balanceador de carga reenvía la conexión en la capa 4, no es él quien negocia TLS, sino el sistema situado detrás frente al destino real. Probar en texto plano en el segmento interno no es entonces una deficiencia de seguridad, sino simplemente la arquitectura.

## Paso 6: Python como alternativa

Si Python está disponible, se ahorra la temporización manual con `sleep`. Basta con la biblioteca estándar; no se debe instalar nada adicional:

```python
#!/usr/bin/env python3
import smtplib, ssl
from email.message import EmailMessage
from email.utils import formatdate, make_msgid

msg = EmailMessage()
msg["From"] = "absender@example.com"
msg["To"] = "empfaenger@example.net"
msg["Subject"] = "Relay-Test"
msg["Date"] = formatdate(localtime=True)
msg["Message-ID"] = make_msgid(domain="example.com")
msg.set_content("Testnachricht\n")

ctx = ssl.create_default_context()
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

s = smtplib.SMTP("192.0.2.25", 25, timeout=30, local_hostname="host.example.com")
s.set_debuglevel(1)
s.ehlo()
if s.has_extn("starttls"):
    s.starttls(context=ctx, server_hostname="mail.example.com")
    s.ehlo()
    print("TLS:", s.sock.version(), s.sock.cipher()[0])
s.send_message(msg)
s.quit()
```

`set_debuglevel(1)` registra el diálogo completo, incluidos todos los códigos de respuesta, y `smtplib` lee cada respuesta de forma síncrona. Una interrupción aparece como `SMTPServerDisconnected` junto con la última línea recibida, en lugar de como una tubería rota silenciosa.

Dos puntos suelen fallar aquí: `server_hostname` es obligatorio al conectarse mediante una dirección IP; de lo contrario, Python comprueba el certificado contra la dirección numérica. Y si desactiva deliberadamente la comprobación, `check_hostname = False` debe ir antes de `verify_mode = ssl.CERT_NONE`; de lo contrario, Python lanza un `ValueError`.

## Dirección del remitente, SPF y alineación

Una prueba falla sorprendentemente a menudo no en el transporte, sino en la dirección de remitente elegida. Hay que comprobar tres puntos de antemano.

El dominio del remitente debe ser un FQDN. Una dirección como `test@meine-testdomain` sin dominio de nivel superior ya es rechazada por muchos MTA en MAIL FROM con `501` o `553`.

El dominio debe autorizar la ruta de envío utilizada. Una consulta del registro SPF muestra si la dirección saliente está cubierta:

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `+short` | Muestra únicamente los valores de los registros, sin cabeceras ni metadatos |
| `TXT` | Tipo de registro consultado |
| `example.com` | Nombre consultado |
| `grep spf1` | Filtra la línea SPF de varios registros TXT |

</details>

Y con DMARC activo, decide la alineación. Si el registro contiene `aspf=s`, el dominio en el sobre (MAIL FROM) y el dominio de la cabecera `From:` deben coincidir exactamente, no basta con que estén relacionados:

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `+short` | Muestra únicamente los valores de los registros, sin cabeceras ni metadatos |
| `TXT _dmarc.example.com` | Tipo de registro y nombre definido para DMARC bajo el dominio |

</details>

Con `p=reject`, un correo de prueba con alineación incorrecta desaparece silenciosamente en el destinatario, aunque su relay lo haya aceptado con `250 queued`. Esta es la causa más frecuente de los mensajes que se consideran enviados correctamente en el lado emisor y, aun así, nunca llegan.

## Cuando hay un balanceador de carga de por medio

En entornos más grandes, una appliance rara vez envía directamente a Internet. Lo habitual es un servidor virtual en un balanceador de carga que acepta la conexión, la reescribe mediante Source-NAT a una dirección definida y solo entonces la reenvía hacia el exterior. Esto tiene una consecuencia incómoda para el diagnóstico.

Un servidor virtual que opera en la capa 4 confirma inmediatamente el handshake TCP antes de haber establecido por sí mismo una conexión con el destino. Si falla esta segunda conexión, en el cliente verá una conexión establecida correctamente y restablecida inmediatamente después: `Connection reset by peer`, sin ningún banner SMTP. El error no está entonces en su lado ni en el destino, sino en el pool detrás del servidor virtual, por ejemplo porque un miembro está marcado como caído o no se puede resolver el FQDN configurado.

Esto también explica por qué una prueba directa contra el destino de Internet debe fallar si la regla de reenvío solo acepta tráfico desde la dirección SNAT ya reescrita. Las conexiones con la dirección de origen original no coinciden con ninguna regla y se descartan. En estos entornos, pruebe siempre contra el servidor virtual previsto, no contra el destino real.

Una sola línea indica qué dirección de origen utiliza su sistema para un destino determinado. El valor detrás de `src` es exactamente el dato que el equipo de red necesita para habilitar el acceso:

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `route get` | Consulta al kernel qué ruta elegiría para un destino concreto |
| `192.0.2.25` | Dirección de destino de la conexión simulada |

</details>

Si el sistema está detrás de NAT, el extremo remoto no ve esta dirección, sino la dirección pública del perímetro. No se puede determinar desde dentro mientras no pase ningún tráfico; figura en la regla NAT.

## Patrones de error de un vistazo

| Observación | Causa probable |
|---|---|
| `Name or service not known` | No hay resolución de nombres en el host |
| Tiempo de espera agotado, salida 124 | El firewall descarta silenciosamente (DROP) |
| `Connection refused` | No hay servicio en el puerto o existe una regla REJECT |
| La conexión está establecida, no hay banner y luego RST | El balanceador de carga acepta, pero el backend no es accesible |
| `Didn't find STARTTLS` | El servidor no ofrece cifrado de transporte |
| `wrong version number`, `errno=32` | Errores posteriores tras forzar TLS sin STARTTLS |
| `501` / `553` en MAIL FROM | El dominio del remitente no es un FQDN o no está permitido |
| `554 relay access denied` | La IP de origen no está habilitada en el relay |
| `250 queued`, pero no hay entrega | Alineación SPF, DKIM o DMARC en el destinatario |

## Pruebas de carga y límites de velocidad

Para pruebas de volumen se aplica una regla que a menudo se pasa por alto en el día a día: el problema no es la cantidad de mensajes, sino la cantidad de conexiones. Los relays típicos permiten algunos cientos de conexiones por minuto, pero decenas de miles de mensajes. Por tanto, mantenga una sesión abierta y envíe muchos sobres a través de ella, en lugar de volver a conectarse para cada mensaje.

En `smtplib` esto simplemente significa reutilizar varias veces el mismo objeto de conexión y reconstruir la sesión de forma controlada tras un número fijo de mensajes. En cambio, quien abre una conexión nueva por correo supera el límite de conexiones mucho antes que el límite de mensajes y provoca rechazos que parecen un problema del extremo remoto.

## Conclusión

La prueba SMTP manual no es un recurso de emergencia para entornos sin herramientas, sino el diagnóstico más preciso disponible en la operación de correo. Separa claramente la resolución de nombres, la accesibilidad, el diálogo de protocolo y el cifrado, y proporciona un resultado inequívoco para cada nivel. Quien primero solo escucha, luego realiza el diálogo a mano y toma en serio los códigos de respuesta, llega en pocos minutos a una conclusión con la que puede documentar un ticket para el equipo de red o del proveedor: con dirección de origen, puerto de destino, comportamiento observado y código de salida.

## Historial de cambios

- **25 de octubre de 2026:** Se sustituyó la fuente RFC 7489 por RFC 9989, que reemplaza la especificación DMARC desde mayo de 2026.

## Fuentes

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Define el diálogo SMTP, el orden de los comandos y el significado de los códigos de respuesta.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Describe STARTTLS como extensión, incluido el comportamiento cuando el servidor no la ofrece.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Estructura y evaluación del registro SPF para autorizar sistemas emisores.

4.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Regula la alineación entre remitentes del sobre y de la cabecera, la etiqueta `aspf` y la evaluación de políticas; sustituye RFC 7489 desde mayo de 2026.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referencia de las opciones utilizadas, entre otras `-starttls`, `-servername` y `-verify_hostname`.

6.  [Documentación de Python: smtplib](https://docs.python.org/3/library/smtplib.html): Biblioteca estándar para sesiones SMTP, incluidas STARTTLS y salida de depuración.

7.  [Manual de referencia de Bash: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documenta `/dev/tcp` como pseudodispositivo propio de bash para conexiones de red.
