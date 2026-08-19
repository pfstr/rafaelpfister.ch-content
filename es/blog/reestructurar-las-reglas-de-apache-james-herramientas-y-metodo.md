---
title: "Reestructurar las reglas de Apache James: herramientas y método"
navTitle: "Reestructurar las reglas"
description: "Las reglas de Mailet que han crecido con los años contienen rutas muertas que ya nadie reconoce. Cómo evaluar las reglas como un grafo, encontrar código inalcanzable de forma fiable y diseñar la reestructuración para que un único Mailet mantenga abierta la vuelta atrás."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min de lectura"
themen:
  - totemomail
  - e-mail-verschluesselung
  - smtp-mailflow
hauptthema: "totemomail"
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
  - "verschluesselung"
related:
  - totemomail-m365
  - totemomail-licensed-user-limit-ldap-cleanup
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "reestructurar-las-reglas-de-apache-james-herramientas-y-metodo"
translationId: "article-b9c98459a0ff6352"
draft: false
translationOf: apache-james-ruleset-strukturiert-neu-aufbauen
url: https://rafaelpfister.ch/es/blog/reestructurar-las-reglas-de-apache-james-herramientas-y-metodo
translationSourceHash: b0274af954ad40614bc74b37b7be1e6e9bee6c856e28105336eddfb967895884
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:10:25.231Z
translationReview: automatic
---

# Reestructurar las reglas de Apache James: herramientas y método

Las pasarelas de correo basadas en Apache James, entre ellas Totemomail, controlan todo su flujo de mensajes mediante reglas en XML. Tras algunos años de funcionamiento, estas reglas adquieren una característica que apenas nadie advierte: una parte considerable nunca se ejecuta. Se añadieron reglas, se colocaron desvíos antes de ellas, ramas quedaron sin salida y, como nada se rompió, todo permaneció ahí.

El problema no es el espacio en disco. Es que nadie puede ya decir qué regla se aplica realmente. Quien planifica un cambio lee un archivo con cientos de Mailets y no sabe cuáles son siquiera relevantes. Precisamente eso puede responderse de forma mecánica.

Este artículo describe el método y las herramientas para ello: evaluar las reglas como un grafo dirigido, encontrar código inalcanzable de forma fiable y plantear la reestructuración de modo que un único Mailet mantenga abierta la vuelta atrás.

## El modelo en cuatro frases

Un conjunto de reglas consta de **procesadores**, es decir, cadenas con nombre. Cada cadena contiene **Mailets** que realizan una acción, y cada Mailet tiene un **Matcher** que decide si se aplica al mensaje actual. Un Mailet de la clase `ToProcessor` entrega el mensaje a otra cadena.

El punto de entrada suele llamarse `root`. A partir de ahí se ramifica todo lo demás.

```xml
<processor name="root">
   <mailet class="ToProcessor" match="RecipientIs?Recipient(s)=journal@example.com">
      <processor>dropJournal</processor>
   </mailet>
   <mailet class="ToProcessor" match="HostIsLocal">
      <processor>incoming</processor>
   </mailet>
   <mailet class="ToProcessor" match="All">
      <processor>outgoing</processor>
   </mailet>
</processor>
```

Por tanto, la estructura es un grafo dirigido: los procesadores son nodos y los destinos de `ToProcessor` son aristas. En cuanto lo ve así, la cuestión del código muerto se convierte en una tarea estándar: un análisis de accesibilidad.

## Dos tipos de código muerto

Antes de medir, debe saber qué busca. Hay dos formas, y la segunda es la más insidiosa.

**Procesadores inalcanzables.** Cadenas enteras a las que ya no apunta ningún `ToProcessor`. Están en el archivo, pero nunca se accede a ellas. Es el caso evidente.

**Resto muerto dentro de una cadena.** Un `ToProcessor` con `match="All"` se aplica a **todos** los mensajes y los reenvía. Todo lo que queda debajo en la misma cadena nunca se alcanza. Lo mismo ocurre con Mailets con `passThrough=false`: consumen el mensaje y se encargan ellos mismos del tratamiento posterior; los Mailets siguientes ya no lo ven.

