---
title: "Renovación de la plataforma HIN 2026: Access Gateway, Client y los plazos hasta el 14 de septiembre"
navTitle: "Access Gateway 2026"
description: "Habilitación de firewall hasta el 14 de agosto, Access Gateway versión 4 a partir del 17 de agosto, endpoints SAML, tokens de hardware y HIN Client hasta el 14 de septiembre. El gateway de correo no se ve afectado y se sustituirá por separado."
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
url: https://rafaelpfister.ch/es/blog/renovacion-de-la-plataforma-hin-2026-access-gateway-client-y-los-plazos-hasta-el-14-de
translationSourceHash: 1a174bd131b8bb29f9b1e1e793d4cf19b3f732e3c6fd779a25193f151ec8c109
translationModel: gpt-5.6-terra
translatedAt: 2026-08-02T06:16:41.212Z
translationReview: automatic
---

# Renovación de la plataforma HIN 2026: Access Gateway, Client y los plazos hasta el 14 de septiembre

En 2026, HIN renovará la plataforma de identidad y acceso. El primer plazo vence el 14 de agosto de 2026; el gran cambio tendrá lugar el 14 de septiembre de 2026.

**Se ven afectados el HIN Access Gateway (AGW), el HIN Client y los medios de autenticación. El HIN Mailgateway no se ve afectado.** También será sustituido, pero en un proyecto independiente con su propio calendario.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Su situación</span>
    <span class="choice__title">Solo AGW en funcionamiento</span>
    <span class="choice__hint">Los plazos indicados a continuación cubren todas las medidas que debe tomar. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Su situación</span>
    <span class="choice__title">Además, necesidad de migración del gateway de correo</span>
    <span class="choice__hint">También está pendiente la sustitución por «Stargate», con despliegue amplio a partir del T3 de 2026. Revisión gratuita de su entorno. →</span>
  </a>
</div>

## Los plazos

