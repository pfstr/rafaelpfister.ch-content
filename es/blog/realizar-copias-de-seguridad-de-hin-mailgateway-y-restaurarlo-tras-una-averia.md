---
title: "Realizar copias de seguridad y recuperar HIN Mailgateway tras un fallo"
navTitle: "Backup y recuperación"
description: "Un clúster protege HIN Mailgateway frente al fallo de un nodo, pero no sustituye una copia de seguridad. Son decisivos la configuración, el material de claves, el orden de restauración y los cambios introducidos por Stargate."
date: "2026-07-08"
kategorie: "HIN-Gateway"
timeToRead: "15 min de lectura"
themen:
  - hin-gateway
slug: "realizar-copias-de-seguridad-de-hin-mailgateway-y-restaurarlo-tras-una-averia"
translationOf: "hin-mailgateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translationReview: automatic
translationSourceHash: cd8f7c8633d8cf882725ad88feed7be5a1730ce563724481e25b4b9ee2ec84e4
translatedAt: 2026-11-06T03:23:20.401Z
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/realizar-copias-de-seguridad-de-hin-mailgateway-y-restaurarlo-tras-una-averia
translationModel: gpt-5.6-terra
---

# Realizar copias de seguridad y recuperar HIN Mailgateway tras un fallo

Muchos HIN Mailgateway en producción funcionan como clúster. Si falla un nodo, el otro toma el relevo. Sin embargo, esta redundancia no protege frente a una regla incorrecta, un certificado eliminado o una importación dañada: [Los datos relevantes para el sistema se replican en todos los nodos](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), incluidos los cambios no deseados.

Por ello, para una recuperación fiable se necesita una copia de seguridad independiente. Dado que HIN Mailgateway se basa técnicamente en una appliance SEPPmail con GINA, se aplican sus mecanismos documentados de copia de seguridad y restauración.

## Qué datos se encuentran en el gateway

El gateway procesa los correos entrantes y salientes según un conjunto central de reglas y los cifra según el destinatario mediante S/MIME, OpenPGP o TLS; para destinatarios sin material de claves propio se utiliza el procedimiento web GINA. Para la copia de seguridad es decisivo que [el contenido de los mensajes no se almacena de forma persistente en el gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): la appliance procesa los correos en tránsito, sin archivarlos.

  

## Qué replica el clúster

SEPPmail admite varias [configuraciones de clúster: alta disponibilidad, balanceo de carga y Geo-Cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); en todos los nodos se sincronizan los parámetros del sistema, los datos de usuario y el material de claves. En el [clúster frontend/backend, el frontend no dispone de una base de datos de configuración propia](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): puede operar en una DMZ sin almacenamiento de datos y solo recibe los datos necesarios para el procesamiento actual; la base de datos, incluidas las claves, se encuentra en el backend. Para [Large File Transfer (LFT) se aplica una excepción](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): a cada miembro, incluidos los frontends, se le asigna un disco del mismo tamaño y los datos LFT se sincronizan en todos los nodos.

  

## Por qué la replicación no es una copia de seguridad

> *La replicación copia el estado actual, incluido el erróneo. Una copia de seguridad conserva un estado conocido y funcional.*

Una importación incorrecta, una clave eliminada o un dominio desactivado se replica en cuestión de segundos en el nodo asociado. Sin una copia de seguridad independiente, deja de existir un punto de recuperación. La estrecha relación entre disponibilidad y consistencia en el clúster se hizo patente con los [problemas de inicio de sesión tras la actualización a 15.0.5](/blog/hin-update-issue-version-15.0.5), provocados por una replicación de clúster alterada.

  

## Qué incluye la copia de seguridad y qué no

