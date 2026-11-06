---
title: "Säkerhetskopiera HIN Mailgateway och återställ efter ett fel"
navTitle: "Backup och återställning"
description: "Ett kluster skyddar HIN Mailgateway mot fel på en nod, men ersätter inte en säkerhetskopia. Avgörande är konfiguration, nyckelmaterial, återställningsordning och förändringarna med Stargate."
date: "2026-07-08"
kategorie: "HIN-gateway"
timeToRead: "15 min läsning"
themen:
  - hin-gateway
slug: "sakerhetskopiera-och-aterstalla-hin-mailgateway-efter-ett-avbrott"
translationOf: "hin-mailgateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translationReview: automatic
translationSourceHash: cd8f7c8633d8cf882725ad88feed7be5a1730ce563724481e25b4b9ee2ec84e4
translatedAt: 2026-11-06T03:23:51.929Z
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/sakerhetskopiera-och-aterstalla-hin-mailgateway-efter-ett-avbrott
translationModel: gpt-5.6-terra
---

# Säkerhetskopiera HIN Mailgateway och återställ efter ett fel

Många HIN Mailgateways i produktion körs som kluster. Om en nod fallerar tar den andra över. Denna redundans hjälper dock inte mot en felaktig regel, ett raderat certifikat eller en skadad import: [Systemrelevanta data replikeras till alla noder](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), inklusive oönskade ändringar.

För en tillförlitlig återställning behövs därför en separat säkerhetskopia. Eftersom HIN Mailgateway tekniskt bygger på en SEPPmail-appliance med GINA gäller dess dokumenterade mekanismer för säkerhetskopiering och återställning.

## Vilka data som finns på gatewayen

