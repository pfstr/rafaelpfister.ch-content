---
title: "Probar SMTP en Linux: desde la conexión TCP hasta el correo entregado"
navTitle: "Probar SMTP"
description: "Cuando una appliance deja de entregar correos, una prueba SMTP manual ayuda más que cualquier registro. Cómo comprobar cada capa con herramientas integradas, qué significan los distintos errores y por qué un balanceador de carga puede distorsionar el diagnóstico."
date: "2026-07-31"
kategorie: "SMTP y flujo de correo"
timeToRead: "10 min de lectura"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
slug: "probar-smtp-en-linux-desde-la-conexion-tcp-hasta-el-correo-entregado"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
url: https://rafaelpfister.ch/es/blog/probar-smtp-en-linux-desde-la-conexion-tcp-hasta-el-correo-entregado
translationSourceHash: 650b4717ca00ffd3d02cebae8f1484027cf0f9b47de1b607caa951cd7264454a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-01T06:13:36.133Z
translationReview: automatic
---

# Probar SMTP en Linux: desde la conexión TCP hasta el correo entregado

Cuando una pasarela de correo deja de entregar mensajes de repente, los registros de la appliance suelen mostrar solo el final de la historia: falla una entrega, crece la cola, un mensaje de error indica un tiempo de espera agotado. La causa real solo se revela mediante una prueba manual desde la línea de comandos. SMTP es un protocolo de texto plano que puede comunicarse completamente a mano, y eso lo convierte en una de las herramientas de diagnóstico más agradecidas en la operación de correo.

La segunda razón para realizar una prueba manual: normalmente no se puede instalar nada en las appliances. No hay gestor de paquetes, ni permisos de root, ni `swaks`. Por eso, todos los pasos siguientes funcionan con lo que ya está disponible en prácticamente cualquier sistema Linux.

## Separar las capas

Un envío de correo fallido puede fallar en cinco niveles distintos, y cada uno genera un patrón de error diferente:

1. **Resolución de nombres:** El host de destino no se puede traducir a una dirección IP.
2. **Conexión TCP:** No se establece la conexión al puerto o esta se restablece.
3. **Diálogo SMTP:** La conexión está establecida, pero el servidor rechaza el remitente, el destinatario o el contenido.
4. **Cifrado de transporte:** Falta STARTTLS, el certificado no es válido o la versión TLS no coincide.
5. **Verificación del remitente:** El correo se acepta y se descarta en el destinatario debido a SPF, DKIM o DMARC.

El diagnóstico mejora enormemente si comprueba estos niveles de forma individual y secuencial, en lugar de enviar directamente un correo de prueba completo. Un intento global fallido solo indica que algo no funciona. La comprobación por capas indica qué es.

## Paso 1: Resolución de nombres

```bash
getent hosts relay.example.com
```

Si la salida permanece vacía, no hay ningún servidor de nombres accesible desde este host o no responde a nombres externos. Esto ocurre más a menudo de lo que parece: las appliances en zonas aisladas a menudo solo reciben un resolvedor interno que conoce exclusivamente sus propias zonas.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

Si falta la resolución, pruebe a continuación directamente con la dirección IP. Esto es totalmente suficiente para el diagnóstico y separa claramente el problema de DNS del problema de transporte. Para el funcionamiento en producción, la falta de resolución sigue siendo, naturalmente, un hallazgo independiente que debe corregirse.

## Paso 2: Accesibilidad del puerto

Para una comprobación TCP pura basta con bash. El pseudodispositivo `/dev/tcp` abre una conexión sin necesidad de tener instalados `nc` ni `telnet`:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

El código de salida es aquí la información fundamental:

| exit | Significado |
|---|---|
| `0` | La conexión está establecida, el puerto está abierto |
| `124` | Tiempo de espera agotado: se descartan paquetes, típico de un firewall con regla DROP |
| `1` | Rechazo inmediato (RST) o falta de ruta |

En la práctica, la diferencia entre 124 y 1 es la pista más importante. Un tiempo de espera agotado significa que alguien descarta silenciosamente en el camino, y casi siempre se trata de una regla de firewall. En cambio, un RST inmediato procede de un sistema que responde, pero no ofrece el servicio.

