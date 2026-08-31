---
title: "Reestructurar las reglas de Apache James: herramientas y método"
navTitle: "Reestructurar las reglas"
description: "Las reglas Mailet que han crecido con los años contienen rutas muertas que ya nadie reconoce. Cómo evaluar las reglas como un grafo, encontrar código inalcanzable de forma fiable y diseñar la reestructuración para que un único Mailet mantenga abierta la vía de regreso."
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
translationSourceHash: ebcf5bf98f1f74aa7784c74c558da4db240e69f02de722a0251dd832d1224403
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:02:48.679Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/reestructurar-las-reglas-de-apache-james-herramientas-y-metodo
---

# Reestructurar las reglas de Apache James: herramientas y método

Las pasarelas de correo basadas en Apache James, entre ellas Totemomail, controlan todo su flujo de mensajes mediante reglas en XML. Tras varios años de funcionamiento, estas reglas adquieren una característica que apenas nadie advierte: una parte considerable de ellas nunca se ejecuta. Se añadieron reglas, se insertaron desvíos antes de ellas, ramas que no llevaban a ninguna parte y, como nada se rompía, todo permaneció ahí.

El problema no es el espacio en disco. Es que nadie puede ya decir qué regla se aplica realmente. Quien planifica un cambio lee un archivo con cientos de Mailets y no sabe cuáles son siquiera relevantes. Precisamente eso se puede responder mecánicamente.

Este artículo describe el método y las herramientas para ello: evaluar las reglas como un grafo dirigido, encontrar código inalcanzable de forma fiable y diseñar la reestructuración de modo que un único Mailet mantenga abierta la vía de regreso.

## El modelo en cuatro frases

Un conjunto de reglas consta de **procesadores**, es decir, cadenas con nombre. Cada cadena contiene **Mailets** que realizan alguna acción, y cada Mailet tiene un **Matcher** que decide si se aplica al mensaje actual. Un Mailet de la clase `ToProcessor` transfiere el mensaje a otra cadena.

El punto de entrada suele llamarse `root`. Desde ahí se ramifica todo lo demás.

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

Antes de medir, debe saber qué busca. Hay dos formas, y la segunda es la más insidiosa.

**Procesadores inalcanzables.** Cadenas completas a las que ya no apunta ningún `ToProcessor`. Están en el archivo, pero nunca se accede a ellas. Es el caso evidente.

**Resto muerto dentro de una cadena.** Un `ToProcessor` con `match="All"` coincide con **cada** mensaje y lo reenvía. Todo lo que aparece debajo en la misma cadena nunca se alcanza. Lo mismo ocurre con Mailets con `passThrough=false`: consumen el mensaje y asumen por sí mismos el procesamiento posterior; los Mailets siguientes ya no lo ven.

Esta segunda forma no se encuentra con una simple búsqueda de texto, porque las líneas parecen totalmente normales. Para ello necesita el orden dentro de la cadena.

## Herramienta 1: extraer el grafo

El comienzo es una evaluación que extrae los procesadores y sus destinos. El siguiente script utiliza únicamente la biblioteca estándar y se ejecuta en cualquier instalación de Python:

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

Observe la diferencia entre la **etiqueta de definición** `<processor name="...">` y la **etiqueta de destino** `<processor>name</processor>` dentro de un Mailet `ToProcessor`. Ambas se llaman igual, pero significan cosas distintas. Quien las confunda obtendrá resultados sin sentido. Esta es también la causa del error que se explica más abajo.

## Herramienta 2: alcanzabilidad desde el punto de entrada

Con el grafo, el análisis consiste en una búsqueda en anchura desde `root`. Todo lo que no se visite en ella está muerto:

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

Una salida típica en reglas que han crecido con el tiempo:

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

Ahora la segunda forma. Recorra cada cadena alcanzable Mailet a Mailet y marque todo lo que siga a la primera salida incondicional:

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

Este hallazgo es más valioso que la lista de procesadores porque se encuentra en medio de cadenas activas. Quien añade una regla y la inserta por debajo de un `ToProcessor match="All"` ha escrito una regla que nunca se aplicará y después se pregunta por qué no tiene efecto.

## Herramienta 4: validación estructural

XML bien formado por sí solo no basta. Estas cuatro comprobaciones detectan los errores que un analizador deja pasar, pero la pasarela no:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Nombres de procesador duplicados
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

Un `ToProcessor` que apunta a un procesador inexistente es el error clásico tras un cambio de nombre. El XML sigue estando bien formado; la pasarela falla al ejecutarse, normalmente con un mensaje poco útil.

