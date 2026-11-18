---
title: "Proofpoint Essentials noviembre de 2026: nuevo permiso de Graph para Message Pull hasta enero de 2027"
navTitle: "Proofpoint Essentials 12/11/2026"
description: "La versión de Proofpoint Essentials del 12 de noviembre de 2026 anuncia que One-Click Message Pull y Automatic Remediation necesitarán el permiso de Microsoft Graph Mail-Advanced.ReadWrite.All a partir del 1 de enero de 2027. Añada el permiso en la aplicación de Entra y conceda el consentimiento de administrador antes de la fecha límite."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 min de lectura"
themen:
  - proofpoint
  - microsoft-365-exchange
produkte:
  - "proofpoint-essentials"
  - "proofpoint-releases"
  - "exchange-online"
  - "entra"
protokolle:
  - "releases"
  - "apis"
slug: "proofpoint-essentials-noviembre-de-2026-nuevo-permiso-de-graph-para-message-pull-hasta-enero-de"
translationId: article-eb5da8564c33d2e9
translationOf: proofpoint-essentials-november-2026
url: https://rafaelpfister.ch/es/blog/proofpoint-essentials-noviembre-de-2026-nuevo-permiso-de-graph-para-message-pull-hasta-enero-de
translationSourceHash: 269840f611e0f96902a1ed742ee1e48ab8d618005a603133463ff394a9a28a3c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:24:34.018Z
translationReview: required
---

# Proofpoint Essentials noviembre de 2026: nuevo permiso de Graph para Message Pull hasta enero de 2027

Proofpoint publicó el 12 de noviembre de 2026 las notas de la versión de Proofpoint Essentials, el servicio en la nube para la seguridad del correo electrónico en pequeñas y medianas empresas. La entrada es la única versión de noviembre de 2026 y no incluye nuevas funciones ni correcciones de errores: anuncia que One-Click Message Pull y Automatic Remediation, después del 1 de enero de 2027, solo funcionarán con el permiso adicional de Microsoft Graph `Mail-Advanced.ReadWrite.All`. El motivo es un cambio de Microsoft que, a partir del 31 de diciembre de 2026, restringe la edición de propiedades sensibles de mensajes enviados o recibidos mediante la API de Graph. El aviso está dirigido a administradores y partners que hayan integrado Proofpoint Essentials con Microsoft 365 y utilicen una de las dos funciones. Esta versión no afecta a los operadores de Proofpoint Protection Server (PPS, local).

## Versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| Versión del 12 de noviembre de 2026 | 12/11/2026 | Aviso sobre permisos | `Mail-Advanced.ReadWrite.All` obligatorio para One-Click Message Pull y Automatic Remediation a partir del 1 de enero de 2027 |

## Permiso de Microsoft Graph Mail-Advanced.ReadWrite.All

Según las notas de la versión, los clientes que quieran seguir utilizando One-Click Message Pull y Automatic Remediation después del 1 de enero de 2027 deben ampliar su integración de Microsoft 365 con el permiso de aplicación `Mail-Advanced.ReadWrite.All` (Microsoft Graph). Los permisos existentes se mantienen sin cambios. No es necesaria ninguna configuración adicional en Proofpoint Essentials.

La causa está en Microsoft. En el Microsoft 365 Developer Blog, el equipo de Exchange anunció el 26 de marzo de 2026 que, a partir del 31 de diciembre de 2026, las aplicaciones solo podrán modificar propiedades sensibles de mensajes que no sean borradores con uno de los permisos `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All` o `Mail-Advanced.ReadWrite.Shared`. Microsoft cita como ejemplos el asunto, el cuerpo del mensaje y los destinatarios. Con `Mail.ReadWrite`, posteriormente solo podrán editarse borradores en estas propiedades. Todos los permisos Mail-Advanced requieren el consentimiento de un administrador del tenant. Según Microsoft, los permisos ya están disponibles, por lo que pueden añadirse antes de la fecha límite.

La documentación de la integración de Microsoft 365 de Proofpoint Essentials sigue indicando únicamente `Mail.ReadWrite` para One-Click Message Pull (a 17 de noviembre de 2026). Con este permiso, Proofpoint Essentials mueve mensajes directamente en el buzón del usuario, por ejemplo, a la cuarentena o desde ella.

## Integración mediante la Microsoft Integration App

Si la aplicación de Azure o Entra para Proofpoint Essentials se creó automáticamente mediante la Microsoft Integration App, añada el permiso según las notas de la versión de la siguiente manera:

1. Inicie sesión en Proofpoint Essentials como administrador con acceso al cliente correspondiente (Partner Admin, Channel Admin u Org Admin).
2. Abra **Account Management > Integrations**.
3. En la tarjeta **Microsoft 365**, haga clic en **Details**.
4. Haga clic en **Update Permissions**.
5. Haga que la posterior solicitud de consentimiento sea confirmada por un administrador de Microsoft 365 con un rol suficiente, por ejemplo, Global Administrator o Privileged Role Administrator.

A continuación, la aplicación añade el permiso `Mail-Advanced.ReadWrite.All`.

## Aplicación de Entra creada manualmente

Si la aplicación se registró manualmente, según las notas de la versión debe seguirse este procedimiento:

1. Inicie sesión en el Centro de administración de Microsoft Entra (`https://entra.microsoft.com`) con una cuenta que pueda conceder consentimientos de administrador (por ejemplo, Global Administrator o Privileged Role Administrator).
2. Abra **Identity > Applications > App registrations**.
3. Abra el registro de la aplicación de Proofpoint Essentials. Si no aparece, cambie a la pestaña **All applications**.
4. Seleccione **API permissions** y, a continuación, **Add a permission > Microsoft Graph > Application permissions**.
5. Busque `Mail-Advanced.ReadWrite.All`, marque el permiso y seleccione **Add permissions**.
6. Seleccione **Grant admin consent for <Organisation>** y confirme.
7. Compruebe que el estado del permiso indique **Granted for <Organisation>**.

Según Proofpoint, todos los permisos existentes se conservan.

## Procedimiento recomendado

Añada `Mail-Advanced.ReadWrite.All` antes del 31 de diciembre de 2026, fecha a partir de la cual Microsoft aplicará la restricción; Proofpoint indica como fecha límite el 1 de enero de 2027. Primero determine si la aplicación de Entra se creó mediante la Microsoft Integration App o manualmente, y siga el procedimiento correspondiente. Para ello, tenga disponible un administrador con el rol Global Administrator o Privileged Role Administrator. Después de conceder el consentimiento, compruebe en el Centro de administración de Entra que el permiso aparezca como concedido. Las notas de la versión no describen en detalle qué ocurrirá después de la fecha límite sin el nuevo permiso; solo indican que el permiso es necesario para seguir utilizando las dos funciones (a 17 de noviembre de 2026).

## Fuentes

1.  [Proofpoint Essentials: Release Notes](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): entrada del 12 de noviembre de 2026 con el nuevo permiso, la fecha límite del 1 de enero de 2027 y los pasos para ambos tipos de integración.

2.  [Proofpoint Essentials: Microsoft 365 Integration MX](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): permisos de Graph actuales de la integración, incluido `Mail.ReadWrite` para One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: Graph API updates to sensitive email properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): anuncio del 26 de marzo de 2026, aplicación a partir del 31 de diciembre de 2026 y permisos Mail-Advanced con consentimiento de administrador.
