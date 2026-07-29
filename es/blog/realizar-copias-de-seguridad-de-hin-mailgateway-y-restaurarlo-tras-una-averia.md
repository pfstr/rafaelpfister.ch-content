---
title: "Realizar copias de seguridad de HIN Mailgateway y restaurarlo tras una avería"
navTitle: "Copia de seguridad y recuperación"
description: "Un clúster protege HIN Mailgateway frente a la caída de un nodo, pero no sustituye una copia de seguridad. Lo decisivo son la configuración, el material de claves, el orden de restauración y los cambios introducidos por Stargate."
date: "2026-07-08"
kategorie: "HIN-Gateway"
timeToRead: "15 min de lectura"
themen:
  - hin-gateway
slug: "realizar-copias-de-seguridad-de-hin-mailgateway-y-restaurarlo-tras-una-averia"
translationOf: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/es/blog/realizar-copias-de-seguridad-de-hin-mailgateway-y-restaurarlo-tras-una-averia"
translationId: article-845fb4bd0e4c592a
translationReview: automatic
translationSourceHash: 39ecd30339131eb74d0748f4bfb31ead3f98aefbd47621974b1e032f1a96b345
translatedAt: 2026-07-29T12:29:38.956Z
---

# Realizar copias de seguridad de HIN Mailgateway y restaurarlo tras una avería

Muchos HIN Mailgateways en producción funcionan como clúster. Si falla un nodo, el otro toma el relevo. Sin embargo, esta redundancia no ayuda frente a una regla errónea, un certificado eliminado o una importación dañada: [Los datos relevantes para el sistema se replican en todos los nodos](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), incluidos los cambios no deseados.

Por ello, para una recuperación fiable se requiere una copia de seguridad independiente. Dado que HIN Mailgateway se basa técnicamente en una appliance SEPPmail con GINA, se aplican sus mecanismos documentados de copia de seguridad y restauración.

## Qué datos se almacenan en el gateway

El gateway procesa correos entrantes y salientes según un conjunto central de reglas y los cifra, según el destinatario, mediante S/MIME, OpenPGP o TLS; para destinatarios sin material de claves propio se utiliza el procedimiento web GINA. Para la copia de seguridad es crucial que [el contenido de los mensajes no se almacene de forma persistente en el gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): la appliance procesa los correos en tránsito, sin archivarlos.

  

## Qué replica el clúster

SEPPmail admite varias [modalidades de clúster: alta disponibilidad, balanceo de carga y clúster geográfico](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); los parámetros del sistema, los datos de usuarios y el material de claves se sincronizan entre todos los nodos. En el [clúster frontend/backend, el frontend no dispone de una base de datos de configuración propia](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): puede funcionar en una DMZ sin almacenamiento de datos y recibe solo los datos necesarios para el procesamiento actual; la base de datos, junto con las claves, reside en el backend. Para [Large File Transfer (LFT) existe una excepción](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): a cada participante, incluidos los frontends, se le asigna un disco del mismo tamaño y los datos LFT se sincronizan en todos los nodos.

  

## Por qué la replicación no es una copia de seguridad

> *La replicación copia el estado actual, incluido el defectuoso. Una copia de seguridad conserva un estado conocido y funcional.*

Una importación errónea, una clave eliminada o un dominio desactivado se replica en segundos en el nodo asociado. Sin una copia de seguridad independiente, deja de existir un punto de recuperación. La estrecha relación entre disponibilidad y consistencia en el clúster se evidenció con los [problemas de inicio de sesión tras la actualización a 15.0.5](/blog/hin-update-issue-version-15.0.5), causados por una replicación de clúster alterada.

  

## Qué incluye la copia de seguridad y qué no

