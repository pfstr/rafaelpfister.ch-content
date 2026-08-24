---
title: "Läs och förstå elräkningen: rad för rad i en EKZ-räkning"
navTitle: "Förstå elräkningen"
description: "Energi, nätanvändning, mätning, avgifter: Vad som faktiskt står på en schweizisk elräkning, vem som fastställer de enskilda priserna och vilka poster som går att påverka. Med en interaktiv exempelräkning enligt EKZ-modellen."
date: "2026-08-20"
kategorie: "El och energi"
timeToRead: "9 min lästid"
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
slug: "las-och-forsta-elrakningen-rad-for-rad-i-en-ekz-rakning"
aiPrompt: "Ich füge dir gleich die Positionen meiner Schweizer Stromrechnung ein. Erkläre mir jede Position einzeln: was sie bedeutet, wer den Preis festlegt (Energielieferant, Netzbetreiber oder Bund/Gemeinde) und ob ich sie beeinflussen kann. Rechne zum Schluss aus, wie sich meine Gesamtkosten pro kWh zusammensetzen, und nenne die zwei grössten Hebel zum Sparen. Meine Rechnung:"
translationOf: stromrechnung-verstehen-ekz
url: https://rafaelpfister.ch/sv/blog/las-och-forsta-elrakningen-rad-for-rad-i-en-ekz-rakning
translationSourceHash: 27cb9362a6871acb6b089c11dc1972f992dc29ad3b21e9f5f85c569493d669b3
translationModel: gpt-5.6-terra
translatedAt: 2026-08-21T04:14:09.006Z
translationReview: required
---

# Läs och förstå elräkningen: rad för rad i en EKZ-räkning

Elräkningen hör till de dokument man betalar utan att läsa. Ändå är den förvånansvärt transparent uppbyggd: Varje post har ett tydligt syfte, en tydlig avsändare och ett tydligt svar på frågan om man kan förändra den. Den som en gång har förstått de fyra blocken kan läsa vilken schweizisk elräkning som helst, eftersom strukturen är lagstadgad och densamma hos alla nätoperatörer.

Den här artikeln går igenom en räkning från EKZ (Elektrizitätswerke des Kantons Zürich), vår nätoperatör, post för post. Den interaktiva exempelräkningen nedan följer upplägget i vår egen kvartalsräkning; siffrorna avser ett typhushåll med 1'800 kWh förbrukning per kvartal, beräknat med EKZ:s faktiska tariffer för 2026.

## Vem som egentligen bidrar till räkningen

På en elräkning finns tre avsändare, även om bara en av dem skickar den:

1. **Energileverantören** säljer kilowattimmarna. I grundförsörjningen är det den lokala leverantören, och priserna granskas av prisövervakaren respektive ElCom. Detta är det enda blocket där du kan välja produkt.
2. **Nätoperatören** transporterar elen. Nätet är ett reglerat monopol: Det går inte att byta, och ElCom granskar tarifferna. Däremot finns det här valbara tariffer och sedan 2026 LEG-rabatten.
3. **Förbundet, kantonen och kommunen** lägger på avgifter: nätpåslag, elreserv, kommunala avgifter. Varken leverantören eller nätoperatören kan ändra något åt dem.

Med detta raster kan varje post nedan placeras in. Håll muspekaren över raderna i exempelräkningen (eller tryck på dem) för att visa förklaringen:

