---
title: "Rinnovo della piattaforma HIN 2026: Access Gateway, Client e scadenze fino al 14 settembre"
navTitle: "Access Gateway 2026"
description: "Abilitazione del firewall entro il 14 agosto, Access Gateway versione 4 dal 17 agosto, endpoint SAML, token hardware e HIN Client entro il 14 settembre. Il mail gateway non è interessato e sarà sostituito separatamente."
date: "2026-08-01"
kategorie: "HIN-Gateway"
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
url: https://rafaelpfister.ch/it/blog/rinnovo-della-piattaforma-hin-2026-access-gateway-client-e-scadenze-fino-al-14-settembre
translationSourceHash: 1a174bd131b8bb29f9b1e1e793d4cf19b3f732e3c6fd779a25193f151ec8c109
translationModel: gpt-5.6-terra
translatedAt: 2026-08-02T06:16:20.439Z
translationReview: automatic
---

# Rinnovo della piattaforma HIN 2026: Access Gateway, Client e scadenze fino al 14 settembre

Nel 2026 HIN rinnova la piattaforma per identità e accesso. La prima scadenza è il 14 agosto 2026, mentre la grande transizione seguirà il 14 settembre 2026.

**Sono interessati HIN Access Gateway (AGW), HIN Client e i metodi di autenticazione. HIN Mailgateway non è interessato.** Sarà anch'esso sostituito, ma nell'ambito di un progetto separato con una propria tabella di marcia.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">La vostra situazione</span>
    <span class="choice__title">Solo AGW in esercizio</span>
    <span class="choice__hint">Le scadenze riportate sotto comprendono tutte le azioni necessarie. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">La vostra situazione</span>
    <span class="choice__title">Necessità di migrazione aggiuntiva per il mail gateway</span>
    <span class="choice__hint">È quindi imminente anche la sostituzione con «Stargate», con rollout su larga scala dal terzo trimestre 2026. Verifica gratuita del vostro ambiente. →</span>
  </a>
</div>

## Le scadenze