Compruebe de inmediato ambos puertos relevantes y, además, cualquier otro destino para ver si el host tiene permitido establecer conexiones salientes:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do set -- $t; timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null; echo "$1:$2 -> exit=$?"; done
```

Si tampoco funciona la comprobación adicional, el sistema no tiene salida directa en general y el tráfico debe pasar por un relay interno o un proxy. Más adelante explicamos por qué este caso es especialmente engañoso.

Si falta `/dev/tcp`, el shell no es bash. En `sh`, `ash` o `ksh` esa función no existe, algo que a menudo se interpreta erróneamente como un problema de red:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Paso 3: Primero escuchar, no enviar

Un servidor SMTP se saluda por sí mismo con un banner `220`. Por ello, la prueba individual más significativa consiste en abrir una conexión y no hacer nada:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

Estos pocos caracteres distinguen dos situaciones completamente diferentes. Si llega un `220 mail.example.com ESMTP`, el extremo remoto está hablando y todos los errores posteriores se encuentran en el diálogo. Si no llega nada, no se debe a un comando mal formulado por su parte, porque no ha enviado ninguno.

El descriptor de archivo permanece abierto en el shell. Ciérrelo antes de iniciar la siguiente prueba; de lo contrario, podría continuar trabajando con una conexión antigua medio muerta:

```bash
exec 3<&- 3>&-
```

## Paso 4: El diálogo SMTP a mano

Si aparece el banner, ejecute el diálogo completo. Es importante contar con un proceso de lectura en ejecución para ver cada respuesta en el momento en que llega. Un script que primero envía todo y luego lee no le mostrará nada si la conexión se interrumpe a mitad del diálogo:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\nDate: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

Dos detalles determinan el éxito o la frustración. SMTP exige CRLF como final de línea; por eso, `printf` con `\r\n` y no `echo`. Y el punto en una línea propia finaliza la parte del mensaje; debe enviarse como `\r\n.\r\n`.

La secuencia esperada: `220` al establecer la conexión, `250` para EHLO, `250 2.1.0` para MAIL FROM, `250 2.1.5` para RCPT TO, `354` para DATA y, al final, `250 2.0.0 Ok: queued as <id>`. Anote el ID de cola. Con él se puede rastrear el mensaje con el proveedor operador si nunca llega al destinatario.

El nombre EHLO merece atención: algunos relays lo verifican frente al DNS directo e inverso y, de lo contrario, responden con `501` o `504`. Utilice el FQDN real del sistema remitente, no el nombre corto.

## Paso 5: STARTTLS y certificado

Para la conexión cifrada, `openssl s_client` realiza la negociación STARTTLS por sí mismo y después entrega el canal a la entrada estándar:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

Si se conecta mediante la dirección IP porque falta DNS, la comprobación del nombre de host no sirve. El nombre del certificado no coincide entonces con la dirección numérica. SNI y el nombre de comprobación pueden establecerse explícitamente, sin ninguna consulta DNS:

```bash
openssl s_client -connect 192.0.2.25:25 -servername mail.example.com -verify_hostname mail.example.com -starttls smtp -tls1_2 -brief </dev/null
```

Aquí aparecen regularmente dos patrones de error que suelen interpretarse mal.

**«Didn't find STARTTLS in server response, trying anyway»** significa que el servidor no ofreció STARTTLS en su respuesta EHLO. `openssl` envía aun así un TLS-ClientHello, el servidor lo ve como basura de protocolo y la conexión termina con `wrong version number` o `write:errno=32` (EPIPE). Ambos mensajes son errores consecuentes. La información real es: no hay STARTTLS. Consulte con el diálogo de texto plano del paso 4 qué capacidades informa realmente el servidor.

**No tener STARTTLS en un salto interno** suele ser totalmente correcto. Si un balanceador de carga reenvía la conexión en la capa 4, no negocia él TLS, sino el sistema situado detrás de él frente al destino real. En ese caso, probar en texto plano en el segmento interno no es una deficiencia de seguridad, sino simplemente la arquitectura.

## Paso 6: Python como alternativa

Si Python está disponible, se ahorra los ajustes de temporización con `sleep`. La biblioteca estándar es suficiente; no es necesario instalar nada adicional:

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

Dos trampas: `server_hostname` es obligatorio al conectar mediante una dirección IP; de lo contrario, Python verifica el certificado frente a la dirección numérica. Y si desactiva deliberadamente la comprobación, `check_hostname = False` debe situarse antes de `verify_mode = ssl.CERT_NONE`; de lo contrario, Python genera un `ValueError`.

## Dirección del remitente, SPF y alineación

Con sorprendente frecuencia, una prueba no falla en el transporte, sino en la dirección de remitente elegida. Hay tres puntos que deben comprobarse previamente.

El dominio del remitente debe ser un FQDN. Una dirección como `test@meine-testdomain` sin dominio de nivel superior es rechazada por muchos MTA ya durante MAIL FROM con `501` o `553`.

El dominio debe autorizar la ruta de envío utilizada. Una consulta del registro SPF muestra si la dirección saliente está cubierta:

```bash
dig +short TXT example.com | grep spf1
```

Y con DMARC activo decide la alineación. Si el registro contiene `aspf=s`, el dominio en el sobre (MAIL FROM) y el dominio en la cabecera `From:` deben coincidir exactamente, no basta con que estén relacionados:

```bash
dig +short TXT _dmarc.example.com
```

Con `p=reject` un correo de prueba con una alineación inadecuada desaparece silenciosamente en el destinatario, aunque su relay lo haya aceptado con `250 queued`. Esta es la causa más frecuente de mensajes que se consideran enviados correctamente en el lado emisor y, aun así, nunca llegan.

## Cuando hay un balanceador de carga entre medias

En entornos grandes, una appliance rara vez envía directamente a Internet. Lo habitual es un servidor virtual en un balanceador de carga que acepta la conexión, reescribe la dirección a una dirección definida mediante Source-NAT y solo entonces la reenvía al exterior. Esto tiene una consecuencia incómoda para el diagnóstico.

Un servidor virtual que opera en la capa 4 confirma inmediatamente el handshake TCP, antes de haber establecido él mismo una conexión con el destino. Si esta segunda conexión falla, verá en el cliente una conexión establecida correctamente y restablecida inmediatamente después: `Connection reset by peer`, sin ningún banner SMTP. Entonces el error no está en su lado ni en el destino, sino en el pool detrás del servidor virtual, por ejemplo porque un miembro está marcado como down o porque no se resuelve el FQDN configurado.

Esto también explica por qué una prueba directa contra el destino de Internet debe fallar si la regla de reenvío solo acepta tráfico procedente de la dirección SNAT ya reescrita. Las conexiones con la dirección de origen original no coinciden con ninguna regla y se descartan. En estos entornos, pruebe siempre contra el servidor virtual previsto, no contra el destino real.

Una sola línea indica qué dirección de origen utiliza su sistema para un destino concreto. El valor detrás de `src` es exactamente el dato que el equipo de red necesita para la habilitación:

```bash
ip route get 192.0.2.25
```

Si el sistema se encuentra detrás de NAT, el extremo remoto no verá esta dirección, sino la dirección pública del perímetro. No puede determinarse desde dentro mientras no pase ningún tráfico; figura en la regla NAT.

## Patrones de error de un vistazo

| Observación | Causa probable |
|---|---|
| `Name or service not known` | No hay resolución de nombres en el host |
| Tiempo de espera agotado, exit 124 | El firewall descarta silenciosamente (DROP) |
| `Connection refused` | No hay servicio en el puerto o regla REJECT |
| La conexión está establecida, no hay banner y luego RST | El balanceador de carga acepta, el backend no es accesible |
| `Didn't find STARTTLS` | El servidor no ofrece cifrado de transporte |
| `wrong version number`, `errno=32` | Errores consecuentes tras forzar TLS sin STARTTLS |
| `501` / `553` en MAIL FROM | El dominio del remitente no es un FQDN o no está permitido |
| `554 relay access denied` | La IP de origen no está autorizada en el relay |
| `250 queued`, pero no hay entrega | SPF, DKIM o alineación DMARC en el destinatario |