Gatewayen behandlar inkommande och utgående e-post enligt en central regeluppsättning och krypterar beroende på mottagare med S/MIME, OpenPGP eller TLS. För mottagare utan eget nyckelmaterial används det webbaserade GINA-förfarandet. För säkerhetskopieringen är det avgörande att [meddelandeinnehåll inte lagras permanent på gatewayen](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Appliancen behandlar e-post i genomflöde utan att arkivera den.

  

## Vad klustret replikerar

SEPPmail har flera [klustertyper – hög tillgänglighet, lastbalansering och geo-kluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); systemparametrar, användardata och nyckelmaterial synkroniseras över alla noder. I ett [frontend/backend-kluster har frontend ingen egen konfigurationsdatabas](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): det kan drivas i en DMZ utan datalagring och får endast de data som behövs för den aktuella bearbetningen; databasen med nycklarna finns på backend. För [Large File Transfer (LFT) gäller ett undantag](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): varje partner, även frontends, tilldelas en lika stor disk och LFT-data synkroniseras till alla noder.

  

## Varför replikering inte är en säkerhetskopia

> *Replikering kopierar det aktuella tillståndet, även det felaktiga. En säkerhetskopia bevarar ett känt fungerande tillstånd.*

En felaktig import, en raderad nyckel eller en inaktiverad domän replikeras till partnernoden inom sekunder. Utan en oberoende säkerhetskopia finns det därefter ingen återställningspunkt kvar. Hur tätt tillgänglighet och konsistens i klustret hänger samman visade sig vid [inloggningsproblemen efter uppdateringen till 15.0.5](/blog/hin-update-issue-version-15.0.5), som orsakades av störd klusterreplikering.

  

## Vad säkerhetskopian innehåller och inte innehåller

[SEPPmail-säkerhetskopian är medvetet slimmad](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): den omfattar endast konfiguration och kryptografiskt nyckelmaterial: [inga meddelanden, ingen e-postkö och uttryckligen inga loggar](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (loggar ska därför skickas via Syslog till ett externt system). Sedan firmware 14.0.0 skapar appliancen säkerhetskopian [automatiskt på natten vid midnatt som backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); den kan hämtas via `Download`, `Send Backup` (e-post till backupgruppen) eller SCP.

| **Ingår i säkerhetskopian** | **Ingår inte i säkerhetskopian** |
| --- | --- |
| Systemkonfiguration och regeluppsättning | E-postinnehåll / meddelandetexter |
| Användar- och GINA-konton | Aktuell e-postkö |
| Nyckelmaterial: S/MIME, X.509, OpenPGP | System- och e-postloggar (säkerhetskopiera externt via Syslog) |
| TLS- och certifikatkonfiguration | Operativsystem / VM-avbildning |


Det innebär att eftersom operativsystemet inte ingår i konfigurationssäkerhetskopian behöver en fullständig DR-strategi även ett sätt att återställa appliance-grunden (ny utrullning från tillverkaravbildning eller VM-snapshot). Konfigurationssäkerhetskopian kompletterar sedan konfiguration och nycklar.

  

## Snapshots är ingen klustersäkerhetskopia

Sedan firmware 14.0.0 skapar appliancen dessutom [lokala snapshots, men endast om en LFT-partition med databas finns](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). På söndagar skapas en fullständig snapshot och från måndag till lördag en inkrementell snapshot per dag; lagringstiden är 14 dagar.

Avgörande för DR-planeringen: I klusterdrift körs dessa snapshots visserligen i bakgrunden, men ingen återställning erbjuds från dem. Snapshots är därmed ett lokalt rollback-stöd på enskilda system, inte klusteråterställning. Den tillförlitliga säkerhetskopian förblir den krypterade konfigurationssäkerhetskopian.

  

## Konfigurera säkerhetskopiering

Förutsättningen för varje hämtningsmetod är att ett backup-lösenord anges under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); utan detta lösenord går det varken att ladda ned, skicka eller tillhandahålla via SCP. Som standard skickas den nattliga säkerhetskopian via e-post till [gruppen «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); en dedikerad backupanvändare behöver en giltig intern e-postadress.

-   Ange backup-lösenordet och [förvara det separat från säkerhetskopian](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): säkerhetskopian innehåller privata nycklar.
    
-   För automatiserad lagring: [hämta säkerhetskopiorna via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): lagra den publika `SSH-RSA`\-nyckeln i administrationen och hämta den `backup.tgz` som tillhandahålls vid midnatt via OS-användaren `backup`.
    
-   Säkerhetskopiera loggar separat (extern Syslog), eftersom de [medvetet inte är en del av säkerhetskopian](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Säkerhetskopieringsstrategi vid klusterdrift

Vid klusterdrift är ordnad säkerhetskopiering och konsekvent versionshantering avgörande.

-   **Dagligen**: hämta den krypterade konfigurationssäkerhetskopian [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) och lagra den externt med versionshantering
    
-   **Veckovis**: fullständig VM- eller systemsäkerhetskopia av båda noderna, tidsförskjutet i stället för samtidigt (operativsystemet ingår inte i konfigurationssäkerhetskopian)
    
-   **Före underhåll eller uppdatering**: stoppa e-postmottagning via [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): inkommande e-post avvisas då tillfälligt med en konfigurerbar SMTP-returkod (standard `421`); inställningen är aktiv även efter en omstart.
    

  

För versionshantering gäller: SEPPmail uppdaterar i frontend/backend-klustret [frontend före backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), och vid uppdateringar i flera steg måste alla partner ha samma versionsnivå innan nästa release installeras. Efter en större uppdatering kan regeluppsättningen behöva genereras på nytt (meddelandet *«Current ruleset created for another version»*).

  

## Återställning och Disaster Recovery

Grundfallet är enkelt: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), starta om och därefter arbetar gatewayen med full funktionalitet. Observera versionsregeln: Endast säkerhetskopian från den direkt föregående firmware-versionen kan importeras till den aktuella versionen (generera sedan regeluppsättningen på nytt); det går inte att importera en säkerhetskopia från en nyare firmware till en äldre version.

I klustret gäller en viktig begränsning:

-   **Återställ aldrig en enskild nod direkt**: En [återställning av en enskild klusterpartner är inte avsedd](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Ta i stället bort den felaktiga maskinen från klustret, sätt upp en ny VM och lägg till den igen: konfiguration och nycklar kommer automatiskt via replikering från den intakta partnern.
    
-   **Total förlust på alla noder**: Distribuera appliancen på nytt från basavbildningen, importera sedan den senast kända fungerande konfigurationssäkerhetskopian och starta om.
    

En säkerhetskopia är bara så tillförlitlig som det senaste lyckade återställningstestet. En teståterställning bör utföras minst två gånger per år i en isolerad miljö, inte mot produktionsklustret.

  

### Checklista för återställning i skarpt läge

1.  Ta bort den felaktiga noden från klustret (ingen direktåterställning av en partner).
    
2.  Sätt upp en ny VM eller tillhandahåll vid total förlust appliancen från basavbildning/VM-snapshot.
    
3.  Endast vid total förlust: importera den senaste fungerande konfigurationssäkerhetskopian (ha lösenordet till hands och beakta versionsregeln).
    
4.  Kontrollera noden isolerat: SMTP-mottagning, TLS, GINA, regeluppsättning.
    
5.  Lägg till den i klustret och övervaka replikeringen; generera regeluppsättningen på nytt om det anges.
    
6.  Dokumentera incidenten och uppdatera backupintervall och versionsnivåer.
    

  

Två underhållsåtgärder kräver särskild försiktighet och alltid en föregående säkerhetskopia: [utökning av LFT-partitionen stänger av appliancen](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), och fabriksåterställningen skriver över hårddisken tio gånger (säkerhetsfrågan kräver koden i omvänd ordning).

  

## Vad som förändras med «Stargate»

HIN ersätter stegvis det tidigare Mailgateway med [den nya HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (projektet «Stargate», som hos Zug-baserade [Vereign AG kallas «Verimesh»](https://www.vereign.com/)). Det är inte en 1:1-ersättning av appliancen, utan ett arkitekturbyte som i grunden påverkar säkerhetskopiering och Disaster Recovery:

-   **Från centraliserat till decentraliserat**: Noder kommunicerar direkt med varandra; ett centralt distributionscentrum försvinner.
    
-   **Decentraliserad nyckelhantering (DKMS)**: Varje organisation hanterar sin egen kryptografiska identitet, utan central Certificate Authority.
    
-   **End-to-end-kryptering** med fragmentering av meddelanden.
    
-   **Resiliens från nätet**: Om en nod fallerar förblir meshet fungerande.
    
-   **Öppet distributionsrepository**: Repositoriet [Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) med tjänsterna för en Stargate-nod är offentligt tillgängligt under AGPLv3; Vereign Client Library (vcl) kan däremot bara hämtas efter inloggning.
    

Tidsplan: Den decentraliserade infrastrukturen används [i produktion inom schweizisk hälso- och sjukvård sedan april 2025](https://www.vereign.com/); för 2026 planeras den gradvisa ersättningen av de befintliga Mailgateways och en bred utrullning. Organisationer med HIN:s egna domäner (`@hin.ch`, `@verband-hin.ch`) kör på HIN-infrastruktur och påverkas knappt av övergången.

  

För driftmanualen innebär detta att den klassiska disciplinen «exportera appliance-konfiguration och nycklar och återställa dem på en ersättningsnod» blir mindre viktig. I stället kommer nodanslutning, identitets- och nyckelförvaring i meshet samt återanslutning av noder till nätet.

  

## Den viktigaste åtskillnaden

Så länge HIN MGW körs på SEPPmail-teknik gäller följande: Klustret kompenserar för maskinvarufel, men ansvaret för konfigurations- och nyckelintegritet ligger kvar hos operatören. Den slimmade konfigurationssäkerhetskopian ska säkras oberoende av klustret (via SCP, versionshanterad, med separat förvarat lösenord), snapshots ersätter den inte i klustret, versionsnivåerna hålls synkroniserade och återställningen testas regelbundet i en isolerad miljö. Övergången till Stargate bör tidigt inkluderas i DR-planeringen, eftersom den flyttar resiliens och nyckelförvaring till det decentraliserade nätet.

## Ändringshistorik

- **25 oktober 2026:** Tillägg: Vereign Client Library kan nu bara hämtas efter inloggning; som offentligt tillgänglig källa under AGPLv3 anges nu repositoriet Stargate Deployment.

## Källor

1.  [SEPPmail-dokumentation – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): innehåll i säkerhetskopian (endast konfiguration och nyckelmaterial), nattlig skapande, automatisk klusteråterställning via replikering.
    
2.  [SEPPmail-dokumentation – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): utförlig referens: backupmeny (Download / Send Backup / Change password, `backup.tgz` vid midnatt), LFT-snapshots (14 dagar, ingen återställning i klustret), återställningsregler och klusterförfarande, Preempt (SMTP-returkod, standard 421), Device Cloning, uppdateringskanaler och uppdateringsordning (frontend före backend), Factory Reset, bulkimport/-export.
    
3.  [SEPPmail-dokumentation – «Skapa backupanvändare»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): gruppen «backup (Backup operator)», kryptering och lösenordshantering.
    
4.  [SEPPmail-dokumentation – «Kopiera backup via SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): hämta `backup.tgz` via SCP med OS-användaren `backup` i stället för e-postutskick.
    
5.  [SEPPmail-dokumentation – «Kluster / hög tillgänglighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): klustertyper och data som synkroniseras över alla noder (systemparametrar, användardata, nyckelmaterial).
    
6.  [SEPPmail-dokumentation – «Frontend/backend-kluster»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): frontend utan konfigurationsdatabas, DMZ-drift, data vid behov; backend som datahållare.
    
7.  [SEPPmail-dokumentation – «Datalagring i klustret (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): lika stor tilläggsdisk för varje partner, synkronisering av LFT-data till alla noder.
    
8.  [HIN AG – «Från Mailgateway till Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): HIN-kommunikation om efterträdaren Stargate: decentraliserade noder, Data Mesh-koncept, tidsplan, end-to-end-kryptering.
    
9.  [Vereign AG – «Verimesh»](https://vereign.com/): Verimesh (tidigare projekt Stargate) i produktion inom schweizisk hälso- och sjukvård sedan april 2025.
    
10. [Vereign GitLab – Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment): offentligt distributionsrepository under AGPLv3 med tjänsterna för en Stargate-nod och hänvisning till [distributionsdokumentationen](https://health-info-net-ag.github.io/Stargate-deployment/).
