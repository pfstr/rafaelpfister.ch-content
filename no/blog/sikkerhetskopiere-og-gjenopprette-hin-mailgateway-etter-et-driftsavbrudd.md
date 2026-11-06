---
title: "Sikre og gjenopprett HIN Mailgateway etter et driftsavbrudd"
navTitle: "Sikkerhetskopiering og gjenoppretting"
description: "Et cluster beskytter HIN Mailgateway mot bortfall av en node, men erstatter ikke en sikkerhetskopi. Avgjørende er konfigurasjon, nøkkelmateriale, rekkefølgen for gjenoppretting og endringene med Stargate."
date: "2026-07-08"
kategorie: "HIN-gateway"
timeToRead: "15 min lesetid"
themen:
  - hin-gateway
slug: "sikkerhetskopiere-og-gjenopprette-hin-mailgateway-etter-et-driftsavbrudd"
translationOf: "hin-mailgateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translationReview: automatic
translationSourceHash: cd8f7c8633d8cf882725ad88feed7be5a1730ce563724481e25b4b9ee2ec84e4
translatedAt: 2026-11-06T03:24:24.533Z
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/sikkerhetskopiere-og-gjenopprette-hin-mailgateway-etter-et-driftsavbrudd
translationModel: gpt-5.6-terra
---

# Sikre og gjenopprett HIN Mailgateway etter et driftsavbrudd

Mange HIN Mailgateways i produksjon kjører som cluster. Hvis én node faller ut, overtar den andre. Denne redundansen hjelper imidlertid ikke mot en feilaktig regel, et slettet sertifikat eller en skadet import: [Systemrelevante data replikeres til alle noder](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), inkludert uønskede endringer.

For en robust gjenoppretting trengs derfor en separat sikkerhetskopi. Siden HIN Mailgateway teknisk sett er basert på en SEPPmail-appliance med GINA, gjelder de dokumenterte mekanismene for sikkerhetskopiering og gjenoppretting.

## Hvilke data som ligger på gatewayen

