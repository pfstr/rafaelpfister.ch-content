---
title: "Ristrutturare le regole di Apache James: strumenti e metodo"
navTitle: "Ristrutturare le regole"
description: "Dopo anni, le regole Mailet stratificate contengono percorsi morti che nessuno riconosce più. Come analizzare le regole come un grafo, individuare in modo affidabile il codice irraggiungibile e strutturare la riorganizzazione affinché un unico Mailet mantenga aperta la via di ritorno."
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
translationSourceHash: ebcf5bf98f1f74aa7784c74c558da4db240e69f02de722a0251dd832d1224403
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:02:16.992Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/ricostruire-in-modo-strutturato-i-set-di-regole-apache-james-strumenti-e-metodo
---

# Ristrutturare le regole di Apache James: strumenti e metodo

I gateway di posta basati su Apache James, tra cui TotemoMail, controllano l'intero flusso dei messaggi tramite un insieme di regole in XML. Dopo alcuni anni di esercizio, questo insieme di regole acquisisce una caratteristica che quasi nessuno nota: una parte considerevole non viene mai eseguita. Sono state aggiunte regole, sono stati inseriti smistamenti prima di esse, rami hanno portato al nulla e, poiché nulla si rompeva, tutto è rimasto com'era.

Il problema non è lo spazio su disco. È che nessuno può più dire quale regola venga effettivamente applicata. Chi pianifica una modifica legge un file con centinaia di Mailet e non sa quali siano davvero rilevanti. Ed è proprio una domanda a cui si può rispondere meccanicamente.

Questo articolo descrive il metodo e gli strumenti: analizzare l'insieme di regole come un grafo diretto, trovare in modo affidabile il codice irraggiungibile e impostare la riorganizzazione affinché un unico Mailet mantenga aperta la via di ritorno.

## Il modello in quattro frasi

Un insieme di regole è composto da **processori**, ovvero catene denominate. Ogni catena contiene **Mailet** che svolgono un'azione e ogni Mailet dispone di un **Matcher** che decide se si applica al messaggio corrente. Un Mailet della classe `ToProcessor` passa il messaggio a un'altra catena.

Il punto di ingresso si chiama generalmente `root`. Da lì si dirama tutto il resto.

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

La struttura è quindi un grafo diretto: i processori sono nodi, le destinazioni di `ToProcessor` sono archi. E non appena lo si vede in questo modo, la questione del codice morto diventa un compito standard: un'analisi di raggiungibilità.

## Due tipi di codice morto

Prima di misurare, dovete sapere cosa state cercando. Esistono due forme e la seconda è quella insidiosa.

**Processori irraggiungibili.** Intere catene alle quali non punta più alcun `ToProcessor`. Sono presenti nel file, ma non vi si entra mai. È il caso evidente.

**Residuo morto all'interno di una catena.** Un `ToProcessor` con `match="All"` si applica a **qualsiasi** messaggio e lo inoltra. Tutto ciò che si trova sotto nella stessa catena non viene mai raggiunto. Lo stesso vale per i Mailet con `passThrough=false`: consumano il messaggio e ne gestiscono autonomamente l'ulteriore trattamento; i Mailet successivi non lo vedono più.

Questa seconda forma non può essere trovata con una semplice ricerca testuale, poiché le righe appaiono del tutto normali. A tal fine serve l'ordine interno della catena.

## Strumento 1: estrarre il grafo

Il punto di partenza è un'analisi che estrae i processori e le relative destinazioni. Lo script seguente utilizza solo la libreria standard e funziona con ogni installazione Python:

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

Notate la differenza tra il **tag di definizione** `<processor name="...">` e il **tag di destinazione** `<processor>name</processor>` all'interno di un Mailet `ToProcessor`. Entrambi hanno lo stesso nome, ma significano cose diverse. Confonderli produce risultati privi di senso. È proprio questa anche l'origine dell'errore descritto più avanti.

## Strumento 2: raggiungibilità dal punto di ingresso

Con il grafo, l'analisi è una ricerca in ampiezza a partire da `root`. Tutto ciò che non viene visitato è morto:

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

Un output tipico per un insieme di regole stratificato:

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

Venti dei 38 processori, con oltre 160 Mailet complessivi, non vengono mai eseguiti. Non è un'eccezione, ma la norma in un ambiente che ha attraversato più ristrutturazioni.

## Strumento 3: trovare il residuo morto nelle catene

Ora la seconda forma. Esaminate ogni catena raggiungibile, Mailet per Mailet, e contrassegnate tutto ciò che segue il primo salto incondizionato:

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

Questo risultato è più prezioso dell'elenco dei processori, perché si trova nel mezzo di catene attive. Chi aggiunge una regola e la inserisce sotto un `ToProcessor match="All"` ha scritto una regola che non verrà mai applicata e poi si stupisce della sua inefficacia.

## Strumento 4: verifica della struttura