| Data | Misura | Riguarda |
|---|---|---|
| 14.08.2026 | Abilitazione del firewall per `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | Gestori AGW |
| 17.08.2026 | Installazione automatica di AGW versione 4 | Gestori AGW |
| da metà agosto | Consigliata l'installazione manuale di HIN Client 4.0 | Tutti gli utenti Client |
| 14.09.2026 | Endpoint SAML migrati | Federazioni, collegamenti EPD |
| 14.09.2026 | Scadenza di token hardware e identità di test | Utenti di token, integrazioni |
| 14.09.2026 | Nuova configurazione dell'app Authenticator | Utenti dell'app |
| 14.09.2026 | Aggiornamento forzato a HIN Client 4.0 | Tutti gli utenti Client |

## Access Gateway non è Mailgateway

Entrambi hanno Gateway nel nome e vengono regolarmente confusi. Access Gateway controlla l'accesso alle applicazioni protette da HIN e non riguarda il traffico e-mail. Mailgateway si trova nel percorso della posta e cifra i messaggi.

## Access Gateway: firewall e versione 4

Entro il 14 agosto l'AGW deve poter raggiungere `idp.id.hin.ch`. Si tratta di una modifica del firewall, non di un'impostazione nel Gateway, e quindi spesso compete al responsabile della rete anziché all'amministratore del Gateway.

Dal 17 agosto verrà installata automaticamente la versione 4. Requisiti: AGW alla versione 3.1.50 o successiva e Kerberos attivo come metodo di autenticazione. Per il collegamento ad Active Directory è necessario un account LDAP con diritti di lettura.

Chi non soddisfa i requisiti non verrà aggiornato, e l'esperienza dimostra che questo si nota solo quando nessuno riesce più ad accedere. È quindi meglio verificare ora la versione anziché a settembre.

## SAML: nuovi endpoint, meno attributi

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Con il passaggio cambiano i formati degli attributi e i binding. L'insieme degli attributi viene ridotto a GLN, nome, data di nascita e sesso.

Questo è il punto che interrompe le integrazioni. Ogni applicazione che utilizza altri attributi per ruoli o separazione dei mandanti non li riceverà più dopo il 14 settembre. L'errore non si manifesta come errore di accesso, ma come autorizzazione mancante nel sistema di destinazione.

Le identità di test scadono nella stessa data; chi desidera quindi sperimentare la transizione in un ambiente di integrazione dovrebbe farlo prima.

Chi gestisce una federazione gestisce quasi sempre anche una propria infrastruttura e-mail. Per queste organizzazioni, il rinnovo della piattaforma coincide nello stesso anno con la [sostituzione del Mailgateway con «Stargate»](/stargate): tecnicamente indipendenti, ma in competizione per le stesse persone e finestre di manutenzione.

## Token, app e HIN Client 4.0

I token hardware non vengono più emessi e scadono il 14 settembre. Alternative: HIN Client, codice SMS o app Authenticator. L'app stessa resta valida fino al 14 settembre e dovrà poi essere nuovamente configurata tramite il portale self-service.

HIN Client verrà aggiornato automaticamente alla versione 4.0 entro e non oltre il 14 settembre; l'installazione manuale è disponibile da metà agosto tramite `download.hin.ch`. Il login avverrà ora tramite browser.

Il punto critico sono i requisiti di sistema: **la versione 4.0 richiede Windows 11 o macOS 14.** I dispositivi più vecchi devono essere aggiornati o sostituiti prima. Per una parte degli studi, la scadenza non è quindi una questione software, ma di approvvigionamento. Chi se ne accorge solo a settembre dovrà fare i conti con tempi di consegna e reinstallazione del software dello studio.

## Cinque domande per fare il punto

1. Quale versione AGW è in esecuzione e Kerberos è attivo?
2. Il firewall consente in uscita `idp.id.hin.ch`?
3. Quante postazioni di lavoro utilizzano ancora Windows 10 o macOS 13 e versioni precedenti?
4. Quanti token hardware sono in uso e a cosa passeranno gli utenti interessati?
5. Un'applicazione utilizza attributi HIN che in futuro verranno eliminati?

Le risposte alle domande 3 e 5 determinano l'impegno necessario. Il resto si completa in poche ore ed è documentato da HIN.

## Il secondo progetto: «Stargate»

Indipendentemente da questo, HIN sostituisce Mailgateway con il nuovo HIN Gateway, internamente progetto «Stargate», tecnicamente un approccio Data Mesh con crittografia end-to-end e gestione decentralizzata delle chiavi. Non si tratta della sostituzione dell'appliance, ma di un cambiamento architetturale.

L'impegno si colloca quindi su un livello completamente diverso. Il rinnovo della piattaforma richiede soprattutto il rispetto delle scadenze per una regola firewall, una versione e la sostituzione di un dispositivo, mentre con Stargate è il percorso di posta produttivo stesso a essere rimesso in discussione: l'insieme di regole consolidato, il materiale delle chiavi, la gestione dei destinatari senza identità HIN e la domanda su quale soluzione adottare se qualcosa non funziona come previsto. Poiché la migrazione si svolge in finestre di quattro ore prenotate e HIN raccomanda un mese di preparazione, un simile appuntamento non ammette questioni aperte.

<aside class="offer-box">
  <span class="offer-box__tag">Verifica gratuita</span>
  <p><strong>Non dovete sapere a che punto siete. È proprio a questo che serve la verifica.</strong> Esamino il vostro ambiente Gateway esistente e vi dico cosa deve essere fatto prima della finestra di migrazione, indipendentemente dal fatto che eseguiate poi la migrazione autonomamente o con assistenza.</p>
  <a class="offer-box__cta" href="/stargate">Registratevi ora</a>
</aside>

## Fonti

1.  [Rinnovo della piattaforma HIN: questi adeguamenti tecnici sono necessari per i membri HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): scadenze ad agosto e settembre, endpoint SAML, insieme di attributi ridotto, abilitazioni del firewall.

2.  [Il nuovo HIN Client è disponibile: cosa cambia per i membri HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): versione 4.0, requisiti del sistema operativo, accesso basato sul browser.

3.  [HIN Gateway: comunicazione sicura all'interno della Community HIN](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): sostituzione di Mailgateway, architettura, modelli operativi, migrazione in finestre temporali prenotate.

4.  [Configurazione di HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): ruolo dell'AGW nella gestione degli accessi.

5.  [Collegamento ad Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos e l'account LDAP con diritti di lettura.

6.  [HIN AG: «Dal Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): informazioni di contesto su «Stargate», nodi decentralizzati, calendario.