<div class="sr-embed">
<div class="sr-grid">
<div class="sr-paper" role="group" aria-label="Interaktive Beispielrechnung">
<div class="sr-head">
<div class="sr-brand">Musterwerk AG</div>
<div class="sr-meta">Din räkning från 01.03.2026 till 31.05.2026<br>
Typhushåll, enfamiljshus, nättariff EKZ Netz 400F, 1'800 kWh</div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Energileverans</div>
<div class="sr-row" tabindex="0" data-sr="energie-winter"><span class="sr-label">Energitariff jan.–mars</span><span class="sr-calc">600 kWh × 13.30 Rp.</span><span class="sr-amount">79.80</span></div>
<div class="sr-row" tabindex="0" data-sr="energie-sommer"><span class="sr-label">Energitariff apr.–juni</span><span class="sr-calc">1'200 kWh × 9.00 Rp.</span><span class="sr-amount">108.00</span></div>
<div class="sr-row" tabindex="0" data-sr="grundtarif"><span class="sr-label">Grundtariff</span><span class="sr-calc">3 månader × CHF 3.00</span><span class="sr-amount">9.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Nätanvändning</div>
<div class="sr-row" tabindex="0" data-sr="netz"><span class="sr-label">Nät 400F</span><span class="sr-calc">1'800 kWh × 7.50 Rp.</span><span class="sr-amount">135.00</span></div>
<div class="sr-row" tabindex="0" data-sr="sdl"><span class="sr-label">Systemtjänster (SDL)</span><span class="sr-calc">1'800 kWh × 0.27 Rp.</span><span class="sr-amount">4.86</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Mätning</div>
<div class="sr-row" tabindex="0" data-sr="messung"><span class="sr-label">Mättariff</span><span class="sr-calc">3 månader × CHF 5.00</span><span class="sr-amount">15.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Påslag och avgifter</div>
<div class="sr-row" tabindex="0" data-sr="bundesabgaben"><span class="sr-label">Federala avgifter</span><span class="sr-calc">1'800 kWh × 2.30 Rp.</span><span class="sr-amount">41.40</span></div>
<div class="sr-row" tabindex="0" data-sr="stromreserve"><span class="sr-label">Elreserv</span><span class="sr-calc">1'800 kWh × 0.41 Rp.</span><span class="sr-amount">7.38</span></div>
<div class="sr-row" tabindex="0" data-sr="solidarisiert"><span class="sr-label">Solidariserade kostnader</span><span class="sr-calc">1'800 kWh × 0.05 Rp.</span><span class="sr-amount">0.90</span></div>
<div class="sr-row" tabindex="0" data-sr="effizienz"><span class="sr-label">Främjande av energieffektivitet</span><span class="sr-calc">1'800 kWh × 0.16 Rp.</span><span class="sr-amount">2.88</span></div>
</div>
<div class="sr-block sr-sums">
<div class="sr-row sr-net" tabindex="0" data-sr="netto"><span class="sr-label">Nettobelopp (exkl. moms)</span><span class="sr-calc"></span><span class="sr-amount">404.22</span></div>
<div class="sr-row" tabindex="0" data-sr="mwst"><span class="sr-label">8.1 % mervärdesskatt</span><span class="sr-calc"></span><span class="sr-amount">32.74</span></div>
<div class="sr-row sr-total" tabindex="0" data-sr="total"><span class="sr-label">Fakturabelopp</span><span class="sr-calc"></span><span class="sr-amount">CHF 436.95</span></div>
</div>
</div>
<aside class="sr-panel" aria-live="polite">
<div class="sr-panel-inner" id="sr-panel-target">
<p class="sr-panel-hint">Håll muspekaren över en post eller tryck på den för att se vad som ligger bakom.</p>
</div>
</aside>
</div>
<div hidden id="sr-explanations">
<div data-exp="energie-winter"><strong>Energi, vinterkvartal</strong><p>Själva elen. Sedan 2026 debiterar EKZ, i stället för hög- och lågpristariff, en enhetstariff per kvartal: 13.30 Rp./kWh under vinterhalvåret (januari till mars, oktober till december). Avsändare: energileverantören. Detta är det enda block där du kan välja en annan produkt.</p></div>
<div data-exp="energie-sommer"><strong>Energi, sommarkvartal</strong><p>Samma energi, annat pris: 9.00 Rp./kWh från april till september. Sommaren är billigare eftersom vattenkraft och solel då finns i överflöd. Säsongspriserna synliggör vad som länge har gällt på elmarknaden: El har olika värde beroende på årstid.</p></div>
<div data-exp="grundtarif"><strong>Energi, grundtariff</strong><p>Fast månadsbelopp för energileveransen (CHF 3.00 per månad), oberoende av förbrukningen. Täcker fakturering och försäljning.</p></div>
<div data-exp="netz"><strong>Nätanvändning, energipris</strong><p>Transporten: byggande, drift och underhåll av elnätet. Ett reglerat monopol; det går inte att byta nätoperatör. Priset beror på det valda nätprodukten: standardtariff 400ST 7.95 Rp./kWh, 400F med nätvänlig styrning 7.50 Rp./kWh, värmepumpstariff 400WP 6.45 Rp./kWh (alla exkl. moms). Det är just på denna post som LEG-rabatten på 20 eller 40 procent tillämpas.</p></div>
<div data-exp="sdl"><strong>Systemtjänster</strong><p>Bidraget till Swissgrid för stabiliteten i transmissionsnätet: frekvenshållning, reglerkraft, reaktiv effekt. 0.27 Rp./kWh, liknande hos alla nätoperatörer.</p></div>
<div data-exp="messung"><strong>Mätning</strong><p>Drift av mätaren och tillhandahållande av mätdata, redovisas sedan 2026 som en egen post (tidigare inkluderad i nätanvändningen). CHF 5.00 per månad. Den smarta elmätaren som betalas här är för övrigt den tekniska förutsättningen för att delta i en LEG.</p></div>
<div data-exp="bundesabgaben"><strong>Federala avgifter (nätpåslag)</strong><p>Det lagstadgade nätpåslaget enligt artikel 35 i energilagen: 2.30 Rp./kWh för att främja förnybar energi och ekologisk upprustning av vattenkraften. Fastställs av förbundet och är lika för alla slutförbrukare.</p></div>
<div data-exp="stromreserve"><strong>Elreserv</strong><p>Swissgrid-tariff för finansiering av vinterreserven: vattenkraftsreserv, reservkraftverk, nödströmsaggregat. En följd av energikrisen 2022. 0.41 Rp./kWh.</p></div>
<div data-exp="solidarisiert"><strong>Solidariserade kostnader</strong><p>Kostnader som fördelas över hela Schweiz för nätförstärkningar (exempelvis för anslutning av solcellsanläggningar) och stödåtgärder. Den minsta posten: 0.05 Rp./kWh.</p></div>
<div data-exp="effizienz"><strong>Främjande av energieffektivitet</strong><p>Kantonal respektive kommunal avgift för energirådgivning och stödprogram, 0.16 Rp./kWh. Beroende på kommun kan dessutom koncessionsavgifter tillkomma här.</p></div>
<div data-exp="netto"><strong>Nettobelopp</strong><p>Summan av alla poster före mervärdesskatt. För detta typhushåll: omkring 22.5 Rp. per förbrukad kWh, varav endast cirka 10.4 Rp. faktiskt är energi.</p></div>
<div data-exp="mwst"><strong>Mervärdesskatt</strong><p>8.1 procent på nettobeloppet, på alla poster inklusive de statliga avgifterna. Ja: mervärdesskatt tas ut på avgifter.</p></div>
<div data-exp="total"><strong>Fakturabelopp</strong><p>Slutbeloppet avrundas till 5 rappen och avviker därför med några rappen från den exakta summan. EKZ redovisar avrundningsdifferensen separat.</p></div>
</div>
</div>

