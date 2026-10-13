---
title: "SSH: transporte, identidades y administración segura"
blatt: "ssh"
description: "SSH para administradores de infraestructura y mensajería: transporte, intercambio de claves e identidad de host, autenticación de usuarios y cargas de trabajo, canales, claves y certificados OpenSSH, agentes, bastiones y reenvío, SFTP/SCP, política de servidor, endurecimiento, registro, OpenSSH para Windows e historia técnica."
fakten:
  - label: Función del sistema
    wert: capa de transporte y canales cifrada, protegida por integridad y autenticada para la administración
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Puerto estándar
    wert: TCP 22; el listener real y la ruta de acceso dependen de la configuración
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Pila de protocolos
    wert: Transport Protocol · User Authentication Protocol · Connection Protocol
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Identidad del servidor
    wert: Host Key o Host Certificate; comprobar antes de las credenciales de usuario
    href: https://www.rfc-editor.org/rfc/rfc4253.html
  - label: Autenticación de usuario
    wert: publickey · password · hostbased · keyboard-interactive; la política puede exigir varios factores
    href: https://www.rfc-editor.org/rfc/rfc4252.html
  - label: Negociación criptográfica
    wert: KEX · algoritmo de Host Key · Cipher · MAC/AEAD · compresión
    href: https://www.rfc-editor.org/rfc/rfc9142.html
  - label: Tipos de clave
    wert: Ed25519 · ECDSA · RSA/SHA-2 · variantes FIDO Security Key; uso según la compatibilidad cliente/servidor
    href: https://www.rfc-editor.org/rfc/rfc8709.html
  - label: Canales de acceso
    wert: Shell · Exec · Subsystem · reenvío TCP-/Streamlocal · reenvío Agent-/X11
    href: https://www.rfc-editor.org/rfc/rfc4254.html
  - label: Transferencia de archivos
    wert: SFTP como subsistema SSH; la CLI scp puede usar semántica SFTP o SCP heredada según la implementación
    href: https://man.openbsd.org/sftp
  - label: Distribución de confianza
    wert: known_hosts/Pinning · SSHFP con DNSSEC · Host-CA · certificados OpenSSH User-/Host
    href: https://www.rfc-editor.org/rfc/rfc4255.html
  - label: Estado operativo
    wert: listener · política efectiva · Host Keys · ruta de autenticación · sesiones/canales · logs · red de origen
    href: https://man.openbsd.org/sshd_config
  - label: Objeto de recuperación
    wert: identidad de host/CA, política de servidor, Authorized Principals/Keys, Break Glass, ruta de auditoría y dependencias del bastión
    href: https://man.openbsd.org/ssh-keygen
werbung:
  - newsletter
ctaThemen:
  - claude
  - powershell
  - haertung
translationSourceHash: 9617920ae627ef338448276f4094696352ba3f158be192d73dd10226c8ee296f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:25:29.391Z
translationReview: automatic
---

# SSH: transporte, identidades y administración segura

Secure Shell es una familia de protocolos para el acceso remoto seguro a través de una red no segura. SSH combina tres tareas: establece un transporte cifrado y protegido por integridad, autentica primero el servidor y después al usuario o la carga de trabajo y, finalmente, multiplexa shells, comandos individuales, subsistemas y reenvíos como canales separados. Este orden es decisivo para los administradores: una clave de usuario sólida no protege si el cliente la ofrece a un servidor no verificado.

SSH es a la vez protocolo y plano de control. Un acceso puede permitir comandos root, transferencia de archivos, túneles de bases de datos, acceso a agentes y rutas de salto hacia redes internas. Por tanto, la evaluación de seguridad no debe terminar en el puerto del listener ni en `PasswordAuthentication no`. La distribución de Host Keys, la política Match efectiva, el origen de las claves, los principals, los derechos de reenvío, la ruta de origen, el registro de sesiones y la recuperación de los anclajes de confianza pertenecen al mismo sistema.

La explicación sigue una conexión SSH desde TCP y el intercambio de versiones, pasando por el intercambio de claves y la comprobación del host, hasta el inicio de sesión de usuario y el canal. Los reenvíos, la operación de claves, los certificados y el diagnóstico se basan en ello.

SSH convierte una conexión de red no segura en un transporte autenticado y cifrado, con canales separados para shell, comandos, transferencia de archivos o túneles. El artículo sigue exactamente este orden: identidad del servidor, intercambio de claves, inicio de sesión de usuario, canales y operación.

