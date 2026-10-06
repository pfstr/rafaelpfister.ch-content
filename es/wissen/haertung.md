---
title: "Endurecimiento: líneas base, límites de confianza y controles administrativos"
blatt: "haertung"
description: "Endurecimiento técnico para administradores de mensajería e infraestructura: líneas base de seguridad, funcionalidad mínima y mínimo privilegio, identidad y plano de gestión, límites de red y protocolos de correo, claves, riesgos de parches y cadena de suministro, registro, detección de desviaciones y verificación."
fakten:
  - label: Objetivo
    wert: reducir de forma controlada la superficie de ataque, la confianza implícita y el radio de impacto
    href: https://csrc.nist.gov/pubs/sp/800/123/final
  - label: Principios de diseño
    wert: Fail-safe Defaults · Complete Mediation · Least Privilege
    href: https://web.mit.edu/Saltzer/www/publications/protection/Basic.html
  - label: Línea base
    wert: estado objetivo documentado, aprobado y verificable
    href: https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software
  - label: Funcionalidad mínima
    wert: solo las funciones, puertos, protocolos, software y servicios necesarios para el negocio
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf
  - label: Modelo de acceso
    wert: sin concesión implícita de confianza únicamente por la ubicación de red o la propiedad
    href: https://csrc.nist.gov/pubs/sp/800/207/final
  - label: Superficies de control
    wert: host · identidad · gestión · red/protocolo · aplicación/datos · cadena de suministro/telemetría
    href: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
  - label: Límites del correo
    wert: relay de Internet · Submission/Access · entrega interna · administración
    href: https://csrc.nist.gov/pubs/sp/800/177/r1/final
  - label: Acceso administrativo
    wert: personal · con autenticación fuerte · privilegios mínimos · trazable
    href: https://www.cisecurity.org/controls/access-control-management
  - label: Red de gestión
    wert: zona de administración restrictiva, separada de los flujos de datos productivos
    href: https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure
  - label: Aplicación de parches
    wert: identificar · priorizar · obtener · instalar · verificar la instalación
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Evidencia de desviaciones
    wert: comparación entre estado objetivo y real de configuración, servicios, cuentas, reglas y registros
    href: https://www.cisecurity.org/controls/audit-log-management
  - label: Referencias
    wert: línea base del fabricante · CIS Benchmark · BSI IT-Grundschutz · propia aprobación de riesgos
    href: https://www.cisecurity.org/cis-benchmarks
werbung:
  - tools
  - newsletter
ctaThemen:
  - haertung
  - messaging
  - security
translationSourceHash: f97c0de00da2f8c1a5d89407341fb9a46b9fc213b01559d4e297feb29c0a3c83
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:13:00.349Z
translationReview: automatic
---

# Endurecimiento: líneas base, límites de confianza y controles administrativos

El endurecimiento es la transformación controlada de un sistema a un **estado objetivo documentado, justificado y verificable**. Restringe funciones, accesos y relaciones de confianza al mínimo operativo, sin perjudicar de forma incontrolada el servicio previsto. El resultado no es la lista más extensa posible de opciones de seguridad activadas, sino una arquitectura en la que cada interfaz accesible, cada privilegio y cada flujo de datos tienen una finalidad, un propietario y una evidencia definidos. NIST describe la seguridad de servidores en consecuencia como la selección, implementación y mantenimiento continuo de controles adecuados; CIS Control 4 exige configuraciones seguras para activos y software ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)).