<style>
</style>

<script>
</script>

## De fyra blocken i detalj

För alla som hellre läser löpande text (och för sökmotorer) följer här samma poster mer utförligt.

### Energileverans: det enda blocket med produktval

Energileveransen är själva elen. I grundförsörjningen, där de allra flesta hushåll befinner sig, fastställer den lokala leverantören tariffen varje år och ElCom granskar den. Hos EKZ heter standardprodukten «EKZ Energie Erneuerbar» och kostar 2026 13.30 Rp./kWh under vinterhalvåret och 9.00 Rp./kWh under sommarhalvåret (exkl. moms).

Anmärkningsvärt är vad som har försvunnit 2026: hög- och lågpristariffen. I stället för ”dyrt på dagen, billigt på natten” gäller nu en enhetstariff som ändras per kvartal. Det klassiska rådet att köra tvättmaskinen på natten är därmed tariffmässigt föråldrat; det är säsongen, inte klockslaget, som räknas. Den som vill ha det mer detaljerat kan byta till den dynamiska valbara tariffen, där priset följer börspriset timme för timme.

Därtill kommer en fast grundtariff på CHF 3.00 per månad.

### Nätanvändning: det reglerade monopolet

Nätanvändningen betalar för byggande, drift och underhåll av ledningar, transformatorstationer och understationer. Det går inte att byta nätoperatör; som motvikt är tarifferna reglerade och kan granskas av ElCom.