## Arquitectura del protocolo

RFC 4251 divide SSH-2 en tres protocolos:

1. **Transport Layer Protocol:** intercambio de versiones, negociación de algoritmos, Key Exchange, autenticación del servidor, cifrado e integridad.
2. **User Authentication Protocol:** autenticación de un usuario cliente frente al servidor sobre el transporte protegido.
3. **Connection Protocol:** multiplexación de canales lógicos para sesiones, Port Forwarding y otros servicios.

Las capas se ejecutan sobre un flujo de bytes fiable, normalmente [TCP](/kb/tcp). El puerto TCP 22 está registrado en IANA como `ssh`; los proxies inversos, los reenvíos de puertos y los bastiones pueden modificar la ruta real ([RFC 4251 – SSH Protocol Architecture](https://www.rfc-editor.org/rfc/rfc4251.html), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ssh.svg?v=20260813" title="Interaktive Infografik: SSH-Verbindungsaufbau von TCP und Versionsaustausch über KEX, Hostidentität und Benutzerautorisierung bis zu Channels, Bastion, Schlüssel- und Zertifikatslebenszyklus sowie Audit und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-ssh.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Intercambio de versiones y negociación de algoritmos

El cliente y el servidor envían primero cadenas de identificación como `SSH-2.0-...`. A continuación intercambian `SSH_MSG_KEXINIT` con listas ordenadas para Key Exchange, algoritmo de Host Key, cifrado en ambas direcciones, MAC, compresión e idioma. Se elige la primera opción común de cada categoría. Por tanto, las listas de algoritmos son una política y un contrato de compatibilidad, no solo un detalle de implementación ([RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)).

RFC 9142 actualiza las recomendaciones para algoritmos SSH. Distingue `MUST`, `SHOULD`, `MAY`, `SHOULD NOT` y `MUST NOT` para las implementaciones; una política productiva también considera cumplimiento, parque de clientes, hardware y ruta de migración ([RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)).

Las líneas de configuración que fijan una lista propia y rígida de cifrados envejecen. OpenSSH puede mostrar sus algoritmos compatibles con `ssh -Q` y la configuración de cliente o servidor resuelta de forma efectiva. Los operadores eliminan específicamente procedimientos heredados conocidos, pero mantienen una ruta de compatibilidad probada para appliances.

## Key Exchange y claves de sesión

El Key Exchange genera un secreto compartido y un Exchange Hash sin enviar la clave de sesión por la red. El hash vincula cadenas de identificación, mensajes KEXINIT, la Host Key del servidor y valores efímeros específicos del algoritmo. El servidor firma el Exchange Hash con su Host Key; con ello, el cliente comprueba conjuntamente la identidad del servidor y el intercambio de claves ([RFC 4253](https://www.rfc-editor.org/rfc/rfc4253.html)).

Los procedimientos modernos utilizan variantes efímeras de Diffie-Hellman y ofrecen Forward Secrecy para las claves de sesión. SSH puede realizar rekey durante una conexión, por ejemplo tras cierta cantidad de datos o tiempo. Rekey y los Connection Channels comparten el mismo transporte; por ello, los problemas se manifiestan como bloqueos o desconexiones de sesiones aparentemente activas.

RFC 8308 define una negociación genérica de extensiones. OpenSSH utiliza extensiones, entre otras cosas, para anunciar algoritmos de firma del servidor o actualizaciones de Host Keys ([RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)). Strict Key Exchange protege con mayor intensidad la secuencia inicial frente a mensajes inesperados y se estandarizó tras el análisis de Terrapin ([RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)).

Después de elegir versión y algoritmos, el cliente debe reconocer primero al servidor. Solo cuando se ha aceptado la Host Key tiene sentido el posterior inicio de sesión de usuario.

## Host Keys: identidad del servidor antes que identidad del usuario

La Host Key identifica al servidor o a un grupo controlado de servidores. En el primer contacto, un cliente sin un anclaje de confianza distribuido previamente no tiene base criptográfica para evaluar el fingerprint mostrado. «Accept new» automatiza el pinning para hosts nuevos, pero no resuelve la distribución inicial segura.

OpenSSH almacena claves y marcadores conocidos en `known_hosts`. Una clave modificada puede significar una rotación legítima, una reinstalación, un pool de balanceador de carga modificado o un ataque. La advertencia no se «resuelve» eliminando a ciegas la línea; la nueva clave se verifica mediante una fuente independiente y se documenta la causa ([ssh(1) – host key verification](https://man.openbsd.org/ssh), [sshd(8)](https://man.openbsd.org/sshd)).

### Modelos de confianza para la identidad de host

| Modelo | Fortaleza | Límite operativo |
|---|---|---|
| fingerprint manual fuera de banda | directo y sencillo para pocos hosts | la distribución y la rotación seguras escalan mal |
| pinning/TOFU de `known_hosts` | detecta cambios posteriores | el primer contacto sigue sin verificarse |
| SSHFP con DNSSEC | DNS publica fingerprints | el cliente debe validar DNSSEC; la ruta del resolver es un límite de confianza |
| Host-CA de OpenSSH | los clientes confían en CA y principals | clave de CA, proceso de firma, validez y asignación de principals |
| gestión de configuración | distribución central y auditable | confianza en pipeline y enrolamiento |

RFC 4255 define los registros SSHFP; sin DNSSEC solo pueden ser una indicación, no un anclaje de confianza único ([RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)). Los certificados OpenSSH vinculan Public Key, número de serie, tipo, principals, validez, Critical Options y Extensions mediante una firma de CA ([IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Hostidentität">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:SSH_HOST -Type SSHFP -DnssecOk
ssh-keyscan -t ed25519,rsa $env:SSH_HOST | Set-Content .\\hostkeys.candidate
ssh-keygen -lf .\\hostkeys.candidate
ssh-keygen -F $env:SSH_HOST</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +dnssec "$SSH_HOST" SSHFP
ssh-keyscan -t ed25519,rsa "$SSH_HOST" &gt; hostkeys.candidate
ssh-keygen -lf hostkeys.candidate
ssh-keygen -F "$SSH_HOST"</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) consultan [DNS](/kb/dns); [`ssh-keyscan`](https://man.openbsd.org/ssh-keyscan) recopila las claves ofrecidas, pero **no las autentica**. [`ssh-keygen`](https://man.openbsd.org/ssh-keygen) calcula fingerprints y busca en `known_hosts`. Los candidatos solo se adoptan después de una verificación independiente.

## Ciclo de vida y rotación de Host Keys

Un servidor puede tener varias Host Keys para diferentes algoritmos. Las Host Keys privadas solo se almacenan en servidores o componentes controlados de balanceo de carga; las copias de seguridad y los clones no deben provocar una identidad compartida no intencionada. En el autoscaling, una Host-CA suele ser más adecuada que Host Keys privadas copiadas.

Una rotación comprende:

1. generar y proteger una nueva clave o un nuevo certificado de host;
2. ofrecerlos en paralelo y distribuir la confianza en fingerprint/CA;
3. comprobar la compatibilidad real de los clientes y el principal de host;
4. eliminar la clave antigua de forma controlada;
5. actualizar inventario, revocación y documentación de Break Glass.

OpenSSH puede anunciar Host Keys adicionales después de una autenticación correcta mediante `UpdateHostKeys`. Esto facilita la rotación, pero presupone que la confianza de host existente ya es correcta ([ssh_config(5)](https://man.openbsd.org/ssh_config)).

Con transporte protegido y servidor confirmado comienza el inicio de sesión de usuario o máquina. Password, Public Key, certificado y otros procedimientos tienen distintos requisitos operativos y de recuperación.

## User Authentication Protocol

Después del transporte, el cliente solicita el servicio `ssh-userauth`. RFC 4252 define los métodos `publickey`, `password`, `hostbased` y `none`; Keyboard-Interactive se especifica en RFC 4256. El servidor puede exigir más autenticación e informa de los siguientes métodos permitidos ([RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html), [RFC 4256 – Keyboard Interactive Authentication](https://www.rfc-editor.org/rfc/rfc4256.html)).

**Método de autenticación** y **autorización** son independientes. Una Public Key válida puede asignarse a una cuenta mediante `authorized_keys`, servicio de directorio, `AuthorizedKeysCommand`, principals de certificado o PAM. Después, los derechos Unix, `sudo`, SELinux/AppArmor, tokens de Windows y políticas de aplicación determinan los derechos efectivos.

OpenSSH `AuthenticationMethods` puede imponer combinaciones como `publickey,keyboard-interactive:pam`. Que sean dos factores independientes depende de las fuentes de credenciales y de la vinculación al dispositivo. Dos métodos que proceden ambos del mismo agente de portátil comprometido ofrecen menos independencia de la que su sintaxis sugiere ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Procedimientos de Public Key y tipos de clave

Con autenticación de Public Key, el cliente envía una firma sobre el identificador de sesión y la solicitud de autenticación; la clave privada no sale del cliente. El servidor comprueba Public Key, firma y autorización. RFC 8332 define firmas RSA con SHA-2 para SSH; el tipo de clave `ssh-rsa` también designa históricamente semántica de firma SHA-1 y no debe equipararse con cualquier uso de una clave RSA ([RFC 8332 – RSA Keys with SHA-2 Signatures](https://www.rfc-editor.org/rfc/rfc8332.html)).

RFC 8709 estandariza Ed25519 y Ed448 para SSH. OpenSSH también admite claves `sk-` basadas en FIDO/U2F, en las que la clave privada está vinculada a una Security Key y que opcionalmente requieren presencia o verificación de usuario ([RFC 8709 – Ed25519 and Ed448 for SSH](https://www.rfc-editor.org/rfc/rfc8709.html), [OpenSSH FIDO/U2F support](https://www.openssh.com/txt/release-8.2)).

La elección sigue la compatibilidad cliente/servidor y el modelo de amenazas. La longitud de clave por sí sola no decide la seguridad: el almacenamiento privado, la passphrase, el acceso al agente, la validez, el principal, la revocación y las opciones de autorización son igual de importantes.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Schlüssel- und Agentinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">ssh-keygen -t ed25519 -a 64 -f $env:USERPROFILE\\.ssh\\id_admin_ed25519
ssh-keygen -lf $env:USERPROFILE\\.ssh\\id_admin_ed25519.pub
Get-Service ssh-agent
ssh-add -l
ssh-add -T $env:USERPROFILE\\.ssh\\id_admin_ed25519.pub</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh-keygen -t ed25519 -a 64 -f "$HOME/.ssh/id_admin_ed25519"
ssh-keygen -lf "$HOME/.ssh/id_admin_ed25519.pub"
ssh-add -l
ssh-add -T "$HOME/.ssh/id_admin_ed25519.pub"
ssh-keygen -y -f "$HOME/.ssh/id_admin_ed25519" | diff - "$HOME/.ssh/id_admin_ed25519.pub"</code></pre>
  </div>
</div>

[`ssh-add`](https://man.openbsd.org/ssh-add) gestiona identidades en el agente y puede probar la capacidad de firma de una Public Key. [`Get-Service`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) muestra el servicio de agente de Windows. Los archivos de claves privadas y los sockets de agente reciben ACL mínimas; los ejemplos no generan claves centrales de CA ni de host.

## `authorized_keys` como objeto de política

Una línea en `authorized_keys` puede incluir opciones antes de la clave. `from=`, `command=`, `restrict`, `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty`, `permitopen=` y `environment=` limitan el uso. `restrict` establece un conjunto defensivo de opciones; pueden seguir autorizaciones explícitas ([sshd(8) – AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)).

Una clave para backup o deployment no recibe acceso genérico de shell. Se vincula a la red de origen, al comando forzado y a los destinos de reenvío necesarios. Los wrappers validan los argumentos como entrada estructurada y no utilizan concatenación de shell no comprobada.

`authorized_keys` es configuración distribuida. El inventario incluye fingerprint, propietario, propósito, cuentas de destino, origen, vencimiento, último uso y revocación. Las claves huérfanas son credenciales duraderas sin visibilidad central.

## Certificados OpenSSH y principals

Los certificados OpenSSH no son certificados X.509. Una SSH-CA firma una Public Key con número de serie, Key ID, tipo, principals y ventana de validez. Los User Certificates autentican usuarios frente a servidores; los Host Certificates autentican servidores frente a clientes. Las Critical Options deben entenderse; de lo contrario, el certificado se rechaza. Las Extensions conceden funciones como PTY o Forwarding ([IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

Los certificados acortan la vida de las credenciales y centralizan la emisión, pero no eliminan la autorización. Los servidores determinan mediante `TrustedUserCAKeys` y Authorized Principals qué CA y principals se aplican a cada cuenta. Los clientes vinculan una Host-CA en `known_hosts` con `@cert-authority` a patrones de nombres.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">ssh-keygen -L -f $env:USERPROFILE\\.ssh\\id_admin_ed25519-cert.pub
ssh-keygen -lf $env:USERPROFILE\\.ssh\\id_admin_ed25519-cert.pub
ssh -G $env:SSH_ALIAS | Select-String 'hostname|user|identityfile|certificatefile|proxyjump'
ssh -vvv $env:SSH_ALIAS</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh-keygen -L -f "$HOME/.ssh/id_admin_ed25519-cert.pub"
ssh-keygen -lf "$HOME/.ssh/id_admin_ed25519-cert.pub"
ssh -G "$SSH_ALIAS" | grep -E '^(hostname|user|identityfile|certificatefile|proxyjump) '
ssh -vvv "$SSH_ALIAS"</code></pre>
  </div>
</div>

[`grep`](https://www.gnu.org/software/grep/manual/grep.html) y [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) solo filtran la configuración efectiva del cliente. La salida de depuración de `ssh -vvv` puede contener nombres de usuario, rutas, infraestructura de destino y fingerprints ofrecidos, y se trata como un artefacto de diagnóstico sensible.

## ssh-agent y Agent Forwarding

Un agente mantiene desbloqueadas claves privadas o handles de firma y responde a solicitudes de firma a través de un socket local. Los procesos con acceso a este socket pueden abusar del agente sin exportar la Private Key. Por tanto, la passphrase y la clave de hardware no limitan toda posible compromisión del agente.

Agent Forwarding transmite el canal del agente a un host remoto. Root o un proceso comprometido en ese host puede realizar solicitudes de firma durante la sesión. OpenSSH advierte expresamente de este riesgo; `ProxyJump` transporta la conexión a través de un bastión sin poner allí a disposición el socket del agente ([ssh(1) – `-A` and `-J`](https://man.openbsd.org/ssh)).

Si un destino solo es accesible mediante un Jump Host, los modelos preferidos son:

- `ProxyJump`/`ProxyCommand` sin almacenamiento de credenciales en el bastión;
- certificados SSH de corta duración con principals específicos del destino;
- una clave independiente por dominio de confianza;
- Session Brokering controlado con auditoría, si la interactividad debe intermediarse centralmente.

Después de un inicio de sesión correcto, SSH abre canales lógicos dentro de la misma conexión. Shell, comando individual, SFTP y Port Forwarding comparten así transporte e identidad, pero permanecen como operaciones separadas.

## Connection Protocol y Channels

Después de una autenticación correcta, SSH abre canales lógicos. RFC 4254 define windowing, datos de canal, solicitudes y tipos. Una sesión puede solicitar un PTY, una shell, un comando `exec` individual o un subsistema como SFTP. El TCP Forwarding abre canales `direct-tcpip` o `forwarded-tcpip` ([RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)).

El multiplexing permite que varias conexiones de cliente compartan un transporte existente. Reduce los costes de KEX y autenticación, pero aumenta el impacto de un Control Socket comprometido. `ControlPath`, los permisos, la persistencia y la vinculación al destino se establecen explícitamente.

Los keepalives tienen distintos niveles: TCPKeepAlive comprueba la ruta TCP, `ServerAliveInterval`/`ClientAliveInterval` utilizan mensajes SSH. No evitan bloqueos de aplicaciones y los valores agresivos pueden finalizar sesiones ante interrupciones breves de red.

## Port Forwarding y límites de confianza

Local Forwarding (`-L`) abre un listener en el lado del cliente y conecta desde la red del servidor hacia el destino. Remote Forwarding (`-R`) abre un listener en el lado del servidor o remoto. Dynamic Forwarding (`-D`) proporciona un proxy SOCKS. Unix-Socket-/Streamlocal-Forwarding complementa TCP.

Forwarding puede eludir la segmentación. Un usuario sin privilegios de shell puede alcanzar interfaces administrativas internas o bases de datos mediante un reenvío permitido. Los servidores lo controlan con `AllowTcpForwarding`, `PermitOpen`, `PermitListen`, `GatewayPorts`, bloques Match y opciones de Authorized Key ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Bastion und Tunnelprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection $env:BASTION_HOST -Port $env:SSH_PORT
ssh -J $env:BASTION_ALIAS -N -L 127.0.0.1:8443:$env:INTERNAL_HOST:443 $env:TARGET_ALIAS
Get-NetTCPConnection -LocalPort 8443 -State Listen
ssh -O check $env:TARGET_ALIAS</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz "$BASTION_HOST" "$SSH_PORT"
ssh -J "$BASTION_ALIAS" -N -L "127.0.0.1:8443:$INTERNAL_HOST:443" "$TARGET_ALIAS"
ss -lntp 'sport = :8443'
ssh -O check "$TARGET_ALIAS"</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban la ruta TCP hacia el bastión. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) confirman el listener local. El túnel `-N` no ejecuta ningún comando remoto; la autorización y el [TLS](/kb/tls) de destino siguen siendo límites independientes.

## SFTP, SCP y permisos de archivos

SFTP es un protocolo binario de transferencia de archivos que normalmente se ejecuta como subsistema SSH. No es FTP sobre SSH. OpenSSH puede ejecutar `internal-sftp` sin un binario de servidor SFTP externo y limitarlo a transferencia de archivos mediante `ChrootDirectory` más `ForceCommand internal-sftp` ([sftp(1)](https://man.openbsd.org/sftp), [sftp-server(8)](https://man.openbsd.org/sftp-server)).

La CLI `scp` utilizaba históricamente el protocolo remoto de tipo SCP/RCP y, en líneas modernas de OpenSSH, utiliza SFTP de forma predeterminada; `-O` fuerza el protocolo heredado. Los wildcards, el procesamiento de shell remoto y la compatibilidad del servidor difieren ([scp(1)](https://man.openbsd.org/scp)). Por ello, los runbooks indican explícitamente la CLI, el modo de protocolo y los metadatos de archivo deseados.

La autorización de transferencia de archivos incluye propiedad del chroot, rutas de destino, UID/GID o ACL de Windows, umask, symlinks, cuota y almacenamiento atómico. La carga a un directorio de correo o automatización suele requerir un límite de cuarentena/renombrado en lugar de procesar directamente archivos escritos parcialmente.

## Política del lado del servidor y bloques Match

`sshd_config` se evalúa de arriba abajo; para cada palabra clave, en principio se aplica el primer valor obtenido. `Match` cambia el ámbito por usuario, grupo, host, dirección, LocalAddress, LocalPort o RDomain. Los archivos Include, valores predeterminados de distribución y opciones de línea de comandos pueden complementar el archivo principal visible ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

Una baseline sólida define:

- usuarios/grupos y redes de origen permitidos;
- métodos de autenticación necesarios y excepción de Break Glass;
- acceso Root-/Administrator y Privilege Escalation;
- Forwarding, Agent, X11, PTY y subsistemas;
- MaxAuthTries, LoginGraceTime, límites de sesión/inicio y keepalives;
- Host Keys, algoritmos y rekey;
- fuentes de AuthorizedKeys/Principals;
- logging, banner y aceptación del entorno;
- desviaciones específicas de Match.

Las «listas de endurecimiento» generales pueden bloquear appliances o no surtir efecto si un valor Match aceptado posteriormente se aplica de otro modo. Lo decisivo es la configuración efectiva para una conexión concreta.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Serverpolicy und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">sshd.exe -t
sshd.exe -T -C user=$env:SSH_USER,host=$env:SSH_HOST,addr=$env:CLIENT_IP
Get-NetTCPConnection -LocalPort $env:SSH_PORT -State Listen
Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 100</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sshd -t
sshd -T -C "user=$SSH_USER,host=$SSH_HOST,addr=$CLIENT_IP"
ss -lntp "sport = :$SSH_PORT"
journalctl --unit ssh --unit sshd --since '-60 minutes' --no-pager</code></pre>
  </div>
</div>

`sshd -t` valida la sintaxis y las claves, `sshd -T -C` muestra la configuración efectiva para parámetros de prueba ([sshd(8)](https://man.openbsd.org/sshd)). [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) y [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) consultan los logs de la plataforma. Un reload correcto y una segunda sesión de administrador probada preceden al cierre de la sesión existente.

## Fuerza bruta, límites de tasa y límites de red

Los servicios SSH expuestos a Internet reciben intentos de inicio de sesión automatizados. El firewall de origen, acceso VPN/Zero-Trust, bastión, `AllowUsers`/`AllowGroups`, autenticación fuerte y vida corta de las credenciales reducen la superficie de ataque. Otro puerto reduce el ruido de logs, pero no modifica ni la autenticación ni la vulnerabilidad.

Las herramientas de bloqueo perimetral pueden limitar intentos repetidos; procesan datos de log no confiables y necesitan sus propias reglas de allowlist, IPv6, NAT, lockout y recuperación. OpenSSH `MaxStartups` limita probabilísticamente las conexiones paralelas no autenticadas; `PerSourceMaxStartups` y `PerSourcePenalties` pueden tener en cuenta el comportamiento de origen. La capacidad y el riesgo de DoS se prueban en condiciones reales de proxy/NAT ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Logging, auditoría y evidencia de sesión

Los logs SSH suelen acreditar origen de la conexión, usuario, método de autenticación/fingerprint, inicio/final de sesión y eventos seleccionados de subsistemas. No contienen automáticamente cada comando de shell ejecutado. El historial de shell está controlado por el usuario y no es una auditoría protegida contra manipulación.

Para la administración privilegiada se correlacionan:

- identidad de origen, dispositivo y decisión de acceso antes de SSH;
- salto de bastión/proxy y host de destino;
- decisión del cliente sobre Host Key/CA;
- principal de usuario, método de autenticación, fingerprint de clave o serie de certificado;
- tipo de sesión/canal, inicio, fin y estado de salida;
- eventos de privilegios `sudo`/Windows y auditoría del sistema de destino;
- ruta de transferencia de archivos, hash o ticket de cambio en deployments sensibles.

La grabación de sesiones modifica la protección de datos, la captura de secretos y el riesgo operativo. Passwords, tokens, contenidos de correo y Private Keys no deben acabar sin control en transcripciones. La retención, el acceso y la protección contra manipulación se definen antes de activarla.

## OpenSSH para Windows

Microsoft proporciona OpenSSH Client y Server como características de Windows. El servidor utiliza el servicio `sshd`, la configuración bajo `%ProgramData%\ssh\sshd_config`, cuentas y ACL de Windows; los logs pueden aparecer en el canal `OpenSSH/Operational` ([Microsoft Learn – OpenSSH overview](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview), [Microsoft Learn – OpenSSH Server configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)).

Las claves de administradores de Windows pueden, según la configuración, encontrarse en un archivo central `administrators_authorized_keys` cuya ACL debe ser estricta. Shell, subsistemas, valor de registro DefaultShell y resolución de rutas difieren de Unix. Un comando de cliente portable no implica una autorización de servidor idéntica.

## Backup, recuperación y Break Glass

La recuperación SSH no abarca solo `sshd_config`:

- Host Keys o Host-CA más claves privadas protegidas;
- User-CA, emisor de certificados, principals y listas de revocación;
- Authorized Keys/Commands y su fuente de directorio/gestión de configuración;
- política de servidor, archivos Include, dependencias de PAM/SSSD/AD y MFA;
- firewall, bastión, DNS/SSHFP y fuente de tiempo;
- cuenta Break-Glass con credencial verificada fuera de línea;
- consola fuera de banda si falla SSH de red;
- logs de auditoría e inventario de claves/certificados.

Restaurar la misma Host Key privada conserva la identidad del servidor, pero puede ser peligroso tras una compromisión. Por ello, la recuperación decide entre **continuidad** y **nuevo enrolamiento con rotación**. Una prueba de restauración utiliza una dirección de destino aislada y comprueba principal de host, métodos de autenticación, política Match, Privilege Escalation y correlación de logs.

El diagnóstico lee el establecimiento de conexión en orden: TCP, intercambio de versiones, Key Exchange, Host Key, autenticación de usuario y apertura de canal.

## Diagnóstico por fase de protocolo

SSH transcurre en fases claras. Determinar primero si se ven afectados TCP, el intercambio de claves, la comprobación de host, el inicio de sesión de usuario o el canal evita cambios no dirigidos en claves y configuración de servidor.

| Fase | Evidencia típica | Clase de error frecuente |
|---|---|---|
| TCP | DNS, ruta, listener, SYN/ACK | firewall, puerto incorrecto, proxy/bastión |
| Versión | cliente `-vvv`, log del servidor | implementación incompatible o ajena |
| KEX | algoritmos negociados | ninguna KEX/Cipher/MAC común, problema Strict-KEX |
| Host Key | fingerprint, principal, CA/known_hosts | host incorrecto, rotación, destino DNS/LB |
| Userauth | métodos ofrecidos, fingerprint/Cert, PAM | clave no autorizada, principal, ACL, MFA |
| Connection | solicitud de canal, PTY, subsistema, Forwarding | ForceCommand, PermitOpen, SFTP/Chroot, política |
| Proceso remoto | estado de salida, stderr, entorno, derechos | ruta de shell, sudo, locale, aplicación |

Un `Permission denied (publickey)` se produce tras un transporte y una confianza de host correctos. Un timeout antes del banner no es un problema de clave. El diagnóstico orientado a fases evita modificar credenciales o listas de algoritmos sin evidencia.

## Historia técnica

Tatu Ylönen desarrolló el SSH original en 1995 como sustituto de herramientas no cifradas de acceso remoto y transferencia de archivos. SSH-1 y SSH-2 diferían fundamentalmente; el IETF SECSH Working Group estandarizó posteriormente la arquitectura SSH-2 en RFC 4251 a 4254 ([OpenSSH – History](https://www.openssh.com/history.html), [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251.html)).

OpenSSH surgió en el proyecto OpenBSD a partir de código SSH libre y se hizo portable a otros sistemas operativos. Reunió cliente, servidor, herramientas de claves, agente, SCP y SFTP, y marcó numerosas extensiones de facto como su propio formato de certificado ([OpenSSH Project](https://www.openssh.com/)).

La criptografía evolucionó desde combinaciones antiguas de Diffie-Hellman, DSA, RSA/SHA-1, CBC y MAC independientes hacia Curve25519/ECDH, Ed25519, RSA/SHA-2 y procedimientos AEAD. Al mismo tiempo, la confianza operativa pasó de claves individuales distribuidas manualmente a autenticadores de hardware, Host-/User-CAs, certificados de corta duración y bastiones basados en identidad. Las capas de protocolo siguieron siendo reconocibles; la operación de confianza y ciclo de vida se hizo más extensa.

## Lista de comprobación para administradores

La lista de comprobación resume ruta de conexión, confianza de host, derechos de usuario, operación de claves y registro. Es adecuada para la aprobación y revisión periódica de un acceso SSH.

| Pregunta | Evidencia operativa |
|---|---|
| ¿Qué ruta? | cliente/contexto, DNS, IP de origen, Bastion/ProxyJump, IP de destino, puerto, proceso listener |
| ¿Qué identidad de servidor? | fingerprint de Host Key o certificado de host, principal, CA, validez, rotación |
| ¿Qué algoritmos? | procedimientos KEX, Hostkey, Cipher/AEAD y MAC negociados, excepciones heredadas |
| ¿Quién se autentica? | cuenta, principal, fingerprint de Public Key/serie de certificado, ruta MFA/PAM, dispositivo de origen |
| ¿Qué está autorizado? | grupos, bloque Match, opciones de Authorized Key, sudo/token, Subsystem/Command |
| ¿Qué canales? | Shell/Exec/SFTP, PTY, Local/Remote/Dynamic Forward, Agent/X11, PermitOpen |
| ¿Qué política se aplica realmente? | `sshd -T -C`, `ssh -G`, Includes, valores predeterminados de distribución y contexto Match |
| ¿Qué se registra? | Connect/Auth/Session/Channel, Privilege Escalation, transferencia y correlación central |
| ¿Cómo se rota? | confianza paralela de Host Key/CA, certificados de usuario cortos, inventario y revocación |
| ¿Cómo se recupera? | consola OOB, Host-/User-CA, política, Principals/Keys, DNS/bastión, Break Glass y auditoría |

## Fuentes

- [RFC 4251 – SSH Protocol Architecture](https://www.rfc-editor.org/rfc/rfc4251.html)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)
- [RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)
- [RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)
- [RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)
- [ssh(1)](https://man.openbsd.org/ssh)
- [sshd(8)](https://man.openbsd.org/sshd)
- [RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)
- [IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [ssh-keyscan(1)](https://man.openbsd.org/ssh-keyscan)
- [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen)
- [ssh_config(5)](https://man.openbsd.org/ssh_config)
- [RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html)
- [RFC 4256 – Keyboard Interactive Authentication](https://www.rfc-editor.org/rfc/rfc4256.html)
- [sshd_config(5)](https://man.openbsd.org/sshd_config)
- [RFC 8332 – RSA Keys with SHA-2 Signatures](https://www.rfc-editor.org/rfc/rfc8332.html)
- [RFC 8709 – Ed25519 and Ed448 for SSH](https://www.rfc-editor.org/rfc/rfc8709.html)
- [OpenSSH FIDO/U2F support](https://www.openssh.com/txt/release-8.2)
- [ssh-add(1)](https://man.openbsd.org/ssh-add)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service)
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html)
- [Microsoft Learn – Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)
- [RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [sftp(1)](https://man.openbsd.org/sftp)
- [sftp-server(8)](https://man.openbsd.org/sftp-server)
- [scp(1)](https://man.openbsd.org/scp)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft Learn – OpenSSH overview](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview)
- [Microsoft Learn – OpenSSH Server configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenSSH – History](https://www.openssh.com/history.html)
- [OpenSSH Project](https://www.openssh.com/)
