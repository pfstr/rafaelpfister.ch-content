---
title: "Leggere e capire la bolletta elettrica: voce per voce con una fattura EKZ"
navTitle: "Capire la bolletta elettrica"
description: "Energia, utilizzo della rete, misurazione, tributi: cosa riporta davvero una bolletta elettrica svizzera, chi stabilisce i singoli prezzi e su quali voci si può intervenire, con un esempio di calcolo interattivo sul modello di EKZ."
date: "2026-08-20"
kategorie: "Elettricità ed energia"
timeToRead: "9 min di lettura"
themen:
  - stromtarife-leg
hauptthema: "stromtarife-leg"
protokolle:
  - "strom"
related:
  - lokale-elektrizitaetsgemeinschaft-leg-erklaert
  - lohnt-sich-leg-beitritt
  - leg-preisrechner
translationId: "article-76c220e720fdffbe"
slug: "leggere-e-capire-la-bolletta-elettrica-voce-per-voce-attraverso-una-fattura-ekz"
aiPrompt: "Ich füge dir gleich die Positionen meiner Schweizer Stromrechnung ein. Erkläre mir jede Position einzeln: was sie bedeutet, wer den Preis festlegt (Energielieferant, Netzbetreiber oder Bund/Gemeinde) und ob ich sie beeinflussen kann. Rechne zum Schluss aus, wie sich meine Gesamtkosten pro kWh zusammensetzen, und nenne die zwei grössten Hebel zum Sparen. Meine Rechnung:"
translationOf: stromrechnung-verstehen-ekz
translationSourceHash: d81a9bfcf0e980271b4b1f54234b918f4658a44fa002a3f7572dfd80df8ba9b1
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T09:54:52.238Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/leggere-e-capire-la-bolletta-elettrica-voce-per-voce-attraverso-una-fattura-ekz
---

# Leggere e capire la bolletta elettrica: voce per voce con una fattura EKZ

La bolletta elettrica è uno di quei documenti che si pagano senza leggerli. Eppure la sua struttura è trasparente: ogni voce ha uno scopo chiaro, un soggetto responsabile chiaro e una risposta chiara alla domanda se sia possibile modificarla. Chi ha compreso una volta i quattro blocchi sa leggere qualsiasi bolletta elettrica svizzera, perché la struttura è prescritta dalla legge ed è identica per tutti i gestori di rete.

Questo articolo analizza voce per voce una fattura di EKZ (Elektrizitätswerke des Kantons Zürich), il nostro gestore di rete. L'esempio di calcolo interattivo qui sotto segue la struttura della nostra fattura trimestrale; le cifre si riferiscono a un nucleo familiare tipo con un consumo di 1'800 kWh nel trimestre, calcolato con le tariffe EKZ effettive del 2026.

## Chi contribuisce effettivamente alla fattura

Su una bolletta elettrica compaiono tre soggetti, anche se a inviarla è uno solo:

1. **Il fornitore di energia** vende i chilowattora. Nel servizio universale si tratta del fornitore locale; i prezzi sono verificati dal Sorvegliante dei prezzi e dalla ElCom. È l'unico blocco in cui è possibile scegliere il prodotto.
2. **Il gestore di rete** trasporta l'elettricità. La rete è un monopolio regolamentato: non si può cambiare, e la ElCom verifica le tariffe. Qui sono però disponibili tariffe a scelta e, dal 2026, lo sconto LEG.
3. **Confederazione, Cantone e Comune** applicano tributi: supplemento di rete, riserva elettrica, tasse comunali. Né il fornitore né il gestore di rete possono modificarli.

Con questo schema è possibile classificare ogni voce sottostante. Passate il mouse sulle righe dell'esempio di fattura (oppure toccatele) per visualizzarne la spiegazione:

