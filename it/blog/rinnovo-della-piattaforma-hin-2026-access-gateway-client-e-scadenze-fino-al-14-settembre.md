---
title: "Rinnovo della piattaforma HIN 2026: Access Gateway, Client e scadenze fino al 26 ottobre"
navTitle: "Access Gateway 2026"
description: "Abilitazione firewall entro il 31 agosto, Access Gateway versione 4 da settembre, endpoint SAML entro il 12 ottobre, token hardware e HIN Client 4.0 entro il 26 ottobre. Il mail gateway non è interessato e sarà sostituito separatamente."
date: "2026-08-01"
kategorie: "HIN Gateway"
timeToRead: "5 min di lettura"
themen:
  - hin-gateway
  - active-directory-entra
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "rinnovo-della-piattaforma-hin-2026-access-gateway-client-e-scadenze-fino-al-14-settembre"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
translationSourceHash: cd34e6297afca9491bd965e40dbaf55b920296e62e57726754824474fcb27f0d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:46:15.175Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/rinnovo-della-piattaforma-hin-2026-access-gateway-client-e-scadenze-fino-al-14-settembre
---

# Rinnovo della piattaforma HIN 2026: Access Gateway, Client e scadenze fino al 26 ottobre

Nel 2026 HIN rinnova la piattaforma per identità e accesso. Il 27 agosto 2026 HIN ha ridefinito le scadenze: la prima è scaduta il 31 agosto 2026, i vecchi endpoint SAML non sono più utilizzabili dal 12 ottobre 2026 e il grande passaggio avverrà il 26 ottobre 2026.

**Sono interessati HIN Access Gateway (AGW), HIN Client e i mezzi di autenticazione. HIN Mailgateway non è interessato.** Sarà anch'esso sostituito, ma nell'ambito di un progetto separato con una propria tabella di marcia.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">La vostra situazione</span>
    <span class="choice__title">Solo AGW in funzione</span>
    <span class="choice__hint">Le scadenze riportate sotto rappresentano tutte le azioni necessarie. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">La vostra situazione</span>
    <span class="choice__title">Ulteriore necessità di migrazione per il Mailgateway</span>
    <span class="choice__hint">È quindi prevista anche la sostituzione con «Stargate», con rollout esteso dal terzo trimestre 2026. Verifica gratuita del vostro ambiente. →</span>
  </a>
</div>

## Le scadenze

| Data | Misura | Riguarda |
|---|---|---|
| 31.08.2026 | Abilitazione firewall per `idp.id.hin.ch` e `cdn.hin.ch` (443/tcp) | Gestori AGW |
| da settembre 2026 | Installazione automatica di AGW versione 4 (escluse le installazioni cluster) | Gestori AGW |
| da settembre al 26.10.2026 | Aggiornamento automatico a HIN Client 4.0 in più tranche | Tutti gli utenti Client |
| 12.10.2026 | I vecchi endpoint SAML non sono più utilizzabili | Federazioni, collegamenti EPD |
| 26.10.2026 | Scadono i token hardware e le identità di test per l'accesso EPD | Utenti di token, integrazione |
| 26.10.2026 | L'attuale Authenticator App utilizzabile per l'ultima volta; nuova configurazione dal 27.10.2026 | Utenti dell'app |
| 26.10.2026 | HIN Client 4.0 obbligatorio | Tutti gli utenti Client |
| dall'inizio di novembre 2026 | Accesso tramite browser in HIN Client | Tutti gli utenti Client |

## Access Gateway non è Mailgateway

Entrambi hanno Gateway nel nome e vengono regolarmente confusi. Access Gateway controlla l'accesso alle applicazioni protette da HIN e non riguarda il traffico e-mail. Mailgateway è collocato nel percorso della posta e cifra i messaggi.

## Access Gateway: firewall e versione 4

Entro il 31 agosto 2026 l'AGW doveva poter raggiungere `idp.id.hin.ch` e `cdn.hin.ch` tramite 443/tcp. HIN raccomanda di impostare la regola sui nomi host anziché sugli indirizzi IP, poiché questi ultimi possono cambiare. Si tratta di una modifica del firewall, non di un'impostazione nel Gateway, e quindi spesso è di competenza del responsabile della rete anziché dell'amministratore del Gateway.

Da settembre 2026 la versione 4 viene installata automaticamente; le installazioni cluster sono escluse e HIN contatta direttamente questi clienti. Requisiti: AGW alla versione 3.1.50 o superiore e Kerberos attivo come metodo di autenticazione. Per il collegamento ad Active Directory è necessario un account LDAP con diritti di lettura.

Chi non soddisfa i requisiti non viene aggiornato e, per esperienza, ci si accorge del problema solo quando nessuno riesce più ad accedere. È quindi meglio verificare ora la versione installata, anziché farlo durante un'interruzione.

## SAML: nuovi endpoint, meno attributi

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Con il passaggio cambiano i formati degli attributi e i binding. L'insieme di attributi viene ridotto a GLN, nome, data di nascita e sesso.

È qui che le integrazioni falliscono. Ogni applicazione che utilizza ulteriori attributi per ruoli o separazione dei mandanti non li riceverà più tramite i nuovi endpoint; quelli vecchi non sono più utilizzabili dal 12 ottobre 2026. L'errore non si manifesta come errore di accesso, bensì come autorizzazione mancante nel sistema di destinazione.