## Contexto: esto es construcción de compiladores

Lo que está haciendo aquí tiene un nombre y una teoría detrás. Un conjunto de reglas es un **grafo de flujo de control**, el mismo modelo con el que los compiladores analizan programas desde hace décadas. Conviene saberlo porque existen algoritmos ya preparados y, más importante aún, afirmaciones claras sobre sus límites.

| Pregunta en las reglas | Modelo | Procedimiento |
|---|---|---|
| ¿Qué procesadores están muertos? | Alcanzabilidad desde el nodo de entrada | Búsqueda en anchura o profundidad, complejidad `O(V+E)` |
| ¿Qué reglas de una cadena están muertas? | Nodos tras un salto incondicional | la misma búsqueda en un grafo más detallado |
| ¿Dónde puede surgir un bucle de correo? | **Ciclo en el grafo** | componentes fuertemente conexos |
| ¿Dónde debe estar una regla para garantizar que se aplique? | **Dominador** del nodo de entrada | árbol de dominadores |

Las dos últimas filas son las más valiosas en la práctica. Un bucle de correo no es un fenómeno operativo misterioso, sino un ciclo en el grafo de enrutamiento; el contador de saltos en tiempo de ejecución es solo el freno de emergencia, mientras que estructuralmente puede encontrar el bucle antes. Y si quiere colocar una regla por la que **deba** pasar cada mensaje, por ejemplo un filtro para dominios de remitentes no enrutables, entonces debe buscar un dominador. No es una cuestión de gustos, sino algo calculable.

### Encontrar ciclos antes de que se conviertan en bucles de correo

La búsqueda en anchura responde a la pregunta sobre el código muerto. Para los bucles necesita la búsqueda en profundidad, porque allí una **arista de retroceso** indica el ciclo. El procedimiento es el marcado clásico de tres colores:

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

Un hallazgo de este tipo no demuestra que exista un bucle, pues las aristas están protegidas por condiciones y quizá nunca se tomen juntas. Sin embargo, es la lista completa de los puntos donde **puede** surgir uno, y eso es precisamente lo que debe conocer antes de una reestructuración. El contador de saltos en tiempo de ejecución es solo el freno de emergencia; aquí ve la construcción.

Igualmente importante es el **límite** del procedimiento. Las aristas están protegidas por Matchers y estos dependen del contenido del mensaje. Por tanto, la alcanzabilidad exacta es indecidible en general; el análisis proporciona una sobreaproximación. De ello se deriva una capacidad informativa asimétrica que debe conocer:

- **«Inalcanzable» es fiable.** Si no conduce ningún camino allí, ningún mensaje puede llegar. Puede eliminar este código.
- **«Alcanzable» solo significa «no excluido estructuralmente».** El grafo no indica si algún mensaje real cumplirá alguna vez las condiciones.

Por tanto, el análisis no sustituye a las pruebas, sino que reduce el espacio de pruebas. Aun así, para la práctica supone una enorme ventaja: de 38 procesadores pasan a ser 18 los que debe revisar.

Aquí no necesita expresamente procedimientos de aprendizaje automático, como Graph Neural Networks o incrustaciones de nodos. Son útiles en grafos grandes con estructura desconocida y patrones estadísticos. Un conjunto de reglas tiene unas pocas docenas de nodos, una estructura completamente conocida y una semántica determinista. Los algoritmos exactos no solo son más baratos aquí, sino que proporcionan pruebas en lugar de probabilidades.

## Errores al procesar automáticamente

Si modifica un conjunto de reglas mediante un script, hay tres errores que aparecen de forma sistemática. Yo mismo he cometido los tres.

**El patrón codicioso que atraviesa los límites de procesador.** Quien quiera eliminar un procesador mediante una expresión regular recurre de forma natural a:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Eso es incorrecto. Dentro de la cadena, cada Mailet `ToProcessor` contiene un `<processor>ziel</processor>`, y el no codicioso `.*?` se detiene exactamente ahí. El resultado: se elimina la mitad del procesador, queda un fragmento formado por `</mailet>` y `</processor>`, y el XML queda destruido. En su lugar, ancle en la sangría de la etiqueta de cierre y compruebe el balance de etiquetas con:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Finales de línea.** La configuración suele usar CRLF. Lea en Python con `rb`, normalice a `\n` para procesar y vuelva a escribir CRLF al final. Quien lo olvida produce un archivo con finales de línea mezclados que, según el producto, se rechaza sin aviso.

**Caracteres especiales.** Mantenga el archivo en ASCII puro y escriba las diéresis como referencias de caracteres (`&#228;` para ä). Así evita cualquier discusión sobre codificaciones entre el editor, el script y la interfaz web de la pasarela.