| Fecha | Medida | Afecta a |
|---|---|---|
| 14.08.2026 | Habilitación de firewall para `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | Operadores de AGW |
| 17.08.2026 | Instalación automática de AGW versión 4 | Operadores de AGW |
| a partir de mediados de agosto | Se recomienda la instalación manual de HIN Client 4.0 | Todos los usuarios de Client |
| 14.09.2026 | Endpoints SAML migrados | Federaciones, conexiones al EPD |
| 14.09.2026 | Caducan los tokens de hardware y las identidades de prueba | Usuarios de tokens, integración |
| 14.09.2026 | Reconfigurar la aplicación Authenticator | Usuarios de la aplicación |
| 14.09.2026 | Actualización obligatoria a HIN Client 4.0 | Todos los usuarios de Client |

## Access Gateway no es Mailgateway

Ambos llevan Gateway en el nombre y se confunden con frecuencia. El Access Gateway controla el acceso a aplicaciones protegidas por HIN y no afecta al tráfico de correo. El Mailgateway se encuentra en la ruta del correo y cifra los mensajes.

## Access Gateway: firewall y versión 4

Hasta el 14 de agosto, el AGW debe poder acceder a `idp.id.hin.ch`. Se trata de un cambio en el firewall, no de una configuración en el gateway, por lo que suele corresponder al responsable de red y no al administrador del gateway.

A partir del 17 de agosto se instalará automáticamente la versión 4. Requisito: AGW en versión 3.1.50 o superior y Kerberos activo como método de autenticación. Para la conexión con Active Directory se necesita una cuenta LDAP con permisos de lectura.

Quien no cumpla los requisitos no será actualizado, y la experiencia demuestra que esto solo se detecta cuando ya nadie puede iniciar sesión. Por tanto, es mejor comprobar ahora la versión que hacerlo en septiembre.

## SAML: nuevos endpoints, menos atributos

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Con el cambio se modifican los formatos de atributos y los bindings. El conjunto de atributos se reduce a GLN, nombre, fecha de nacimiento y sexo.

Este es el punto que rompe las integraciones. Cualquier aplicación que utilice otros atributos para roles o separación de mandantes dejará de recibirlos después del 14 de septiembre. El error no se manifestará como un fallo de inicio de sesión, sino como la ausencia de permisos en el sistema de destino.

Las identidades de prueba caducan en la misma fecha; quien desee probar el cambio en un entorno de integración debería hacerlo antes.

Quien opera una federación casi siempre opera también su propia infraestructura de correo. Para estas organizaciones, la renovación de la plataforma coincide en el mismo año con la [sustitución del Mailgateway por «Stargate»](/stargate): técnicamente son independientes, pero compiten por las mismas personas y ventanas de mantenimiento.

## Token, aplicación y HIN Client 4.0

Ya no se emiten tokens de hardware y caducan el 14 de septiembre. Alternativas: HIN Client, código SMS o aplicación Authenticator. La propia aplicación seguirá siendo válida hasta el 14 de septiembre y después deberá configurarse de nuevo a través del portal de autoservicio.

HIN Client se actualizará automáticamente a la versión 4.0 a más tardar el 14 de septiembre; la instalación manual estará disponible desde mediados de agosto a través de `download.hin.ch`. El inicio de sesión se realizará ahora mediante el navegador.

El punto crítico son los requisitos del sistema: **la versión 4.0 requiere Windows 11 o macOS 14.** Los dispositivos más antiguos deben actualizarse o sustituirse antes. Para una parte de las consultas, el plazo no es por tanto una tarea de software, sino de adquisición. Quien lo advierta solo en septiembre tendrá que lidiar con plazos de entrega y la reinstalación del software de la consulta.

## Cinco preguntas para determinar la situación

1. ¿Qué versión de AGW está en funcionamiento y está activo Kerberos?
2. ¿El firewall permite conexiones salientes a `idp.id.hin.ch`?
3. ¿Cuántos puestos de trabajo siguen funcionando con Windows 10 o macOS 13 y versiones anteriores?
4. ¿Cuántos tokens de hardware están en uso y a qué alternativa cambiarán las personas afectadas?
5. ¿Alguna aplicación utiliza atributos HIN que dejarán de estar disponibles?

Las respuestas a las preguntas 3 y 5 determinan el esfuerzo. El resto se resuelve en pocas horas y está documentado por HIN.

## El segundo proyecto: «Stargate»

De forma independiente, HIN sustituye el Mailgateway por el nuevo HIN Gateway, denominado internamente proyecto «Stargate», un enfoque técnico de Data Mesh con cifrado de extremo a extremo y gestión descentralizada de claves. No se trata de sustituir el appliance, sino de un cambio de arquitectura.

Por tanto, el esfuerzo se sitúa en un nivel completamente distinto. La renovación de la plataforma exige sobre todo cumplir los plazos de una regla de firewall, una versión y la sustitución de un dispositivo, mientras que con Stargate se pone en cuestión la propia ruta productiva de correo: el conjunto de reglas desarrollado con el tiempo, el material de claves, el tratamiento de destinatarios sin identidad HIN y la cuestión de a qué recurrir si algo no funciona como se esperaba. Dado que la migración se realiza en ventanas reservadas de cuatro horas y HIN recomienda un mes de preparación, una cita de este tipo no admite asuntos pendientes.

<aside class="offer-box">
  <span class="offer-box__tag">Revisión gratuita</span>
  <p><strong>No necesita saber cuál es su situación. Precisamente para eso está la revisión.</strong> Revisaré su entorno de gateway existente y le indicaré qué debe hacerse antes de la ventana de migración, independientemente de que luego migre por su cuenta o cuente con acompañamiento.</p>
  <a class="offer-box__cta" href="/stargate">Registrarse ahora</a>
</aside>

## Fuentes

1.  [Renovación de la plataforma HIN: estos ajustes técnicos son necesarios para los miembros de HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): plazos en agosto y septiembre, endpoints SAML, conjunto reducido de atributos, habilitaciones de firewall.

2.  [Ya está disponible el nuevo HIN Client: esto cambia para los miembros de HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): versión 4.0, requisitos del sistema operativo, inicio de sesión basado en navegador.

3.  [HIN Gateway: comunicación segura dentro de la comunidad HIN](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): sustitución del Mailgateway, arquitectura, modelos operativos, migración en ventanas reservadas.

4.  [Configuración del HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): función del AGW en la gestión de accesos.

5.  [Conexión con Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos y la cuenta LDAP con permisos de lectura.

6.  [HIN AG: «Del Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): contexto de «Stargate», nodos descentralizados, calendario.
