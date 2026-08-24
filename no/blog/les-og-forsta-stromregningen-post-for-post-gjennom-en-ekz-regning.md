---
title: "Les og forstå strømregningen: post for post gjennom en EKZ-regning"
navTitle: "Forstå strømregningen"
description: "Energi, nettbruk, måling, avgifter: Hva som faktisk står på en sveitsisk strømregning, hvem som fastsetter de enkelte prisene og hvilke poster det er mulig å påvirke. Med interaktiv eksempelregning basert på EKZ-modellen."
date: "2026-08-20"
kategorie: "Strøm og energi"
timeToRead: "9 min lesetid"
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
slug: "les-og-forsta-stromregningen-post-for-post-gjennom-en-ekz-regning"
aiPrompt: "Ich füge dir gleich die Positionen meiner Schweizer Stromrechnung ein. Erkläre mir jede Position einzeln: was sie bedeutet, wer den Preis festlegt (Energielieferant, Netzbetreiber oder Bund/Gemeinde) und ob ich sie beeinflussen kann. Rechne zum Schluss aus, wie sich meine Gesamtkosten pro kWh zusammensetzen, und nenne die zwei grössten Hebel zum Sparen. Meine Rechnung:"
translationOf: stromrechnung-verstehen-ekz
url: https://rafaelpfister.ch/no/blog/les-og-forsta-stromregningen-post-for-post-gjennom-en-ekz-regning
translationSourceHash: 27cb9362a6871acb6b089c11dc1972f992dc29ad3b21e9f5f85c569493d669b3
translationModel: gpt-5.6-terra
translatedAt: 2026-08-21T04:14:53.774Z
translationReview: required
---

# Les og forstå strømregningen: post for post gjennom en EKZ-regning

Strømregningen er blant dokumentene man betaler uten å lese. Likevel er den overraskende ærlig bygget opp: Hver post har et tydelig formål, en tydelig avsender og et tydelig svar på spørsmålet om man kan endre noe ved den. Når man først har forstått de fire blokkene, kan man lese enhver sveitsisk strømregning, for oppbyggingen er lovpålagt og lik hos alle nettselskaper.

Denne artikkelen går post for post gjennom en regning fra EKZ (Elektrizitätswerke des Kantons Zürich), nettselskapet vårt. Den interaktive eksempelregningen nedenfor følger oppbyggingen av vår egen kvartalsregning; tallene gjelder en eksempelhusholdning med et forbruk på 1'800 kWh per kvartal, beregnet med EKZs reelle tariffer for 2026.

## Hvem som egentlig bidrar til regningen

På en strømregning finnes det tre avsendere, selv om bare én sender den:

1. **Energileverandøren** selger kilowattimene. I grunnforsyningen er dette den lokale leverandøren, og prisene kontrolleres av prisovervåkeren, eller ElCom. Dette er den eneste blokken med produktvalg.
2. **Nettselskapet** transporterer strømmen. Nettet er et regulert monopol: Det kan ikke byttes, og ElCom kontrollerer tariffene. Til gjengjeld finnes det her valgbare tariffer og siden 2026 LEG-rabatten.
3. **Forbund, kanton og kommune** legger på avgifter: nettillegg, strømreserve, kommunale gebyrer. Verken leverandøren eller nettselskapet kan endre disse.

Med dette rutenettet kan hver post nedenfor plasseres. Hold musen over linjene i eksempelregningen (eller trykk på dem) for å vise forklaringen:

