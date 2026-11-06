---
slug: "hin-mailgateway-15-0-5-solucionar-el-fallo-de-inicio-de-sesion-tras-la-actualizacion-del-cluster"
title: "HIN Mailgateway 15.0.5: solucionar el fallo de inicio de sesión tras la actualización del clúster"
navTitle: "Error de inicio de sesión 15.0.5"
description: "Tras actualizar un clúster de HIN Mailgateway a la versión 15.0.5, el inicio de sesión falla en ambos nodos después de unos minutos. Este procedimiento permite volver a poner las appliances en funcionamiento de forma controlada."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min de lectura"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 126aded044bfb102e020aaf45b7b77f6658f77a6de88a71d1554b1fabeefce2a
translatedAt: 2026-11-06T03:28:56.904Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/hin-mailgateway-15-0-5-solucionar-el-fallo-de-inicio-de-sesion-tras-la-actualizacion-del-cluster
---

# HIN Mailgateway 15.0.5: solucionar el fallo de inicio de sesión tras la actualización del clúster

Al actualizar un HIN Mailgateway de la versión 14.1.4.2 a la 15.0.5, un error en la replicación del clúster puede provocar que el inicio de sesión deje de funcionar en ambas appliances. Los sistemas individuales no se ven afectados. El fabricante conoce el problema y tiene previsto corregirlo en una versión posterior.

**Actualización del 29 de julio de 2026:** La corrección anunciada ya está disponible. La versión de parche 15.0.6 suprime el rehashing de contraseñas cuando los miembros del clúster ejecutan versiones de firmware diferentes. Esta es exactamente la configuración que había provocado el fallo descrito aquí. El contexto se explica en el artículo sobre [SEPPmail 15.0.6 y 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); el siguiente procedimiento de recuperación sigue siendo relevante para los clústeres que aún se actualicen a 15.0.5.

## Síntomas

Inmediatamente después de la actualización, la interfaz web todavía se puede abrir. Aproximadamente diez minutos después, el inicio de sesión falla en ambos nodos del clúster. El hecho de que el error aparezca con retraso y en ambos sistemas indica que la causa es la configuración del clúster replicada.

## Recuperación

Los siguientes pasos modifican la configuración del clúster. Antes deben estar disponibles copias de seguridad actualizadas y el identificador del clúster.

1. Restaurar los snapshots creados simultáneamente de ambos nodos del clúster.
2. Después de la restauración, dejar apagado uno de los nodos.
3. En el nodo en funcionamiento, descargar primero el identificador del clúster y después disolver el clúster.
4. Atención: después de disolverlo, la appliance se reinicia inmediatamente y sin más confirmación.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Actualizar el primer nodo a la versión 15.0.5 y apagarlo después.
6. Iniciar el segundo nodo y repetir allí los mismos pasos.
7. Solo cuando ambos sistemas funcionen individualmente y tengan la misma versión, volver a crear el clúster conforme a la documentación del fabricante.

Este procedimiento evita que una configuración defectuosa vuelva a replicarse entre los nodos durante la actualización.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: según la documentación de SEPPmail, el orden de actualización en el clúster es backend antes que frontend, y no al revés.

## Fuentes

1. [Documentación de SEPPmail – «Clúster / Alta disponibilidad»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipos de clúster y replicación de la configuración en todos los nodos.
2. [Documentación de SEPPmail – «Administración»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): orden de actualización en el clúster (backend antes que frontend) y el requisito de versiones idénticas.
3. [HIN Mailgateway: Backup y recuperación ante desastres en el clúster](/blog/hin-mailgateway-backup-disaster-recovery): análisis detallado de la replicación del clúster, la copia de seguridad y la restauración.