Un valor predeterminado de producto no es automáticamente inseguro ni constituye automáticamente la línea base correcta para producción. Los fabricantes deben cubrir amplios rangos funcionales y de compatibilidad. En cambio, el operador conoce la exposición, las necesidades de protección, las dependencias, la capacidad de recuperación y los riesgos residuales aceptados. Por ello, una línea base combina las recomendaciones del fabricante, una referencia CIS o BSI adecuada y la propia decisión arquitectónica. Las desviaciones no se mantienen de forma implícita, sino que se documentan con su causa, riesgo, control compensatorio, propietario y fecha de vencimiento. Los CIS Benchmarks son recomendaciones de configuración basadas en consenso; el BSI separa los requisitos generales de servidor de los requisitos relacionados con el correo ([CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks), [BSI SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

El endurecimiento comienza con el servicio real y sus vías de administración, no con una lista arbitraria de valores del registro. Primero se inventarían componentes, identidades, datos y rutas de red; de ello se derivan la línea base, las excepciones y los controles verificables.

## De la lista de comprobación al modelo de superficies de control

Para los administradores, una revisión estructurada por superficies de control es más sólida que una única lista de comprobación del host. El siguiente modelo resume controles de NIST SP 800-53: gestión de configuración y funcionalidad mínima, acceso y autenticación, protección de comunicaciones, integridad del sistema, auditoría, así como controles de cadena de suministro y recuperación. Es un modelo de revisión, no una norma adicional ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [NIST SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

| Superficie de control | Objeto protegido | Valores objetivo típicos | Evidencia operativa |
|---|---|---|---|
| Host y entorno de ejecución | Sistema operativo, contenedores, servicios, permisos de archivos, funciones de kernel/entorno de ejecución | paquetes mínimos, listeners mínimos, procesos sin privilegios, permisos de archivos seguros | inventario de servicios y puertos, análisis de línea base, comprobación de integridad |
| Identidad y autorización | Personas, cuentas de servicio, roles, tokens, certificados | identidad administrativa personal, MFA, mínimo privilegio, identidades de servicio separadas | revisión de cuentas/roles, eventos de autenticación y privilegios |
| Plano de gestión | GUI, API, SSH, PowerShell, SNMP, canal de copia de seguridad y actualización | zona administrativa dedicada, protocolos cifrados, denegación predeterminada, proceso Break-Glass | rutas de gestión accesibles, registros AAA, cambios de configuración |
| Red y protocolos | Listeners, salida, TLS, DNS, rutas de relay, Submission y acceso | flujos explícitos por rol, sin protocolos de texto sin cifrar innecesarios, certificados verificados | reglas de firewall, pruebas de paquetes/TLS, monitorización de DNS y flujo de correo |
| Aplicación y datos | Cola, buzón, política, analizador, datos temporales, claves | roles separados, permisos de archivos restrictivos, valores predeterminados seguros, permisos de analizador y salida limitados | pruebas negativas funcionales, registros de cola/política, revisión de secretos y claves |
| Cadena de suministro y telemetría | Imágenes, paquetes, firmas, dependencias, registros, tiempo | artefactos compatibles, procedencia verificada, proceso de parches, registros centrales no alterados | inventario, hash/firma, informe de parches y desviaciones, prueba de alertas |

NIST Zero Trust añade un límite importante: un usuario, servicio o dispositivo no obtiene confianza solo por estar en la red interna o pertenecer a la organización. La autenticación y autorización se evalúan antes de acceder a un recurso. La segmentación sigue siendo útil, pero no sustituye a la identidad, la política y la decisión continua ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-haertung.svg?v=20260813" title="Interaktive Infografik: Härtungsmodell für Messaging-Systeme mit externen Mailgrenzen, Managementebene, Identität, Host, Anwendung, Daten, Lieferkette, Telemetrie und Verifikationsschleife" loading="lazy">
  <a href="/images/kb-interaktiv-haertung.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## La pila tecnológica como inventario de endurecimiento

El endurecimiento no posee su propia pila de lenguajes de programación o productos. Se aplica a la **pila tecnológica realmente operada**. Por ello, el inventario incluye como mínimo firmware e hipervisor, sistema operativo o base de contenedores, entorno de ejecución y lenguaje de programación, servidores web y de correo, bibliotecas y analizadores, base de datos, cola y almacén de objetos, componentes de identidad y claves, protocolos de gestión y rutas de registro y actualización. NIST CM-8 exige un inventario de los componentes del sistema; CM-7 vincula este inventario con la limitación a las funciones necesarias ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)).

Para cada capa se registran fabricante, procedencia, estado de soporte, módulos activos, privilegios, listeners, destinos de salida, fuente de configuración, vía de parcheo y objeto de recuperación. Solo así se puede aplicar una recomendación como «desactivar servicios innecesarios» a un proceso concreto y sus dependencias, sin afectar al flujo de correo ni a la recuperabilidad.

## Límites de confianza de una plataforma de mensajería

Una plataforma de mensajería posee varias vías de entrada técnicamente diferentes. No deben tratarse con una sola regla de «solo conexiones autenticadas»:

- **Relay de Internet:** un MTA accesible públicamente recibe en [SMTP](/kb/smtp) puerto 25 mensajes de MTAs no conocidos previamente. Aquí, la verificación de destinatarios, la política de relay, los estados de protocolo, los límites de recursos, la reputación y los controles de contenido limitan el riesgo; la autenticación de usuario no es el modelo de confianza general.
- **Message Submission:** usuarios y aplicaciones envían mensajes nuevos como remitentes identificados. Submission separa este rol del relay; la autenticación, autorización, límites de tasa y [TLS](/kb/tls) forman parte de la política.
- **Acceso al correo:** IMAP, POP o HTTP acceden a datos existentes en buzones. RFC 8314 considera obsoleto el texto sin cifrar para Submission y acceso al correo, y prefiere TLS implícito.
- **Rutas internas de servicio:** gateways, directorios, bases de datos, almacenes de objetos, colas y escáneres se comunican entre sí como servicios. La ubicación de red por sí sola no es una prueba de identidad; cada conexión necesita una ruta mínima y dirigida de datos y autorizaciones.
- **Gestión y actualizaciones:** la GUI administrativa, la API, SSH, la administración remota, la copia de seguridad y la obtención de software tienen un radio de impacto mayor que una ruta de cliente normal y pertenecen a una zona independiente de gestión y confianza.

NIST SP 800-177 trata la autenticación de dominio, TLS y la criptografía de contenido como mecanismos de seguridad complementarios en torno al SMTP que sigue utilizándose. RFC 8314 separa deliberadamente el relay de Submission y Access. De ello se deduce que el endurecimiento debe comprobar por rol **quién puede iniciar una conexión, qué identidad porta, qué datos procesa y con quién puede seguir comunicándose** ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final), [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

El inventario muestra qué debe protegerse. Una línea base lo traduce en configuraciones concretas que se versionan, prueban y modifican de forma trazable cuando existen excepciones justificadas.

## Ciclo de vida de la línea base y desviación controlada

Una línea base eficaz sigue un ciclo de vida:

1. **Inventariar:** registrar producto, rol, versión de software, módulos, listeners, cuentas, flujos de datos, claves y dependencias.
2. **Elegir la referencia:** asignar la línea base del fabricante, el CIS Benchmark, el módulo BSI y los requisitos legales al rol concreto.
3. **Adaptar:** eliminar reglas no aplicables, añadir reglas más estrictas y justificar las desviaciones según el riesgo.
4. **Pilotar:** comprobar función, rendimiento, flujo de correo, monitorización, copia de seguridad y [Disaster Recovery](/kb/backup-dr) en un entorno representativo.
5. **Desplegar de forma declarativa:** usar GPO, gestión de configuración, imagen, Policy-as-Code o API del fabricante en lugar de cambios manuales individuales.
6. **Comprobar continuamente:** detectar desviaciones, cuentas nuevas, listeners, paquetes, certificados, reglas y cambios de línea base.
7. **Retirar del servicio:** eliminar de forma controlada acceso, DNS, certificados, claves, datos, copias de seguridad y monitorización.

El Microsoft Security Compliance Toolkit puede almacenar, analizar, comparar, editar y aplicar como GPO las líneas base recomendadas de Windows. No sustituye la adaptación: primero se prueba una línea base en un grupo piloto para verificar su función y efectos secundarios. Lo mismo se aplica a las recomendaciones CIS y BSI ([Microsoft Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

## Funcionalidad mínima: servicios, puertos y software

El control NIST CM-7 exige configurar un sistema con las capacidades necesarias para el negocio y prohibir o restringir funciones, puertos, protocolos, software o servicios. La pregunta técnica no es «¿es seguro el puerto 443?», sino: **¿qué proceso escucha en qué dirección, para qué rol, desde qué zona y con qué modelo de parches e identidad?** ([NIST SP 800-53 Rev. 5, CM-7](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

Las interfaces web innecesarias, los puntos finales de depuración, los protocolos de descubrimiento, los listeners de bases de datos locales y los servicios de gestión heredados se desactivan. Los servicios necesarios se vinculan, en la medida de lo posible, solo a las interfaces previstas. Un MTA puede escuchar públicamente en SMTP, pero no su base de datos. Un puerto de API administrativa puede ser necesario, pero no pertenece automáticamente a Internet. CISA recomienda desactivar para la infraestructura de comunicaciones servicios no necesarios o sin cifrar como Telnet, FTP, TFTP, HTTP y variantes antiguas de SNMP, e inventariar continuamente los servicios accesibles públicamente ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Inventariar listeners y servicios activos

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Service | Where-Object Status -eq Running |
  Sort-Object Name
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
systemctl list-units --type=service --state=running
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestran listeners y procesos locales. [`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service) y [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) muestran servicios activos. La comparación entre el estado objetivo y el real requiere posteriormente una matriz aprobada de puertos y servicios; un listener desconocido es un hallazgo, pero aún no un análisis de causa.

Tras eliminar las funciones innecesarias, permanecen las cuentas y servicios que realmente pueden actuar. Sus derechos, vías de inicio de sesión y secretos determinan la mayor parte de la superficie de ataque administrativa.

## Identidades, cuentas y mínimo privilegio

Las cuentas se separan por rol: identidad de usuario normal, identidad administrativa personal, cuenta de servicio no interactiva y cuenta de emergencia estrictamente controlada. Los inicios de sesión administrativos compartidos impiden una atribución fiable. Las cuentas cotidianas con privilegios elevados permanentes aumentan el radio de impacto del phishing y de compromisos del navegador y del cliente. CIS Control 5 abarca cuentas de usuario, administración y servicio; CIS Control 6 cubre la asignación, mantenimiento y revocación de sus credenciales y privilegios ([CIS Control 5: Account Management](https://www.cisecurity.org/controls/account-management), [CIS Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)).

La identidad centralizada mejora los procesos Joiner/Mover/Leaver, pero no sustituye una vía de emergencia local. Una interrupción de LDAP, Kerberos o SSO no debe imposibilitar el acceso autorizado de recuperación. Por tanto, las cuentas Break-Glass son una excepción deliberadamente pequeña: documentadas offline, fuertemente protegidas, no usadas en el día a día, con alerta inmediata al utilizarlas y probadas regularmente. Las identidades de servicio no reciben inicio de sesión interactivo y solo disponen de los derechos, destinos de red y secretos de su tarea. Cuando sea posible, se prefieren tokens de corta duración, Managed Identities o certificados a contraseñas estáticas; su ciclo de vida y recuperación siguen formando parte de la operación.

MFA reduce el riesgo de contraseñas robadas, pero no sustituye los privilegios mínimos ni la recuperación segura. NIST Zero Trust exige una decisión de acceso para el sujeto y, si procede, el dispositivo antes de la sesión; la ubicación de red o la posesión por sí solas no bastan ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

### Revisar cuentas locales y privilegiadas

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokales Konteninventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires
Get-LocalGroupMember -Group Administrators
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
getent passwd
getent group sudo wheel
```

  </div>
</div>

[`Get-LocalUser`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser) y [`Get-LocalGroupMember`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember) consultan las cuentas locales de Windows y las pertenencias a grupos. [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) consulta las bases de datos de servicio de nombres configuradas y, por ello, puede mostrar cuentas locales y resueltas centralmente. Una revisión también debe incluir los roles reales en el producto, tokens de API, claves SSH, certificados e IAM en la nube.

## Plano de gestión y rutas de administración

El plano de gestión puede modificar configuración, claves, enrutamiento, actualizaciones y registros, y merece un límite más estricto que la ruta de datos útiles. CISA recomienda una red de gestión fuera de banda, separada física o lógicamente del flujo de datos operativo, reglas de denegación predeterminada, estaciones de trabajo administrativas dedicadas y registro AAA centralizado. También se deben limitar las conexiones de gestión laterales entre dispositivos ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

Para los sistemas de mensajería, esto significa:

- La GUI administrativa, la API, SSH y la administración remota solo son accesibles desde zonas administrativas definidas o mediante una ruta bastión controlada.
- Los certificados, cuentas y reglas de firewall de gestión y flujo de correo se gestionan por separado.
- Las conexiones salientes del plano de gestión se restringen a destinos de actualización, identidad, tiempo, registros y copia de seguridad.
- Los cambios de configuración requieren identidad personal, preferiblemente MFA, auditoría y, para riesgos elevados, aprobación de cuatro ojos.
- Una ruta de emergencia funciona sin la plataforma habitual de identidad o gestión, pero no se opera como acceso permanente encubierto.

SSH es solo un transporte para la administración; su seguridad depende de la autenticación, los grupos de usuarios permitidos, los algoritmos de claves, el reenvío, los permisos de archivos y los permisos del destino. Examinar la configuración efectiva del servidor en lugar de solo el archivo de texto permite detectar inclusiones y valores predeterminados.

### Mostrar la configuración efectiva del servidor SSH

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für effektive OpenSSH-Konfiguration">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
& "$env:WINDIR\System32\OpenSSH\sshd.exe" -T
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sshd -T
```

  </div>
</div>

[`Get-WindowsCapability`](https://learn.microsoft.com/powershell/module/dism/get-windowscapability) muestra el componente OpenSSH instalado; Microsoft documenta rutas y particularidades de [`sshd_config`](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration). [`sshd -T`](https://man.openbsd.org/sshd) muestra la configuración efectiva. Las opciones no deben establecerse a ciegas siguiendo listas de Internet: la disponibilidad del acceso de emergencia, los tipos de claves utilizados y la automatización deben incluirse en las pruebas.

## Rutas de red: denegación predeterminada con dirección explícita

Una regla de firewall se documenta como un contrato dirigido: **origen, destino, protocolo/puerto, iniciador, identidad, finalidad, propietario y fecha de vencimiento**. «El servidor de correo puede acceder a Internet» no es una especificación técnica. Un listener SMTP entrante necesita destinos de salida distintos de los de un trabajador de sandbox de malware o una API administrativa. El filtrado de salida limita el control y mando, la exfiltración y las descargas incontroladas; debe considerar conscientemente DNS, tiempo, validación de certificados, actualizaciones y destinos de entrega.

La segmentación reduce el espacio de movimiento tras una vulneración. Es especialmente importante entre el borde de Internet, el procesamiento de correo, el almacenamiento de buzones/datos, el directorio, la gestión, la copia de seguridad y la monitorización. NIST Zero Trust advierte al mismo tiempo de utilizar la posición de red como única base de confianza. CISA recomienda denegación predeterminada para las rutas de gestión y una zona separada del tráfico de datos del cliente ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Revisar el firewall del host y la dirección de las reglas

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Firewall-Regeln">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetFirewallProfile |
  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Get-NetFirewallRule -Enabled True |
  Select-Object DisplayName, Direction, Action, Profile
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nft list ruleset
```

  </div>
</div>

[`Get-NetFirewallProfile`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile) y [`Get-NetFirewallRule`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule) muestran perfiles y reglas activas de Windows. [`nft`](https://netfilter.org/projects/nftables/manpage.html) muestra las reglas de nftables, incluidas las cadenas y la dirección. La salida se comprueba contra la matriz de flujos de datos aprobada; una política de denegación predeterminada sin los destinos necesarios de DNS, tiempo o certificados no es un estado de endurecimiento satisfactorio.

## Protocolos de correo y confianza en el transporte

Relay, Submission y Access requieren reglas distintas de TLS y autenticación. Para Submission y acceso, RFC 8314 recomienda TLS 1.2 o superior y prefiere TLS implícito; ya no debe ofrecerse acceso en texto sin cifrar. Para el relay SMTP, STARTTLS describe en cambio una negociación salto a salto. Sin una política adicional, un MTA remitente puede seguir entregando en texto sin cifrar si TLS no está disponible. [DANE](/kb/tls) y MTA-STS proporcionan políticas de transporte diferentes y más explícitas ([RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

[SPF, DKIM y DMARC](/kb/mail-auth) autentican relaciones de dominio y políticas, no cuentas de usuario ni el contenido en sí. [S/MIME y OpenPGP](/kb/verschluesselung) protegen partes del mensaje, pero no cambian nada respecto a un acceso administrativo inseguro o un almacén de claves comprometido. Una revisión de endurecimiento mantiene separadas estas prestaciones de seguridad y controla sus dependencias: DNS, certificados, claves, tiempo, informes y reglas de excepción. NIST SP 800-177 sitúa precisamente estos mecanismos complementarios en torno a SMTP y DNS ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final)).

### Comprobar accesibilidad y comportamiento de TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail- und Management-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban la ruta TCP. [`curl`](https://curl.se/docs/manpage.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestran STARTTLS, la cadena de certificados y errores. Solo la política del MTA y los registros responden si, ante un error, se retrasa, rechaza o recurre a texto sin cifrar.

## Aplicación, datos, analizadores y claves

Los servidores de correo procesan deliberadamente formatos complejos no confiables. Mensajes MIME, archivos, documentos, imágenes y HTML llegan a analizadores, escáneres, convertidores, vistas previas y sandboxes. Por ello, el endurecimiento no limita solo los puertos de red, sino también los derechos de proceso, el acceso al sistema de archivos, el almacenamiento temporal, CPU/RAM/tamaño de archivo, la recursión, la capacidad de ejecución y la salida de los componentes de análisis. Un escáner necesita acceso a un objeto de comprobación, pero no automáticamente a todos los buzones, secretos administrativos o la API de gestión.

Las colas y los directorios temporales contienen información confidencial. Se comprueban explícitamente los permisos de archivos, el cifrado, las reglas de eliminación y retención, así como los volcados de depuración. Los registros deben permitir reconstruir estados y decisiones, pero no recopilar contraseñas, tokens, claves privadas ni contenido innecesario de mensajes. Las claves se separan por finalidad: TLS, DKIM, S/MIME/OpenPGP, JWT/API y cifrado de copias de seguridad tienen ciclos de vida, autorizaciones y reglas de recuperación distintos. NIST SP 800-53 vincula el mínimo privilegio, la integridad del sistema, la protección de comunicaciones y la auditoría; BSI APP.5.3 concreta las necesidades de protección para clientes y servidores de correo electrónico ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

## Parches, imágenes y cadena de suministro

La gestión de parches es mantenimiento preventivo, no una emergencia esporádica. NIST define el proceso como identificación, priorización, obtención, instalación y verificación de parches, actualizaciones y mejoras. Para componentes de correo y gestión expuestos, la información sobre vulnerabilidades, la superficie de ataque accesible, la explotación activa, la criticidad de los datos y las compensaciones disponibles deben influir en la prioridad ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

La vía de actualización es en sí misma un límite de confianza. Los paquetes, imágenes, contenedores, complementos, firmas antivirus y firmware de dispositivos se obtienen de fuentes autenticadas y se verifican con la firma del fabricante o el hash publicado. Las dependencias y los cambios de repositorio forman parte del inventario. NIST SP 800-161 trata los riesgos de productos y servicios cuyo desarrollo, integración y suministro el operador solo puede inspeccionar o controlar de forma limitada ([NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

Una actualización de endurecimiento se prueba en una etapa representativa: inicio, flujo de correo, cola, TLS, directorio, política, monitorización, copia de seguridad y reversión. «No aplicar parches porque el correo es crítico» intercambia un riesgo operativo conocido por un riesgo de seguridad creciente. Un diseño mejor proporciona redundancia, ventanas de mantenimiento, compilaciones reproducibles y vías de recuperación probadas.

### Integridad de artefactos y eventos relevantes para la seguridad

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefakt- und Ereignisprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-gateway-update.bin -Algorithm SHA256
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=(Get-Date).AddHours(-4)} |
  Select-Object TimeCreated, Id, ProviderName, Message
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-gateway-update.bin
journalctl --since '-4 hours' --priority=notice..alert
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparan un artefacto con un hash esperado procedente de una fuente autenticada del fabricante. [`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) y [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leen eventos; la detección en producción requiere además una política de auditoría correcta, recopilación centralizada, sincronización horaria y alertas definidas.

Una configuración endurecida solo sigue siendo eficaz si se hacen visibles los cambios, los controles fallidos y las desviaciones. Por ello, el registro y la detección de desviaciones forman parte de la operación y no solo de la comprobación posterior.

## Registro, telemetría y desviaciones

Un estado endurecido no es permanente sin observación. Las señales relevantes incluyen, entre otras:

- inicios de sesión correctos y fallidos, uso de MFA y Break-Glass;
- cambios en cuentas, roles, tokens, certificados y claves;
- cambios de configuración y desviaciones respecto a la línea base;
- nuevos listeners, servicios, paquetes, tareas, contenedores o destinos salientes;
- bloqueos de firewall, conexiones de salida inesperadas y accesos de gestión;
- errores de políticas TLS, DNS, SMTP, cola y autenticación de correo;
- sensores desactivados, lagunas de registros, falta de almacenamiento y desviaciones horarias.

Los registros se recopilan de forma centralizada y protegida contra el acceso, para que un host comprometido no pueda eliminar simplemente sus rastros junto con el estado del sistema. CIS Control 8 exige un proceso de gestión de registros, almacenamiento suficiente, hora estandarizada, registros de auditoría detallados y centralizados, así como revisiones. Una alerta solo se considera implementada cuando un evento controlado la activa, la ve el equipo responsable y conduce a un runbook de respuesta ([CIS Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)).

La detección de desviaciones compara el estado real con la línea base versionada. La comparación abarca más que hashes de archivos: configuración efectiva, cuentas, grupos, roles IAM, certificados, reglas de firewall, listeners, servicios, paquetes instalados, imágenes, tareas programadas y políticas del proveedor. Los cambios de emergencia se incorporan posteriormente o se revierten automáticamente; de lo contrario, el estado de excepción «temporal» se convierte en el nuevo valor predeterminado no documentado.

## Evolución técnica

Saltzer y Schroeder formularon en 1975 principios fundamentales de protección, como mecanismos pequeños y simples, valores predeterminados seguros, comprobación completa de autorizaciones, separación de privilegios y mínimo privilegio. Su punto de partida no fue un sistema operativo concreto, sino la arquitectura de la divulgación controlada de información en sistemas multiusuario ([Saltzer/Schroeder: Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)).

Con la expansión de los servidores de red, el endurecimiento se desplazó también hacia servicios remotos, protocolos, parcheo, auditoría y mantenimiento seguro de configuraciones. NIST SP 800-123 resumió sistemáticamente esta práctica de servidores en 2008. Los CIS Benchmarks basados en consenso, los módulos BSI-Grundschutz y las líneas base de fabricantes hicieron que las configuraciones objetivo seguras fueran más reproducibles y comparables ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

Más tarde, las arquitecturas de nube, SaaS, API e híbridas debilitaron la suposición de un perímetro interno claro. NIST SP 800-207 describió en 2020 Zero Trust como una arquitectura orientada a recursos sin confianza implícita derivada de la ubicación de red o la propiedad. Paralelamente, la cadena de suministro de software, la procedencia de las imágenes y las desviaciones automatizadas de la línea base se convirtieron en superficies de control propias. Por ello, el endurecimiento moderno combina la minimización clásica del host con identidad, política de servicio a servicio, configuración declarativa, evidencia de cadena de suministro, telemetría y recuperación probada ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

## Lista de comprobación para administradores

El endurecimiento solo concluye cuando las medidas elegidas son verificables durante la operación normal y en caso de recuperación. Por ello, la lista de comprobación vincula configuración, responsabilidad y evidencia.

- [ ] Están documentados el rol, las necesidades de protección, los flujos de datos y los límites de confianza del sistema.
- [ ] Las recomendaciones del fabricante, CIS y BSI se han asignado a una línea base versionada.
- [ ] Cada desviación posee justificación, control compensatorio, propietario y fecha de vencimiento.
- [ ] Los listeners, servicios, paquetes, módulos y destinos salientes se han reducido al mínimo necesario.
- [ ] Las identidades de usuario, administración, servicio y Break-Glass están separadas y se revisan regularmente.
- [ ] Los accesos administrativos utilizan identidad personal, MFA, privilegios mínimos y auditoría centralizada.
- [ ] El plano de gestión y el flujo de correo productivo se encuentran en zonas separadas y restrictivas.
- [ ] Relay, Submission, Access y las rutas internas de servicio tienen políticas propias de TLS, autenticación y límites de tasa.
- [ ] Los analizadores, escáneres, datos temporales, colas, claves y secretos cuentan con permisos mínimos de proceso y archivos.
- [ ] Los parches y las imágenes proceden de fuentes autenticadas; se verifica su procedencia e integridad.
- [ ] Las pruebas de línea base, flujo de correo, copia de seguridad y reversión se ejecutan antes del despliegue general.
- [ ] Los registros son centrales, temporalmente coherentes, están protegidos contra cambios y vinculados a alertas probadas.
- [ ] Se detectan automáticamente las desviaciones en cuentas, configuración, reglas, servicios, certificados y software.
- [ ] La recuperación y el acceso de emergencia se han probado en la práctica bajo las condiciones endurecidas.

## Fuentes

- [NIST – SP 800-123, Guide to General Server Security](https://csrc.nist.gov/pubs/sp/800/123/final)
- [CIS – Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)
- [CIS – Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [BSI – SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3)
- [BSI – APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)
- [NIST – SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST – SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)
- [NIST – SP 800-207, Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://csrc.nist.gov/pubs/sp/800/177/r1/final)
- [IETF RFC 8314 – TLS for Email Submission and Access](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10)
- [CIS – Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)
- [CISA – Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [CIS – Control 5: Account Management](https://www.cisecurity.org/controls/account-management)
- [CIS – Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)
- [Microsoft Learn – Get-LocalUser](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser)
- [Microsoft Learn – Get-LocalGroupMember](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember)
- [Linux man-pages – getent](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft Learn – Get-WindowsCapability](https://learn.microsoft.com/powershell/module/dism/get-windowscapability)
- [Microsoft Learn – OpenSSH Server Configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenBSD – sshd manpage](https://man.openbsd.org/sshd)
- [Microsoft Learn – Get-NetFirewallProfile](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile)
- [Microsoft Learn – Get-NetFirewallRule](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule)
- [Netfilter – nft manpage](https://netfilter.org/projects/nftables/manpage.html)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [NIST – SP 800-161 Rev. 1, Cybersecurity Supply Chain Risk Management](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [CIS – Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)
- [Saltzer/Schroeder – Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)