Un XML ben formato da solo non basta. Queste quattro verifiche intercettano gli errori che un parser lascia passare ma che il gateway non accetta:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Nomi dei processori duplicati
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

Un `ToProcessor` che punta a un processore inesistente è l'errore classico dopo una rinomina. L'XML rimane ben formato, ma il gateway fallisce solo in fase di esecuzione, di solito con un messaggio poco utile.

## Inquadramento: è costruzione di compilatori

Ciò che state facendo ha un nome e una teoria alle spalle. Un insieme di regole è un **grafo di controllo del flusso**, ossia lo stesso modello con cui i compilatori analizzano i programmi da decenni. Vale la pena saperlo perché mette a disposizione algoritmi pronti e, cosa più importante, affermazioni chiare sui loro limiti.

| Domanda nell'insieme di regole | Modello | Procedura |
|---|---|---|
| Quali processori sono morti? | Raggiungibilità dal nodo di ingresso | Ricerca in ampiezza o in profondità, complessità `O(V+E)` |
| Quali regole in una catena sono morte? | Nodi dopo un salto incondizionato | la stessa ricerca su un grafo più dettagliato |
| Dove può nascere un mail loop? | **Ciclo nel grafo** | componenti fortemente connesse |
| Dove deve trovarsi una regola affinché venga sicuramente applicata? | **Dominatore** del nodo di ingresso | albero dei dominatori |

Le ultime due righe sono le più preziose sul piano pratico. Un mail loop non è un misterioso fenomeno operativo, bensì un ciclo nel grafo di routing; il contatore degli hop in fase di esecuzione è solo il freno di emergenza, mentre a livello strutturale individuate il ciclo prima. E se volete collocare una regola che **ogni** messaggio deve attraversare, ad esempio un filtro per domini mittente non instradabili, dovete cercare un dominatore. Non è una questione di gusto, ma qualcosa di calcolabile.

### Trovare i cicli prima che diventino mail loop

La ricerca in ampiezza risponde alla domanda sul codice morto. Per i loop serve la ricerca in profondità, poiché una **retrocosta** indica un ciclo. Il metodo è la classica marcatura a tre colori:

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

Un tale rilevamento non prova l'esistenza di un loop, perché gli archi sono protetti da condizioni e forse non vengono mai percorsi insieme. Tuttavia, è l'elenco completo dei punti in cui un loop **può** formarsi, ed è esattamente ciò che dovete conoscere prima di una ristrutturazione. Il contatore degli hop in fase di esecuzione è solo il freno di emergenza; qui vedete la costruzione.

Altrettanto importante è il **limite** del metodo. Gli archi sono controllati da Matcher che dipendono dal contenuto del messaggio. La raggiungibilità esatta è quindi in generale indecidibile; l'analisi fornisce una sovra-approssimazione. Ne deriva un valore informativo asimmetrico che dovete conoscere:

- **«Irraggiungibile» è affidabile.** Se non conduce alcun percorso, nessun messaggio può arrivarci. Potete eliminare questo codice.
- **«Raggiungibile» significa solo «non escluso strutturalmente».** Il grafo non indica se un messaggio reale soddisferà mai le condizioni.

L'analisi non sostituisce quindi il test, ma riduce lo spazio di test. In pratica, rimane comunque un vantaggio enorme: da 38 processori ne restano 18 da verificare.

Non vi servono esplicitamente metodi di apprendimento automatico, come Graph Neural Networks o embedding di nodi. Sono utili per grafi grandi con struttura sconosciuta e pattern statistici. Un insieme di regole ha alcune decine di nodi, una struttura interamente nota e una semantica deterministica. Gli algoritmi esatti non sono solo più economici, ma forniscono prove anziché probabilità.

## Fonti di errore nell'elaborazione automatica

Se modificate un insieme di regole tramite script, vi sono tre errori che si verificano regolarmente. Li ho commessi tutti e tre personalmente.

**Il pattern avido oltre i confini dei processori.** Chi vuole rimuovere un processore con un'espressione regolare ricorre naturalmente a:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

È sbagliato. All'interno della catena, ogni Mailet `ToProcessor` contiene un `<processor>ziel</processor>`, e il `.*?` non avido si ferma esattamente lì. Il risultato: viene rimosso mezzo processore, rimane un residuo di `</mailet>` e `</processor>`, e l'XML è distrutto. Ancorate invece al rientro del tag di chiusura e verificate il bilanciamento dei tag rispetto a:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Fine riga.** La configurazione usa solitamente CRLF. In Python leggete con `rb`, normalizzate in `\n` per l'elaborazione e alla fine riscrivete CRLF. Chi lo dimentica produce un file con terminazioni di riga miste, che a seconda del prodotto viene rifiutato senza alcun messaggio.

