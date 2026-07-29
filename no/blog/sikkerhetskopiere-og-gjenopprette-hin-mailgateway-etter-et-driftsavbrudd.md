---
title: "Sikkerhetskopiere og gjenopprette HIN Mailgateway etter et driftsavbrudd"
navTitle: "Backup og gjenoppretting"
description: "Et cluster beskytter HIN Mailgateway mot nodefeil, men erstatter ikke en sikkerhetskopi. Avgjørende er konfigurasjon, nøkkelmateriale, gjenopprettingsrekkefølge og endringene med Stargate."
date: "2026-07-08"
kategorie: "HIN-gateway"
timeToRead: "15 min lesetid"
themen:
  - hin-gateway
slug: "sikkerhetskopiere-og-gjenopprette-hin-mailgateway-etter-et-driftsavbrudd"
translationOf: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/no/blog/sikkerhetskopiere-og-gjenopprette-hin-mailgateway-etter-et-driftsavbrudd"
translationId: article-845fb4bd0e4c592a
translationReview: automatic
translationSourceHash: 39ecd30339131eb74d0748f4bfb31ead3f98aefbd47621974b1e032f1a96b345
translatedAt: 2026-07-29T12:29:38.971Z
---

# Sikkerhetskopiere og gjenopprette HIN Mailgateway etter et driftsavbrudd

Mange HIN Mailgatewayer i produksjon kjører som cluster. Hvis én node svikter, overtar den andre. Denne redundansen hjelper imidlertid ikke mot en feilaktig regel, et slettet sertifikat eller en ødelagt import: [Systemrelevante data replikeres til alle noder](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), inkludert uønskede endringer.

For en robust gjenoppretting kreves det derfor en separat sikkerhetskopi. Siden HIN Mailgateway teknisk sett er basert på en SEPPmail-appliance med GINA, gjelder de dokumenterte mekanismene for sikkerhetskopiering og gjenoppretting.

## Hvilke data som ligger på gatewayen

