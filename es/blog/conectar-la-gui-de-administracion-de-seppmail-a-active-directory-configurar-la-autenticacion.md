---
title: "Conectar la GUI de administración de SEPPmail a Active Directory: configurar la autenticación LDAP a partir de la versión 15.0.6"
navTitle: "Inicio de sesión LDAP de administración"
description: "Desde el firmware 15.0.6, los administradores de la appliance SEPPmail pueden autenticarse mediante un servidor LDAP externo como Active Directory, incluido el mapeo de grupos al grupo local admin. Configuración paso a paso en User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min de lectura"
themen:
  - seppmail
slug: "conectar-la-gui-de-administracion-de-seppmail-a-active-directory-configurar-la-autenticacion"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: b9a1bdfe75f57458eb57c2e1dcb32fdfd25e1279eb720d4cd4754d49b5aac624
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:23:58.684Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/conectar-la-gui-de-administracion-de-seppmail-a-active-directory-configurar-la-autenticacion
---

# Conectar la GUI de administración de SEPPmail a Active Directory: configurar la autenticación LDAP a partir de la versión 15.0.6

Hasta el firmware 15.0.5, la interfaz de administración de SEPPmail Secure E-Mail Gateway solo admitía cuentas locales. Quienes querían trabajar de forma ordenada creaban un usuario local independiente para cada administrador y lo añadían al grupo admin. Esto funciona, pero presenta las desventajas habituales de las cuentas locales: contraseñas propias para cada appliance, ausencia de un offboarding centralizado y falta de aplicación de las directivas de contraseña del servicio de directorio. Con la versión de parche 15.0.6, esto cambia. Si se desea, la GUI de administración autentica a los administradores mediante un servidor LDAP externo como Active Directory y asigna grupos de AD a grupos locales de la appliance.

Los demás cambios de la versión se resumen en el artículo sobre [SEPPmail 15.0.6 y 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Aquí se trata únicamente de la nueva autenticación externa.

## Qué ofrece esta función

Según las notas de la versión ampliadas, la versión 15.0.6 añade una nueva sección, **External Authentication**, en **User > Advanced Settings**. Con ella, la GUI de administración se autentica mediante un servidor LDAP externo y los grupos externos, como los grupos de seguridad de AD, se asignan a grupos locales de la appliance.

Los usuarios autenticados externamente aparecen localmente en la appliance y se comportan como usuarios locales, con una diferencia: su contraseña no se puede cambiar en la appliance, ya que se encuentra en el servidor LDAP externo. Por tanto, el control de las contraseñas pasa por completo al directorio.

Es importante distinguirlo de la función anterior: la appliance ya admitía autenticación externa, pero solo para la interfaz web GINA, configurada para cada Managed Domain, en la sección External authentication de la configuración del dominio. La novedad de la versión 15.0.6 es que el acceso a la propia interfaz de administración también funciona mediante LDAP.

Aún estoy comprobando si HIN Mailgateway también ha recibido el inicio de sesión LDAP y añadiré el resultado al artículo posteriormente. Dado que las appliances HIN se basan en el mismo firmware de SEPPmail, parto de que así será.

## Requisitos

Antes de la configuración, deben estar preparados cuatro elementos:

- **Firmware 15.0.7 o posterior:** La función llega con la versión 15.0.6. Desde el 2 de septiembre de 2026 está disponible la versión de parche 15.0.7, que corrige, entre otras cosas, una omisión de la obligación de MFA (SEC-100) y una inyección de comandos en el procesamiento de la configuración (CVE-2026-84830). La appliance debería contar al menos con esta versión; la rama de funcionalidades 15.1, actualmente con el hotfix 15.1.0.1 del 29 de septiembre de 2026, también incluye la función.
- **Un directorio compatible con LDAP:** Un Active Directory, OpenLDAP o equivalente. Si los usuarios solo están en Entra ID, que no habla LDAP por sí mismo, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) proporciona el puente.
- **Una cuenta de enlace en el directorio:** Una cuenta de servicio dedicada y sin privilegios, con acceso de lectura, con la que la appliance realiza la búsqueda LDAP. No debe ser un administrador de dominio.
- **Un grupo de AD para los administradores del gateway:** Por ejemplo, un grupo de seguridad SEPPmail-Admins, que posteriormente se asigna al grupo local admin. La pertenencia a este grupo determina entonces el acceso administrativo completo.

TLS está activado de forma predeterminada en la configuración de conexión y debería permanecer así; las credenciales de los administradores no deben circular sin cifrar por la red. La appliance debe poder alcanzar el servidor LDAP en el puerto configurado, normalmente 636 para LDAPS.

## Configuración en User > Advanced Settings

La configuración se encuentra en la GUI de administración, en **User > Advanced Settings**, dentro de la sección **External Authentication**, y consta de cuatro bloques.

**1. Connection Settings:** La casilla *Authenticate users to external LDAP server (e.g. Active Directory)* activa la función. A continuación se indican la dirección del servidor, el puerto, la opción *TLS required*, así como el Bind DN y el Bind Password de la cuenta de servicio.

**2. User Attributes:** Aquí se define cómo encuentra la appliance los objetos de usuario: la LDAP Object Class, normalmente person en Active Directory; la Search Base, es decir, la OU o el contenedor donde se encuentran las cuentas de administrador; y el atributo de correo electrónico, cuyo valor predeterminado es mail.

**3. Group Attributes:** Del mismo modo, aquí se especifican los objetos de grupo para que la appliance pueda resolver las pertenencias a grupos.

