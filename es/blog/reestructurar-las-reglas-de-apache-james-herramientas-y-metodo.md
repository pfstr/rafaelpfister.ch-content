---
title: "Reestructurar las reglas de Apache James: herramientas y método"
navTitle: "Reestructurar las reglas"
description: "Las reglas de Mailets que han crecido con los años contienen rutas muertas que ya nadie reconoce. Cómo analizar las reglas como un grafo, encontrar código inalcanzable de forma fiable y diseñar la reestructuración de modo que un único Mailet mantenga abierta la vuelta atrás."
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
translationSourceHash: acf116bf560f396b8a3f679fd2ca62a600e127ccf6cd7522c4a8534bb9876c0e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:36:05.482Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/reestructurar-las-reglas-de-apache-james-herramientas-y-metodo
---

# Reestructurar las reglas de Apache James: herramientas y método

Las pasarelas de correo basadas en Apache James, entre ellas Totemomail, controlan todo su flujo de mensajes mediante unas reglas en XML. Tras algunos años de funcionamiento, estas reglas adquieren una característica que casi nadie advierte: una parte considerable nunca se ejecuta. Se añadieron reglas, se colocaron bifurcaciones delante de ellas, las ramas quedaron sin salida y, como nada se rompió, todo permaneció ahí.

El problema no es el espacio de almacenamiento. Es que nadie puede decir ya qué regla se aplica realmente. Quien planifica un cambio lee un archivo con cientos de Mailets y no sabe cuáles son siquiera relevantes. Precisamente eso puede responderse de forma mecánica.

Este artículo describe el método y las herramientas para ello: analizar las reglas como un grafo dirigido, encontrar código inalcanzable de forma fiable y plantear la reestructuración de manera que un único Mailet mantenga abierta la vuelta atrás.

## El modelo en cuatro frases

Unas reglas se componen de **procesadores**, es decir, cadenas con nombre. Cada cadena contiene **Mailets**, que realizan alguna acción, y cada Mailet tiene un **Matcher** que decide si se aplica al mensaje actual. Un Mailet de la clase `ToProcessor` entrega el mensaje a otra cadena.

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

De este modo, la estructura es un grafo dirigido: los procesadores son nodos y los destinos de `ToProcessor` son aristas. Y en cuanto lo ve así, la cuestión del código muerto pasa a ser una tarea estándar: un análisis de alcanzabilidad.

## Dos tipos de código muerto

Antes de medir, debe saber qué busca. Hay dos formas, y la segunda es la más traicionera.

**Procesadores inalcanzables.** Cadenas completas a las que ya no apunta ningún `ToProcessor`. Están en el archivo, pero nunca se accede a ellas. Es el caso evidente.

**Resto muerto dentro de una cadena.** Un `ToProcessor` con `match="All"` coincide con **todos** los mensajes y los reenvía. Todo lo que aparece debajo en la misma cadena nunca se alcanza. Lo mismo ocurre con Mailets con `passThrough=false`: consumen el mensaje y asumen ellos mismos el tratamiento posterior; los Mailets siguientes no lo ven.

Esta segunda forma no la encuentra una simple búsqueda de texto, pues las líneas parecen totalmente normales. Para ello necesita el orden dentro de la cadena.

## Herramienta 1: extraer el grafo

El punto de partida es un análisis que extrae los procesadores y sus destinos. El siguiente script utiliza únicamente la biblioteca estándar y funciona con cualquier instalación de Python:

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

Observe la diferencia entre la **etiqueta de definición** `<processor name="...">` y la **etiqueta de destino** `<processor>name</processor>` dentro de un Mailet `ToProcessor`. Ambas se llaman igual, pero significan cosas diferentes. Si las confunde, obtendrá resultados absurdos. De ahí procede también la fuente de error explicada más abajo.

## Herramienta 2: alcanzabilidad desde el punto de entrada

Con el grafo, el análisis consiste en una búsqueda en anchura desde `root`. Todo lo que no se visite estará muerto:

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

Ahora, la segunda forma. Recorra cada cadena alcanzable Mailet por Mailet y marque todo lo que siga a la primera salida incondicional:

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

Este hallazgo es más valioso que la lista de procesadores porque se encuentra dentro de cadenas activas. Quien añade una regla y la inserta debajo de un `ToProcessor match="All"` ha escrito una regla que nunca se aplicará y luego se pregunta por qué no tiene efecto.

## Herramienta 4: comprobación estructural

El XML bien formado por sí solo no basta. Estas cuatro comprobaciones detectan los errores que un analizador deja pasar, pero que la pasarela no:

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

# 3. Todo destino ToProcessor debe existir
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

Un `ToProcessor` que apunta a un procesador inexistente es el error clásico tras un cambio de nombre. El XML sigue estando bien formado, pero la pasarela falla al ejecutarse, normalmente con un mensaje poco útil.

