---
slug: "hin-mailgateway-15-0-5-risolvere-l-errore-di-accesso-dopo-l-aggiornamento-del-cluster"
title: "HIN Mailgateway 15.0.5: risolvere il problema di accesso dopo l’aggiornamento del cluster"
navTitle: "Errore di accesso 15.0.5"
description: "Dopo l’aggiornamento di un cluster HIN Mailgateway alla versione 15.0.5, l’accesso smette di funzionare su entrambi i nodi dopo pochi minuti. Questa procedura ripristina in modo controllato il funzionamento delle appliance."
date: "2026-06-19"
kategorie: "Gateway HIN"
timeToRead: "3 min di lettura"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 126aded044bfb102e020aaf45b7b77f6658f77a6de88a71d1554b1fabeefce2a
translatedAt: 2026-11-06T03:28:49.374Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/hin-mailgateway-15-0-5-risolvere-l-errore-di-accesso-dopo-l-aggiornamento-del-cluster
---

# HIN Mailgateway 15.0.5: risolvere il problema di accesso dopo l’aggiornamento del cluster

Durante l’aggiornamento di un HIN Mailgateway dalla versione 14.1.4.2 alla 15.0.5, un errore nella replica del cluster può impedire l’accesso su entrambe le appliance. I sistemi singoli non sono interessati. Il produttore è a conoscenza del problema e prevede una correzione in una versione successiva.

**Aggiornamento del 29 luglio 2026:** La correzione annunciata è disponibile. La patch release 15.0.6 sopprime il rehashing delle password quando i membri del cluster eseguono versioni firmware differenti. Si tratta esattamente della configurazione che aveva causato il problema qui descritto. Il contesto è illustrato nell’articolo su [SEPPmail 15.0.6 e 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); la seguente procedura di ripristino rimane rilevante per i cluster che effettuano ancora l’aggiornamento alla 15.0.5.

## Comportamento del problema

Subito dopo l’aggiornamento, l’interfaccia web è ancora accessibile. Circa dieci minuti più tardi, l’accesso fallisce su entrambi i nodi del cluster. Il fatto che l’errore si manifesti con ritardo e su entrambi i sistemi indica che la causa è la configurazione del cluster replicata.

## Ripristino

I passaggi seguenti modificano la configurazione del cluster. Prima di procedere, devono essere disponibili backup aggiornati e l’identificatore del cluster.

1. Ripristinare gli snapshot creati contemporaneamente di entrambi i nodi del cluster.
2. Dopo il ripristino, lasciare spento un nodo.
3. Sul nodo in esecuzione, scaricare prima l’identificatore del cluster e poi sciogliere il cluster.
4. Attenzione: dopo lo scioglimento, l’appliance si riavvia immediatamente e senza ulteriore richiesta di conferma.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Aggiornare il primo nodo alla versione 15.0.5 e quindi spegnerlo.
6. Avviare il secondo nodo e ripetere gli stessi passaggi.
7. Ricostruire il cluster secondo la documentazione del produttore solo quando entrambi i sistemi funzionano singolarmente e hanno la stessa versione.

Questa procedura impedisce che una configurazione difettosa venga nuovamente replicata tra i nodi durante l’aggiornamento.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: secondo la documentazione SEPPmail, l’ordine di aggiornamento nel cluster è backend prima del frontend, non viceversa.

## Fonti

1. [Documentazione SEPPmail – «Cluster / Alta disponibilità»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipi di cluster e replica della configurazione su tutti i nodi.
2. [Documentazione SEPPmail – «Amministrazione»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): ordine di aggiornamento nel cluster (backend prima del frontend) e requisito di versioni identiche.
3. [HIN Mailgateway: Backup e Disaster Recovery nel cluster](/blog/hin-mailgateway-backup-disaster-recovery): analisi approfondita della replica del cluster, del backup e del ripristino.
