---
title: "Renovación de la plataforma HIN 2026: Access Gateway, Client y los plazos hasta el 26 de octubre"
navTitle: "Access Gateway 2026"
description: "Habilitación del firewall hasta el 31 de agosto, Access Gateway versión 4 a partir de septiembre, endpoints SAML hasta el 12 de octubre, tokens de hardware y HIN Client 4.0 hasta el 26 de octubre. El gateway de correo no se ve afectado y se sustituirá por separado."
date: "2026-08-01"
kategorie: "Gateway HIN"
timeToRead: "5 min de lectura"
themen:
  - hin-gateway
  - active-directory-entra
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "renovacion-de-la-plataforma-hin-2026-access-gateway-client-y-los-plazos-hasta-el-14-de"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
translationSourceHash: cd34e6297afca9491bd965e40dbaf55b920296e62e57726754824474fcb27f0d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:46:35.269Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/renovacion-de-la-plataforma-hin-2026-access-gateway-client-y-los-plazos-hasta-el-14-de
---

# Renovación de la plataforma HIN 2026: Access Gateway, Client y los plazos hasta el 26 de octubre

En 2026, HIN renovará la plataforma de identidad y acceso. HIN redefinió los plazos el 27 de agosto de 2026: el primer plazo venció el 31 de agosto de 2026, los antiguos endpoints SAML ya no pueden utilizarse desde el 12 de octubre de 2026 y el gran cambio tendrá lugar el 26 de octubre de 2026.

**Se ven afectados HIN Access Gateway (AGW), HIN Client y los medios de autenticación. HIN Mailgateway no se ve afectado.** También será sustituido, pero en un proyecto independiente con su propio calendario.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Su situación</span>
    <span class="choice__title">Solo AGW en funcionamiento</span>
    <span class="choice__hint">Los plazos indicados a continuación representan todas las acciones que debe realizar. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Su situación</span>
    <span class="choice__title">También necesita migrar el gateway de correo</span>
    <span class="choice__hint">También deberá realizarse la sustitución por «Stargate», con un despliegue amplio a partir del T3 de 2026. Revisión gratuita de su entorno. →</span>
  </a>
</div>

## Los plazos

| Fecha | Medida | Afecta a |
|---|---|---|
| 31.08.2026 | Habilitación del firewall para `idp.id.hin.ch` y `cdn.hin.ch` (443/tcp) | Operadores de AGW |
| desde septiembre de 2026 | Instalación automática de AGW versión 4 (sin instalaciones en clúster) | Operadores de AGW |
| De septiembre al 26.10.2026 | Actualización automática a HIN Client 4.0 por fases | Todos los usuarios de Client |
| 12.10.2026 | Los antiguos endpoints SAML dejan de ser utilizables | Federaciones, conexiones EPD |
| 26.10.2026 | Caducan los tokens de hardware e identidades de prueba para el acceso EPD | Usuarios de tokens, integración |
| 26.10.2026 | La aplicación Authenticator anterior puede utilizarse por última vez; nueva configuración a partir del 27.10.2026 | Usuarios de la aplicación |
| 26.10.2026 | Se requiere HIN Client 4.0 | Todos los usuarios de Client |
| desde principios de noviembre de 2026 | Inicio de sesión mediante el navegador en HIN Client | Todos los usuarios de Client |

## Access Gateway no es Mailgateway

Ambos llevan Gateway en el nombre y se confunden habitualmente. Access Gateway controla el acceso a aplicaciones protegidas por HIN y no afecta al tráfico de correo. Mailgateway se encuentra en la ruta del correo y cifra los mensajes.

## Access Gateway: firewall y versión 4

Hasta el 31 de agosto de 2026, el AGW debía poder acceder a `idp.id.hin.ch` y `cdn.hin.ch` mediante 443/tcp. HIN recomienda configurar la regla con los nombres de host en lugar de direcciones IP, ya que estas pueden cambiar. Se trata de un cambio en el firewall, no de una configuración en el gateway, y por ello suele corresponder al responsable de red y no al administrador del gateway.

Desde septiembre de 2026, la versión 4 se instala automáticamente; las instalaciones en clúster están excluidas y HIN contactará directamente con esos clientes. Requisito: AGW en versión 3.1.50 o superior, y Kerberos activo como método de autenticación. Para la conexión con Active Directory se requiere una cuenta LDAP con permisos de lectura.

Quien no cumpla los requisitos no será actualizado, y la experiencia demuestra que esto solo se detecta cuando ya nadie puede iniciar sesión. Por ello, es mejor comprobar ahora la versión instalada que esperar a una incidencia.

## SAML: nuevos endpoints, menos atributos

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Con el cambio se modifican los formatos de atributos y los bindings. El conjunto de atributos se reduce a GLN, nombre, fecha de nacimiento y sexo.

Aquí es donde fallan las integraciones. Las aplicaciones que utilizan atributos adicionales para roles o separación de mandantes dejarán de recibirlos a través de los nuevos endpoints; los antiguos ya no pueden utilizarse desde el 12 de octubre de 2026. El error no se manifiesta como un error de inicio de sesión, sino como una autorización ausente en el sistema de destino.

Las identidades de prueba para el acceso EPD perderán su validez el 26 de octubre de 2026 y deberán solicitarse de nuevo. Para la integración EPD, HIN exige identidades verificadas mediante identificación por vídeo; todavía no hay un entorno de prueba disponible. Además, a partir del 26 de octubre de 2026 pueden recibirse assertions sin GLN.