La [copia de seguridad de SEPPmail es deliberadamente ligera](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): incluye exclusivamente la configuración y el material criptográfico de claves: [ni mensajes, ni cola de correo y expresamente ningún log](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (por ello, los logs deben enviarse mediante Syslog a un sistema externo). Desde el firmware 14.0.0, la appliance crea la copia de seguridad [automáticamente por la noche a medianoche como backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); puede recuperarse mediante `Download`, `Send Backup` (correo electrónico al grupo de copias de seguridad) o SCP.

| **Incluido en la copia de seguridad** | **No incluido en la copia de seguridad** |
| --- | --- |
| Configuración del sistema y conjunto de reglas | Contenido de los correos / textos de los mensajes |
| Cuentas de usuario y GINA | Cola de correo actual |
| Material de claves: S/MIME, X.509, OpenPGP | Logs del sistema y de correo (proteger externamente mediante Syslog) |
| Configuración TLS y de certificados | Sistema operativo / imagen de VM |


De ello se deduce que, puesto que el sistema operativo no está incluido en la copia de seguridad de configuración, una estrategia completa de DR necesita además una forma de restaurar la base de la appliance (nuevo despliegue desde la imagen del fabricante o instantánea de VM). La copia de seguridad de configuración añade posteriormente la configuración y las claves.

  

## Las instantáneas no son una copia de seguridad del clúster

Desde el firmware 14.0.0, la appliance crea además [instantáneas locales, aunque solo si existe una partición LFT con base de datos](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Los domingos se crea una instantánea completa y de lunes a sábado una instantánea incremental; la retención es de 14 días.

Para la planificación de DR es decisivo que, aunque estas instantáneas se ejecutan en segundo plano en el funcionamiento en clúster, no se ofrece ninguna restauración a partir de ellas. Por tanto, las instantáneas son una ayuda local para revertir cambios en sistemas individuales, no una recuperación de clúster. La protección fiable sigue siendo la copia de seguridad de configuración cifrada.

  

## Configurar la copia de seguridad

El requisito para cualquier vía de recuperación es establecer una contraseña de copia de seguridad en [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); sin esta contraseña no se descarga, envía ni proporciona nada mediante SCP. De forma predeterminada, la copia de seguridad nocturna se envía por correo electrónico al [grupo «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); un usuario de copia de seguridad dedicado necesita una dirección de correo interna válida.

-   Establecer la contraseña de copia de seguridad y [guardarla separadamente de la copia de seguridad](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): la copia contiene claves privadas.
    
-   Para el almacenamiento automatizado, [recuperar las copias de seguridad mediante SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): almacenar la clave pública `SSH-RSA`\- en la administración y recuperar mediante el usuario del sistema operativo `backup` el archivo `backup.tgz` disponible a medianoche.
    
-   Proteger los logs por separado (Syslog externo), ya que [deliberadamente no forman parte de la copia de seguridad](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Estrategia de copia de seguridad en funcionamiento en clúster

En el funcionamiento en clúster, una copia de seguridad ordenada y una gestión coherente de versiones son decisivas.

-   **Diariamente**: recuperar la copia de seguridad de configuración cifrada [mediante SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) y almacenarla externamente con versionado
    
-   **Semanalmente**: copia de seguridad completa de VM o del sistema de ambos nodos, escalonada en el tiempo en lugar de simultánea (el sistema operativo no forma parte de la copia de seguridad de configuración)
    
-   **Antes de mantenimiento o actualización**: detener la aceptación de correo mediante [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): los correos entrantes se rechazan temporalmente con un código de retorno SMTP configurable (predeterminado: `421`); la configuración permanece activa incluso tras un reinicio.
    

  

En cuanto a la gestión de versiones: en el clúster frontend/backend, SEPPmail actualiza [el frontend antes que el backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), y en las actualizaciones de varias etapas todos los miembros deben tener la misma versión antes de pasar a la siguiente versión. Tras una actualización mayor puede ser necesario regenerar el conjunto de reglas (mensaje *«Current ruleset created for another version»*).

  

## Restauración y recuperación ante desastres

El caso básico es sencillo: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), reiniciar y, a continuación, el gateway vuelve a funcionar con todas sus funciones. Debe tenerse en cuenta la regla de versiones: solo se puede importar en la versión actual la copia de seguridad de la versión de firmware inmediatamente anterior (después hay que regenerar el conjunto de reglas); no es posible importar una copia de seguridad de un firmware más reciente en una versión anterior.

En el clúster se aplica una restricción importante:

-   **Nunca restaurar directamente un nodo individual**: no está prevista la [restauración de un único miembro del clúster](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). En su lugar, se debe eliminar la máquina defectuosa del clúster, configurar una nueva VM y volver a añadirla: la configuración y las claves llegan automáticamente por replicación desde el miembro intacto.
    
-   **Pérdida total en todos los nodos**: volver a desplegar la appliance desde la imagen base, importar después la última copia de seguridad de configuración conocida y funcional y reiniciar.
    

Una copia de seguridad solo es tan fiable como la última prueba de restauración realizada con éxito. Debe realizarse una restauración de prueba al menos dos veces al año en un entorno aislado, no contra el clúster de producción.

  

### Lista de verificación de restauración para una emergencia

1.  Eliminar el nodo defectuoso del clúster (sin restauración directa de un miembro).
    
2.  Configurar una nueva VM o, en caso de pérdida total, proporcionar la appliance desde la imagen base o la instantánea de VM.
    
3.  Solo en caso de pérdida total: importar la última copia de seguridad de configuración funcional (tener preparada la contraseña y respetar la regla de versiones).
    
4.  Comprobar el nodo de forma aislada: aceptación SMTP, TLS, GINA, conjunto de reglas.
    
5.  Incorporarlo al clúster y observar la replicación; si aparece el aviso, regenerar el conjunto de reglas.
    
6.  Documentar el incidente y actualizar el intervalo de copia de seguridad y las versiones.
    

  

Dos acciones de mantenimiento requieren especial precaución y siempre una copia de seguridad previa: la [ampliación de la partición LFT apaga la appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), y el restablecimiento de fábrica sobrescribe el disco duro diez veces (la confirmación de seguridad exige introducir el código escrito al revés).

  

## Qué cambia con «Stargate»

HIN está sustituyendo gradualmente el Mailgateway actual por el [nuevo HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (proyecto «Stargate», denominado [«Verimesh» en la empresa Vereign AG de Zug](https://www.vereign.com/)). No se trata de un reemplazo 1:1 de la appliance, sino de un cambio arquitectónico que afecta de forma fundamental a la copia de seguridad y la recuperación ante desastres:

-   **De centralizado a descentralizado**: los nodos se comunican directamente entre sí; desaparece un centro de distribución central.
    
-   **Gestión descentralizada de claves (DKMS)**: cada organización administra su propia identidad criptográfica, sin una Certificate Authority central.
    
-   **Cifrado de extremo a extremo** con fragmentación de los mensajes.
    
-   **Resiliencia desde la red**: si falla un nodo, la malla sigue funcionando.
    
-   **Repositorio de despliegue abierto**: el repositorio [Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) con los servicios de un nodo Stargate es públicamente accesible bajo AGPLv3; en cambio, la Vereign Client Library (vcl) solo está disponible tras iniciar sesión.
    

Calendario: la infraestructura descentralizada está [en uso productivo en el sistema sanitario suizo desde abril de 2025](https://www.vereign.com/); para 2026 está prevista la sustitución gradual de los Mailgateways actuales y un despliegue amplio. Las organizaciones con dominios propios de HIN (`@hin.ch`, `@verband-hin.ch`) funcionan en la infraestructura de HIN y apenas se ven afectadas por la transición.

  

Para el manual de operaciones, esto significa que la disciplina clásica de «exportar la configuración y las claves de la appliance y restaurarlas en un nodo de sustitución» pierde importancia. En su lugar entran el enrolamiento de nodos, la custodia de identidades y claves en la malla, y la reincorporación de nodos a la red.

  

## La separación más importante

Mientras el HIN MGW funcione con tecnología SEPPmail, se aplica lo siguiente: el clúster compensa los fallos de hardware, pero la responsabilidad de la integridad de la configuración y las claves sigue recayendo en el operador. La copia de seguridad ligera de configuración debe protegerse independientemente del clúster (mediante SCP, versionada y con la contraseña guardada por separado), las instantáneas no la sustituyen en el clúster, las versiones se mantienen sincronizadas y la restauración se prueba regularmente de forma aislada. La transición a Stargate debe incorporarse pronto a la planificación de DR, ya que desplaza la resiliencia y la custodia de claves a la red descentralizada.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: la Vereign Client Library solo está disponible tras iniciar sesión; como fuente públicamente accesible bajo AGPLv3 se indica ahora el repositorio Stargate Deployment.

## Fuentes

1.  [Documentación de SEPPmail – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): contenido de la copia de seguridad (solo configuración y material de claves), creación nocturna, recuperación automática del clúster mediante replicación.
    
2.  [Documentación de SEPPmail – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): referencia detallada: menú de copia de seguridad (Download / Send Backup / Change password, `backup.tgz` a medianoche), instantáneas LFT (14 días, sin restauración en el clúster), reglas de restauración y procedimiento de clúster, Preempt (código de retorno SMTP, predeterminado 421), clonación de dispositivos, canales de actualización y orden de actualización (frontend antes que backend), restablecimiento de fábrica, importación/exportación masiva.
    
3.  [Documentación de SEPPmail – «Crear usuario de copia de seguridad»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): grupo «backup (Backup operator)», cifrado y gestión de contraseñas.
    
4.  [Documentación de SEPPmail – «Copiar copia de seguridad mediante SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): recuperación de `backup.tgz` mediante SCP a través del usuario del sistema operativo `backup` en lugar del envío por correo.
    
5.  [Documentación de SEPPmail – «Clúster / Alta disponibilidad»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipos de clúster y datos sincronizados en todos los nodos (parámetros del sistema, datos de usuario, material de claves).
    
6.  [Documentación de SEPPmail – «Clúster frontend/backend»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): frontend sin base de datos de configuración, funcionamiento en DMZ, datos bajo demanda; backend como titular de los datos.
    
7.  [Documentación de SEPPmail – «Almacenamiento de datos en el clúster (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): disco adicional del mismo tamaño para cada miembro, sincronización de los datos LFT en todos los nodos.
    
8.  [HIN AG – «Del Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): comunicación de HIN sobre el sucesor Stargate: nodos descentralizados, concepto Data Mesh, calendario, cifrado de extremo a extremo.
    
9.  [Vereign AG – «Verimesh»](https://vereign.com/): Verimesh (anteriormente proyecto Stargate) en uso productivo en el sistema sanitario suizo desde abril de 2025.
    
10. [Vereign GitLab – Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment): repositorio de despliegue público bajo AGPLv3 con los servicios de un nodo Stargate y referencia a la [documentación de despliegue](https://health-info-net-ag.github.io/Stargate-deployment/).