<div class="sr-embed">
<div class="sr-grid">
<div class="sr-paper" role="group" aria-label="Interaktive Beispielrechnung">
<div class="sr-head">
<div class="sr-brand">Musterwerk AG</div>
<div class="sr-meta">Deres regning fra 01.03.2026 til 31.05.2026<br>Eksempelhusholdning, enebolig, netttariff EKZ Netz 400F, 1'800 kWh</div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Energileveranse</div>
<div class="sr-row" tabindex="0" data-sr="energie-winter"><span class="sr-label">Energitariff jan.–mars</span><span class="sr-calc">600 kWh × 13.30 Rp.</span><span class="sr-amount">79.80</span></div>
<div class="sr-row" tabindex="0" data-sr="energie-sommer"><span class="sr-label">Energitariff apr.–juni</span><span class="sr-calc">1'200 kWh × 9.00 Rp.</span><span class="sr-amount">108.00</span></div>
<div class="sr-row" tabindex="0" data-sr="grundtarif"><span class="sr-label">Grunntariff</span><span class="sr-calc">3 måneder × CHF 3.00</span><span class="sr-amount">9.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Nettbruk</div>
<div class="sr-row" tabindex="0" data-sr="netz"><span class="sr-label">Nett 400F</span><span class="sr-calc">1'800 kWh × 7.50 Rp.</span><span class="sr-amount">135.00</span></div>
<div class="sr-row" tabindex="0" data-sr="sdl"><span class="sr-label">Systemtjenester (SDL)</span><span class="sr-calc">1'800 kWh × 0.27 Rp.</span><span class="sr-amount">4.86</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Måling</div>
<div class="sr-row" tabindex="0" data-sr="messung"><span class="sr-label">Måletariff</span><span class="sr-calc">3 måneder × CHF 5.00</span><span class="sr-amount">15.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Tillegg og avgifter</div>
<div class="sr-row" tabindex="0" data-sr="bundesabgaben"><span class="sr-label">Føderale avgifter</span><span class="sr-calc">1'800 kWh × 2.30 Rp.</span><span class="sr-amount">41.40</span></div>
<div class="sr-row" tabindex="0" data-sr="stromreserve"><span class="sr-label">Strømreserve</span><span class="sr-calc">1'800 kWh × 0.41 Rp.</span><span class="sr-amount">7.38</span></div>
<div class="sr-row" tabindex="0" data-sr="solidarisiert"><span class="sr-label">Solidariserte kostnader</span><span class="sr-calc">1'800 kWh × 0.05 Rp.</span><span class="sr-amount">0.90</span></div>
<div class="sr-row" tabindex="0" data-sr="effizienz"><span class="sr-label">Fremme av energieffektivitet</span><span class="sr-calc">1'800 kWh × 0.16 Rp.</span><span class="sr-amount">2.88</span></div>
</div>
<div class="sr-block sr-sums">
<div class="sr-row sr-net" tabindex="0" data-sr="netto"><span class="sr-label">Nettobeløp (ekskl. mva.)</span><span class="sr-calc"></span><span class="sr-amount">404.22</span></div>
<div class="sr-row" tabindex="0" data-sr="mwst"><span class="sr-label">8.1 % merverdiavgift</span><span class="sr-calc"></span><span class="sr-amount">32.74</span></div>
<div class="sr-row sr-total" tabindex="0" data-sr="total"><span class="sr-label">Fakturabeløp</span><span class="sr-calc"></span><span class="sr-amount">CHF 436.95</span></div>
</div>
</div>
<aside class="sr-panel" aria-live="polite">
<div class="sr-panel-inner" id="sr-panel-target">
<p class="sr-panel-hint">Hold musen over en post eller trykk på den for å se hva som ligger bak.</p>
</div>
</aside>
</div>
<div hidden id="sr-explanations">
<div data-exp="energie-winter"><strong>Energi, vinterkvartal</strong><p>Selve strømmen. Siden 2026 beregner EKZ, i stedet for høy- og lavtariff, en enhetstariff per kvartal: 13.30 Rp./kWh i vinterhalvåret (januar til mars, oktober til desember). Avsender: energileverandøren. Dette er den eneste blokken der et annet produkt kan velges.</p></div>
<div data-exp="energie-sommer"><strong>Energi, sommerkvarter</strong><p>Samme energi, annen pris: 9.00 Rp./kWh fra april til september. Sommeren er billigere fordi vannkraft og solstrøm da finnes i rikelige mengder. Sesongprisene synliggjør det som lenge har vært tilfelle i strømmarkedet: Strøm har ulik verdi avhengig av årstiden.</p></div>
<div data-exp="grundtarif"><strong>Grunntariff energi</strong><p>Fast månedsbeløp for energileveransen (CHF 3.00 per måned), uavhengig av forbruket. Dekker fakturering og salg.</p></div>
<div data-exp="netz"><strong>Nettbruk, energiledd</strong><p>Transporten: bygging, drift og vedlikehold av strømnettet. Et regulert monopol; nettselskapet kan ikke byttes. Prisen avhenger av valgt nettprodukt: standardtariff 400ST 7.95 Rp./kWh, 400F med nettnyttig styring 7.50 Rp./kWh, varmepumpetariff 400WP 6.45 Rp./kWh (alle ekskl. mva.). Det er nettopp på denne posten LEG-rabatten på 20 eller 40 prosent gis.</p></div>
<div data-exp="sdl"><strong>Systemtjenester</strong><p>Bidraget til Swissgrid for stabiliteten i overføringsnettet: frekvensregulering, regulerkraft, reaktiv energi. 0.27 Rp./kWh, omtrent likt hos alle nettselskaper.</p></div>
<div data-exp="messung"><strong>Måling</strong><p>Drift av måleren og tilgjengeliggjøring av måledata, oppført som egen post siden 2026 (tidligere inkludert i nettbruken). CHF 5.00 per måned. Smartmåleren som betales her, er for øvrig den tekniske forutsetningen for å delta i en LEG.</p></div>
<div data-exp="bundesabgaben"><strong>Føderale avgifter (nettillegg)</strong><p>Det lovbestemte nettillegget etter art. 35 i energiloven: 2.30 Rp./kWh til fremme av fornybar energi og økologisk rehabilitering av vannkraften. Fastsettes av forbundet og er likt for alle sluttbrukere.</p></div>
<div data-exp="stromreserve"><strong>Strømreserve</strong><p>Swissgrid-tariff for finansiering av vinterreserven: vannkraftreserve, reservekraftverk, nødstrømsaggregater. En følge av energikrisen i 2022. 0.41 Rp./kWh.</p></div>
<div data-exp="solidarisiert"><strong>Solidariserte kostnader</strong><p>Kostnader som fordeles over hele Sveits for nettforsterkninger (for eksempel for tilkobling av solcelleanlegg) og støttetiltak. Den minste posten: 0.05 Rp./kWh.</p></div>
<div data-exp="effizienz"><strong>Fremme av energieffektivitet</strong><p>Kantonal eller kommunal avgift til energirådgivning og støtteprogrammer, 0.16 Rp./kWh. Avhengig av kommunen kan det i tillegg stå konsesjonsavgifter her.</p></div>
<div data-exp="netto"><strong>Nettobeløp</strong><p>Summen av alle poster før merverdiavgift. For denne eksempelhusholdningen: rundt 22.5 Rp. per forbrukte kWh, hvorav bare omtrent 10.4 Rp. faktisk er energi.</p></div>
<div data-exp="mwst"><strong>Merverdiavgift</strong><p>8.1 prosent av nettobeløpet, på alle poster inkludert de statlige avgiftene. Ja: Det beregnes merverdiavgift på avgifter.</p></div>
<div data-exp="total"><strong>Fakturabeløp</strong><p>Sluttbeløpet rundes til 5 rappen, og avviker derfor med noen få rappen fra den eksakte summen. EKZ oppgir avrundingsdifferansen separat.</p></div>
</div>
</div>