La [copia de seguridad de SEPPmail es deliberadamente ligera](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): comprende exclusivamente la configuración y el material de claves criptográficas: [sin mensajes, sin cola de correo y expresamente sin registros](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (por ello, los registros deben enviarse mediante Syslog a un sistema externo). Desde el firmware 14.0.0, la appliance genera la copia de seguridad [automáticamente a medianoche como backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); puede obtenerse mediante `Download`, `Send Backup` (correo electrónico al grupo de copias de seguridad) o SCP.

| **Incluido en la copia de seguridad** | **No incluido en la copia de seguridad** |
| --- | --- |
| Configuración del sistema y conjunto de reglas | Contenido de correo electrónico / textos de mensajes |
| Cuentas de usuario y GINA | Cola de correo actual |
| Material de claves: S/MIME, X.509, OpenPGP | Registros del sistema y de correo (proteger externamente mediante Syslog) |
| Configuración de TLS y certificados | Sistema operativo / imagen de VM |


De ello se deduce que, como el sistema operativo no está incluido en la copia de seguridad de configuración, una estrategia completa de recuperación ante desastres requiere además una forma de restaurar la base de la appliance (nuevo despliegue desde la imagen del fabricante o instantánea de VM). Posteriormente, la copia de seguridad de configuración restablece la configuración y las claves.

  

## Las instantáneas no son una copia de seguridad del clúster

Desde el firmware 14.0.0, la appliance también crea [instantáneas locales, aunque solo si existe una partición LFT con base de datos](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Los domingos se crea una instantánea completa y, de lunes a sábado, una incremental cada día; la retención es de 14 días.

Lo decisivo para la planificación de recuperación ante desastres: aunque estas instantáneas se ejecutan en segundo plano en un clúster, no se ofrece restauración a partir de ellas. Por tanto, las instantáneas son una ayuda local para revertir cambios en sistemas individuales, no una recuperación de clúster. La protección fiable sigue siendo la copia de seguridad de configuración cifrada.

  

## Configurar la copia de seguridad

El requisito para cualquier método de obtención es establecer una contraseña de copia de seguridad en [Administración › Copia de seguridad › Cambiar contraseña](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); sin esta contraseña no se descarga, envía ni proporciona mediante SCP. De forma predeterminada, la copia nocturna se envía por correo electrónico al [grupo «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); un usuario dedicado a copias de seguridad necesita una dirección de correo interno válida.

-   Establecer la contraseña de copia de seguridad y [guardarla separada de la copia](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): la copia contiene claves privadas.
    
-   Para el almacenamiento automatizado, [recuperar las copias mediante SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): almacenar la clave pública `SSH-RSA`\- en la administración y descargar mediante el usuario del sistema operativo `backup` el archivo `backup.tgz` disponible a medianoche.
    
-   Proteger los registros por separado (Syslog externo), ya que [deliberadamente no forman parte de la copia de seguridad](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Estrategia de copia de seguridad en clúster

En un clúster, son decisivas una copia ordenada y una gestión coherente de versiones.

-   **Diariamente**: recuperar la copia de seguridad de configuración cifrada [mediante SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) y almacenarla externamente con control de versiones
    
-   **Semanalmente**: copia de seguridad completa de VM o sistema de ambos nodos, escalonada en el tiempo en lugar de simultánea (el sistema operativo no forma parte de la copia de seguridad de configuración)
    
-   **Antes de mantenimiento o actualización**: detener la aceptación de correo mediante [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): los correos entrantes se rechazan temporalmente con un código de retorno SMTP configurable (predeterminado `421`); la configuración sigue activa incluso tras reiniciar.
    

  

Respecto a la gestión de versiones: en un clúster frontend/backend, SEPPmail actualiza [el frontend antes que el backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), y en actualizaciones de varias fases todos los participantes deben tener la misma versión antes de pasar a la siguiente versión. Tras una actualización mayor, puede ser necesario regenerar el conjunto de reglas (mensaje *«Current ruleset created for another version»*).

  

## Restauración y recuperación ante desastres

El caso básico es sencillo: [Importar archivo de copia de seguridad](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), reiniciar y, a continuación, el gateway funciona con todas sus prestaciones. Debe tenerse en cuenta la regla de versiones: solo se puede importar en la versión actual la copia de seguridad de la versión de firmware inmediatamente anterior (después debe regenerarse el conjunto de reglas); no es posible importar la copia de una versión de firmware más reciente en una versión anterior.

En el clúster existe una limitación importante:

-   **No restaurar nunca directamente un nodo individual**: no está previsto [restaurar un único participante del clúster](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). En su lugar, retire la máquina defectuosa del clúster, cree una nueva VM y vuelva a añadirla: la configuración y las claves llegan automáticamente por replicación desde el participante intacto.
    
-   **Pérdida total en todos los nodos**: vuelva a desplegar la appliance desde la imagen base, importe después la última copia de seguridad de configuración conocida y funcional, y reinicie.
    

Una copia de seguridad es tan fiable como la última prueba de restauración realizada con éxito. Debe realizarse una restauración de prueba al menos dos veces al año en un entorno aislado, no contra el clúster de producción.

  

### Lista de comprobación de restauración para casos de emergencia

1.  Retirar el nodo defectuoso del clúster (sin restauración directa de un participante).
    
2.  Crear una nueva VM o, en caso de pérdida total, proporcionar la appliance desde la imagen base/instantánea de VM.
    
3.  Solo en caso de pérdida total: importar la última copia de seguridad de configuración funcional (tener preparada la contraseña y respetar la regla de versiones).
    
4.  Comprobar el nodo de forma aislada: aceptación SMTP, TLS, GINA, conjunto de reglas.
    
5.  Incorporarlo al clúster y observar la replicación; si aparece el aviso, regenerar el conjunto de reglas.
    
6.  Documentar el incidente y actualizar el intervalo de copia de seguridad y las versiones.
    

  

Dos acciones de mantenimiento requieren especial cautela y siempre una copia de seguridad previa: [ampliar la partición LFT apaga la appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), y el restablecimiento de fábrica sobrescribe el disco duro diez veces (la consulta de seguridad exige introducir el código en orden inverso).

  

## Qué cambia con «Stargate»

HIN está sustituyendo gradualmente el Mailgateway actual por el [nuevo HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (proyecto «Stargate», gestionado como [«Verimesh» por Vereign AG](https://www.vereign.com/)). No se trata de una sustitución uno a uno de la appliance, sino de un cambio arquitectónico que afecta fundamentalmente a la copia de seguridad y a la recuperación ante desastres:

-   **De centralizado a descentralizado**: los nodos se comunican directamente entre sí; desaparece un centro de distribución central.
    
-   **Gestión descentralizada de claves (DKMS)**: cada organización administra su propia identidad criptográfica, sin una autoridad de certificación central.
    
-   **Cifrado de extremo a extremo** con fragmentación de los mensajes.
    
-   **Resiliencia desde la red**: si falla un nodo, la malla sigue funcionando.
    
-   **Implementación de referencia abierta**: la [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) es de código abierto bajo AGPLv3.
    

Calendario: la infraestructura descentralizada está [en producción en el sector sanitario suizo desde abril de 2025](https://www.vereign.com/); para 2026 está prevista la sustitución gradual de los Mailgateways actuales y un despliegue amplio. Las organizaciones con dominios propios de HIN (`@hin.ch`, `@verband-hin.ch`) funcionan en infraestructura de HIN y apenas se ven afectadas por la transición.

  

Para el manual de operaciones, esto significa que la disciplina clásica de «exportar la configuración y las claves de la appliance y restaurarlas en un nodo de sustitución» pierde importancia. En su lugar entran el registro de nodos, la custodia de identidades y claves en la malla, y la reincorporación de nodos a la red.

  

## La separación más importante

Mientras HIN MGW funcione con tecnología SEPPmail, se aplica lo siguiente: el clúster compensa los fallos de hardware, pero la responsabilidad por la integridad de la configuración y las claves permanece en el operador. La copia de seguridad ligera de configuración debe protegerse independientemente del clúster (mediante SCP, con control de versiones y contraseña almacenada por separado); las instantáneas no la sustituyen en el clúster, las versiones deben permanecer sincronizadas y la restauración debe probarse periódicamente de forma aislada. La migración a Stargate debe incorporarse pronto a la planificación de recuperación ante desastres, ya que desplaza la resiliencia y la custodia de claves a la red descentralizada.

## Fuentes

1.  [Documentación de SEPPmail – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): contenido de la copia de seguridad (solo configuración y material de claves), creación nocturna, recuperación automática del clúster mediante replicación.
    
2.  [Documentación de SEPPmail – «Administración»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): referencia detallada: menú de copia de seguridad (Download / Send Backup / Change password, `backup.tgz` a medianoche), instantáneas LFT (14 días, sin restauración en clúster), reglas de restauración y procedimiento de clúster, Preempt (código de retorno SMTP, predeterminado 421), clonación de dispositivos, canales de actualización y orden de actualización (frontend antes que backend), restablecimiento de fábrica, importación/exportación masiva.
    
3.  [Documentación de SEPPmail – «Crear usuario de copia de seguridad»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): grupo «backup (Backup operator)», cifrado y gestión de contraseñas.
    
4.  [Documentación de SEPPmail – «Copiar copia de seguridad mediante SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): recuperación de `backup.tgz` mediante SCP a través del usuario del sistema operativo `backup` en lugar de envío por correo.
    
5.  [Documentación de SEPPmail – «Clúster / Alta disponibilidad»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipos de clúster y datos sincronizados entre todos los nodos (parámetros del sistema, datos de usuarios, material de claves).
    
6.  [Documentación de SEPPmail – «Clúster frontend/backend»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): frontend sin base de datos de configuración, funcionamiento en DMZ, datos bajo demanda; backend como repositorio de datos.
    
7.  [Documentación de SEPPmail – «Almacenamiento de datos en el clúster (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): disco adicional del mismo tamaño para cada participante, sincronización de los datos LFT en todos los nodos.
    
8.  [HIN AG – «Del Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): comunicación de HIN sobre el sucesor Stargate: nodos descentralizados, concepto Data Mesh, calendario, cifrado de extremo a extremo.
    
9.  [Vereign AG – «Verimesh» / Vereign Client Library (vcl, etiqueta 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1): base técnica de Stargate: gestión descentralizada de claves (DKMS), cifrado de extremo a extremo con fragmentación de mensajes, código abierto bajo AGPLv3.