Gatewayen behandler inn- og utgående e-post etter et sentralt regelsett og krypterer avhengig av mottaker med S/MIME, OpenPGP eller TLS; for mottakere uten eget nøkkelmateriale brukes den nettbaserte GINA-metoden. For sikkerhetskopien er det avgjørende at [meldingsinnhold ikke lagres persistent på gatewayen](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Appliancen behandler e-poster løpende uten å arkivere dem.

  

## Hva clusteret replikerer

SEPPmail har flere [clustervarianter – høy tilgjengelighet, lastbalansering og geo-cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); systemparametere, brukerdata og nøkkelmateriale synkroniseres på tvers av alle noder. I et [frontend/backend-cluster har frontenden ingen egen konfigurasjonsdatabase](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Den kan driftes i en DMZ uten datalagring og mottar bare dataene som er nødvendige for aktuell behandling; databasen med nøklene ligger på backenden. For [Large File Transfer (LFT) gjelder et unntak](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Hver partner, også frontender, tildeles en disk av samme størrelse, og LFT-data synkroniseres til alle noder.

  

## Hvorfor replikering ikke er en sikkerhetskopi

> *Replikering kopierer gjeldende tilstand, også den feilaktige. En sikkerhetskopi bevarer en kjent, fungerende tilstand.*

En feilaktig import, en slettet nøkkel eller et deaktivert domene replikeres til partnernoden i løpet av sekunder. Uten en uavhengig sikkerhetskopi finnes det deretter ikke noe gjenopprettingspunkt. Hvor tett tilgjengelighet og konsistens henger sammen i clusteret, viste seg ved [innloggingsproblemene etter oppdateringen til 15.0.5](/blog/hin-update-issue-version-15.0.5), som ble utløst av forstyrret clusterreplikering.

  

## Hva sikkerhetskopien inneholder – og ikke inneholder

[SEPPmail-sikkerhetskopien er bevisst slank](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Den omfatter utelukkende konfigurasjon og kryptografisk nøkkelmateriale: [ingen meldinger, ingen e-postkø og uttrykkelig ingen logger](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (logger bør derfor sendes via Syslog til et eksternt system). Siden firmware 14.0.0 oppretter appliancen sikkerhetskopien [automatisk om natten ved midnatt som backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); den kan hentes via `Download`, `Send Backup` (e-post til sikkerhetskopigruppen) eller SCP.

| **Inkludert i sikkerhetskopien** | **Ikke inkludert i sikkerhetskopien** |
| --- | --- |
| Systemkonfigurasjon og regelsett | E-postinnhold / meldingstekster |
| Bruker- og GINA-kontoer | Gjeldende e-postkø |
| Nøkkelmateriale: S/MIME, X.509, OpenPGP | System- og e-postlogger (sikres eksternt via Syslog) |
| TLS- og sertifikatkonfigurasjon | Operativsystem / VM-image |


Dette innebærer: Fordi operativsystemet ikke inngår i konfigurasjonssikkerhetskopien, krever en komplett DR-strategi også en måte å gjenopprette appliance-grunnlaget på (ny utrulling fra produsentens image eller VM-snapshot). Konfigurasjonssikkerhetskopien kompletterer deretter konfigurasjonen og nøklene.

  

## Snapshots er ikke en cluster-sikkerhetskopi

Siden firmware 14.0.0 oppretter appliancen i tillegg [lokale snapshots, men bare dersom det finnes en LFT-partisjon med database](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). På søndager opprettes et komplett snapshot, og mandag til lørdag opprettes ett inkrementelt snapshot per dag; oppbevaringstiden er 14 dager.

Avgjørende for DR-planleggingen: I clusterdrift kjører disse snapshotene riktignok i bakgrunnen, men det tilbys ingen gjenoppretting fra dem. Snapshots er dermed et lokalt tilbakeføringshjelpemiddel på enkeltsystemer, ikke cluster-gjenoppretting. Den pålitelige sikringen forblir den krypterte konfigurasjonssikkerhetskopien.

  

## Konfigurere sikkerhetskopi

Forutsetningen for alle hentemetoder er at et sikkerhetskopipassord er angitt under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); uten dette passordet blir sikkerhetskopien verken lastet ned, sendt eller gjort tilgjengelig via SCP. Som standard sendes den nattlige sikkerhetskopien via e-post til [gruppen «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); en dedikert sikkerhetskopibruker trenger en gyldig intern e-postadresse.

-   Angi sikkerhetskopipassord og [oppbevar det separat fra sikkerhetskopien](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Sikkerhetskopien inneholder private nøkler.
    
-   For automatisert lagring kan du [hente sikkerhetskopiene via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Lagre den offentlige `SSH-RSA`\-nøkkelen i administrasjonen og hent `backup.tgz` som gjøres tilgjengelig ved midnatt, via OS-brukeren `backup`.
    
-   Sikre logger separat (ekstern Syslog), siden de [bevisst ikke er en del av sikkerhetskopien](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Sikkerhetskopistrategi i clusterdrift

I clusterdrift er ordnet sikkerhetskopiering og konsekvent versjonsstyring avgjørende.

-   **Daglig**: Hent den krypterte konfigurasjonssikkerhetskopien [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) og lagre den eksternt med versjonering
    
-   **Ukentlig**: Fullstendig VM- eller systemsikkerhetskopi av begge nodene, tidsforskjøvet i stedet for samtidig (operativsystemet er ikke en del av konfigurasjonssikkerhetskopien)
    
-   **Før vedlikehold eller oppdatering**: Stans e-postmottak via [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Innkommende e-poster blir da midlertidig avvist med en konfigurerbar SMTP-returkode (standard `421`); innstillingen forblir aktiv også etter en omstart.
    

  

Når det gjelder versjonsstyring: I et frontend/backend-cluster oppdaterer SEPPmail [frontenden før backenden](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), og ved oppdateringer i flere trinn må alle partnere ha samme versjonsnivå før de oppgraderes til neste utgave. Etter en hovedoppdatering kan det være nødvendig å generere regelsettet på nytt (melding *«Current ruleset created for another version»*).

  

## Gjenoppretting og katastrofegjenoppretting

Grunntilfellet er enkelt: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), omstart, og deretter arbeider gatewayen med full funksjonalitet. Versjonsregelen må følges: Bare sikkerhetskopien fra den umiddelbart foregående firmware-versjonen kan importeres i den aktuelle versjonen (generer deretter regelsettet på nytt); det er ikke mulig å importere en sikkerhetskopi fra nyere firmware til en eldre versjon.

I clusteret gjelder en viktig begrensning:

-   **Gjenopprett aldri én enkelt node direkte**: En [gjenoppretting av én enkelt clusterpartner er ikke forutsatt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Fjern i stedet den defekte maskinen fra clusteret, opprett en ny VM og legg den til igjen: Konfigurasjon og nøkler kommer automatisk via replikering fra den intakte partneren.
    
-   **Totalt tap på alle noder**: Rull ut appliancen på nytt fra basis-imaget, importer deretter den sist kjente fungerende konfigurasjonssikkerhetskopien og start på nytt.
    

En sikkerhetskopi er bare så pålitelig som den siste vellykkede gjenopprettingstesten. En testgjenoppretting bør utføres minst to ganger i året i et isolert miljø, ikke mot produksjonsclusteret.

  

### Sjekkliste for gjenoppretting i en nødsituasjon

1.  Fjern den defekte noden fra clusteret (ingen direkte gjenoppretting av en partner).
    
2.  Opprett en ny VM eller, ved totalt tap, klargjør appliancen fra basis-image/VM-snapshot.
    
3.  Bare ved totalt tap: Importer siste fungerende konfigurasjonssikkerhetskopi (ha passordet klart, og følg versjonsregelen).
    
4.  Kontroller noden isolert: SMTP-mottak, TLS, GINA, regelsett.
    
5.  Legg den inn i clusteret og overvåk replikeringen; generer regelsettet på nytt ved melding.
    
6.  Dokumenter hendelsen og oppdater sikkerhetskopiintervallet og versjonsnivåene.
    

  

To vedlikeholdsoperasjoner krever særlig forsiktighet og alltid en forhåndssikkerhetskopi: [Utvidelse av LFT-partisjonen slår av appliancen](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), og fabrikktilbakestilling overskriver harddisken ti ganger (sikkerhetsspørsmålet krever koden skrevet baklengs).

  

## Hva som endres med «Stargate»

HIN erstatter gradvis den nåværende Mailgatewayen med den [nye HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (prosjekt «Stargate», omtalt som «Verimesh» hos Zug-baserte [Vereign AG](https://www.vereign.com/)). Dette er ikke en 1:1-erstatning av appliancen, men et arkitekturskifte som i hovedsak berører sikkerhetskopiering og katastrofegjenoppretting:

-   **Fra sentralisert til desentralisert**: Noder kommuniserer direkte med hverandre; et sentralt distribusjonssenter bortfaller.
    
-   **Desentralisert nøkkeladministrasjon (DKMS)**: Hver organisasjon administrerer sin egen kryptografiske identitet, uten en sentral Certificate Authority.
    
-   **Ende-til-ende-kryptering** med fragmentering av meldingene.
    
-   **Robusthet fra nettet**: Hvis en node svikter, forblir meshet funksjonelt.
    
-   **Åpen referanseimplementering**: [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) kan undersøkes som åpen kildekode under AGPLv3.
    

Tidsplan: Den desentraliserte infrastrukturen har vært [i produktiv drift i det sveitsiske helsevesenet siden april 2025](https://www.vereign.com/); for 2026 er det planlagt en gradvis utfasing av de nåværende Mailgatewayene og bred utrulling. Organisasjoner med HIN-egne domener (`@hin.ch`, `@verband-hin.ch`) kjører på HIN-infrastruktur og påvirkes i liten grad av overgangen.

  

For driftshåndboken betyr dette at den klassiske disiplinen «eksportere appliance-konfigurasjon og nøkler og gjenopprette dem på en erstatningsnode» blir mindre viktig. I stedet kommer node-registrering, forvaltning av identiteter og nøkler i meshet samt gjenopptakelse av noder i nettet.

  

## Det viktigste skillet

Så lenge HIN MGW kjører på SEPPmail-teknologi, gjelder følgende: Clusteret kompenserer for maskinvarefeil, men ansvaret for integriteten til konfigurasjon og nøkler ligger fortsatt hos operatøren. Den slanke konfigurasjonssikkerhetskopien må sikres uavhengig av clusteret (via SCP, versjonert, med separat oppbevart passord), snapshots erstatter den ikke i clusteret, versjonsnivåene holdes synkronisert, og gjenoppretting testes regelmessig isolert. Overgangen til Stargate bør inngå tidlig i DR-planleggingen, siden den flytter robusthet og nøkkelforvaltning til det desentraliserte nettet.

## Kilder

1.  [SEPPmail-dokumentasjon – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Innhold i sikkerhetskopien (kun konfigurasjon og nøkkelmateriale), nattlig opprettelse, automatisk cluster-gjenoppretting via replikering.
    
2.  [SEPPmail-dokumentasjon – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Utførlig referanse: Sikkerhetskopimeny (Download / Send Backup / Change password, `backup.tgz` ved midnatt), LFT-snapshots (14 dager, ingen gjenoppretting i cluster), gjenopprettingsregler og clusterprosedyre, Preempt (SMTP-returkode, standard 421), Device Cloning, oppdateringskanaler og oppdateringsrekkefølge (frontend før backend), Factory Reset, bulkimport/-eksport.
    
3.  [SEPPmail-dokumentasjon – «Opprette sikkerhetskopibruker»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Gruppen «backup (Backup operator)», kryptering og passordadministrasjon.
    
4.  [SEPPmail-dokumentasjon – «Kopiere sikkerhetskopi via SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Henting av `backup.tgz` via SCP gjennom OS-brukeren `backup` i stedet for e-postsending.
    
5.  [SEPPmail-dokumentasjon – «Cluster / høy tilgjengelighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Clustertyper og dataene som synkroniseres på tvers av alle noder (systemparametere, brukerdata, nøkkelmateriale).
    
6.  [SEPPmail-dokumentasjon – «Frontend/backend-cluster»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Frontend uten konfigurasjonsdatabase, DMZ-drift, data ved behov; backend som dataholder.
    
7.  [SEPPmail-dokumentasjon – «Datalagring i clusteret (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Ekstra disk av samme størrelse for hver partner, synkronisering av LFT-data til alle noder.
    
8.  [HIN AG – «Fra Mailgateway til Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): HIN-kommunikasjon om etterfølgeren Stargate: desentraliserte noder, Data Mesh-konsept, tidsplan, ende-til-ende-kryptering.
    
9.  [Vereign AG – «Verimesh» / Vereign Client Library (vcl, tagg 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1): Teknisk grunnlag for Stargate: desentralisert nøkkeladministrasjon (DKMS), ende-til-ende-kryptering med meldingsfragmentering, åpen kildekode under AGPLv3.