Esta segunda forma no se detecta con una simple búsqueda de texto, pues las líneas parecen completamente normales. Para ello necesita el orden dentro de la cadena.

## Herramienta 1: leer el grafo

El punto de partida es una evaluación que extrae los procesadores y sus destinos. El siguiente script utiliza únicamente la biblioteca estándar y se ejecuta en cualquier instalación de Python:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Extraer los procesadores y sus bloques
starts = [(m.start(), m.group(1))
          for m in re.finditer(r'<processor name="([^"]+)">', daten)]

bloecke = {}
for i, (pos, name) in enumerate(starts):
    ende = starts[i + 1][0] if i + 1 < len(starts) else daten.find("</spoolmanager>")
    bloecke[name] = daten[pos:ende]

def ziele(block):
    """Alle ToProcessor-Ziele einer Kette."""
    return re.findall(r"<processor>\s*([^<>]+?)\s*</processor>", block)

for name, block in bloecke.items():
    print(f"{name} -> {', '.join(ziele(block)) or '(kein Ziel)'}")
```

Observe la diferencia entre la **etiqueta de definición** `<processor name="...">` y la **etiqueta de destino** `<processor>name</processor>` dentro de un Mailet `ToProcessor`. Ambas se llaman igual, pero significan cosas distintas. Si las confunde, obtendrá resultados sin sentido. De ahí procede también la trampa que verá más adelante.

## Herramienta 2: accesibilidad desde el punto de entrada

Con el grafo, el análisis consiste en una búsqueda en anchura desde `root`. Todo lo que no se visite está muerto:

```python
erreichbar = set()
stapel = ["root"]

while stapel:
    knoten = stapel.pop()
    if knoten in erreichbar:
        continue
    erreichbar.add(knoten)
    for ziel in ziele(bloecke.get(knoten, "")):
        if ziel not in erreichbar:
            stapel.append(ziel)

def anzahl_mailets(block):
    return len(re.findall(r"<mailet ", block))

tot = [n for n in bloecke if n not in erreichbar]

print(f"Prozessoren gesamt: {len(bloecke)}")
print(f"Erreichbar:         {len(erreichbar)}")
print(f"Tot:                {len(tot)}")

for name in tot:
    print(f"  - {name} ({anzahl_mailets(bloecke[name])} Mailets)")
```

Una salida típica en unas reglas que han crecido con el tiempo:

```text
Prozessoren gesamt: 38
Erreichbar:         18
Tot:                20
  - addExtSender (7 Mailets)
  - decrypt (6 Mailets)
  - externalDelivery (14 Mailets)
  - outgoingProcessExceptionTriggers (12 Mailets)
  ...
```

Veinte de 38 procesadores, con más de 160 Mailets en total, que nunca se ejecutan. No es una excepción, sino lo habitual en un entorno que ha pasado por varias reestructuraciones.

## Herramienta 3: encontrar el resto muerto dentro de las cadenas

Ahora la segunda forma. Recorra cada cadena accesible Mailet por Mailet y marque todo lo que siga al primer desvío incondicional:

```python
def toter_rest(block):
    """Index des ersten Mailets, ab dem nichts mehr erreicht wird."""
    mailets = re.findall(r"<mailet\b.*?(?:/>|</mailet>)", block, re.S)
    for i, m in enumerate(mailets):
        ist_all = 'match="All"' in m
        ist_weiche = 'class="ToProcessor"' in m
        konsumiert = "<passThrough>false</passThrough>" in m
        if ist_all and (ist_weiche or konsumiert):
            return i + 1, len(mailets)
    return None, len(mailets)

for name in sorted(erreichbar):
    ab, gesamt = toter_rest(bloecke[name])
    if ab is not None and ab < gesamt:
        print(f"{name}: Mailets {ab + 1} bis {gesamt} werden nie erreicht")
