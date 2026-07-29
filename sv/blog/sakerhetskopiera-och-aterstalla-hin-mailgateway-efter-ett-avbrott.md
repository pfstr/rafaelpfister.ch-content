---
title: "Säkerhetskopiera och återställa HIN Mailgateway efter ett avbrott"
navTitle: "Backup och återställning"
description: "Ett kluster skyddar HIN Mailgateway mot bortfall av en nod, men ersätter inte en säkerhetskopia. Avgörande är konfiguration, nyckelmaterial, återställningsordning och förändringarna genom Stargate."
date: "2026-07-08"
kategorie: "HIN-gateway"
timeToRead: "15 min lästid"
themen:
  - "hin-gateway"
slug: "sakerhetskopiera-och-aterstalla-hin-mailgateway-efter-ett-avbrott"
translationOf: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/sv/blog/sakerhetskopiera-och-aterstalla-hin-mailgateway-efter-ett-avbrott"
---

# Säkerhetskopiera och återställa HIN Mailgateway efter ett avbrott

Många HIN Mailgateways i produktion körs som kluster. Om en nod faller bort tar den andra över. Denna redundans hjälper dock inte mot en felaktig regel, ett raderat certifikat eller en skadad import: [Systemrelevanta data replikeras till alla noder](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), inklusive oönskade ändringar.

För en robust återställning krävs därför en separat säkerhetskopia. Eftersom HIN Mailgateway tekniskt bygger på en SEPPmail-appliance med GINA gäller dess dokumenterade mekanismer för backup och återställning.

## Vilka data som finns på gatewayen