**4. Mapping Settings:** La parte decisiva. En *Remote Group* se selecciona el grupo del servidor LDAP y, en *Local Group*, uno o varios grupos locales a los que se asigna. Para el acceso administrativo completo es el grupo admin; sus miembros tienen los mismos privilegios que el usuario predeterminado admin. Quien desee diferenciar permisos puede asignarlos en su lugar a grupos restringidos como readonly admin o a grupos de la appliance vinculados a funciones específicas.

Antes de guardar, conviene utilizar la prueba integrada **Login Test**: con el nombre de usuario y la contraseña de una cuenta de prueba, permite verificar la conexión, la búsqueda y la autenticación antes de activar la configuración.

## Configuraciones de ejemplo

Los siguientes valores deben adaptarse al entorno propio; el dominio de ejemplo es example.com. Los nombres de los campos corresponden a la sección External Authentication de la appliance.

### Active Directory

| Campo | Valor |
|---|---|
| Servidor | dc01.example.com |
| Puerto | 636 |
| TLS required | activado |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Contraseña de la cuenta de servicio |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Notas sobre Active Directory: cualquier Domain Controller accesible es adecuado como servidor; en entornos con varias ubicaciones, se recomienda un DC en la misma ubicación o un alias que apunte a varios DC. El puerto 636 corresponde a LDAPS; para ello, el certificado del DC debe poder ser validado por la appliance. La Search Base debe ser lo bastante restrictiva como para contener las cuentas de administrador, pero no todo el directorio. El atributo mail debe estar configurado en las cuentas de AD.

### OpenLDAP

| Campo | Valor |
|---|---|
| Servidor | ldap01.example.com |
| Puerto | 636 |
| TLS required | activado |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Contraseña de la cuenta de servicio |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Notas sobre OpenLDAP: en configuraciones habituales, los usuarios se almacenan como inetOrgPerson en ou=people. Para los grupos, groupOfNames es la opción fiable, ya que la pertenencia se representa mediante el atributo member con el DN completo. En cambio, los grupos posixGroup solo registran sus miembros como memberUid, es decir, con el nombre de usuario en lugar del DN; no está documentado si la appliance puede resolverlo y debe verificarse con Login Test antes de realizar el cambio. Si el servidor solo utiliza STARTTLS en el puerto 389, debe indicarse el puerto correspondiente en el campo Server; la conexión no debe utilizarse sin cifrado en ningún caso.

## Indicaciones de funcionamiento

Hay tres puntos que requieren atención antes de convertir el inicio de sesión LDAP en la única vía de acceso a la appliance:

- **Mantener un acceso local de emergencia.** Las contraseñas de los usuarios externos se encuentran en el servidor LDAP. Si el directorio no está disponible —por un problema de red, mantenimiento de AD o porque el gateway debe resolver un problema precisamente en esa red—, sigue siendo necesaria una cuenta local de administrador con una contraseña almacenada de forma segura. Por tanto, el usuario predeterminado admin no debe eliminarse, sino mantenerse como acceso de emergencia documentado.
- **MFA sigue siendo relevante.** La versión 15.0.6 también ha revisado el inicio de sesión con MFA: el segundo factor ya no se añade a la contraseña, sino que se solicita en un campo independiente. La autenticación externa no sustituye al segundo factor. Una vulnerabilidad que permitía, en determinadas condiciones, el acceso antes de completar todos los pasos de MFA no se corrigió hasta la versión 15.0.7.
- **Offboarding mediante el directorio.** Esta es la verdadera ventaja de la integración: cuando un administrador deja la empresa, basta con desactivar la cuenta de AD o eliminarla del grupo asignado. Ya no es necesario mantener cuentas locales en cada appliance. No obstante, los objetos de usuario autenticados externamente que son visibles localmente deben compararse periódicamente con el directorio.

## Conclusión

La autenticación LDAP para la GUI de administración cubre una carencia que la appliance tenía desde hacía tiempo: ahora los accesos de administrador pueden gestionarse centralmente en el directorio en lugar de hacerlo dispositivo por dispositivo. Junto con el campo independiente para MFA, la versión 15.0.6 mejora significativamente el inicio de sesión en la interfaz de administración en una sola versión. Quien introduzca esta función debe mantener el mapeo de grupos deliberadamente restrictivo y conservar el acceso local de emergencia.

## Historial de cambios

- **25 de octubre de 2026:** Se elevó la versión mínima recomendada de 15.0.6.1 a 15.0.7, que corrige una omisión de la obligación de MFA (SEC-100) y CVE-2026-84830. Añadido: la rama de funcionalidades 15.1, actualmente 15.1.0.1, también incluye la función.

## Fuentes

1.  [Notas de la versión ampliadas de SEPPmail 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): entrada sobre la autenticación de la GUI de administración con descripción de la función, ubicación de configuración y comportamiento de los usuarios autenticados externamente; correcciones de seguridad de 15.0.7 (SEC-100, CVE-2026-84830).

2.  [Documentación de SEPPmail – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): referencia de los campos de la sección External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentación de SEPPmail – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): grupos predefinidos de la appliance; los miembros del grupo admin tienen acceso administrativo sin restricciones.

4.  [Documentación de SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): notas oficiales de la versión 15.0.6 con la entrada sobre la autenticación de la GUI de administración mediante servidores LDAP externos, así como sobre las versiones posteriores 15.0.7, 15.1.0 y 15.1.0.1.

5.  [SEPPmail 15.0.6 y 15.0.6.1: correcciones de seguridad y nuevas funciones de administración](/blog/seppmail-releases-15-0-6-und-15-0-6-1): resumen de todos los cambios de ambas versiones.
