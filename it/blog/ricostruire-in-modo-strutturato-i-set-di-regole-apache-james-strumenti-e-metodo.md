---
title: "Ricostruire in modo strutturato i set di regole Apache James: strumenti e metodo"
navTitle: "Ricostruire il set di regole"
description: "Dopo anni, i set di regole Mailet evoluti contengono percorsi morti che nessuno riconosce più. Come analizzare il set di regole come un grafo, individuare in modo affidabile il codice irraggiungibile e strutturare la riorganizzazione affinché un unico Mailet mantenga aperta la via di ritorno."
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
url: https://rafaelpfister.ch/it/blog/ricostruire-in-modo-strutturato-i-set-di-regole-apache-james-strumenti-e-metodo
translationSourceHash: b0274af954ad40614bc74b37b7be1e6e9bee6c856e28105336eddfb967895884
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:09:53.615Z
translationReview: automatic
---

# Ricostruire in modo strutturato i set di regole Apache James: strumenti e metodo

I gateway di posta basati su Apache James, tra cui Totemomail, controllano l'intero flusso dei messaggi tramite un set di regole XML. Dopo alcuni anni di esercizio, questo set di regole acquisisce una caratteristica che quasi nessuno nota: una parte considerevole non viene mai eseguita. Sono state aggiunte regole, sono stati inseriti instradamenti a monte, rami sono finiti nel vuoto e, poiché nulla si rompeva, tutto è rimasto così.

Il problema non è lo spazio su disco. È che nessuno riesce più a dire quale regola venga effettivamente applicata. Chi pianifica una modifica legge un file con centinaia di Mailet e non sa quali siano davvero rilevanti. Ed è proprio una domanda a cui si può rispondere meccanicamente.

Questo articolo descrive il metodo e gli strumenti necessari: analizzare il set di regole come un grafo diretto, individuare in modo affidabile il codice irraggiungibile e impostare la riorganizzazione affinché un unico Mailet mantenga aperta la via di ritorno.

## Il modello in quattro frasi

Un set di regole è composto da **processori**, ovvero catene denominate. Ogni catena contiene **Mailet** che eseguono un'azione e ogni Mailet ha un **Matcher** che decide se si applica al messaggio corrente. Un Mailet della classe `ToProcessor` trasferisce il messaggio a un'altra catena.

Il punto di ingresso si chiama di solito `root`. Da lì si dirama tutto il resto.

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

La struttura è quindi un grafo diretto: i processori sono nodi, le destinazioni di `ToProcessor` sono archi. E non appena la si vede in questo modo, la domanda sul codice morto diventa un compito standard: un'analisi di raggiungibilità.

## Due tipi di codice morto

Prima di misurare, occorre sapere cosa cercare. Esistono due forme, e la seconda è quella insidiosa.

**Processori irraggiungibili.** Intere catene alle quali non punta più alcun `ToProcessor`. Rimangono nel file, ma non vi si entra mai. È il caso evidente.

**Resto morto all'interno di una catena.** Un `ToProcessor` con `match="All"` si applica a **ogni** messaggio e lo inoltra. Tutto ciò che segue nella stessa catena non viene mai raggiunto. Lo stesso vale per i Mailet con `passThrough=false`: consumano il messaggio e gestiscono autonomamente l'ulteriore elaborazione; i Mailet successivi non lo vedono più.

Questa seconda forma non viene individuata da una semplice ricerca testuale, poiché le righe appaiono del tutto normali. A questo scopo serve l'ordine all'interno della catena.

## Strumento 1: leggere il grafo

Il punto di partenza è un'analisi che estrae i processori e le loro destinazioni. Lo script seguente usa solo la libreria standard e funziona con qualsiasi installazione Python:

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

Notate la differenza tra il **tag di definizione** `<processor name="...">` e il **tag di destinazione** `<processor>name</processor>` all'interno di un Mailet `ToProcessor`. Entrambi hanno lo stesso nome, ma significano cose diverse. Chi li confonde ottiene risultati privi di senso. È proprio su questo che si basa anche l'insidia illustrata più avanti.

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

Un output tipico per un set di regole evoluto:

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

Venti processori su 38, con oltre 160 Mailet complessivi, che non vengono mai eseguiti. Non è un'eccezione, ma il caso normale in un ambiente che ha già attraversato più riorganizzazioni.

## Strumento 3: individuare il resto morto all'interno delle catene

Ora la seconda forma. Esaminate ogni catena raggiungibile Mailet per Mailet e contrassegnate tutto ciò che segue la prima uscita incondizionata:

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

Questo risultato è più prezioso della lista dei processori, perché si trova nel mezzo delle catene attive. Chi aggiunge una regola e la inserisce sotto un `ToProcessor match="All"` ha scritto una regola che non verrà mai applicata e in seguito si stupisce della sua inefficacia.

## Strumento 4: verifica strutturale