Gatewayen behandlar inkommande och utgående e-post enligt ett centralt regelverk och krypterar beroende på mottagare med S/MIME, OpenPGP eller TLS; för mottagare utan eget nyckelmaterial används det webbaserade GINA-förfarandet. För säkerhetskopian är det avgörande att [meddelandeinnehåll inte lagras persistent på gatewayen](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Appliancen behandlar e-post i genomflöde utan att arkivera den.

  

## Vad klustret replikerar

SEPPmail har flera [klustervarianter – hög tillgänglighet, lastbalansering och geo-kluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); systemparametrar, användardata och nyckelmaterial synkroniseras över alla noder. I [frontend/backend-klustret har frontend ingen egen konfigurationsdatabas](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Den kan drivas i en DMZ utan datalagring och får endast de data som behövs för den aktuella behandlingen; databasen med nycklar finns på backend. För [Large File Transfer (LFT) gäller ett undantag](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Varje partner, inklusive frontends, tilldelas en lika stor disk och LFT-data synkroniseras till alla noder.

  

## Varför replikering inte är backup

> *Replikering kopierar det aktuella tillståndet, även det felaktiga. En säkerhetskopia bevarar ett känt fungerande tillstånd.*

En felaktig import, en raderad nyckel eller en inaktiverad domän replikeras till partnernoden inom några sekunder. Utan en oberoende säkerhetskopia finns därefter ingen återställningspunkt kvar. Hur tätt tillgänglighet och konsistens hänger ihop i klustret visades av [inloggningsproblemen efter uppdateringen till 15.0.5](/blog/hin-update-issue-version-15.0.5), som orsakades av störd klusterreplikering.

  

## Vad som ingår i säkerhetskopian och vad som inte gör det

[SEPPmail-säkerhetskopian är medvetet slimmad](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Den omfattar endast konfiguration och kryptografiskt nyckelmaterial: [inga meddelanden, ingen e-postkö och uttryckligen inga loggar](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (loggar ska därför skickas till ett externt system via Syslog). Sedan firmware 14.0.0 skapar appliancen säkerhetskopian [automatiskt varje natt vid midnatt som backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); den kan hämtas via `Download`, `Send Backup` (e-post till backupgruppen) eller SCP.

| **Ingår i säkerhetskopian** | **Ingår inte i säkerhetskopian** |
| --- | --- |
| Systemkonfiguration och regelverk | E-postinnehåll / meddelandetexter |
| Användar- och GINA-konton | Aktuell e-postkö |
| Nyckelmaterial: S/MIME, X.509, OpenPGP | System- och e-postloggar (säkerhetskopiera externt via Syslog) |
| TLS- och certifikatkonfiguration | Operativsystem / VM-avbildning |


Detta innebär: Eftersom operativsystemet inte ingår i konfigurationssäkerhetskopian behöver en fullständig DR-strategi dessutom ett sätt att återställa appliance-basen (nyinstallation från tillverkarens avbildning eller VM-snapshot). Konfigurationssäkerhetskopian kompletterar sedan konfiguration och nycklar.

  

## Snapshots är ingen klusterbackup

Sedan firmware 14.0.0 skapar appliancen även [lokala snapshots, men endast om det finns en LFT-partition med databas](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). På söndagar skapas en fullständig snapshot och måndag till lördag en inkrementell snapshot per dag; lagringstiden är 14 dagar.

Avgörande för DR-planeringen: I klusterdrift körs dessa snapshots visserligen i bakgrunden, men ingen återställning erbjuds från dem. Snapshots är därför ett lokalt rollback-stöd på enskilda system, inte klusteråterställning. Den tillförlitliga säkerhetskopian förblir den krypterade konfigurationssäkerhetskopian.

  

## Konfigurera backup

Förutsättningen för varje hämtningsmetod är att ett backup-lösenord har angetts under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); utan detta lösenord kan säkerhetskopian varken laddas ned, skickas eller göras tillgänglig via SCP. Som standard skickas den nattliga säkerhetskopian via e-post till [gruppen «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); en dedikerad backupanvändare behöver en giltig intern e-postadress.

-   Ange backup-lösenordet och [förvara det separat från säkerhetskopian](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Säkerhetskopian innehåller privata nycklar.
    
-   För automatiserad lagring, [hämta säkerhetskopiorna via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): lagra den offentliga `SSH-RSA`\-nyckeln i administrationen och hämta den `backup.tgz` som görs tillgänglig vid midnatt via OS-användaren `backup`.
    
-   Säkerhetskopiera loggar separat (extern Syslog), eftersom de [avsiktligt inte är en del av säkerhetskopian](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Backupstrategi vid klusterdrift

Vid klusterdrift är en ordnad säkerhetskopiering och konsekvent versionshantering avgörande.

-   **Dagligen**: hämta den krypterade konfigurationssäkerhetskopian [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) och lagra den externt med versionshantering
    
-   **Veckovis**: fullständig VM- eller systemsäkerhetskopia av båda noderna, tidsförskjutet i stället för samtidigt (operativsystemet ingår inte i konfigurationssäkerhetskopian)
    
-   **Före underhåll eller uppdatering**: stoppa e-postmottagning via [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Inkommande e-post avvisas då tillfälligt med en konfigurerbar SMTP-returkod (standard `421`); inställningen förblir aktiv även efter en omstart.
    

  

För versionshantering gäller: SEPPmail uppdaterar i frontend/backend-klustret [frontend före backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), och vid uppdateringar i flera steg måste alla partner ha samma versionsnivå innan nästa release installeras. Efter en större uppdatering kan regelverket behöva genereras om (meddelandet *«Current ruleset created for another version»*).

  

## Återställning och disaster recovery

Grundfallet är enkelt: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), omstart, därefter fungerar gatewayen med full funktionalitet. Versionsregeln måste beaktas: Endast säkerhetskopian från den omedelbart föregående firmwareversionen kan importeras till den aktuella versionen (generera sedan regelverket på nytt); det går inte att importera säkerhetskopian från en nyare firmware till en äldre version.

I klustret gäller en viktig begränsning:

-   **Återställ aldrig en enskild nod direkt**: En [återställning av en enskild klusterpartner är inte avsedd](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Ta i stället bort den defekta maskinen från klustret, sätt upp en ny VM och lägg till den igen: Konfiguration och nycklar kommer automatiskt via replikering från den intakta partnern.
    
-   **Totalförlust på alla noder**: Installera om appliancen från basavbildningen, importera därefter den senaste kända fungerande konfigurationssäkerhetskopian och starta om.
    

En säkerhetskopia är endast så tillförlitlig som det senaste lyckade återställningstestet. En teståterställning bör genomföras minst två gånger per år i en isolerad miljö, inte mot produktionsklustret.

  

### Checklista för återställning i nödfall

1.  Ta bort den defekta noden från klustret (ingen direktåterställning av en partner).
    
2.  Sätt upp en ny VM eller, vid totalförlust, tillhandahåll appliancen från basavbildning/VM-snapshot.
    
3.  Endast vid totalförlust: importera den senaste fungerande konfigurationssäkerhetskopian (ha lösenordet tillgängligt och beakta versionsregeln).
    
4.  Kontrollera noden isolerat: SMTP-mottagning, TLS, GINA, regelverk.
    
5.  Lägg till den i klustret och övervaka replikeringen; generera om regelverket om ett meddelande visas.
    
6.  Dokumentera incidenten och justera backupintervall och versionsnivåer.
    

  

Två underhållsåtgärder kräver särskild försiktighet och alltid en föregående säkerhetskopia: [Utökning av LFT-partitionen stänger av appliancen](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), och fabriksåterställningen skriver över hårddisken tio gånger (säkerhetsfrågan kräver koden i omvänd stavning).

  

## Vad som förändras med «Stargate»

HIN ersätter stegvis den nuvarande Mailgatewayen med den [nya HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (projektet «Stargate», som hos Zug-baserade [Vereign AG benämns «Verimesh»](https://www.vereign.com/)). Det är inte en 1:1-ersättning av appliancen, utan ett arkitekturbyte som i grunden påverkar backup och disaster recovery:

-   **Från centralt till decentraliserat**: Noder kommunicerar direkt med varandra; ett centralt distributionscentrum försvinner.
    
-   **Decentraliserad nyckelhantering (DKMS)**: Varje organisation hanterar sin egen kryptografiska identitet, utan central Certificate Authority.
    
-   **End-to-end-kryptering** med fragmentering av meddelanden.
    
-   **Resiliens från nätverket**: Om en nod faller bort förblir meshet funktionellt.
    
-   **Öppen referensimplementation**: [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) är öppen källkod under AGPLv3.
    

Tidsplan: Den decentraliserade infrastrukturen är [i produktiv drift inom den schweiziska hälso- och sjukvården sedan april 2025](https://www.vereign.com/); för 2026 planeras den stegvisa ersättningen av de nuvarande Mailgateways och en bred utrullning. Organisationer med HIN-egna domäner (`@hin.ch`, `@verband-hin.ch`) kör på HIN-infrastruktur och påverkas knappt av övergången.

  

För drifthandboken innebär detta: Den klassiska disciplinen att «exportera appliance-konfiguration och nycklar och återställa dem på en ersättningsnod» blir mindre viktig. I stället kommer node enrollment, identitets- och nyckelförvaring i meshet samt återanslutning av noder till nätverket.

  

## Den viktigaste åtskillnaden

Så länge HIN MGW körs på SEPPmail-teknik gäller följande: Klustret kompenserar för maskinvarufel, men ansvaret för konfigurations- och nyckelintegritet ligger kvar hos operatören. Den slimmade konfigurationssäkerhetskopian måste säkras oberoende av klustret (via SCP, versionshanterad, med lösenordet förvarat separat), snapshots ersätter den inte i klustret, versionsnivåerna hålls synkroniserade och återställningen testas regelbundet isolerat. Övergången till Stargate bör tidigt inkluderas i DR-planeringen, eftersom den flyttar resiliens och nyckelförvaring till det decentraliserade nätverket.

## Källor

1.  [SEPPmail-dokumentation – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Backupinnehåll (endast konfiguration och nyckelmaterial), nattlig skapande, automatisk klusteråterställning via replikering.
    
2.  [SEPPmail-dokumentation – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Utförlig referens: backupmeny (Download / Send Backup / Change password, `backup.tgz` vid midnatt), LFT-snapshots (14 dagar, ingen återställning i klustret), återställningsregler och klusterförfarande, Preempt (SMTP-returkod, standard 421), Device Cloning, uppdateringskanaler och uppdateringsordning (frontend före backend), Factory Reset, massimport/-export.
    
3.  [SEPPmail-dokumentation – «Skapa backupanvändare»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Gruppen «backup (Backup operator)», kryptering och lösenordshantering.
    
4.  [SEPPmail-dokumentation – «Kopiera backup via SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Hämta `backup.tgz` via SCP med OS-användaren `backup` i stället för e-postutskick.
    
5.  [SEPPmail-dokumentation – «Kluster / hög tillgänglighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Klustertyper och data som synkroniseras över alla noder (systemparametrar, användardata, nyckelmaterial).
    
6.  [SEPPmail-dokumentation – «Frontend/backend-kluster»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Frontend utan konfigurationsdatabas, DMZ-drift, data vid behov; backend som datahållare.
    
7.  [SEPPmail-dokumentation – «Datalagring i klustret (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Lika stor extra disk per partner, synkronisering av LFT-data till alla noder.
    
8.  [HIN AG – «Från Mailgateway till Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): HIN-kommunikation om efterträdaren Stargate: decentraliserade noder, Data Mesh-koncept, tidsplan, end-to-end-kryptering.
    
9.  [Vereign AG – «Verimesh» / Vereign Client Library (vcl, tagg 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1): Teknisk grund för Stargate: decentraliserad nyckelhantering (DKMS), end-to-end-kryptering med meddelandefragmentering, öppen källkod under AGPLv3.
