---
title: "Renovación de la plataforma HIN 2026: Access Gateway, Client y los plazos hasta el 14 de septiembre"
navTitle: "Access Gateway 2026"
description: "Habilitación del firewall hasta el 14 de agosto, Access Gateway versión 4 a partir del 17 de agosto, endpoints SAML, tokens de hardware y HIN Client hasta el 14 de septiembre. El gateway de correo no se ve afectado y se sustituirá por separado."
date: "2026-08-01"
kategorie: "HIN-Gateway"
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
translationSourceHash: 6ab0928c0961c1a185aa0b658e3c5fd0dfcdee1e27366063d007917a18f33ef2
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:12:09.054Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/renovacion-de-la-plataforma-hin-2026-access-gateway-client-y-los-plazos-hasta-el-14-de
---

# Renovación de la plataforma HIN 2026: Access Gateway, Client y los plazos hasta el 14 de septiembre

HIN renovará en 2026 la plataforma de identidad y acceso. El primer plazo vence el 14 de agosto de 2026; el gran cambio tendrá lugar el 14 de septiembre de 2026.

**Se ven afectados el HIN Access Gateway (AGW), el HIN Client y los métodos de autenticación. El HIN Mailgateway no se ve afectado.** También será sustituido, pero en un proyecto independiente con su propio calendario.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Su situación</span>
    <span class="choice__title">Solo AGW en funcionamiento</span>
    <span class="choice__hint">Los plazos indicados abajo cubren todas las acciones necesarias. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Su situación</span>
    <span class="choice__title">Necesidad adicional de migración del gateway de correo</span>
    <span class="choice__hint">También está pendiente la sustitución por «Stargate», con despliegue general a partir del tercer trimestre de 2026. Comprobación gratuita de su entorno. →</span>
  </a>
</div>

## Los plazos