## Contexto: esto es construcción de compiladores

Lo que hace aquí tiene un nombre y una teoría detrás. Unas reglas son un **grafo de flujo de control**, es decir, el mismo modelo con el que los compiladores analizan programas desde hace décadas. Conviene saberlo porque pone a su disposición algoritmos ya existentes y, más importante aún, afirmaciones claras sobre sus límites.

| Pregunta en las reglas | Modelo | Procedimiento |
|---|---|---|
| ¿Qué procesadores están muertos? | Alcanzabilidad desde el nodo de entrada | Búsqueda en anchura o profundidad, coste `O(V+E)` |
| ¿Qué reglas de una cadena están muertas? | Nodos después de un salto incondicional | la misma búsqueda en un grafo más detallado |
| ¿Dónde puede surgir un bucle de correo? | **Ciclo en el grafo** | componentes fuertemente conexos |
| ¿Dónde debe estar una regla para que se aplique con garantía? | **Dominador** del nodo de entrada | árbol de dominadores |

Las dos últimas líneas son las más valiosas en la práctica. Un bucle de correo no es un misterioso fenómeno operativo, sino un ciclo en el grafo de enrutamiento; el contador de saltos en tiempo de ejecución es solo el freno de emergencia; estructuralmente, puede encontrar el bucle antes. Y si quiere colocar una regla por la que **debe** pasar cada mensaje, por ejemplo un filtro para dominios de remitentes no enrutable, pregunte por un dominador. No es una cuestión de gusto, sino algo calculable.

### Encontrar ciclos antes de que se conviertan en bucles de correo

La búsqueda en anchura responde a la pregunta sobre el código muerto. Para los bucles necesita la búsqueda en profundidad, pues allí una **arista de retroceso** indica el ciclo. El procedimiento es el clásico marcado de tres colores:

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

Un hallazgo así no prueba que exista un bucle, pues las aristas están protegidas por condiciones y quizá nunca se tomen juntas. Sin embargo, es la lista completa de los lugares donde **puede** surgir uno, y eso es exactamente lo que debe conocer antes de una reestructuración. El contador de saltos en tiempo de ejecución es solo el freno de emergencia; aquí ve la construcción.

Igualmente importante es el **límite** del procedimiento. Las aristas están protegidas por Matchers y dependen del contenido del mensaje. Por tanto, la alcanzabilidad exacta es indecidible en general; el análisis proporciona una sobreaproximación. De ello se deriva una capacidad de afirmación asimétrica que debe conocer:

- **«Inalcanzable» es fiable.** Si no conduce ningún camino hasta allí, ningún mensaje puede llegar. Puede eliminar ese código.
- **«Alcanzable» solo significa «no excluido estructuralmente».** El grafo no indica si algún mensaje real llegará a cumplir las condiciones.

Por tanto, el análisis no sustituye las pruebas, sino que reduce su espacio. Aun así, en la práctica supone una ganancia enorme: de 38 procesadores quedan 18 que debe comprobar.

No necesita expresamente métodos de aprendizaje automático, como Graph Neural Networks o incrustaciones de nodos. Merecen la pena en grafos grandes con estructura desconocida y patrones estadísticos. Unas reglas tienen unas pocas decenas de nodos, una estructura completamente conocida y semántica determinista. Los algoritmos exactos no solo son más económicos aquí: proporcionan pruebas en lugar de probabilidades.

## Fuentes de error al procesar automáticamente

Si modifica unas reglas mediante un script, hay tres errores que ocurren de forma fiable. He cometido los tres yo mismo.

**El patrón codicioso que cruza límites de procesadores.** Quien quiere eliminar un procesador con una expresión regular recurre de forma natural a:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Esto es incorrecto. Dentro de la cadena, cada Mailet `ToProcessor` contiene un `<processor>ziel</processor>`, y el no codicioso `.*?` se detiene precisamente ahí. El resultado: se elimina medio procesador, queda un fragmento de `</mailet>` y `</processor>`, y el XML queda destruido. En su lugar, ancle el patrón en la sangría de la etiqueta de cierre y compruebe el balance de etiquetas frente a:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Finales de línea.** La configuración suele utilizar CRLF. Lea en Python con `rb`, normalice a `\n` para procesar y vuelva a escribir CRLF al final. Quien lo olvida genera un archivo con finales de línea mezclados, que según el producto se rechaza sin aviso.

**Caracteres especiales.** Mantenga el archivo en ASCII puro y escriba las diéresis como referencias de caracteres (`&#228;` para ä). Así evitará cualquier discusión sobre codificaciones entre el editor, el script y la interfaz web de la pasarela.

Después de cada cambio, compruebe como mínimo que el XML esté bien formado, que los finales de línea no hayan cambiado y que el número de procesadores se mantenga. Tres líneas de control ahorran una reversión.

