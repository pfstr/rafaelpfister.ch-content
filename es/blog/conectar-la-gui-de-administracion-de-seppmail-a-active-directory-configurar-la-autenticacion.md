---
title: "Conectar la GUI de administración de SEPPmail a Active Directory: configurar la autenticación LDAP a partir de la versión 15.0.6"
navTitle: "Inicio de sesión LDAP de administración"
description: "Desde el firmware 15.0.6, los administradores de la appliance SEPPmail pueden autenticarse mediante un servidor LDAP externo como Active Directory, incluido el mapeo de grupos a la grupo local admin. Configuración paso a paso en User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min de lectura"
themen:
  - seppmail
slug: "conectar-la-gui-de-administracion-de-seppmail-a-active-directory-configurar-la-autenticacion"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: aad5af6607824c7af146d3214d622067cdb1dfe98b82358fbc7566a32256464a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:23:50.538Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/conectar-la-gui-de-administracion-de-seppmail-a-active-directory-configurar-la-autenticacion
---

# Conectar la GUI de administración de SEPPmail a Active Directory: configurar la autenticación LDAP a partir de la versión 15.0.6

Hasta el firmware 15.0.5, la interfaz de administración de SEPPmail Secure E-Mail Gateway solo conocía cuentas locales. Quien quería trabajar de forma ordenada creaba un usuario local propio para cada administrador y lo añadía al grupo admin. Esto funciona, pero tiene las desventajas habituales de las cuentas locales: contraseñas propias para cada appliance, ausencia de un proceso centralizado de offboarding y falta de aplicación de las directivas de contraseña del servicio de directorio. Con la versión de parche 15.0.6, esto cambia. Si se desea, la GUI de administración autentica a los administradores mediante un servidor LDAP externo como Active Directory y asigna grupos de AD a grupos locales de la appliance.

Los demás cambios de la versión se resumen en el artículo sobre [SEPPmail 15.0.6 y 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Aquí solo se trata de la nueva autenticación externa.

## Qué ofrece la función

Según las Extended Release Notes, la versión 15.0.6 añade una nueva sección, **External Authentication**, en **User > Advanced Settings**. Con ella, la GUI de administración se autentica mediante un servidor LDAP externo y los grupos externos (por ejemplo, grupos de seguridad de AD) se asignan a grupos locales de la appliance.

Los usuarios autenticados externamente aparecen de forma local en la appliance y se comportan como usuarios locales, con una diferencia: su contraseña no puede modificarse en la appliance, ya que se encuentra en el servidor LDAP externo. Por tanto, la gestión de contraseñas pasa por completo al directorio.

Es importante distinguirlo de la funcionalidad existente: la appliance ya admitía autenticación externa, pero solo para la interfaz web GINA, configurada por Managed Domain (sección External authentication en la configuración del dominio). La novedad de la versión 15.0.6 es que también el acceso a la propia interfaz de administración se realiza mediante LDAP.

Aún estoy probando si HIN Mailgateway también ha recibido el inicio de sesión LDAP y completaré el artículo después. Dado que las appliances HIN se basan en el mismo firmware de SEPPmail, parto de esa premisa.

## Requisitos

Antes de la configuración, deben estar preparados cuatro elementos:

- **Firmware 15.0.6.1:** La función llega con la versión 15.0.6; debido a los dos errores de RuleEngine de esta versión, la elección adecuada es directamente el hotfix 15.0.6.1.
- **Un directorio compatible con LDAP:** Un Active Directory, OpenLDAP o equivalente. Si los usuarios solo están en Entra ID, que no habla LDAP por sí mismo, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) actúa de puente.
- **Una cuenta de enlace en el directorio:** Una cuenta de servicio dedicada y sin privilegios, con acceso de lectura, mediante la cual la appliance realiza la búsqueda LDAP. No debe ser un administrador de dominio.
- **Un grupo de AD para los administradores del Gateway:** Por ejemplo, un grupo de seguridad SEPPmail-Admins que posteriormente se asigna al grupo local admin. La pertenencia a este grupo determina entonces el acceso administrativo completo.

TLS está activado de forma predeterminada en los ajustes de conexión y debe mantenerse así; las credenciales de los administradores no deben circular sin cifrar por la red. La appliance debe poder acceder al servidor LDAP en el puerto configurado (normalmente, 636 para LDAPS).

## Configuración en User > Advanced Settings

La configuración se encuentra en la GUI de administración, en **User > Advanced Settings**, dentro de la sección **External Authentication**, y consta de cuatro bloques.

**1. Connection Settings:** La casilla *Authenticate users to external LDAP server (e.g. Active Directory)* activa la función. A continuación se indican la dirección del servidor, el puerto, la opción *TLS required*, así como el Bind DN y el Bind Password de la cuenta de servicio.

**2. User Attributes:** Aquí se define cómo encuentra la appliance los objetos de usuario: la LDAP Object Class (en Active Directory, normalmente person), la Search Base (la OU o el contenedor donde se encuentran las cuentas de administrador) y el atributo de correo electrónico (predeterminado: mail).

