---
title: "Ricostruire in modo strutturato le regole Apache James: strumenti e metodo"
navTitle: "Ricostruire le regole"
description: "Le regole Mailet stratificate negli anni contengono percorsi morti che nessuno riconosce più. Come analizzare le regole come un grafo, individuare in modo affidabile il codice irraggiungibile e strutturare la riorganizzazione in modo che un solo Mailet mantenga aperta la via del ritorno."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min di lettura"
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
slug: "ricostruire-in-modo-strutturato-i-set-di-regole-apache-james-strumenti-e-metodo"
translationId: "article-b9c98459a0ff6352"
draft: false
translationOf: apache-james-ruleset-strukturiert-neu-aufbauen
translationSourceHash: acf116bf560f396b8a3f679fd2ca62a600e127ccf6cd7522c4a8534bb9876c0e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:35:37.403Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/ricostruire-in-modo-strutturato-i-set-di-regole-apache-james-strumenti-e-metodo
---

# Ricostruire in modo strutturato le regole Apache James: strumenti e metodo

I gateway di posta basati su Apache James, tra cui Totemomail, gestiscono l'intero flusso dei messaggi tramite regole in XML. Dopo alcuni anni di esercizio, queste regole acquisiscono una caratteristica che quasi nessuno nota: una parte considerevole non viene mai eseguita. Sono state aggiunte regole, inseriti instradamenti prima di esse, rami sono finiti nel vuoto e, poiché nulla si rompeva, tutto è rimasto com'era.

Il problema non è lo spazio di memoria. È che nessuno sa più dire quale regola si applichi effettivamente. Chi pianifica una modifica legge un file con centinaia di Mailet e non sa quali siano davvero rilevanti. Proprio a questa domanda si può rispondere meccanicamente.

Questo articolo descrive il metodo e gli strumenti necessari: analizzare le regole come un grafo diretto, individuare in modo affidabile il codice irraggiungibile e impostare la riorganizzazione affinché un singolo Mailet mantenga aperta la via del ritorno.

## Il modello in quattro frasi

Una configurazione di regole è composta da **processori**, ovvero catene denominate. Ogni catena contiene **Mailet** che eseguono operazioni, e ogni Mailet ha un **Matcher** che decide se si applica al messaggio corrente. Un Mailet della classe `ToProcessor` passa il messaggio a un'altra catena.

Il punto di ingresso si chiama solitamente `root`. Da lì si dirama tutto il resto.

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

La struttura è quindi un grafo diretto: i processori sono nodi, le destinazioni di `ToProcessor` sono archi. E non appena lo si considera in questo modo, la domanda sul codice morto diventa un problema standard, ovvero un'analisi di raggiungibilità.

## Due tipi di codice morto

Prima di misurare, occorre sapere cosa cercare. Esistono due forme, e la seconda è quella insidiosa.

**Processori irraggiungibili.** Intere catene alle quali non punta più alcun `ToProcessor`. Sono nel file, ma non vi si entra mai. È il caso evidente.

**Residuo morto all'interno di una catena.** Un `ToProcessor` con `match="All"` corrisponde a **ogni** messaggio e lo inoltra. Tutto ciò che segue nella stessa catena non viene mai raggiunto. Lo stesso vale per i Mailet con `passThrough=false`: consumano il messaggio e gestiscono autonomamente il trattamento successivo, per cui i Mailet seguenti non lo vedono più.

Una semplice ricerca testuale non trova questa seconda forma, perché le righe sembrano del tutto normali. A questo scopo serve l'ordine all'interno della catena.

## Strumento 1: leggere il grafo

Il punto di partenza è un'analisi che estrae i processori e le loro destinazioni. Il seguente script usa solo la libreria standard e funziona su qualsiasi installazione Python:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Estrarre i processori e i relativi blocchi
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

Notate la differenza tra il **tag di definizione** `<processor name="...">` e il **tag di destinazione** `<processor>name</processor>` all'interno di un Mailet `ToProcessor`. Entrambi hanno lo stesso nome, ma significano cose diverse. Chi li confonde ottiene risultati privi di senso. Proprio da questo deriva anche la fonte di errore descritta più avanti.

## Strumento 2: raggiungibilità dal punto di ingresso

Con il grafo, l'analisi consiste in una ricerca in ampiezza a partire da `root`. Tutto ciò che non viene visitato è morto:

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

Un output tipico per una configurazione di regole stratificata negli anni:

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

Venti dei 38 processori, con oltre 160 Mailet complessivi, non vengono mai eseguiti. Non è un'eccezione, bensì la norma in un ambiente che ha attraversato diverse riorganizzazioni.

## Strumento 3: individuare il residuo morto all'interno delle catene

Ora la seconda forma. Esaminate ogni catena raggiungibile Mailet per Mailet e contrassegnate tutto ciò che segue il primo salto incondizionato:

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

Questo riscontro è più prezioso dell'elenco dei processori, perché si trova in mezzo a catene attive. Chi aggiunge una regola e la inserisce sotto un `ToProcessor match="All"` ha scritto una regola che non si applicherà mai e poi si stupisce della sua inefficacia.

