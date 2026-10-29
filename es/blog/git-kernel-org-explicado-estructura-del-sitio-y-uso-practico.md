---
title: "git.kernel.org explicado: estructura del sitio y uso práctico"
navTitle: "git.kernel.org"
description: "git.kernel.org es el alojamiento Git del desarrollo del kernel de Linux. Esta guía explica la lista de repositorios, el esquema de rutas, las vistas summary, log, tree y commit, los parámetros de búsqueda en la URL y las URL cortas a las que remiten las CVE del kernel."
date: "2026-10-29"
kategorie: "Kernel de Linux"
timeToRead: "9 min de lectura"
themen:
  - linux-kernel
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-explicado-estructura-del-sitio-y-uso-practico"
translationId: "article-91dab903d58c6173"
translationOf: git-kernel-org-erklaert
url: https://rafaelpfister.ch/es/blog/git-kernel-org-explicado-estructura-del-sitio-y-uso-practico
translationSourceHash: 34a3d0d525d79d1ef4c5ff31824dabdd6e910a1dd5a22e60c77b202f97a71b7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T14:34:17.541Z
translationReview: automatic
---

[git.kernel.org](https://git.kernel.org/) es el servidor Git de kernel.org. Aquí se encuentran la rama principal de desarrollo de Linus Torvalds, los kernels Stable, linux-next, cientos de árboles de subsistemas y desarrolladores, así como herramientas como `git` mismo, `e2fsprogs` o las páginas de manual de Linux. La interfaz web es una versión adaptada de [cgit](https://git.zx2c4.com/cgit/about/) (indicada en el pie de página como `cgit 1.3.1-korg`). Es de solo lectura: no hay cuentas, issues ni pull requests. El desarrollo del kernel se lleva a cabo mediante listas de correo, cuyo archivo está en [lore.kernel.org](https://lore.kernel.org/); git.kernel.org muestra el resultado, es decir, los commits que los mantenedores han incorporado a sus árboles.

Los motivos habituales para visitarlo son una entrada CVE que remite a un commit del kernel, la pregunta de si una corrección concreta está incluida en una versión del kernel o la consulta de un único archivo fuente sin clonar todo el repositorio.

## La página de inicio: lista de repositorios

La página de inicio enumera todos los repositorios públicos en una tabla. La agrupación sigue la ruta de archivos en el servidor: cada encabezado intermedio como `pub/scm/fs` o `pub/scm/linux` es un directorio, y debajo aparecen los repositorios con su ruta relativa a ese directorio.

<details class="options-details">
<summary>Resumen de las columnas</summary>

| Columna | Contenido |
|---|---|
| Name | Ruta del repositorio dentro del grupo, por ejemplo `xfs/xfs-linux.git` en `pub/scm/fs` |
| Description | Contenido del archivo `description` en el repositorio. «Unnamed repository; edit this file …» es el texto estándar de Git y solo significa que el operador no ha definido una descripción |
| Owner | persona o grupo responsable, por ejemplo «XFS FS Group» o «Infra Team» |
| Idle | tiempo desde el último commit en cualquier rama; un valor pequeño indica un árbol con mantenimiento activo |
| Links | accesos directos a las vistas `summary`, `log` y `tree` del repositorio |

</details>

Los grupos más importantes:

| Grupo | Contenido |
|---|---|
| `pub/scm/linux` | Árboles del kernel: Mainline, Stable, linux-next, así como los árboles personales y de subsistemas de los mantenedores |
| `pub/scm/fs` | Sistemas de archivos junto con herramientas de espacio de usuario (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Bibliotecas como `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | El propio proyecto Git (`git.git`) con documentación HTML y páginas de manual |
| `pub/scm/docs` | Fuentes de documentación, entre ellas `man-pages` y las claves PGP de los desarrolladores del kernel (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | Infraestructura de kernel.org, por ejemplo el fork de cgit y los registros de transparencia del servidor Gitolite |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, herramientas para desarrolladores como `sparse` y `pahole`, `uemacs` de Linus Torvalds |

La lista es muy extensa porque solo bajo `pub/scm/linux/kernel/git/` cada mantenedor con cuenta puede alojar sus propios árboles. El campo de búsqueda situado arriba a la derecha filtra la lista por nombre, descripción y propietario. La búsqueda también puede abrirse directamente como URL, por ejemplo `https://git.kernel.org/?q=xfs`.

## El esquema de rutas de los árboles del kernel

Los repositorios del kernel siguen el patrón:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` es el nombre de usuario del mantenedor en kernel.org, no necesariamente su nombre completo (`akpm` es Andrew Morton, `acme` es Arnaldo Carvalho de Melo). Para la mayoría de las consultas bastan tres árboles:

| Árbol | Ruta | Finalidad |
|---|---|---|
| Mainline | `torvalds/linux.git` | Rama de desarrollo de Linus Torvalds; aquí se crean las versiones `v7.3`, `v7.4-rc1` etc. |
| Stable | `stable/linux.git` | Versiones mantenidas con correcciones (`v7.3.1`, `v7.3.2` …) y los kernels Longterm; una rama por cada serie de versiones, por ejemplo `linux-7.3.y` |
| linux-next | `next/linux-next.git` | Estado integrado diariamente de todos los árboles de subsistemas para la próxima ventana de integración |

Por regla general, una corrección pasa por estos árboles en este orden: primero llega al árbol del subsistema responsable, aparece en linux-next, Linus Torvalds la incorpora a Mainline durante la ventana de integración mediante un correo de pull request y después se retroporta a las ramas Stable. El árbol de subsistema responsable figura en el archivo `MAINTAINERS` del código fuente del kernel (líneas con `T: git …`).

## Las vistas de un repositorio

Al hacer clic en el nombre de un repositorio se abre la vista `summary`. La barra de pestañas superior lleva a las demás vistas; a la izquierda, un cuadro de selección permite elegir la rama a la que se refieren todas las vistas.

<details class="options-details">
<summary>Resumen de las vistas</summary>

| Vista | Contenido |
|---|---|
| about | README del repositorio, si está disponible |
| summary | Ramas con el último commit, las etiquetas más recientes con enlace de descarga, los diez últimos commits y las URL de clonación |
| refs | Lista completa de todas las ramas y etiquetas |
| log | Historial de commits con campo de búsqueda; columnas de antigüedad, mensaje de commit, autor, archivos modificados y líneas |
| tree | Árbol de archivos de la rama o commit seleccionado, con enlaces a `plain` (archivo sin procesar) y `log` (historial del archivo) |
| commit | Metadatos y diff de un único commit |
| diff | Diferencia entre dos commits o entre un commit y su predecesor |
| stats | Estadísticas de commits por autor durante un periodo seleccionable |

</details>

En la vista `summary`, el bloque de etiquetas sirve para consultar la versión actual: Mainline muestra release candidates (`v7.3-rc7`) y versiones (`v7.3`). Los enlaces de descarga generan un archivo bajo demanda y por ello no siempre están disponibles; para versiones ya publicadas, los tarballs en [kernel.org](https://kernel.org/) o `cdn.kernel.org` son el medio de descarga previsto, también con firma PGP.

Al final de la vista `summary` se encuentran las URL de clonación. Además de `https://` y del `git://` sin cifrar, se indica un espejo en Google (`kernel.googlesource.com`) que ofrece el mismo contenido.

## Buscar en el historial

El campo de búsqueda de la vista `log` dispone de un cuadro de selección con cuatro modos. Cada modo también está disponible como parámetro de URL, de modo que las búsquedas pueden guardarse como marcadores o enlazarse en tickets:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Parámetro | Efecto |
|---|---|
| `qt=grep` | busca en los mensajes de commit (selección «log msg») |
| `qt=author` | busca en el campo del autor (nombre o dirección de correo electrónico) |
| `qt=committer` | busca en el campo del committer, es decir, la persona que incorporó el commit |
| `qt=range` | muestra un rango en notación Git, por ejemplo `q=v7.3-rc6..v7.3-rc7` para todos los commits entre dos etiquetas |
| `q=` | término de búsqueda o rango |
| `h=` | rama a la que se refiere la búsqueda, por ejemplo `h=linux-7.3.y` en el árbol Stable |
| `ofs=` | desplazamiento para la paginación (git.kernel.org muestra 200 commits por página) |

</details>

El rango `v7.3-rc6..v7.3-rc7` responde directamente en el navegador a la pregunta «¿Qué ha cambiado entre dos versiones?». En el árbol Stable, `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` muestra todas las correcciones añadidas con una actualización Stable.

## Leer un commit

Un único commit puede direccionarse mediante su hash SHA-1 completo:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

La vista muestra autor, committer, fecha, el commit padre (`parent`), el mensaje completo del commit y el diff. Al final de un mensaje de commit del kernel se encuentran trailers como `Signed-off-by:`, `Reviewed-by:` y `Fixes: <sha> ("Titel")`. La línea `Fixes:` indica el commit que introdujo el error; así se puede delimitar desde qué versión un sistema está afectado. `Link:` suele remitir a la discusión correspondiente en lore.kernel.org, y `Cc: stable@vger.kernel.org` marca la corrección para retroportarla a los kernels Stable.

Si se sustituye `commit` en la URL por `patch`, el servidor entrega el commit como archivo de parche en formato de correo (`text/plain`), que puede aplicarse con `git am` o `patch`.

## URL cortas en entradas CVE

Desde que el propio proyecto del kernel asigna números CVE, las entradas de la lista CVE y de la NVD remiten a URL cortas con la forma `https://git.kernel.org/stable/c/<sha>`. git.kernel.org las redirige mediante HTTP 301 a la vista del commit en el árbol correspondiente:

| URL corta | Destino |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit en el árbol Stable `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit en Mainline `torvalds/linux.git` |

Por regla general, una entrada CVE incluye varios enlaces de este tipo: uno por cada rama Stable a la que se ha retroportado la corrección y uno para Mainline. Los hashes difieren porque cada retroportación es un commit independiente. El mensaje de commit de la retroportación indica el hash del original en Mainline, bien como línea `commit <sha> upstream.` o como `[ Upstream commit <sha> ]`.

## Comprobar si una corrección está incluida en una versión

La interfaz web no muestra en qué etiquetas está incluido un commit. Para ello se necesita un clon local. Para este fin basta con un clon sin contenidos de archivos, que solo descarga el historial de commits:

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `clone` | crea una copia local del repositorio |
| `--filter=blob:none` | Clon parcial: descarga commits y estructuras de directorios; los contenidos de archivos (blobs), solo cuando se necesitan |
| `--no-checkout` | no crea un directorio de trabajo; no es necesario para consultar el historial |
| URL | URL de clonación HTTPS de la vista `summary` del árbol Stable |

</details>

A continuación, un comando en el directorio clonado responde a la pregunta sobre la primera versión con la corrección:

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `describe` | describe un commit en relación con una etiqueta |
| `--contains` | busca la etiqueta más antigua que contiene el commit, en lugar de la etiqueta más reciente anterior |
| `<sha>` | hash del commit, por ejemplo de la URL corta de la entrada CVE |

</details>

La salida tiene la forma `v7.3.2~14`: la corrección está incluida desde `v7.3.2`. `git tag --contains <sha>` enumera todas las etiquetas con el commit. Quien solo necesite un único archivo o un estado para consultar puede usar en el navegador las vistas `tree` y `plain` en lugar de un clon.

Para el código fuente completo destinado a compilación basta con un clon superficial de una etiqueta:

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--depth 1` | descarga solo el último commit sin historial; la descarga se reduce de varios GB a unos cientos de MB |
| `--branch v7.3` | selecciona la etiqueta o rama que se extrae |
| URL | URL de clonación de Mainline |

</details>

## Seguir los cambios

Cada repositorio ofrece un feed Atom de la rama predeterminada en `…/<repository>.git/atom/`, y para Mainline es `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. Un lector de feeds muestra así los nuevos commits sin tener que visitar la página con regularidad. Para Mainline, el feed genera muchas entradas al día durante la ventana de integración; para versiones y actualizaciones Stable, la vista general de versiones en [kernel.org](https://kernel.org/) o su feed es la fuente más tranquila.

## Fuentes

1.  [git.kernel.org](https://git.kernel.org/): Lista de repositorios con grupos y columnas Owner e Idle; campo de búsqueda de la página de inicio.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/): Mainline con vistas summary, log, tree, URL de clonación y feed Atom.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/): Kernels Stable y Longterm, una rama por cada serie de versiones.

4.  [cgit](https://git.zx2c4.com/cgit/about/): Upstream de la interfaz web, documentación de los parámetros de URL.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html): Reglas para retroportaciones, trailer `Cc: stable` y referencia al commit upstream.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html): Asignación de CVE por el proyecto del kernel y estructura de las entradas.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html): Significado de los trailers `Fixes:`, `Link:`, `Signed-off-by:`.

8.  [git-describe](https://git-scm.com/docs/git-describe): Página de manual, opción `--contains`.

9.  [git-clone](https://git-scm.com/docs/git-clone): Página de manual, opciones `--filter`, `--depth`, `--branch`, `--no-checkout`.

10.  [lore.kernel.org](https://lore.kernel.org/): Archivo de las listas de correo del kernel, al que remiten los trailers `Link:`.