Men inom monopolet finns valmöjligheter som kan löna sig. EKZ erbjuder 2026 tre nätprodukter för hushåll:

| Nätprodukt | Energipris (exkl. moms) | Villkor |
| --- | --- | --- |
| EKZ Netz 400ST (standard) | 7.95 Rp./kWh | inget |
| EKZ Netz 400F | 7.50 Rp./kWh | EKZ får styra flexibla laster (varmvattenberedare, värmepump) på ett nätvänligt sätt |
| EKZ Netz 400WP | 6.45 Rp./kWh | värmeapplikationer med styrning |

Den som tillåter styrning av sin varmvattenberedare sparar alltså knappt en halv rappen per kilowattimme jämfört med standarden. Och sedan 2026 finns ett andra verktyg på denna post: Den som ansluter sig till en lokal elgemenskap (LEG) får en lagstadgad rabatt på 20 eller 40 procent på energipriset för nätanvändningen för den lokalt handlade elen. Vad en LEG är står i [en separat artikel](/blog/lokale-elektrizitaetsgemeinschaft-leg-erklaert); om det lönar sig, [i nästa artikel](/blog/lohnt-sich-leg-beitritt).

Posten ”Systemtjänster” (0.27 Rp./kWh) går till Swissgrid för stabiliteten i hela systemet: frekvenshållning, reglerenergi, svartstartsförmåga.

### Mätning: nu synlig

Sedan 2026 redovisar EKZ mätkostnaderna separat: CHF 5.00 per månad för mätare, dataöverföring och tillhandahållande av mätvärden. Tidigare ingick detta osynligt i nätanvändningen. Den smarta elmätare som betalas här mäter med kvartsnoggrannhet och är den tekniska grunden för allt nytt som elmarknaden just nu lär sig: dynamiska tariffer, LEG-avräkning, lastförskjutning.

### Påslag och avgifter: det statliga blocket

Fyra poster som varken leverantören eller nätoperatören kan påverka:

- **Federala avgifter** (2.30 Rp./kWh): nätpåslaget enligt energilagen, finansierar främjandet av förnybar energi och upprustning av vattenkraften.
- **Elreserv** (0.41 Rp./kWh): landets försäkringspremie mot bristsituationer för el, införd efter vintern 2022/23. Betalar för vattenkraftsreserv och reservkraftverk.
- **Solidariserade kostnader** (0.05 Rp./kWh): nätförstärkningar som fördelas över hela Schweiz, exempelvis för anslutningar av solcellsanläggningar.
- **Främjande av energieffektivitet** (0.16 Rp./kWh): kantonala och kommunala stödprogram och energirådgivning. Beroende på bostadsort tillkommer kommunala koncessionsavgifter.

Längst ned finns slutligen mervärdesskatten: 8.1 procent på allt, även avgifterna.

## Vad som återstår per kilowattimme

Om man räknar om exempelräkningen till en enskild kilowattimme (sommarkvartal, tariff 400F) blir bilden ungefär så här: 9.0 Rp. energi, 7.8 Rp. nät och systemtjänster, 2.9 Rp. avgifter, plus proportionell grund- och mättariff samt moms. Resultat: Av omkring 21 till 22 rappen per kWh är inte ens hälften själva elen. Den som diskuterar elpriser diskuterar till hälften nät och stat.

Det är just därför det lönar sig att titta på posterna: Det enskilt största verktyget för ett hushåll är själva förbrukningen, därefter kommer valet av nätprodukt, energiprodukt och nu även LEG. För den sistnämnda har vi byggt en [priskalkylator](/tools/leg-rechner) som fortsätter beräkningen i denna artikel.

*Alla tariffer: EKZ 2026, exkl. moms, källa: EKZ:s tariffer 2026. Andra nätområden har andra priser, men samma räkningstruktur.*