| Fecha | Medida | Afecta a |
|---|---|---|
| 14.08.2026 | Habilitación del firewall para `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | Operadores de AGW |
| 17.08.2026 | Instalación automática de AGW versión 4 | Operadores de AGW |
| desde mediados de agosto | Se recomienda la instalación manual de HIN Client 4.0 | Todos los usuarios de Client |
| 14.09.2026 | Migración de endpoints SAML | Federaciones, conexiones al EPD |
| 14.09.2026 | Caducidad de tokens de hardware e identidades de prueba | Usuarios de tokens, integración |
| 14.09.2026 | Reconfiguración de la app Authenticator | Usuarios de la app |
| 14.09.2026 | Actualización obligatoria a HIN Client 4.0 | Todos los usuarios de Client |

## Access Gateway no es Mailgateway

Ambos llevan Gateway en el nombre y se confunden con frecuencia. El Access Gateway controla el acceso a aplicaciones protegidas por HIN y no afecta al tráfico de correo. El Mailgateway se encuentra en la ruta del correo y cifra los mensajes.

## Access Gateway: firewall y versión 4

Hasta el 14 de agosto, el AGW debe poder acceder a `idp.id.hin.ch`. Se trata de un cambio en el firewall, no de una configuración en el gateway, por lo que suele recaer en el responsable de red y no en el administrador del gateway.

A partir del 17 de agosto se instalará automáticamente la versión 4. Requisitos: AGW en la versión 3.1.50 o posterior y Kerberos activado como método de autenticación. Para la conexión con Active Directory se necesita una cuenta LDAP con permisos de lectura.

Quien no cumpla los requisitos no se actualizará, y la experiencia demuestra que esto solo se detecta cuando ya nadie puede iniciar sesión. Por eso, conviene comprobar ahora la versión instalada en lugar de hacerlo en septiembre.

## SAML: nuevos endpoints, menos atributos

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Con el cambio se modifican los formatos de atributos y los bindings. El conjunto de atributos se reduce a GLN, nombre, fecha de nacimiento y sexo.

Este es el punto en el que fallan las integraciones. Toda aplicación que utilice atributos adicionales para roles o separación de mandantes dejará de recibirlos después del 14 de septiembre. El error no se mostrará como un fallo de inicio de sesión, sino como falta de permisos en el sistema de destino.

Las identidades de prueba caducan en la misma fecha; por tanto, quien desee probar el cambio en un entorno de integración debe hacerlo antes.

Quien opera una federación casi siempre opera también una infraestructura de correo propia. Para estas organizaciones, la renovación de la plataforma coincide en el mismo año con la [sustitución del Mailgateway por «Stargate»](/stargate): técnicamente independientes, pero en competencia por las mismas personas y ventanas de mantenimiento.

## Tokens, app y HIN Client 4.0

Ya no se emitirán tokens de hardware y estos caducarán el 14 de septiembre. Alternativas: HIN Client, código SMS o app Authenticator. La propia app seguirá siendo válida hasta el 14 de septiembre y después deberá reconfigurarse mediante el portal de autoservicio.

HIN Client se actualizará automáticamente a la versión 4.0 como muy tarde el 14 de septiembre; la instalación manual estará disponible desde mediados de agosto a través de `download.hin.ch`. El inicio de sesión se realizará ahora mediante el navegador.

El punto crítico son los requisitos del sistema: **la versión 4.0 requiere Windows 11 o macOS 14.** Los equipos más antiguos deben actualizarse o sustituirse antes. Para parte de las consultas, el plazo no representa una tarea de software, sino de adquisición. Quien se dé cuenta de ello solo en septiembre tendrá que lidiar con plazos de entrega y la reinstalación del software de la consulta.

## Cinco preguntas para determinar la situación

1. ¿Qué versión de AGW está en funcionamiento y está Kerberos activado?
2. ¿El firewall permite conexiones salientes a `idp.id.hin.ch`?
3. ¿Cuántos puestos de trabajo siguen utilizando Windows 10 o macOS 13 y versiones anteriores?
4. ¿Cuántos tokens de hardware están en uso y a qué alternativa pasarán las personas afectadas?
5. ¿Alguna aplicación utiliza atributos HIN que dejarán de estar disponibles?

Las respuestas a las preguntas 3 y 5 determinan el esfuerzo. El resto se resuelve en pocas horas y está documentado por HIN.

## El segundo proyecto: «Stargate»

De forma independiente, HIN sustituye el Mailgateway por el nuevo HIN Gateway, denominado internamente proyecto «Stargate», técnicamente un enfoque de malla de datos con cifrado de extremo a extremo y gestión descentralizada de claves. No se trata de sustituir el appliance, sino de un cambio de arquitectura.

Por tanto, el esfuerzo se sitúa en un nivel completamente distinto. La renovación de la plataforma exige sobre todo cumplir los plazos de una regla de firewall, una versión y la sustitución de un equipo, mientras que con Stargate la propia ruta de correo productiva queda en cuestión: el conjunto de reglas consolidado, el material criptográfico, el tratamiento de destinatarios sin identidad HIN y la cuestión de a qué recurrir si algo no funciona como se esperaba. Dado que la migración se realiza en ventanas reservadas de cuatro horas y HIN recomienda un mes de preparación, una cita de este tipo no admite asuntos pendientes.

<aside class="offer-box">
  <span class="offer-box__tag">Comprobación gratuita</span>
  <p><strong>No tiene que saber cuál es su situación. Precisamente para eso sirve la comprobación.</strong> Revisaré su entorno de gateway actual y le indicaré qué debe hacerse antes de la ventana de migración, independientemente de que después realice la migración por su cuenta o con acompañamiento.</p>
  <a class="offer-box__cta" href="/stargate">Registrarse ahora</a>
</aside>

## Fuentes

1.  [Renovación de la plataforma HIN: estos ajustes técnicos son necesarios para los miembros de HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): plazos en agosto y septiembre, endpoints SAML, conjunto reducido de atributos, habilitaciones de firewall.

2.  [Ya está disponible el nuevo HIN Client: esto cambia para los miembros de HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): versión 4.0, requisitos del sistema operativo, inicio de sesión basado en navegador.

3.  [HIN Gateway: comunicación segura dentro de la comunidad HIN](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): sustitución del Mailgateway, arquitectura, modelos operativos, migración en ventanas reservadas.

4.  [Configuración del HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): función del AGW en la gestión de acceso.

5.  [Conexión con Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos y la cuenta LDAP con permisos de lectura.

6.  [HIN AG: «Del Mailgateway a la malla de datos»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): contexto de «Stargate», nodos descentralizados, calendario.