Un XML ben formato è solo metà del lavoro. Queste quattro verifiche intercettano gli errori che un parser lascia passare, ma che il gateway non accetta:

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

Un `ToProcessor` che punta a un processore inesistente è l'errore classico dopo una rinomina. L'XML rimane ben formato, il gateway inciampa soltanto in fase di esecuzione e di solito con un messaggio poco utile.

## Un inciso: è costruzione di compilatori, non bricolage

Ciò che state facendo qui ha un nome e una teoria alle spalle. Un set di regole è un **grafo di controllo del flusso**, ovvero lo stesso modello con cui i compilatori analizzano programmi da decenni. Vale la pena saperlo, perché sono disponibili algoritmi già pronti e, cosa più importante, affermazioni chiare sui loro limiti.

| Domanda nel set di regole | Modello | Metodo |
|---|---|---|
| Quali processori sono morti? | Raggiungibilità dal nodo di ingresso | Ricerca in ampiezza o in profondità, complessità `O(V+E)` |
| Quali regole in una catena sono morte? | Nodi dopo un salto incondizionato | la stessa ricerca su un grafo più fine |
| Dove può nascere un mail loop? | **Ciclo nel grafo** | componenti fortemente connesse |
| Dove deve trovarsi una regola affinché venga sicuramente applicata? | **Dominatore** del nodo di ingresso | albero dei dominatori |

Le ultime due righe sono le più preziose nella pratica. Un mail loop non è un misterioso fenomeno operativo, bensì un ciclo nel grafo di routing; il contatore degli hop in fase di esecuzione è solo il freno di emergenza, mentre strutturalmente potete individuare il loop prima. E se volete collocare una regola attraverso la quale **deve** passare ogni messaggio, ad esempio un filtro per domini mittente non instradabili, allora dovete cercare un dominatore. Non è una questione di gusto, ma è calcolabile.

### Individuare i cicli prima che diventino mail loop

La ricerca in ampiezza risponde alla domanda sul codice morto. Per i loop serve la ricerca in profondità, perché una **back edge** rivela un ciclo. Il metodo è la classica marcatura a tre colori:

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

Un risultato di questo tipo non è la prova di un loop, perché gli archi sono protetti da Matcher e forse non verranno mai percorsi insieme. Tuttavia è l'elenco completo dei punti in cui un loop **può** formarsi, ed è esattamente ciò che dovete conoscere prima di una riorganizzazione. Il contatore degli hop in fase di esecuzione è solo il freno di emergenza; qui vedete la costruzione.

Altrettanto importante è il **limite** del metodo. Gli archi sono protetti da Matcher, che dipendono dal contenuto del messaggio. La raggiungibilità esatta è quindi in generale indecidibile; l'analisi fornisce una sovra-approssimazione. Ne deriva un valore informativo asimmetrico che dovete conoscere:

- **«Irraggiungibile» è affidabile.** Se non vi conduce alcun percorso, nessun messaggio può arrivarci. Potete eliminare questo codice.
- **«Raggiungibile» significa solo «non escluso strutturalmente».** Il grafo non dice se un messaggio reale soddisferà mai le condizioni.

L'analisi non sostituisce quindi il test, ma riduce lo spazio di test. Nella pratica è comunque un enorme vantaggio: da 38 processori ne rimangono 18 che dovete effettivamente verificare.

Non vi servono espressamente metodi di apprendimento automatico, come Graph Neural Networks o embedding dei nodi. Questi sono utili per grafi grandi con struttura sconosciuta e pattern statistici. Un set di regole ha alcune decine di nodi, una struttura completamente nota e una semantica deterministica. Qui gli algoritmi esatti non sono solo più economici: forniscono prove anziché probabilità.

## Insidie nell'elaborazione automatizzata

Quando modificate un set di regole tramite script, ci sono tre errori che si verificano con affidabilità. Li ho commessi tutti e tre personalmente.

**Il classico: il pattern greedy oltre i confini dei processori.** Chi vuole rimuovere un processore tramite un'espressione regolare ricorre naturalmente a:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

È sbagliato. All'interno della catena, ogni Mailet `ToProcessor` contiene un `<processor>ziel</processor>`, e il non-greedy `.*?` si ferma esattamente lì. Il risultato: viene rimosso mezzo processore, rimane un residuo di `</mailet>` e `</processor>`, e l'XML è distrutto. Ancorate invece al rientro del tag di chiusura e controllate il bilanciamento dei tag con:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Fine riga.** La configurazione usa di norma CRLF. In Python leggete con `rb`, normalizzate a `\n` per l'elaborazione e alla fine riscrivete nuovamente in CRLF. Chi lo dimentica produce un file con terminatori di riga misti, che a seconda del prodotto viene rifiutato senza commenti.