Quien opera una federación casi siempre opera también su propia infraestructura de correo. Para estas organizaciones, la renovación de la plataforma coincide en el mismo año con la [sustitución de Mailgateway por «Stargate»](/stargate): son técnicamente independientes, pero compiten por las mismas personas y ventanas de mantenimiento.

## Token, aplicación y HIN Client 4.0

Ya no se emiten tokens de hardware y estos caducan el 26 de octubre de 2026. Alternativas: HIN Client, código SMS o aplicación Authenticator. La aplicación Authenticator anterior puede utilizarse hasta el 26 de octubre de 2026; la nueva aplicación podrá configurarse como segundo factor en el portal de autoservicio a partir del 27 de octubre de 2026. HIN recomienda configurar previamente el inicio de sesión con contraseña y código SMS como acceso de emergencia.

HIN Client se actualizará automáticamente a la versión 4.0 por fases entre septiembre y el 26 de octubre de 2026. Si el 26 de octubre Client no muestra la versión 4.0, deberá instalarse manualmente mediante `download.hin.ch`. El inicio de sesión mediante el navegador estará disponible a partir de principios de noviembre de 2026.

El punto crítico son los requisitos del sistema: **la versión 4.0 requiere Windows 11 con TPM o macOS 14** y, en los Mac con Intel, también un Security Chip; sin TPM, solo queda el inicio de sesión con código SMS. Los equipos antiguos deben actualizarse o sustituirse previamente. Para parte de las consultas, el plazo no es por tanto una tarea de software, sino de adquisición. Quien lo detecte poco antes del 26 de octubre tendrá que lidiar con plazos de entrega y la reinstalación del software de la consulta.

## Cinco preguntas para evaluar la situación

1. ¿Qué versión de AGW está en funcionamiento y está Kerberos activo?
2. ¿El firewall permite conexiones salientes a `idp.id.hin.ch`?
3. ¿Cuántos puestos de trabajo siguen utilizando Windows 10 o macOS 13 y versiones anteriores?
4. ¿Cuántos tokens de hardware están en uso y a qué alternativa cambiarán las personas afectadas?
5. ¿Alguna aplicación utiliza atributos HIN que dejarán de estar disponibles?

Las respuestas a las preguntas 3 y 5 determinan el esfuerzo. El resto puede resolverse en pocas horas y está documentado por HIN.

## El segundo proyecto: «Stargate»

De forma independiente, HIN sustituye Mailgateway por el nuevo HIN Gateway, denominado internamente proyecto «Stargate», técnicamente un enfoque de Data Mesh con cifrado de extremo a extremo y gestión descentralizada de claves. No se trata de sustituir el appliance, sino de un cambio de arquitectura.

Por tanto, el esfuerzo se sitúa en un nivel completamente distinto. La renovación de la plataforma exige sobre todo cumplir los plazos para una regla de firewall, una versión y la sustitución de un equipo, mientras que con Stargate se replantea la propia ruta de correo en producción: el conjunto de reglas desarrollado con el tiempo, el material de claves, el tratamiento de destinatarios sin identidad HIN y la cuestión de qué alternativa utilizar si algo no funciona como se espera. Dado que la migración se realiza en ventanas reservadas de cuatro horas y HIN recomienda un mes de preparación, una cita de este tipo no admite cuestiones pendientes.

<aside class="offer-box">
  <span class="offer-box__tag">Revisión gratuita</span>
  <p><strong>No tiene que saber cuál es su situación. Precisamente para eso sirve la revisión.</strong> Revisaré su entorno de gateway actual y le indicaré qué debe hacerse antes de la ventana de migración, independientemente de si después realiza la migración por su cuenta o con acompañamiento.</p>
  <a class="offer-box__cta" href="/stargate">Registrarse ahora</a>
</aside>

## Historial de cambios

- **25 de octubre de 2026:** Plazos actualizados conforme a la redefinición de HIN del 27 de agosto de 2026: habilitación del firewall (ahora también para cdn.hin.ch) hasta el 31 de agosto, antiguos endpoints SAML hasta el 12 de octubre, tokens de hardware, aplicación Authenticator anterior y HIN Client 4.0 hasta el 26 de octubre de 2026. Añadido: requisito de TPM para HIN Client 4.0 e inicio de sesión mediante navegador a partir de principios de noviembre de 2026.

## Fuentes

1.  [Renovación de la plataforma HIN: estos ajustes técnicos son necesarios para los miembros de HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): plazos de agosto a octubre de 2026 (estado: 27 de agosto de 2026), endpoints SAML, conjunto reducido de atributos, habilitaciones del firewall, aplicación Authenticator y tokens de hardware.

2.  [Ya está disponible el nuevo HIN Client: estos son los cambios para los miembros de HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): versión 4.0, despliegue hasta el 26 de octubre de 2026, requisitos de sistema operativo y TPM, inicio de sesión basado en navegador a partir de noviembre de 2026.

3.  [HIN Gateway: comunicación segura dentro de la comunidad HIN](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): sustitución de Mailgateway, arquitectura, modelos operativos, migración en franjas horarias reservadas.

4.  [Configuración de HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): función del AGW en la gestión de accesos.

5.  [Conexión de Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos y la cuenta LDAP con permisos de lectura.

6.  [HIN AG: «Del Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): contexto sobre «Stargate», nodos descentralizados, calendario.