## Strumento 4: verifica della struttura

Il solo XML ben formato non basta. Questi quattro controlli intercettano gli errori che un parser lascia passare, ma che il gateway non accetta:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Nomi di processore duplicati
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Ogni Mailet deve essere figlio diretto di un processore e avere class + match
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Ogni destinazione ToProcessor deve esistere
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

Un `ToProcessor` verso un processore inesistente è l'errore classico dopo una ridenominazione. L'XML resta ben formato, il gateway fallisce solo a runtime e di solito con un messaggio poco utile.

## Inquadramento: è costruzione di compilatori

Ciò che state facendo ha un nome e una teoria alle spalle. Una configurazione di regole è un **grafo di flusso di controllo**, ossia lo stesso modello con cui i compilatori analizzano i programmi da decenni. Vale la pena saperlo perché mette a disposizione algoritmi pronti e, cosa più importante, affermazioni chiare sui loro limiti.

| Domanda nelle regole | Modello | Procedura |
|---|---|---|
| Quali processori sono morti? | Raggiungibilità dal nodo di ingresso | Ricerca in ampiezza o in profondità, complessità `O(V+E)` |
| Quali regole in una catena sono morte? | Nodi dopo un salto incondizionato | la stessa ricerca su un grafo più fine |
| Dove può formarsi un mail loop? | **Ciclo nel grafo** | componenti fortemente connesse |
| Dove deve trovarsi una regola affinché si applichi con certezza? | **Dominatore** del nodo di ingresso | albero dei dominatori |

Le ultime due righe sono le più preziose nella pratica. Un mail loop non è un misterioso fenomeno operativo, ma un ciclo nel grafo di routing; il contatore degli hop a runtime è solo il freno d'emergenza, mentre qui potete individuare il loop prima a livello strutturale. E se volete collocare una regola che **ogni** messaggio deve attraversare, ad esempio un filtro per domini mittente non instradabili, allora dovete cercare un dominatore. Non è una questione di gusti, ma qualcosa di calcolabile.

### Trovare i cicli prima che diventino mail loop

La ricerca in ampiezza risponde alla domanda sul codice morto. Per i loop serve la ricerca in profondità, perché in questo caso un **arco all'indietro** segnala il ciclo. Il metodo è la classica marcatura a tre colori:

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

Un simile riscontro non è una prova di un loop, poiché gli archi sono sorvegliati da condizioni e potrebbero non essere mai percorsi insieme. È però l'elenco completo dei punti in cui un loop **può** formarsi, ed è proprio ciò che dovete conoscere prima di una riorganizzazione. Il contatore degli hop a runtime è solo il freno d'emergenza; qui vedete la costruzione.

Altrettanto importante è il **limite** del metodo. Gli archi sono sorvegliati dai Matcher, che dipendono dal contenuto del messaggio. La raggiungibilità esatta è quindi generalmente indecidibile, e l'analisi fornisce una sovra-approssimazione. Ne consegue una validità asimmetrica delle affermazioni che dovete conoscere:

- **«Irraggiungibile» è affidabile.** Se nessun percorso conduce lì, nessun messaggio può arrivarci. Potete eliminare questo codice.
- **«Raggiungibile» significa solo «non escluso strutturalmente».** Il grafo non dice se un messaggio reale soddisferà mai le condizioni.

L'analisi non sostituisce quindi il test, ma riduce lo spazio dei test. Nella pratica resta comunque un enorme vantaggio: da 38 processori ne rimangono 18 che dovete effettivamente verificare.

Non servono esplicitamente metodi di machine learning, come Graph Neural Networks o embedding dei nodi. Sono utili per grafi grandi con struttura sconosciuta e pattern statistici. Una configurazione di regole ha alcune decine di nodi, una struttura completamente nota e una semantica deterministica. Qui gli algoritmi esatti non solo sono più economici, ma forniscono prove anziché probabilità.

## Fonti di errore nell'elaborazione automatizzata

Se modificate una configurazione di regole mediante script, ci sono tre errori che si verificano puntualmente. Li ho commessi tutti e tre personalmente.

**Il pattern avido oltre i confini dei processori.** Chi vuole rimuovere un processore tramite espressione regolare ricorre naturalmente a:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Questo è sbagliato. All'interno della catena, ogni Mailet `ToProcessor` contiene un `<processor>ziel</processor>`, e il non avido `.*?` si ferma proprio lì. Il risultato: viene rimosso mezzo processore, resta un frammento composto da `</mailet>` e `</processor>`, e l'XML è distrutto. Ancorate invece il pattern all'indentazione del tag di chiusura e verificate il bilanciamento dei tag con:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Fine riga.** La configurazione usa solitamente CRLF. In Python leggete con `rb`, normalizzate a `\n` per l'elaborazione e alla fine riscrivete nuovamente CRLF. Chi lo dimentica produce un file con fine riga misti, che a seconda del prodotto viene rifiutato senza commenti.

**Caratteri speciali.** Mantenete il file in puro ASCII e scrivete le vocali accentate come riferimenti a caratteri (`&#228;` per ä). Questo evita ogni discussione sulle codifiche tra editor, script e interfaccia web del gateway.