Gatewayen behandler inn- og utgående e-post etter et sentralt regelverk og krypterer avhengig av mottaker med S/MIME, OpenPGP eller TLS; for mottakere uten eget nøkkelmateriale brukes den nettbaserte GINA-metoden. For sikkerhetskopien er det avgjørende at [meldingsinnhold ikke lagres permanent på gatewayen](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Appliancen behandler e-post fortløpende uten å arkivere den.

  

## Hva clusteret replikerer

SEPPmail har flere [clustervarianter – high availability, load balancing og geo-cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); systemparametere, brukerdata og nøkkelmateriale synkroniseres på tvers av alle noder. I et [frontend/backend-cluster har frontenden ingen egen konfigurasjonsdatabase](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Den kan driftes i en DMZ uten datalagring og mottar bare dataene som er nødvendige for den aktuelle behandlingen; databasen med nøklene ligger på backenden. For [Large File Transfer (LFT) gjelder et unntak](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Hver partner, også frontender, får tilordnet en like stor disk, og LFT-dataene synkroniseres til alle noder.

  

## Hvorfor replikering ikke er sikkerhetskopiering

> *Replikering kopierer gjeldende tilstand, også den feilaktige. En sikkerhetskopi bevarer en kjent, fungerende tilstand.*

En feilaktig import, en slettet nøkkel eller et deaktivert domene replikeres til partnernoden i løpet av sekunder. Uten en uavhengig sikkerhetskopi finnes det deretter ikke noe gjenopprettingspunkt. Hvor tett tilgjengelighet og konsistens henger sammen i clusteret, ble synlig ved [innloggingsproblemene etter oppdateringen til 15.0.5](/blog/hin-update-issue-version-15.0.5), som ble utløst av forstyrret clusterreplikering.

  

## Hva sikkerhetskopien inneholder – og ikke inneholder

[SEPPmail-sikkerhetskopien er bevisst slank](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Den omfatter utelukkende konfigurasjon og kryptografisk nøkkelmateriale: [ingen meldinger, ingen e-postkø og uttrykkelig ingen logger](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (logger skal derfor sendes via Syslog til et eksternt system). Siden firmware 14.0.0 oppretter appliancen sikkerhetskopien [automatisk om natten ved midnatt som backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); den kan hentes via `Download`, `Send Backup` (e-post til sikkerhetskopigruppen) eller SCP.

| **Inkludert i sikkerhetskopien** | **Ikke inkludert i sikkerhetskopien** |
| --- | --- |
| Systemkonfigurasjon og regelverk | E-postinnhold / meldingstekster |
| Bruker- og GINA-kontoer | Gjeldende e-postkø |
| Nøkkelmateriale: S/MIME, X.509, OpenPGP | System- og e-postlogger (sikres eksternt via Syslog) |
| TLS- og sertifikatkonfigurasjon | Operativsystem / VM-image |


Dette innebærer: Siden operativsystemet ikke inngår i konfigurasjonssikkerhetskopien, krever en fullstendig DR-strategi i tillegg en måte å gjenopprette appliance-grunnlaget på (ny utrulling fra produsentens image eller VM-snapshot). Konfigurasjonssikkerhetskopien supplerer deretter konfigurasjon og nøkler.

  

## Snapshots er ikke en cluster-sikkerhetskopi

Siden firmware 14.0.0 oppretter appliancen også [lokale snapshots, men bare dersom det finnes en LFT-partisjon med database](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). På søndager opprettes et fullstendig snapshot, og mandag til lørdag ett inkrementelt snapshot per dag; oppbevaringstiden er 14 dager.

Avgjørende for DR-planleggingen: I clusterdrift kjører disse snapshotene riktignok i bakgrunnen, men det tilbys ingen gjenoppretting fra dem. Snapshots er dermed en lokal hjelp for tilbakeføring på enkeltsystemer, ikke cluster-gjenoppretting. Den pålitelige sikkerhetskopien forblir den krypterte konfigurasjonssikkerhetskopien.

  

## Konfigurere sikkerhetskopiering

Forutsetningen for hver hentemetode er at det er angitt et sikkerhetskopipassord under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); uten dette passordet kan sikkerhetskopien verken lastes ned, sendes eller gjøres tilgjengelig via SCP. Som standard sendes den nattlige sikkerhetskopien per e-post til [gruppen «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); en dedikert sikkerhetskopibruker trenger en gyldig intern e-postadresse.

-   Angi sikkerhetskopipassord og [oppbevar det separat fra sikkerhetskopien](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Sikkerhetskopien inneholder private nøkler.
    
-   For automatisert lagring, [hent sikkerhetskopiene via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): lagre den offentlige `SSH-RSA`\-nøkkelen i administrasjonen og hent `backup.tgz` som gjøres tilgjengelig ved midnatt, via OS-brukeren `backup`.
    
-   Sikre logger separat (ekstern Syslog), siden de [bevisst ikke er en del av sikkerhetskopien](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Sikkerhetskopistrategi i clusterdrift

I clusterdrift er strukturert sikkerhetskopiering og konsekvent versjonsstyring avgjørende.

-   **Daglig**: hent den krypterte konfigurasjonssikkerhetskopien [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) og lagre den eksternt med versjonering
    
-   **Ukentlig**: fullstendig VM- eller systemsikkerhetskopi av begge noder, tidsforskjøvet i stedet for samtidig (operativsystemet er ikke en del av konfigurasjonssikkerhetskopien)
    
-   **Før vedlikehold eller oppdatering**: stopp e-postmottak med [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Innkommende e-post avvises da midlertidig med en konfigurerbar SMTP-returkode (standard `421`); innstillingen forblir aktiv også etter omstart.
    

  

Når det gjelder versjonsstyring: SEPPmail oppdaterer i et frontend/backend-cluster [frontenden før backenden](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), og ved oppdateringer i flere trinn må alle partnere ha samme versjonsnivå før det oppgraderes til neste release. Etter en større oppdatering kan det være nødvendig å generere regelsettet på nytt (meldingen *«Current ruleset created for another version»*).

  

## Gjenoppretting og katastrofeberedskap

Grunntilfellet er enkelt: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), start på nytt, og deretter fungerer gatewayen fullt ut. Versjonsregelen må følges: Bare sikkerhetskopien fra den umiddelbart foregående firmwareversjonen kan importeres til den gjeldende (generer deretter regelsettet på nytt); det er ikke mulig å importere en sikkerhetskopi fra nyere firmware til en eldre versjon.

I clusteret gjelder en viktig begrensning:

-   **Gjenopprett aldri en enkelt node direkte**: En [gjenoppretting av én enkelt clusterpartner er ikke forutsatt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Fjern i stedet den defekte maskinen fra clusteret, sett opp en ny VM og legg den til igjen: Konfigurasjon og nøkler kommer automatisk ved replikering fra den intakte partneren.
    
-   **Totalt tap på alle noder**: Rull ut appliancen på nytt fra basis-imaget, importer deretter den sist kjente fungerende konfigurasjonssikkerhetskopien og start på nytt.
    

En sikkerhetskopi er bare så pålitelig som den siste vellykkede gjenopprettingstesten. En testgjenoppretting bør gjennomføres minst to ganger i året i et isolert miljø, ikke mot produksjonsclusteret.

  

### Sjekkliste for gjenoppretting i en nødsituasjon

1.  Fjern defekt node fra clusteret (ingen direkte gjenoppretting av en partner).
    
2.  Sett opp en ny VM, eller ved totalt tap: klargjør appliancen fra basis-image/VM-snapshot.
    
3.  Kun ved totalt tap: importer siste fungerende konfigurasjonssikkerhetskopi (ha passordet klart, følg versjonsregelen).
    
4.  Kontroller noden isolert: SMTP-mottak, TLS, GINA, regelverk.
    
5.  Legg den til i clusteret og overvåk replikeringen; generer regelsettet på nytt ved melding.
    
6.  Dokumenter hendelsen og oppdater sikkerhetskopieringsintervall og versjonsnivåer.
    

  

To vedlikeholdsoperasjoner krever særlig forsiktighet og alltid en sikkerhetskopi på forhånd: [Utvidelse av LFT-partisjonen slår av appliancen](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), og factory reset overskriver harddisken ti ganger (sikkerhetsspørsmålet krever koden skrevet i omvendt rekkefølge).

  

## Hva som endres med «Stargate»

HIN erstatter gradvis det eksisterende Mailgateway med det [nye HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (prosjekt «Stargate», hos Zug-baserte [Vereign AG omtalt som «Verimesh»](https://www.vereign.com/)). Dette er ikke en 1:1-erstatning av appliancen, men et arkitekturskifte som berører sikkerhetskopiering og katastrofeberedskap i kjernen:

-   **Fra sentralisert til desentralisert**: Noder kommuniserer direkte med hverandre; et sentralt distribusjonssenter bortfaller.
    
-   **Desentralisert nøkkeladministrasjon (DKMS)**: Hver organisasjon administrerer sin egen kryptografiske identitet uten en sentral Certificate Authority.
    
-   **Ende-til-ende-kryptering** med fragmentering av meldingene.
    
-   **Resiliens fra nettet**: Hvis én node faller ut, forblir meshet funksjonelt.
    
-   **Åpent deployment-repository**: Repositoryet [Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) med tjenestene til en Stargate-node er offentlig tilgjengelig under AGPLv3; Vereign Client Library (vcl) er derimot bare tilgjengelig etter innlogging.
    

Tidsplan: Den desentraliserte infrastrukturen har vært [i produktiv bruk i det sveitsiske helsevesenet siden april 2025](https://www.vereign.com/); for 2026 er det planlagt gradvis utskifting av de eksisterende Mailgateways og en bred utrulling. Organisasjoner med HIN-egne domener (`@hin.ch`, `@verband-hin.ch`) kjører på HIN-infrastruktur og blir knapt berørt av omstillingen.

  

For driftshåndboken betyr dette: Den klassiske disiplinen «eksportere appliance-konfigurasjon og nøkler og gjenopprette dem på en erstatningsnode» blir mindre viktig. I stedet kommer node-enrollment, oppbevaring av identiteter og nøkler i meshet samt gjenopptakelse av noder i nettet.

  

## Det viktigste skillet

Så lenge HIN MGW kjører på SEPPmail-teknologi, gjelder følgende: Clusteret kompenserer for maskinvarefeil, men ansvaret for konfigurasjons- og nøkkelintegritet ligger fortsatt hos operatøren. Den slanke konfigurasjonssikkerhetskopien skal sikres uavhengig av clusteret (via SCP, versjonert, med passord oppbevart separat), snapshots erstatter den ikke i clusteret, versjonsnivåene holdes synkronisert, og gjenopprettingen testes regelmessig isolert. Overgangen til Stargate bør tas med i DR-planleggingen på et tidlig tidspunkt, siden den flytter resiliens og nøkkeloppbevaring til det desentraliserte nettet.

## Endringslogg

- **25. oktober 2026:** Lagt til: Vereign Client Library er bare tilgjengelig etter innlogging; repositoryet Stargate Deployment er nå oppgitt som offentlig tilgjengelig kilde under AGPLv3.

## Kilder

1.  [SEPPmail-dokumentasjon – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Innhold i sikkerhetskopien (kun konfigurasjon og nøkkelmateriale), nattlig oppretting, automatisk cluster-gjenoppretting via replikering.
    
2.  [SEPPmail-dokumentasjon – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Detaljert referanse: Sikkerhetskopimeny (Download / Send Backup / Change password, `backup.tgz` ved midnatt), LFT-snapshots (14 dager, ingen gjenoppretting i clusteret), gjenopprettingsregler og clusterprosedyre, Preempt (SMTP-returkode, standard 421), Device Cloning, oppdateringskanaler og oppdateringsrekkefølge (frontend før backend), Factory Reset, bulkimport/-eksport.
    
3.  [SEPPmail-dokumentasjon – «Opprette backup-bruker»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Gruppen «backup (Backup operator)», kryptering og passordadministrasjon.
    
4.  [SEPPmail-dokumentasjon – «Kopiere backup via SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Henting av `backup.tgz` via SCP gjennom OS-brukeren `backup` i stedet for e-postsending.
    
5.  [SEPPmail-dokumentasjon – «Cluster / høy tilgjengelighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Clustertyper og dataene som synkroniseres på tvers av alle noder (systemparametere, brukerdata, nøkkelmateriale).
    
6.  [SEPPmail-dokumentasjon – «Frontend/backend-cluster»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Frontend uten konfigurasjonsdatabase, DMZ-drift, on-demand-data; backend som dataholder.
    
7.  [SEPPmail-dokumentasjon – «Datalagring i clusteret (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Like stor tilleggsdisk per partner, synkronisering av LFT-data til alle noder.
    
8.  [HIN AG – «Fra Mailgateway til Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): HIN-kommunikasjon om etterfølgeren Stargate: desentraliserte noder, Data Mesh-konsept, tidsplan, ende-til-ende-kryptering.
    
9.  [Vereign AG – «Verimesh»](https://vereign.com/): Verimesh (tidligere prosjekt Stargate) i produktiv bruk i det sveitsiske helsevesenet siden april 2025.
    
10. [Vereign GitLab – Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment): Offentlig deployment-repository under AGPLv3 med tjenestene til en Stargate-node og henvisning til [deployment-dokumentasjonen](https://health-info-net-ag.github.io/Stargate-deployment/).