```

Este hallazgo es más valioso que la lista de procesadores, porque se encuentra en medio de cadenas activas. Quien añade una regla y la inserta debajo de un `ToProcessor match="All"` ha escrito una regla que nunca se aplica y luego se sorprende de que no tenga efecto.

## Herramienta 4: comprobación estructural

Un XML bien formado es solo la mitad del trabajo. Estas cuatro comprobaciones detectan los errores que un parser deja pasar, pero la pasarela no:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Nombres de procesadores duplicados
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Cada Mailet debe ser hijo directo de un procesador y tener class + match
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Debe existir cada destino de ToProcessor
zielnamen = set()
for m in dom.getElementsByTagName("mailet"):
    if m.getAttribute("class") == "ToProcessor":
        for el in m.getElementsByTagName("processor"):
            text = "".join(c.data for c in el.childNodes if c.nodeType == c.TEXT_NODE)
            zielnamen.add(text.strip())

print("Doppelte Namen:      ", doppelt or "keine")
print("Strukturfehler:      ", fehler or "keine")
print("Ziele ohne Definition:", sorted(zielnamen - set(namen)) or "keine")
```

Un `ToProcessor` que apunta a un procesador inexistente es el error clásico tras un cambio de nombre. El XML sigue estando bien formado, pero la pasarela tropieza con ello solo en tiempo de ejecución, normalmente con un mensaje poco útil.

## Una observación: esto es construcción de compiladores, no bricolaje

Lo que está haciendo aquí tiene un nombre y una teoría detrás. Un conjunto de reglas es un **grafo de flujo de control**, el mismo modelo con el que los compiladores analizan programas desde hace décadas. Conviene saberlo porque pone a disposición algoritmos ya establecidos y, más importante aún, afirmaciones claras sobre sus límites.

| Pregunta en las reglas | Modelo | Procedimiento |
|---|---|---|
| ¿Qué procesadores están muertos? | Accesibilidad desde el nodo de entrada | Búsqueda en anchura o profundidad, complejidad `O(V+E)` |
| ¿Qué reglas de una cadena están muertas? | Nodos tras un salto incondicional | la misma búsqueda en un grafo más detallado |
| ¿Dónde puede surgir un bucle de correo? | **Ciclo en el grafo** | componentes fuertemente conexos |
| ¿Dónde debe situarse una regla para que se aplique con certeza? | **Dominador** del nodo de entrada | árbol de dominadores |

Las dos últimas filas son las más valiosas en la práctica. Un bucle de correo no es un fenómeno operativo misterioso, sino un ciclo en el grafo de enrutamiento; el contador de saltos en tiempo de ejecución es solo el freno de emergencia, mientras que estructuralmente puede encontrar el bucle antes. Y si desea colocar una regla por la que **todo** mensaje deba pasar, por ejemplo un filtro para dominios de remitentes no enrutables, entonces debe buscar un dominador. No es una cuestión de gusto, sino algo calculable.

### Encontrar ciclos antes de que se conviertan en bucles de correo

La búsqueda en anchura responde a la pregunta sobre código muerto. Para los bucles necesita la búsqueda en profundidad, pues una **arista de retroceso** revela el ciclo. El procedimiento es el marcado clásico de tres colores:

```python
def zyklen_finden(bloecke, ziele):
    WEISS, GRAU, SCHWARZ = 0, 1, 2
    farbe = {n: WEISS for n in bloecke}
    pfad, gefunden = [], []

    def besuche(knoten):
        farbe[knoten] = GRAU
        pfad.append(knoten)
        for ziel in ziele(bloecke.get(knoten, "")):
            if ziel not in farbe:
                continue
            if farbe[ziel] == GRAU:                 # Rueckwaertskante = Zyklus
                gefunden.append(pfad[pfad.index(ziel):] + [ziel])
            elif farbe[ziel] == WEISS:
                besuche(ziel)
        farbe[knoten] = SCHWARZ
        pfad.pop()

    for knoten in bloecke:
        if farbe[knoten] == WEISS:
            besuche(knoten)
    return gefunden

for zyklus in zyklen_finden(bloecke, ziele):
    print(" -> ".join(zyklus))
```