Dopo ogni modifica, verificate almeno che il file sia ben formato, che i fine riga non siano cambiati e che il numero di processori sia invariato. Tre righe di controllo risparmiano un rollback.

## Il metodo per la riorganizzazione: albero parallelo con uno scambio

Ora veniamo alla vera ricostruzione. Il percorso più ovvio, trasformare gradualmente la configurazione esistente, è il peggiore: non potete tornare indietro in modo pulito e non potete più leggere lo stato precedente.

Si è invece dimostrato efficace l'albero parallelo:

**Passo 1: creare il nuovo albero accanto al vecchio.** Create i nuovi processori con un suffisso nel nome, ad esempio `rootV2`, `incomingV2`, `outgoingV2`. Il vecchio albero rimane completamente presente e invariato.

**Passo 2: un unico scambio.** All'inizio del precedente punto di ingresso c'è esattamente un Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

In questo modo tutto il traffico passa attraverso il nuovo albero. Il vecchio è irraggiungibile, ma ancora completamente presente. **La via del ritorno consiste nel rimuovere queste tre righe**, ed è comprensibile in ogni situazione, anche per chi non ha effettuato la riorganizzazione.

**Passo 3: raggiungibilità come collaudo.** Eseguite l'analisi dello strumento 2 e controllate tre punti: il nuovo punto di ingresso è referenziato esattamente una volta, tutti i nuovi processori sono raggiungibili e il vecchio albero è completamente irraggiungibile. È un criterio di collaudo oggettivo invece di un controllo visivo.

**Passo 4: riordinare solo dopo la prova sul campo.** Quando il nuovo albero è confermato in esercizio, rimuovete il vecchio ed eliminate i suffissi. Solo allora perdete la via del ritorno nel file, e fino a quel momento non ne avete avuto bisogno.

Per passaggi intermedi che volete osservare senza ancora attivarli definitivamente, sono adatti Mailet di sola osservazione: registrano informazioni, ma non modificano il routing. In questo modo raccogliete i dati mancanti per la decisione senza rischi.

## Integrare subito anche la visibilità

Durante la ricostruzione, conviene considerare due aspetti che faranno la differenza in esercizio.

**Non scartate mai direttamente nella catena principale.** Un Mailet che scarta un messaggio lascia nella cronologia del messaggio solo l'indicazione che è stato eliminato, senza motivo. Diramate invece verso un processore espressamente denominato, ad esempio `dropNonRoutable`. Il solo nome compare nella cronologia e indica già cosa è successo.

**Non tutte le registrazioni finiscono nella cronologia del messaggio.** Molti prodotti conoscono due meccanismi: uno per il log del server e uno per la cronologia visibile anche al supporto. Solo il secondo è visibile nella cronologia. Chi imposta esclusivamente il primo ha sì registrato l'evento, ma nel trace continua ad apparire soltanto «messaggio eliminato». Formulate le voci della cronologia in chiaro e nominate la regola: «scartato intenzionalmente dalla regola per domini mittente non instradabili, nessun errore di consegna» risparmia moltissime richieste di chiarimento in esercizio.

## Il cluster fa parte del compito

Un aspetto regolarmente sottovalutato: se il gateway gira su più nodi, la configurazione deve essere presente **in modo identico e persistente dopo il riavvio su tutti i nodi**. Se è attiva solo su un nodo, il comportamento dipende da quale nodo elabora il messaggio, e i vostri test misurano il caso.

Particolarmente spiacevole è il caso in cui una modifica funziona, ma non è stata resa persistente. Il nodo opera allora correttamente fino al riavvio, per poi tornare allo stato precedente. Dopo ogni deployment, controllate quindi entrambe le cose: stesso stato su tutti i nodi e persistenza dello stato dopo un riavvio.

## In sintesi

Trattate la configurazione di regole come un grafo, non come un file di testo. Una ricerca in ampiezza a partire dal punto di ingresso separa in poche righe di codice ciò che è vivo da ciò che è morto, e l'analisi all'interno delle catene individua inoltre le regole che sono presenti ma non vengono mai raggiunte dopo un salto incondizionato.

Per la riorganizzazione stessa, l'albero parallelo con un unico scambio è il metodo con il miglior rapporto tra impegno e sicurezza. E l'analisi di raggiungibilità vi fornisce al tempo stesso il criterio di collaudo.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornati i link delle Fonti ai nuovi indirizzi della documentazione Apache James; i riferimenti a Mailet e Matcher sono ora riuniti in un'unica fonte.

## Fonti

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): struttura dello Spoolmanager, processori, Mailet e Matcher nonché ordine di elaborazione.

2.  [Apache James: Mailets and Matchers Reference](https://james.apache.org/mailet/standard/mailet-report.html): riferimento dei Mailet forniti, incluso ToProcessor, e dei Matcher, tra cui All, HostIsLocal e RecipientIs.

3.  [Apache James: Mailet API](https://james.apache.org/mailet/api/index.html): descrizione dell'interfaccia Mailet e dell'esecuzione nel contenitore Mailet, base per comprendere consumo e inoltro.