**3. Group Attributes:** De forma análoga, aquí se indican los datos para los objetos de grupo, de modo que la appliance pueda resolver las pertenencias a grupos.

**4. Mapping Settings:** La parte decisiva. En *Remote Group* se selecciona el grupo del servidor LDAP y en *Local Group* uno o varios grupos locales a los que se asigna. Para disponer de acceso administrativo completo, se trata del grupo admin; sus miembros tienen los mismos privilegios que el usuario predeterminado admin. Quien quiera diferenciar, puede asignar en su lugar grupos restringidos como readonly admin o grupos de la appliance basados en funciones.

Antes de guardar, conviene usar el **Login Test** integrado: con el nombre de usuario y la contraseña de una cuenta de prueba puede comprobarse si la conexión, la búsqueda y la autenticación funcionan antes de activar la configuración.

## Configuraciones de ejemplo

Los siguientes valores deben adaptarse al entorno propio (dominio de ejemplo example.com). Los nombres de los campos corresponden a la sección External Authentication de la appliance.

### Active Directory

| Campo | Valor |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
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

Notas sobre Active Directory: cualquier Domain Controller accesible puede utilizarse como servidor; en entornos con varias ubicaciones se recomienda un DC en la misma ubicación o un alias que apunte a varios DC. El puerto 636 corresponde a LDAPS; para ello, el certificado del DC debe poder validarse desde la appliance. La Search Base debe delimitarse de forma que incluya las cuentas de administrador, pero no todo el directorio. El atributo mail debe estar configurado en las cuentas de AD.

### OpenLDAP

| Campo | Valor |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
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

Notas sobre OpenLDAP: en configuraciones típicas, los usuarios se encuentran como inetOrgPerson en ou=people. Para los grupos, groupOfNames es la opción fiable, ya que la pertenencia se representa allí mediante el atributo member con el DN completo. En cambio, los grupos posixGroup solo incluyen a sus miembros como memberUid (nombre de usuario en lugar de DN); no está documentado si la appliance puede resolverlo, por lo que debe comprobarse con el Login Test antes de realizar el cambio. Si el servidor funciona únicamente con STARTTLS en el puerto 389, debe introducirse el puerto correspondiente en el campo Server; la conexión no debe funcionar sin cifrado en ningún caso.

## Indicaciones de funcionamiento

Hay tres puntos que merecen atención antes de que el inicio de sesión LDAP se convierta en la única vía de acceso a la appliance:

- **Mantener un acceso local de emergencia.** Las contraseñas de los usuarios externos se encuentran en el servidor LDAP. Si el directorio no está disponible (por un problema de red, mantenimiento de AD o porque el Gateway debe solucionar precisamente un problema de esa red), debe seguir existiendo una cuenta de administrador local con una contraseña almacenada de forma segura. Por tanto, no se debe eliminar el usuario predeterminado admin, sino mantenerlo como acceso de emergencia documentado.
- **La MFA sigue siendo relevante.** La versión 15.0.6 también ha revisado el inicio de sesión con MFA: el segundo factor ya no se añade a la contraseña, sino que se solicita en un campo propio. La autenticación externa no sustituye al segundo factor.
- **Offboarding mediante el directorio.** La verdadera ventaja de la integración: si un administrador abandona la empresa, basta con desactivar la cuenta de AD o eliminarla del grupo asignado. Ya no es necesario mantener cuentas locales en cada appliance. No obstante, los objetos de usuario autenticados externamente visibles de forma local deberían compararse periódicamente con el directorio.

## Conclusión

La autenticación LDAP para la GUI de administración cubre una carencia que la appliance tuvo durante mucho tiempo: ahora los accesos de administrador pueden gestionarse de forma centralizada en el directorio, en lugar de hacerlo por dispositivo. Junto con el campo MFA independiente, la versión 15.0.6 mejora notablemente el inicio de sesión en la interfaz de administración en una sola versión. Quien introduzca esta función debe mantener el mapeo de grupos deliberadamente restrictivo y conservar el acceso local de emergencia.

## Fuentes

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): entrada sobre la autenticación de la GUI de administración con descripción de la función, ubicación de la configuración y comportamiento de los usuarios autenticados externamente.

2.  [Documentación de SEPPmail – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): referencia de los campos de la sección External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentación de SEPPmail – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): grupos predefinidos de la appliance; los miembros del grupo admin tienen acceso administrativo sin restricciones.

4.  [Documentación de SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): notas de versión oficiales de la 15.0.6 con la entrada sobre la autenticación de la GUI de administración mediante servidores LDAP externos.

5.  [SEPPmail 15.0.6 y 15.0.6.1: correcciones de seguridad y nuevas funciones de administración](/blog/seppmail-releases-15-0-6-und-15-0-6-1): resumen de todos los cambios de ambas versiones.