```text
outgoing -> processOutgoing -> outgoing
```

Un hallazgo así no prueba que exista un bucle, pues las aristas están condicionadas por Matcher y quizá nunca se tomen juntas. Sin embargo, es la lista completa de lugares donde **puede** producirse uno, y eso es exactamente lo que debe conocer antes de una reestructuración. El contador de saltos en tiempo de ejecución es solo el freno de emergencia; aquí ve la construcción.

Igualmente importante es el **límite** del procedimiento. Las aristas están condicionadas por Matcher, y estos dependen del contenido del mensaje. Por tanto, la accesibilidad exacta es indecidible en general; el análisis proporciona una sobreaproximación. De ello se deriva una capacidad informativa asimétrica que debe conocer:

- **«Inalcanzable» es fiable.** Si no conduce ningún camino hasta allí, ningún mensaje puede llegar. Puede eliminar ese código.
- **«Alcanzable» solo significa «no excluido estructuralmente».** El grafo no indica si algún mensaje real llegará a cumplir las condiciones.

Por tanto, el análisis no sustituye a las pruebas, sino que reduce el espacio de pruebas. Aun así, en la práctica es una ganancia enorme: de 38 procesadores quedan 18 que realmente debe comprobar.

No necesita expresamente métodos de aprendizaje automático, como Graph Neural Networks o incrustaciones de nodos. Estos resultan útiles en grafos grandes con estructura desconocida y patrones estadísticos. Un conjunto de reglas tiene unas pocas decenas de nodos, una estructura completamente conocida y semántica determinista. Aquí los algoritmos exactos no solo son más baratos, sino que proporcionan pruebas en lugar de probabilidades.

## Errores habituales en el procesamiento automatizado

Si modifica unas reglas mediante script, hay tres errores que aparecen de forma fiable. Yo mismo he cometido los tres.

**El clásico: el patrón codicioso que cruza límites de procesador.** Quien quiera eliminar un procesador con una expresión regular recurrirá de forma intuitiva a:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Eso es incorrecto. Dentro de la cadena, cada Mailet `ToProcessor` contiene un `<processor>ziel</processor>`, y el `.*?` no codicioso se detiene precisamente ahí. El resultado: se elimina la mitad del procesador, queda un resto de `</mailet>` y `</processor>`, y el XML queda destruido. En su lugar, ancle el patrón a la sangría de la etiqueta de cierre y compruebe el balance de etiquetas frente a:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Finales de línea.** La configuración suele utilizar CRLF. En Python, lea con `rb`, normalice a `\n` para la edición y vuelva a escribir CRLF al final. Si lo olvida, producirá un archivo con finales de línea mezclados que, según el producto, puede rechazarse sin aviso.

**Caracteres especiales.** Mantenga el archivo en ASCII puro y escriba las diéresis como referencias de caracteres (`&#228;` para ä). Así evitará cualquier discusión sobre codificaciones entre el editor, el script y la interfaz web de la pasarela.

Tras cada cambio, compruebe como mínimo que el XML esté bien formado, que los finales de línea no hayan cambiado y que el número de procesadores se mantenga. Tres líneas de control ahorran una reversión.

## El método para la reestructuración: árbol paralelo con un desvío

Pasemos ahora a la reconstrucción propiamente dicha. El camino obvio, reconstruir las reglas existentes paso a paso, es el peor: no puede volver atrás limpiamente y ya no puede leer el estado anterior.

En su lugar, ha demostrado funcionar el árbol paralelo:

**Paso 1: construir el árbol nuevo en paralelo.** Cree los procesadores nuevos con un sufijo de nombre, por ejemplo `rootV2`, `incomingV2`, `outgoingV2`. El árbol antiguo permanece completo y sin cambios.