<style>
</style>

<script>
</script>

## De fire blokkene i detalj

For alle som foretrekker løpende tekst (og for søkemotorer), følger de samme postene her i detalj.

### Energileveranse: den eneste blokken med produktvalg

Energileveransen er selve strømmen. I grunnforsyningen, der de aller fleste husholdninger befinner seg, fastsetter den lokale leverandøren tariffen årlig, og ElCom kontrollerer den. Hos EKZ heter standardproduktet «EKZ Energie Erneuerbar» og koster i 2026 13.30 Rp./kWh i vinterhalvåret og 9.00 Rp./kWh i sommerhalvåret (ekskl. mva.).

Det bemerkelsesverdige er hva som forsvant i 2026: høy- og lavtariffen. I stedet for «dyrt om dagen, billig om natten» gjelder nå en enhetstariff som endres per kvartal. Det klassiske rådet om å la vaskemaskinen gå om natten er dermed utdatert tariffmessig; det er sesongen som teller, ikke klokkeslettet. De som ønsker det mer detaljert, kan bytte til den dynamiske valgfrie tariffen, der prisen følger børsprisen time for time.

I tillegg kommer en fast grunntariff på CHF 3.00 per måned.

### Nettbruk: det regulerte monopolet

Nettbruken betaler for bygging, drift og vedlikehold av ledninger, transformatorstasjoner og understasjoner. Nettselskapet kan ikke byttes; som motvekt er tariffene regulert og kan kontrolleres av ElCom.