**Caratteri speciali.** Mantenete il file in puro ASCII e scrivete le umlaut come riferimenti a carattere (`&#228;` per ä). Questo vi risparmia qualsiasi discussione sulle codifiche tra editor, script e interfaccia web del gateway.

Dopo ogni modifica, controllate almeno la buona formatura, l'assenza di modifiche ai terminatori di riga e il numero invariato di processori. Tre righe di controllo risparmiano un rollback.

## Il metodo per la riorganizzazione: albero parallelo con uno switch

Passiamo ora alla vera ricostruzione. La strada più ovvia, ovvero trasformare gradualmente il set di regole esistente, è la peggiore: non potete tornare indietro in modo pulito e non potete più leggere lo stato precedente.

Si è invece dimostrato efficace l'albero parallelo:

**Passo 1: costruire accanto il nuovo albero.** Create i nuovi processori con un suffisso nel nome, ad esempio `rootV2`, `incomingV2`, `outgoingV2`. Il vecchio albero rimane completo e invariato.

**Passo 2: un unico switch.** All'inizio del punto di ingresso esistente si trova esattamente un Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

In questo modo tutto il traffico passa attraverso il nuovo albero. Quello vecchio è irraggiungibile, ma rimane completamente presente. **La via di ritorno consiste nel rimuovere queste tre righe**, ed è comprensibile in ogni situazione, anche per una persona che non ha effettuato la riorganizzazione.

**Passo 3: raggiungibilità come collaudo.** Eseguite l'analisi dello strumento 2 e verificate tre punti: il nuovo punto di ingresso viene referenziato esattamente una volta, tutti i nuovi processori sono raggiungibili e il vecchio albero è completamente irraggiungibile. È un criterio di collaudo oggettivo invece di un controllo visivo.

**Passo 4: riordinare solo dopo aver dato prova di sé.** Quando il nuovo albero è confermato in esercizio, rimuovete quello vecchio ed eliminate i suffissi. Solo allora perdete la via di ritorno nel file, e fino a quel momento non ne avete avuto bisogno.

Per i passaggi intermedi che volete osservare ma non ancora attivare, sono adatti Mailet di sola osservazione: registrano, ma non modificano il routing. In questo modo raccogliete senza rischi i dati mancanti per la decisione.

## Integrare fin da subito la visibilità

Durante la ricostruzione, vale la pena considerare due aspetti che faranno la differenza in esercizio.

**Non scartate mai direttamente nella catena principale.** Un Mailet che scarta un messaggio lascia nella cronologia del messaggio solo l'indicazione che è stato eliminato, senza motivo. Diramate invece verso un processore appositamente denominato, ad esempio `dropNonRoutable`. Già il solo nome compare nella cronologia e dice cosa è successo.

**Non ogni registrazione finisce nella cronologia del messaggio.** Molti prodotti conoscono due meccanismi: uno per il log del server e uno per la cronologia, che vede anche il supporto. Solo il secondo è visibile nella cronologia. Chi imposta esclusivamente il primo ha sì registrato l'evento, ma nel trace continua a comparire soltanto «Messaggio eliminato». Formulate le voci della cronologia in linguaggio chiaro e nominate la regola: «scartato deliberatamente dalla regola per domini mittente non instradabili, nessun errore di consegna» evita moltissime richieste di chiarimento durante l'esercizio.

## Il cluster fa parte del compito

Un punto regolarmente sottovalutato: se il gateway gira su più nodi, la configurazione deve essere memorizzata **in modo identico e persistente dopo il riavvio su tutti i nodi**. Se è attiva solo su un nodo, il comportamento dipende dal nodo che elabora il messaggio e i vostri test misurano il caso.

Particolarmente sgradevole è il caso in cui una modifica funziona, ma non è stata resa persistente. Il nodo lavora correttamente fino al riavvio, poi torna allo stato precedente. Dopo ogni deployment, verificate quindi entrambi gli aspetti: stesso stato su tutti i nodi e sopravvivenza dello stato a un riavvio.

## In sintesi

Trattate il set di regole come un grafo, non come un file di testo. Una ricerca in ampiezza dal punto di ingresso separa in poche righe di codice il vivo dal morto, e l'analisi all'interno delle catene individua inoltre le regole che sono presenti ma non vengono mai raggiunte dopo un'uscita incondizionata.

Per la riorganizzazione stessa, l'albero parallelo con un unico switch è il metodo con il miglior rapporto tra impegno e sicurezza. E l'analisi di raggiungibilità fornisce al contempo il relativo criterio di collaudo.

## Fonti

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): struttura dello spool manager, processori, Mailet e Matcher, nonché ordine di elaborazione.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): riferimento ai Mailet inclusi, compresi ToProcessor e i parametri per inoltro e consumo.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): riferimento ai Matcher, tra cui All, HostIsLocal e le varianti relative ai destinatari.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): contratto tra Mailet e container, base per comprendere consumo e inoltro.