Le identità di test per l'accesso EPD perdono validità il 26 ottobre 2026 e devono essere richieste nuovamente. Per l'integrazione EPD, HIN richiede identità verificate tramite videoidentificazione; non è ancora disponibile un ambiente di test. Dal 26 ottobre 2026 possono inoltre arrivare assertion senza GLN.

Chi gestisce una federazione gestisce quasi sempre anche una propria infrastruttura e-mail. Per queste organizzazioni, il rinnovo della piattaforma coincide nello stesso anno con la [sostituzione del Mailgateway con «Stargate»](/stargate): tecnicamente indipendenti, ma in competizione per le stesse persone e finestre di manutenzione.

## Token, app e HIN Client 4.0

I token hardware non vengono più emessi e scadono il 26 ottobre 2026. Alternative: HIN Client, codice SMS o Authenticator App. L'attuale Authenticator App può essere utilizzata fino al 26 ottobre 2026; la nuova app potrà essere configurata come secondo fattore nel portale self-service dal 27 ottobre 2026. HIN raccomanda di configurare prima l'accesso con password e codice SMS come accesso di emergenza.

HIN Client viene aggiornato automaticamente alla versione 4.0, a tranche, da settembre al 26 ottobre 2026. Se il 26 ottobre il Client non mostra la versione 4.0, occorre installarlo manualmente tramite `download.hin.ch`. L'accesso tramite browser seguirà dall'inizio di novembre 2026.

Il punto critico sono i requisiti di sistema: **la versione 4.0 richiede Windows 11 con TPM o macOS 14**, e per i Mac Intel anche un Security Chip; senza TPM resta possibile l'accesso con codice SMS. I dispositivi più vecchi devono essere aggiornati o sostituiti in anticipo. Per una parte degli studi, la scadenza non rappresenta quindi un compito software, bensì di approvvigionamento. Chi se ne accorge poco prima del 26 ottobre dovrà fare i conti con tempi di consegna e reinstallazione del software gestionale dello studio.

## Cinque domande per fare il punto

1. Quale versione AGW è in esecuzione e Kerberos è attivo?
2. Il firewall consente in uscita `idp.id.hin.ch`?
3. Quante postazioni di lavoro utilizzano ancora Windows 10 o macOS 13 e versioni precedenti?
4. Quanti token hardware sono in uso e a cosa passeranno le persone interessate?
5. Un'applicazione utilizza attributi HIN che in futuro non saranno più disponibili?

Le risposte alle domande 3 e 5 determinano l'impegno necessario. Il resto si completa in poche ore ed è documentato da HIN.

## Il secondo progetto: «Stargate»

Indipendentemente da ciò, HIN sostituisce Mailgateway con il nuovo HIN Gateway, internamente denominato progetto «Stargate», tecnicamente un approccio Data Mesh con crittografia end-to-end e gestione decentralizzata delle chiavi. Non si tratta quindi della sostituzione dell'appliance, ma di un cambio di architettura.

L'impegno si colloca quindi su un piano completamente diverso. Il rinnovo della piattaforma richiede soprattutto il rispetto delle scadenze per una regola firewall, una versione e una sostituzione di dispositivi, mentre con Stargate è in discussione il percorso produttivo della posta stesso: l'insieme di regole cresciuto nel tempo, il materiale delle chiavi, la gestione dei destinatari senza identità HIN e la questione di cosa fare in caso qualcosa non funzioni come previsto. Poiché la migrazione si svolge in finestre di quattro ore prenotate e HIN raccomanda un mese di preparazione, un appuntamento del genere non ammette questioni aperte.

<aside class="offer-box">
  <span class="offer-box__tag">Verifica gratuita</span>
  <p><strong>Non dovete sapere a che punto siete. È proprio a questo che serve la verifica.</strong> Esamino il vostro ambiente Gateway esistente e vi dico cosa occorre fare prima della finestra di migrazione, indipendentemente dal fatto che migriate autonomamente o con assistenza in seguito.</p>
  <a class="offer-box__cta" href="/stargate">Registratevi ora</a>
</aside>

## Cronologia delle modifiche

- **25 ottobre 2026:** Scadenze aggiornate secondo la ridefinizione di HIN del 27 agosto 2026: abilitazione firewall (ora anche per cdn.hin.ch) entro il 31 agosto, vecchi endpoint SAML entro il 12 ottobre, token hardware, attuale Authenticator App e HIN Client 4.0 entro il 26 ottobre 2026. Aggiunti: requisito TPM per HIN Client 4.0 e accesso tramite browser dall'inizio di novembre 2026.

## Fonti

1.  [Rinnovo della piattaforma HIN: questi adeguamenti tecnici sono necessari per i membri HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): scadenze da agosto a ottobre 2026 (situazione al 27 agosto 2026), endpoint SAML, insieme di attributi ridotto, abilitazioni firewall, Authenticator App e token hardware.

2.  [Il nuovo HIN Client è arrivato: cosa cambia per i membri HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): versione 4.0, rollout entro il 26 ottobre 2026, requisiti del sistema operativo e TPM, accesso basato su browser da novembre 2026.

3.  [HIN Gateway: comunicazione sicura all'interno della HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): sostituzione di Mailgateway, architettura, modelli operativi, migrazione in finestre temporali prenotate.

4.  [Configurazione di HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): ruolo dell'AGW nella gestione degli accessi.

5.  [Collegamento ad Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos e l'account LDAP con diritti di lettura.

6.  [HIN AG: «Dal Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): contesto su «Stargate», nodi decentralizzati, calendario.