**Paso 2: un único desvío.** Al inicio del punto de entrada actual se sitúa exactamente un Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Así, todo el tráfico pasa por el árbol nuevo. El antiguo es inalcanzable, pero sigue estando completo. **La vuelta atrás consiste en eliminar estas tres líneas**, y es comprensible en cualquier situación, incluso para una persona que no haya realizado la reestructuración.

**Paso 3: accesibilidad como aceptación.** Ejecute el análisis de la herramienta 2 y compruebe tres puntos: el nuevo punto de entrada se referencia exactamente una vez, todos los procesadores nuevos son accesibles y el árbol antiguo es completamente inalcanzable. Es un criterio objetivo de aceptación en lugar de una inspección visual.

**Paso 4: limpiar solo después de que se haya probado.** Cuando el árbol nuevo se haya confirmado en producción, elimine el antiguo y quite los sufijos. Solo entonces perderá la vuelta atrás en el archivo, y hasta entonces no la habrá necesitado.

Para pasos intermedios que quiera observar, pero aún no activar, son adecuados los Mailets de mera observación: registran información, pero no modifican el enrutamiento. De este modo recopila los datos que faltan para tomar la decisión, sin riesgo.

## Incorporar también la visibilidad

Durante la reconstrucción, conviene tener en cuenta dos aspectos que luego marcan la diferencia en operación.

**Nunca descarte directamente en la cadena principal.** Un Mailet que descarta un mensaje deja en el historial de mensajes únicamente el aviso de que se eliminó, sin indicar el motivo. En su lugar, ramifique hacia un procesador con un nombre específico, por ejemplo `dropNonRoutable`. El nombre por sí solo aparece en el historial y ya indica lo que ocurre.

**No todo registro aparece en el historial de mensajes.** Muchos productos disponen de dos mecanismos: uno para el registro del servidor y otro para el historial que también ve el soporte. Solo el segundo es visible en el historial. Quien configure exclusivamente el primero habrá registrado el evento, pero en el trace seguirá apareciendo únicamente «mensaje eliminado». Formule las entradas del historial en lenguaje claro e indique la regla: «descartado deliberadamente por la regla para dominios de remitentes no enrutables, sin error de entrega» ahorra muchas consultas durante la operación.

## El clúster forma parte de la tarea

Un aspecto que se subestima con regularidad: si la pasarela se ejecuta en varios nodos, la configuración debe almacenarse **de forma idéntica en todos los nodos y persistir tras reinicios**. Si solo está activa en un nodo, el comportamiento dependerá de qué nodo procese el mensaje, y sus pruebas medirán el azar.

Resulta especialmente desagradable que un cambio funcione, pero no se haya persistido. El nodo funcionará correctamente hasta que se reinicie y luego volverá al estado anterior. Por ello, después de cada despliegue compruebe ambas cosas: el mismo estado en todos los nodos y que el estado sobreviva a un reinicio.

## Resumen

Trate las reglas como un grafo, no como un archivo de texto. Una búsqueda en anchura desde el punto de entrada separa en pocas líneas de código lo vivo de lo muerto, y el análisis dentro de las cadenas encuentra además las reglas que están presentes, pero nunca se alcanzan tras un desvío incondicional.

Para la reestructuración propiamente dicha, el árbol paralelo con un único desvío es el método con la mejor relación entre esfuerzo y seguridad. Además, el análisis de accesibilidad le proporciona directamente el criterio de aceptación.

## Fuentes

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): estructura del gestor de spool, procesadores, Mailets y Matcher, así como el orden de procesamiento.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): referencia de los Mailets incluidos, incluido ToProcessor y los parámetros para reenvío y consumo.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): referencia de los Matcher, entre otros All, HostIsLocal y las variantes relacionadas con destinatarios.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): contrato entre Mailet y contenedor, base para comprender el consumo y el reenvío.
