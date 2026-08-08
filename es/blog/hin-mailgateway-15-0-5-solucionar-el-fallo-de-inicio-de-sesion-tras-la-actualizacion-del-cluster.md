---
slug: "hin-mailgateway-15-0-5-solucionar-el-fallo-de-inicio-de-sesion-tras-la-actualizacion-del-cluster"
title: "HIN Mailgateway 15.0.5: solucionar el fallo de inicio de sesión tras la actualización del clúster"
navTitle: "Error de inicio de sesión 15.0.5"
description: "Tras actualizar un clúster de HIN Mailgateway a la versión 15.0.5, el inicio de sesión falla en ambos nodos al cabo de unos minutos. Este procedimiento permite volver a poner las appliances en funcionamiento de forma controlada."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min de lectura"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: d40e2a9637cb0711ca33ed8ffde7827192362f668bebafb7b1f6862fda736d04
translatedAt: 2026-08-08T14:23:35.102Z
url: https://rafaelpfister.ch/es/blog/hin-mailgateway-15-0-5-solucionar-el-fallo-de-inicio-de-sesion-tras-la-actualizacion-del-cluster
translationModel: gpt-5.6-terra
---

# HIN Mailgateway 15.0.5: solucionar el fallo de inicio de sesión tras la actualización del clúster

Al actualizar un HIN Mailgateway de la versión 14.1.4.2 a la 15.0.5, un error en la replicación del clúster puede bloquear el inicio de sesión en ambas appliances. Los sistemas individuales no se ven afectados. El fabricante conoce el problema y prevé una corrección para una versión posterior.

**Actualización del 29 de julio de 2026:** La corrección anunciada ya está disponible. La versión de parche 15.0.6 suprime el rehashing de contraseñas cuando los miembros del clúster ejecutan versiones de firmware diferentes. Esta es exactamente la situación que había provocado el fallo descrito aquí. El contexto se explica en el artículo sobre [SEPPmail 15.0.6 y 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); el siguiente procedimiento de recuperación sigue siendo relevante para los clústeres que todavía se actualicen a la versión 15.0.5.

## Síntoma

Inmediatamente después de la actualización, la interfaz web todavía puede abrirse. Unos diez minutos más tarde, el inicio de sesión falla en ambos nodos del clúster. El hecho de que el error aparezca con retraso y en ambos sistemas apunta a la configuración replicada del clúster como causa.

## Recuperación

Los siguientes pasos modifican la configuración del clúster. Antes deben estar disponibles copias de seguridad actualizadas y el identificador del clúster.

1. Restaurar los snapshots de ambos nodos del clúster creados simultáneamente.
2. Tras la restauración, dejar apagado uno de los nodos.
3. En el nodo en funcionamiento, descargar primero el identificador del clúster y después disolver el clúster.
4. Atención: tras disolverlo, la appliance se reinicia inmediatamente y sin más confirmación.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Actualizar el primer nodo a la versión 15.0.5 y después apagarlo.
6. Iniciar el segundo nodo y repetir allí los mismos pasos.
7. Solo cuando ambos sistemas funcionen individualmente y tengan la misma versión, volver a crear el clúster conforme a la documentación del fabricante.

Este procedimiento evita que una configuración defectuosa se replique de nuevo entre los nodos durante la actualización.

## Fuentes

1. [Documentación de SEPPmail – «Clúster / Alta disponibilidad»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipos de clúster y replicación de la configuración en todos los nodos.
2. [Documentación de SEPPmail – «Administración»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): orden de actualización en el clúster (frontend antes que backend) y el requisito de versiones idénticas.
3. [HIN Mailgateway: Backup y Disaster Recovery en el clúster](/blog/hin-mailgateway-backup-disaster-recovery): análisis detallado de la replicación del clúster, la copia de seguridad y la restauración.