**Caratteri speciali.** Mantenete il file in puro ASCII e scrivete gli umlaut come riferimenti a caratteri (`&#228;` per ä). Questo evita ogni discussione sulle codifiche tra editor, script e interfaccia web del gateway.

Dopo ogni modifica, verificate almeno la corretta formattazione, le terminazioni di riga invariate e il numero invariato di processori. Tre righe di controllo risparmiano un ripristino.

## Il metodo per la ristrutturazione: albero parallelo con uno scambio

Ora passiamo alla ricostruzione vera e propria. Il percorso più ovvio, modificare passo dopo passo l'insieme di regole esistente, è il peggiore: non potete tornare indietro in modo pulito e non potete più leggere lo stato precedente.

Si è invece dimostrato efficace l'albero parallelo:

**Passo 1: costruire accanto il nuovo albero.** Create i nuovi processori con un suffisso nel nome, ad esempio `rootV2`, `incomingV2`, `outgoingV2`. Il vecchio albero rimane completamente presente e invariato.

**Passo 2: un unico scambio.** All'inizio del punto di ingresso esistente si trova esattamente un Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

In questo modo tutto il traffico passa attraverso il nuovo albero. Quello vecchio è irraggiungibile, ma rimane completamente presente. **La via di ritorno consiste nel rimuovere queste tre righe**, ed è comprensibile in qualsiasi situazione, anche per una persona che non ha eseguito la ristrutturazione.

**Passo 3: raggiungibilità come collaudo.** Eseguite l'analisi dello strumento 2 e verificate tre punti: il nuovo punto di ingresso viene referenziato esattamente una volta, tutti i nuovi processori sono raggiungibili e il vecchio albero è completamente irraggiungibile. È un criterio di collaudo oggettivo anziché un'ispezione visiva.

**Passo 4: riordinare solo dopo il collaudo.** Quando il nuovo albero è confermato in esercizio, rimuovete quello vecchio ed eliminate i suffissi. Solo allora perdete la via di ritorno nel file, e fino a quel momento non ne avete avuto bisogno.

Per passaggi intermedi che volete osservare ma non ancora attivare in modo vincolante, sono adatti Mailet di sola osservazione: registrano, ma non modificano il routing. In questo modo raccogliete i dati mancanti per la decisione, senza rischi.

## Integrare fin da subito la visibilità

Durante la ricostruzione conviene considerare due aspetti che faranno la differenza in esercizio.

**Non scartate mai direttamente nella catena principale.** Un Mailet che scarta un messaggio lascia nella cronologia dei messaggi solo l'indicazione che è stato eliminato, senza motivo. Diramate invece verso un processore appositamente denominato, ad esempio `dropNonRoutable`. Il solo nome compare nella cronologia e indica già cosa è successo.

**Non tutta la registrazione finisce nella cronologia del messaggio.** Molti prodotti prevedono due meccanismi: uno per il log del server e uno per la cronologia visibile anche al supporto. Solo il secondo è visibile nella cronologia. Chi imposta esclusivamente il primo ha sì registrato l'evento, ma nel trace continua a comparire soltanto «Messaggio eliminato». Formulate le voci della cronologia in linguaggio chiaro e nominate la regola: «scartato deliberatamente dalla regola per domini mittente non instradabili, nessun errore di consegna» evita moltissime richieste di chiarimento in esercizio.

## Il cluster è parte del compito

Un punto regolarmente sottovalutato: se il gateway gira su più nodi, la configurazione deve essere salvata **in modo identico e persistente dopo il riavvio su tutti i nodi**. Se è attiva solo su un nodo, il comportamento dipende da quale nodo elabora il messaggio e i vostri test misurano il caso.

Particolarmente spiacevole è il caso in cui una modifica funzioni ma non sia stata resa persistente. Il nodo lavora quindi correttamente fino al riavvio, per poi tornare allo stato precedente. Dopo ogni deployment, verificate perciò entrambe le cose: stessa versione su tutti i nodi e persistenza della versione dopo un riavvio.

## In sintesi

Trattate l'insieme di regole come un grafo, non come un file di testo. Una ricerca in ampiezza dal punto di ingresso separa in poche righe di codice ciò che è vivo da ciò che è morto, e l'analisi all'interno delle catene individua inoltre le regole che sono presenti ma non vengono mai raggiunte dopo un salto incondizionato.

Per la ristrutturazione stessa, l'albero parallelo con un unico scambio è il metodo con il miglior rapporto tra impegno e sicurezza. E l'analisi di raggiungibilità fornisce anche il relativo criterio di collaudo.

## Fonti

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): struttura dello spool manager, processori, Mailet e Matcher nonché ordine di elaborazione.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): riferimento dei Mailet forniti, incluso ToProcessor e i parametri per inoltro e consumo.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): riferimento dei Matcher, tra cui All, HostIsLocal e le varianti basate sul destinatario.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): contratto tra Mailet e container, base per comprendere consumo e inoltro.