Después de cada cambio, compruebe como mínimo que esté bien formado, que los finales de línea no hayan cambiado y que el número de procesadores no haya variado. Tres líneas de control ahorran una reversión.

## El método para la reestructuración: árbol paralelo con un desvío

Ahora, la verdadera reconstrucción. El camino más obvio, transformar gradualmente las reglas existentes, es el peor: no puede volver atrás limpiamente y ya no puede leer el estado anterior.

En su lugar, ha demostrado su eficacia el árbol paralelo:

**Paso 1: construir el nuevo árbol al lado.** Cree los nuevos procesadores con un sufijo de nombre, por ejemplo `rootV2`, `incomingV2`, `outgoingV2`. El árbol antiguo permanece completo e inalterado.

**Paso 2: un único desvío.** Al comienzo del punto de entrada existente hay exactamente un Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Así, todo el tráfico pasa por el nuevo árbol. El antiguo es inalcanzable, pero sigue presente por completo. **La vuelta atrás consiste en eliminar estas tres líneas**, y resulta comprensible en cualquier situación, incluso para alguien que no haya realizado la reestructuración.

**Paso 3: alcanzabilidad como aceptación.** Ejecute el análisis de la herramienta 2 y compruebe tres puntos: el nuevo punto de entrada se referencia exactamente una vez, todos los procesadores nuevos son alcanzables y el árbol antiguo es completamente inalcanzable. Es un criterio de aceptación objetivo en lugar de una inspección visual.

**Paso 4: limpiar solo después de comprobarlo.** Cuando el nuevo árbol esté confirmado en producción, elimine el antiguo y retire los sufijos. Solo entonces perderá la vía de regreso en el archivo, y hasta ese momento no la habrá necesitado.

Para pasos intermedios que quiera observar pero aún no activar, son adecuados los Mailets de mera observación: registran datos, pero no modifican el enrutamiento. Así recopila los datos necesarios para la decisión sin riesgo.

## Incorporar también la visibilidad

Al reconstruir, conviene tener en cuenta dos aspectos que después marcan la diferencia en producción.

**No descarte nunca directamente en la cadena principal.** Un Mailet que descarta un mensaje solo deja en el historial de mensajes el aviso de que se ha eliminado, sin motivo. En su lugar, ramifique a un procesador con un nombre específico, por ejemplo `dropNonRoutable`. El nombre ya aparece en el historial e indica lo que sucede.

**No todos los registros aparecen en el historial del mensaje.** Muchos productos conocen dos mecanismos: uno para el registro del servidor y otro para el historial que también ve el soporte. Solo el segundo es visible en el historial. Quien establezca exclusivamente el primero habrá registrado algo, pero en el trace seguirá apareciendo únicamente «mensaje eliminado». Formule las entradas del historial en lenguaje claro e indique la regla: «descartado deliberadamente por la regla para dominios de remitentes no enrutables, sin error de entrega» evita muchas consultas en producción.

## El clúster forma parte de la tarea

Hay un punto que se subestima regularmente: si la pasarela funciona en varios nodos, la configuración debe estar almacenada **de forma idéntica y persistente tras reinicios en todos los nodos**. Si solo está activa en uno, el comportamiento depende de qué nodo procese el mensaje, y sus pruebas medirán el azar.

Especialmente incómodo es el caso en que un cambio funciona, pero no se ha persistido. Entonces el nodo opera correctamente hasta que se reinicia y después vuelve al estado anterior. Por ello, tras cada despliegue compruebe ambas cosas: mismo estado en todos los nodos y que ese estado sobreviva a un reinicio.

## En resumen

Trate las reglas como un grafo, no como un archivo de texto. Una búsqueda en anchura desde el punto de entrada separa en pocas líneas de código lo vivo de lo muerto, y el análisis dentro de las cadenas encuentra además las reglas que están ahí pero que nunca se alcanzan tras una salida incondicional.

Para la propia reestructuración, el árbol paralelo con un único desvío es el método con la mejor relación entre esfuerzo y seguridad. Y el análisis de alcanzabilidad le proporciona también el criterio de aceptación.

## Fuentes

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): estructura del gestor de spool, procesadores, Mailets y Matchers, así como el orden de procesamiento.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): referencia de los Mailets incluidos, incluido ToProcessor y los parámetros de reenvío y consumo.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): referencia de los Matchers, entre ellos All, HostIsLocal y las variantes relacionadas con destinatarios.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): contrato entre Mailet y contenedor, base para comprender el consumo y el reenvío.