## El método para la reestructuración: árbol paralelo con una bifurcación

Pasemos ahora a la reconstrucción propiamente dicha. El camino obvio de transformar las reglas existentes paso a paso es el peor: no puede volver atrás de forma limpia y ya no puede leer el estado anterior.

En su lugar, ha dado buen resultado el árbol paralelo:

**Paso 1: crear un árbol nuevo al lado.** Cree los nuevos procesadores con un sufijo de nombre, por ejemplo `rootV2`, `incomingV2`, `outgoingV2`. El árbol antiguo permanece completo e inalterado.

**Paso 2: una única bifurcación.** Al principio del punto de entrada existente hay exactamente un Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Con ello, todo el tráfico pasa por el árbol nuevo. El antiguo es inalcanzable, pero sigue estando completo. **La vuelta atrás consiste en eliminar estas tres líneas**, y esto se entiende en cualquier situación, incluso para una persona que no haya realizado la reestructuración.

**Paso 3: alcanzabilidad como aceptación.** Ejecute el análisis de la herramienta 2 y compruebe tres puntos: el nuevo punto de entrada se referencia exactamente una vez, todos los procesadores nuevos son alcanzables y el árbol antiguo es completamente inalcanzable. Es un criterio objetivo de aceptación en lugar de una inspección visual.

**Paso 4: limpiar solo después de que haya demostrado su valía.** Cuando el árbol nuevo esté confirmado en producción, elimine el antiguo y suprima los sufijos. Solo entonces pierde la vuelta atrás en el archivo, y hasta entonces no la ha necesitado.

Para pasos intermedios que quiera observar pero aún no activar plenamente, son adecuados los Mailets de mera observación: registran, pero no modifican el enrutamiento. De este modo recopila los datos que faltan para decidir, sin riesgo.

## Incorporar también la visibilidad

Al reconstruir, conviene tener en cuenta dos aspectos que más tarde marcan la diferencia durante la operación.

**Nunca descarte directamente en la cadena principal.** Un Mailet que descarta un mensaje deja en el historial del mensaje solo la indicación de que se eliminó, sin motivo. Ramifique en su lugar a un procesador con nombre propio, por ejemplo `dropNonRoutable`. El nombre aparece por sí solo en el historial y ya indica lo que sucede.

**No todo registro termina en el historial del mensaje.** Muchos productos conocen dos mecanismos: uno para el registro del servidor y otro para el historial que también ve el soporte. Solo el segundo es visible en el historial. Quien establece exclusivamente el primero habrá registrado algo, pero en el seguimiento seguirá apareciendo solo «mensaje eliminado». Formule las entradas del historial en lenguaje claro e indique la regla: «descartado deliberadamente por la regla para dominios de remitentes no enrutable, sin error de entrega» ahorra muchas consultas durante la operación.

## El clúster forma parte de la tarea

Un punto que se subestima regularmente: si la pasarela se ejecuta en varios nodos, la configuración debe estar almacenada **de forma idéntica en todos los nodos y ser persistente tras reinicios**. Si solo está activa en uno, el comportamiento dependerá de qué nodo procese el mensaje y sus pruebas medirán el azar.

Es especialmente desagradable cuando un cambio funciona, pero no se ha persistido. Entonces el nodo trabaja correctamente hasta que se reinicia y después vuelve al estado anterior. Por ello, tras cada despliegue compruebe ambas cosas: el mismo estado en todos los nodos y que dicho estado sobreviva a un reinicio.

## Resumen

Trate las reglas como un grafo, no como un archivo de texto. Una búsqueda en anchura desde el punto de entrada separa en pocas líneas de código lo vivo de lo muerto, y el análisis dentro de las cadenas encuentra además las reglas que están ahí, pero nunca se alcanzan tras una salida incondicional.

Para la reestructuración en sí, el árbol paralelo con una única bifurcación es el método con la mejor relación entre esfuerzo y seguridad. Y el análisis de alcanzabilidad le proporciona a la vez el criterio de aceptación.

## Historial de cambios

- **25 de octubre de 2026:** enlaces de Fuentes actualizados a las nuevas direcciones de la documentación de Apache James; las referencias a Mailets y Matchers se reúnen ahora en una sola fuente.

## Fuentes

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): estructura del gestor de cola, procesadores, Mailets y Matchers, así como el orden de procesamiento.

2.  [Apache James: Mailets and Matchers Reference](https://james.apache.org/mailet/standard/mailet-report.html): referencia de los Mailets suministrados, incluido ToProcessor, y de los Matchers, entre ellos All, HostIsLocal y RecipientIs.

3.  [Apache James: Mailet API](https://james.apache.org/mailet/api/index.html): descripción de la interfaz de Mailet y de la ejecución en el contenedor de Mailets, base para comprender el consumo y el reenvío.