## Pruebas de carga y límites de tasa

Para pruebas de volumen se aplica una regla que a menudo se pasa por alto en el día a día: el problema no es el número de mensajes, sino el número de conexiones. Los relays típicos permiten algunos cientos de conexiones por minuto, pero decenas de miles de mensajes. Por ello, mantenga una sesión abierta y envíe muchos sobres a través de ella, en lugar de volver a conectarse para cada mensaje.

En `smtplib` esto significa simplemente reutilizar varias veces el mismo objeto de conexión y restablecer la sesión de forma controlada tras un número fijo de mensajes. En cambio, quien abre una conexión nueva por correo alcanza el límite de conexiones mucho antes que el límite de mensajes y provoca rechazos que parecen un problema del extremo remoto.

## Conclusión

La prueba SMTP manual no es un recurso de emergencia para entornos sin herramientas, sino el diagnóstico más preciso disponible en la operación de correo. Separa claramente la resolución de nombres, la accesibilidad, el diálogo de protocolo y el cifrado, y proporciona un resultado inequívoco para cada nivel. Quien primero solo escucha, luego realiza el diálogo a mano y toma en serio los códigos de respuesta, obtiene en pocos minutos una conclusión con la que documentar un ticket ante el equipo de red o el proveedor: con dirección de origen, puerto de destino, comportamiento observado y código de salida.

## Fuentes

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Define el diálogo SMTP, el orden de los comandos y el significado de los códigos de respuesta.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Describe STARTTLS como extensión, incluido el comportamiento cuando el servidor no lo ofrece.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Estructura y evaluación del registro SPF para autorizar sistemas emisores.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Regula la alineación entre el remitente del sobre y el de la cabecera, así como la evaluación de políticas.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referencia de las opciones utilizadas, entre ellas `-starttls`, `-servername` y `-verify_hostname`.

6.  [Documentación de Python: smtplib](https://docs.python.org/3/library/smtplib.html): Biblioteca estándar para sesiones SMTP, incluidos STARTTLS y salida de depuración.

7.  [Manual de referencia de Bash: Redirecciones](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documenta `/dev/tcp` como pseudodispositivo propio de bash para conexiones de red.