<div class="sr-embed">
<div class="sr-grid">
<div class="sr-paper" role="group" aria-label="Interaktive Beispielrechnung">
<div class="sr-head">
<div class="sr-brand">Musterwerk AG</div>
<div class="sr-meta">La vostra fattura dal 01.03.2026 al 31.05.2026<br>Nucleo familiare tipo, casa unifamiliare, tariffa di rete EKZ Netz 400F, 1'800 kWh</div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Fornitura di energia</div>
<div class="sr-row" tabindex="0" data-sr="energie-winter"><span class="sr-label">Tariffa energetica gen.–mar.</span><span class="sr-calc">600 kWh × 13.30 Rp.</span><span class="sr-amount">79.80</span></div>
<div class="sr-row" tabindex="0" data-sr="energie-sommer"><span class="sr-label">Tariffa energetica apr.–giu.</span><span class="sr-calc">1'200 kWh × 9.00 Rp.</span><span class="sr-amount">108.00</span></div>
<div class="sr-row" tabindex="0" data-sr="grundtarif"><span class="sr-label">Tariffa base</span><span class="sr-calc">3 mesi × CHF 3.00</span><span class="sr-amount">9.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Utilizzo della rete</div>
<div class="sr-row" tabindex="0" data-sr="netz"><span class="sr-label">Rete 400F</span><span class="sr-calc">1'800 kWh × 7.50 Rp.</span><span class="sr-amount">135.00</span></div>
<div class="sr-row" tabindex="0" data-sr="sdl"><span class="sr-label">Servizi di sistema (SDL)</span><span class="sr-calc">1'800 kWh × 0.27 Rp.</span><span class="sr-amount">4.86</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Misurazione</div>
<div class="sr-row" tabindex="0" data-sr="messung"><span class="sr-label">Tariffa di misurazione</span><span class="sr-calc">3 mesi × CHF 5.00</span><span class="sr-amount">15.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Supplementi e tributi</div>
<div class="sr-row" tabindex="0" data-sr="bundesabgaben"><span class="sr-label">Tributi federali</span><span class="sr-calc">1'800 kWh × 2.30 Rp.</span><span class="sr-amount">41.40</span></div>
<div class="sr-row" tabindex="0" data-sr="stromreserve"><span class="sr-label">Riserva elettrica</span><span class="sr-calc">1'800 kWh × 0.41 Rp.</span><span class="sr-amount">7.38</span></div>
<div class="sr-row" tabindex="0" data-sr="solidarisiert"><span class="sr-label">Costi solidarizzati</span><span class="sr-calc">1'800 kWh × 0.05 Rp.</span><span class="sr-amount">0.90</span></div>
<div class="sr-row" tabindex="0" data-sr="effizienz"><span class="sr-label">Promozione dell'efficienza energetica</span><span class="sr-calc">1'800 kWh × 0.16 Rp.</span><span class="sr-amount">2.88</span></div>
</div>
<div class="sr-block sr-sums">
<div class="sr-row sr-net" tabindex="0" data-sr="netto"><span class="sr-label">Importo netto (escl. IVA)</span><span class="sr-calc"></span><span class="sr-amount">404.22</span></div>
<div class="sr-row" tabindex="0" data-sr="mwst"><span class="sr-label">IVA 8.1 %</span><span class="sr-calc"></span><span class="sr-amount">32.74</span></div>
<div class="sr-row sr-total" tabindex="0" data-sr="total"><span class="sr-label">Importo della fattura</span><span class="sr-calc"></span><span class="sr-amount">CHF 436.95</span></div>
</div>
</div>
<aside class="sr-panel" aria-live="polite">
<div class="sr-panel-inner" id="sr-panel-target">
<p class="sr-panel-hint">Passate su una voce o toccatela per scoprire cosa c'è dietro.</p>
</div>
</aside>
</div>
<div hidden id="sr-explanations">
<div data-exp="energie-winter"><strong>Energia, trimestre invernale</strong><p>L'elettricità vera e propria. Dal 2026, invece di una tariffa alta e una bassa, EKZ applica una tariffa unica per trimestre: 13.30 Rp./kWh nel semestre invernale (da gennaio a marzo, da ottobre a dicembre). Responsabile: il fornitore di energia. È l'unico blocco in cui si può scegliere un altro prodotto.</p></div>
<div data-exp="energie-sommer"><strong>Energia, trimestre estivo</strong><p>La stessa energia, un prezzo diverso: 9.00 Rp./kWh da aprile a settembre. L'estate costa meno perché l'energia idroelettrica e solare sono allora abbondanti. I prezzi stagionali rendono visibile ciò che vale da tempo sul mercato elettrico: l'elettricità ha un valore diverso a seconda della stagione.</p></div>
<div data-exp="grundtarif"><strong>Tariffa base dell'energia</strong><p>Importo mensile fisso per la fornitura di energia (CHF 3.00 al mese), indipendente dal consumo. Copre la fatturazione e la distribuzione.</p></div>
<div data-exp="netz"><strong>Utilizzo della rete, prezzo di lavoro</strong><p>Il trasporto: costruzione, esercizio e manutenzione della rete elettrica. È un monopolio regolamentato e non è possibile cambiare gestore di rete. Il prezzo dipende dal prodotto di rete scelto: tariffa standard 400ST 7.95 Rp./kWh, 400F con gestione favorevole alla rete 7.50 Rp./kWh, tariffa per pompe di calore 400WP 6.45 Rp./kWh (tutte escl. IVA). Proprio a questa voce si applica lo sconto LEG del 20 o 40 per cento.</p></div>
<div data-exp="sdl"><strong>Servizi di sistema</strong><p>Il contributo a Swissgrid per la stabilità della rete di trasmissione: mantenimento della frequenza, potenza di regolazione, energia reattiva. 0.27 Rp./kWh, simile presso tutti i gestori di rete.</p></div>
<div data-exp="messung"><strong>Misurazione</strong><p>Esercizio del contatore e messa a disposizione dei dati di misurazione, indicati come voce separata dal 2026 (in precedenza inclusi nell'utilizzo della rete). CHF 5.00 al mese. Lo Smart Meter qui pagato è, tra l'altro, il requisito tecnico per partecipare a una LEG.</p></div>
<div data-exp="bundesabgaben"><strong>Tributi federali (supplemento di rete)</strong><p>Il supplemento di rete previsto dalla legge ai sensi dell'art. 35 della legge sull'energia: 2.30 Rp./kWh per la promozione delle energie rinnovabili e il risanamento ecologico dell'energia idroelettrica. Stabilito dalla Confederazione, è uguale per ogni consumatore finale.</p></div>
<div data-exp="stromreserve"><strong>Riserva elettrica</strong><p>Tariffa Swissgrid per finanziare la riserva invernale: riserva idroelettrica, centrali elettriche di riserva, gruppi elettrogeni di emergenza. Una conseguenza della crisi energetica del 2022. 0.41 Rp./kWh.</p></div>
<div data-exp="solidarisiert"><strong>Costi solidarizzati</strong><p>Costi ripartiti a livello svizzero per potenziamenti della rete (ad esempio per l'allacciamento di impianti solari) e misure di sostegno. La voce più piccola: 0.05 Rp./kWh.</p></div>
<div data-exp="effizienz"><strong>Promozione dell'efficienza energetica</strong><p>Tributo cantonale o comunale per consulenza energetica e programmi di incentivazione, 0.16 Rp./kWh. A seconda del Comune, possono figurare inoltre tasse di concessione.</p></div>
<div data-exp="netto"><strong>Importo netto</strong><p>Somma di tutte le voci prima dell'IVA. Per questo nucleo familiare tipo: circa 22.5 Rp. per ogni kWh consumato, di cui solo circa 10.4 Rp. sono effettivamente energia.</p></div>
<div data-exp="mwst"><strong>IVA</strong><p>8.1 per cento sull'importo netto, su tutte le voci incluse le imposte statali. Ciò significa che l'IVA è applicata anche ai tributi.</p></div>
<div data-exp="total"><strong>Importo della fattura</strong><p>L'importo finale è arrotondato ai 5 centesimi, perciò si discosta di qualche centesimo dalla somma esatta. EKZ indica separatamente la differenza di arrotondamento.</p></div>
</div>
</div>

<style>
</style>

<script>
</script>

## I quattro blocchi nel dettaglio

Per chi preferisce leggere un testo scorrevole (e per i motori di ricerca), ecco le stesse voci spiegate in modo approfondito.

### Fornitura di energia: l'unico blocco con scelta del prodotto

La fornitura di energia è l'elettricità stessa. Nel servizio universale, in cui rientra la grandissima maggioranza delle economie domestiche, il fornitore locale stabilisce annualmente la tariffa e la ElCom la verifica. Per EKZ, il prodotto standard si chiama «EKZ Energie Erneuerbar» e nel 2026 costa 13.30 Rp./kWh nel semestre invernale e 9.00 Rp./kWh nel semestre estivo (escl. IVA).

È degno di nota ciò che è scomparso nel 2026: la tariffa alta e quella bassa. Al posto di «cara di giorno, conveniente di notte» vale ora una tariffa unica che cambia ogni trimestre. Il classico consiglio di far funzionare la lavatrice di notte è quindi superato dal punto di vista tariffario; conta la stagione, non l'ora. Chi desidera maggiore precisione può passare alla tariffa dinamica facoltativa, nella quale il prezzo segue ogni ora il prezzo di borsa.

A ciò si aggiunge una tariffa base fissa di CHF 3.00 al mese.

### Utilizzo della rete: il monopolio regolamentato

L'utilizzo della rete finanzia costruzione, esercizio e manutenzione delle linee, delle cabine di trasformazione e delle sottostazioni. Non è possibile cambiare gestore di rete; come compensazione, le tariffe sono regolamentate e verificabili dalla ElCom.

All'interno del monopolio esistono tuttavia possibilità di scelta che possono convenire. Nel 2026 EKZ propone tre prodotti di rete per le economie domestiche:

| Prodotto di rete | Prezzo di lavoro (escl. IVA) | Condizione |
| --- | --- | --- |
| EKZ Netz 400ST (standard) | 7.95 Rp./kWh | nessuna |
| EKZ Netz 400F | 7.50 Rp./kWh | EKZ può gestire in modo favorevole alla rete i carichi flessibili (boiler, pompa di calore) |
| EKZ Netz 400WP | 6.45 Rp./kWh | applicazioni di riscaldamento con gestione |

Chi consente la gestione del proprio boiler risparmia quindi poco meno di mezzo centesimo per chilowattora rispetto alla tariffa standard. E dal 2026 questa voce offre una seconda leva: chi aderisce a una comunità elettrica locale (LEG) ottiene sul prezzo di lavoro per l'utilizzo della rete uno sconto legale del 20 o 40 per cento per l'elettricità scambiata localmente. Che cos'è una LEG è spiegato in un [articolo dedicato](/blog/lokale-elektrizitaetsgemeinschaft-leg-erklaert); per sapere se conviene, [nel prossimo articolo](/blog/lohnt-sich-leg-beitritt).

La voce «servizi di sistema» (0.27 Rp./kWh) va a Swissgrid per la stabilità dell'intero sistema: mantenimento della frequenza, energia di regolazione, capacità di avviamento in nero.

### Misurazione: ora visibile

Dal 2026 EKZ indica separatamente i costi di misurazione: CHF 5.00 al mese per il contatore, la trasmissione dei dati e la messa a disposizione dei valori misurati. Prima erano inclusi invisibilmente nell'utilizzo della rete. Lo Smart Meter qui pagato misura con precisione al quarto d'ora ed è la base tecnica per le attuali novità del mercato elettrico: tariffe dinamiche, fatturazione LEG, spostamento dei carichi.

### Supplementi e tributi: il blocco statale

Quattro voci sulle quali né il fornitore né il gestore di rete hanno influenza:

- **Tributi federali** (2.30 Rp./kWh): il supplemento di rete previsto dalla legge sull'energia, finanzia la promozione delle energie rinnovabili e il risanamento dell'energia idroelettrica.
- **Riserva elettrica** (0.41 Rp./kWh): il premio assicurativo del Paese contro le situazioni di penuria di elettricità, introdotto dopo l'inverno 2022/23. Finanzia la riserva idroelettrica e le centrali elettriche di riserva.
- **Costi solidarizzati** (0.05 Rp./kWh): potenziamenti della rete ripartiti a livello svizzero, ad esempio per gli allacciamenti di impianti solari.
- **Promozione dell'efficienza energetica** (0.16 Rp./kWh): programmi cantonali e comunali di incentivazione e consulenza energetica. A seconda del luogo di residenza, si aggiungono tasse di concessione comunali.

Infine, in fondo alla fattura, l'IVA: 8.1 per cento su tutto, inclusi i tributi.

## Quanto resta per chilowattora

Se si rapporta l'esempio di fattura a un singolo chilowattora (trimestre estivo, tariffa 400F), il quadro è approssimativamente questo: 9.0 Rp. di energia, 7.8 Rp. di rete e servizi di sistema, 2.9 Rp. di tributi, più la quota parte della tariffa base, della tariffa di misurazione e dell'IVA. Risultato: dei circa 21-22 centesimi per kWh, nemmeno la metà è l'elettricità vera e propria. Chi discute dei prezzi dell'elettricità, per metà discute di rete e Stato.

Proprio per questo vale la pena osservare le singole voci: la leva principale per un'economia domestica è il consumo stesso; seguono la scelta del prodotto di rete, il prodotto energetico e ora anche la LEG. Per quest'ultima abbiamo realizzato un [calcolatore di prezzi](/tools/leg-rechner) che prosegue il calcolo di questo articolo.

*Tutte le tariffe: EKZ 2026, escl. IVA, fonte: raccolta tariffaria EKZ 2026. Altre zone di rete hanno prezzi diversi, ma la stessa struttura della fattura.*