Innenfor monopolet finnes det likevel valgmuligheter som kan lønne seg. EKZ har i 2026 tre nettprodukter for husholdninger:

| Nettprodukt | Energiledd (ekskl. mva.) | Vilkår |
| --- | --- | --- |
| EKZ Netz 400ST (standard) | 7.95 Rp./kWh | ingen |
| EKZ Netz 400F | 7.50 Rp./kWh | EKZ kan styre fleksible laster (bereder, varmepumpe) på en nettnyttig måte |
| EKZ Netz 400WP | 6.45 Rp./kWh | varmeapplikasjoner med styring |

Den som tillater styring av varmtvannsberederen, sparer altså nesten en halv rappen per kilowattime sammenlignet med standarden. Og siden 2026 finnes det en annen mulighet på denne posten: Den som blir med i et lokalt elektrisitetsfellesskap (LEG), får en lovfestet rabatt på 20 eller 40 prosent på energileddet for nettbruk for strømmen som handles lokalt. Hva en LEG er, står i [en egen artikkel](/blog/lokale-elektrizitaetsgemeinschaft-leg-erklaert); om det lønner seg, [i den neste](/blog/lohnt-sich-leg-beitritt).

Posten «systemtjenester» (0.27 Rp./kWh) går til Swissgrid for stabiliteten i hele systemet: frekvensregulering, regulerkraft, svartstartkapasitet.

### Måling: nå synlig

Siden 2026 oppgir EKZ målekostnadene separat: CHF 5.00 per måned for måler, dataoverføring og tilgjengeliggjøring av måleverdiene. Tidligere var dette usynlig inkludert i nettbruken. Smartmåleren som betales her, måler hvert kvarter og er det tekniske grunnlaget for alt strømmarkedet nå lærer: dynamiske tariffer, LEG-avregning, lastflytting.

### Tillegg og avgifter: den statlige blokken

Fire poster som verken leverandøren eller nettselskapet har innflytelse på:

- **Føderale avgifter** (2.30 Rp./kWh): nettillegget etter energiloven finansierer fremme av fornybar energi og rehabilitering av vannkraften.
- **Strømreserve** (0.41 Rp./kWh): landets forsikringspremie mot strømmangelsituasjoner, innført etter vinteren 2022/23. Betaler for vannkraftreserve og reservekraftverk.
- **Solidariserte kostnader** (0.05 Rp./kWh): nettforsterkninger som fordeles over hele Sveits, for eksempel for tilkobling av solcelleanlegg.
- **Fremme av energieffektivitet** (0.16 Rp./kWh): kantonale og kommunale støtteprogrammer og energirådgivning. Avhengig av bosted kommer kommunale konsesjonsavgifter i tillegg.

Helt nederst kommer til slutt merverdiavgiften: 8.1 prosent av alt, også avgiftene.

## Hva som gjenstår per kilowattime

Regner man om eksempelregningen til én enkelt kilowattime (sommerkvartal, tariff 400F), får man omtrent dette bildet: 9.0 Rp. energi, 7.8 Rp. nett og systemtjenester, 2.9 Rp. avgifter, i tillegg forholdsmessig grunntariff, måletariff og mva. Resultatet: Av rundt 21 til 22 rappen per kWh er ikke engang halvparten selve strømmen. Den som diskuterer strømpriser, diskuterer halvparten nett og stat.

Det er nettopp derfor det lønner seg å se på postene: Den største enkeltmuligheten for en husholdning ligger i selve forbruket, deretter kommer valg av nettprodukt, energiprodukt og nå også LEG. For sistnevnte har vi bygget en [priskalkulator](/tools/leg-rechner), som viderefører regningen i denne artikkelen.

*Alle tariffer: EKZ 2026, ekskl. mva., kilde: EKZs tarifsamling 2026. Andre nettområder har andre priser, men samme regningsoppbygging.*
