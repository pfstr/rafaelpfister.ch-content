---
slug: "hin-mailgateway-15-0-5-risolvere-l-errore-di-accesso-dopo-l-aggiornamento-del-cluster"
title: "HIN Mailgateway 15.0.5: risolvere il malfunzionamento del login dopo l'aggiornamento del cluster"
navTitle: "Errore di login 15.0.5"
description: "Dopo l'aggiornamento di un cluster HIN Mailgateway alla versione 15.0.5, il login non funziona più su entrambi i nodi dopo pochi minuti. Questa procedura ripristina il funzionamento delle appliance in modo controllato."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min di lettura"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: d40e2a9637cb0711ca33ed8ffde7827192362f668bebafb7b1f6862fda736d04
translatedAt: 2026-08-08T14:23:28.517Z
url: https://rafaelpfister.ch/it/blog/hin-mailgateway-15-0-5-risolvere-l-errore-di-accesso-dopo-l-aggiornamento-del-cluster
translationModel: gpt-5.6-terra
---

# HIN Mailgateway 15.0.5: risolvere il malfunzionamento del login dopo l'aggiornamento del cluster

Durante l'aggiornamento di un HIN Mailgateway dalla versione 14.1.4.2 alla 15.0.5, un errore nella replica del cluster può bloccare il login su entrambe le appliance. I sistemi singoli non sono interessati. Il produttore è a conoscenza del problema e prevede una correzione in una versione successiva.

**Aggiornamento del 29 luglio 2026:** La correzione annunciata è disponibile. La patch release 15.0.6 sopprime il rehashing della password quando i membri del cluster utilizzano versioni firmware differenti. È esattamente la configurazione che aveva causato il malfunzionamento descritto qui. Il contesto è illustrato nell'articolo su [SEPPmail 15.0.6 e 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); la seguente procedura di ripristino rimane rilevante per i cluster che vengono ancora aggiornati alla versione 15.0.5.

## Sintomo

Subito dopo l'aggiornamento, l'interfaccia web può ancora essere aperta. Circa dieci minuti più tardi, il login fallisce su entrambi i nodi del cluster. Il fatto che l'errore si verifichi con ritardo e su entrambi i sistemi indica come causa la configurazione del cluster replicata.

## Ripristino

I passaggi seguenti modificano la configurazione del cluster. Prima di procedere, devono essere disponibili backup aggiornati e l'identificatore del cluster.

1. Ripristinare gli snapshot creati contemporaneamente di entrambi i nodi del cluster.
2. Dopo il ripristino, lasciare spento un nodo.
3. Sul nodo in esecuzione, scaricare prima l'identificatore del cluster e quindi sciogliere il cluster.
4. Attenzione: dopo lo scioglimento, l'appliance si riavvia immediatamente e senza ulteriori richieste di conferma.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Aggiornare il primo nodo alla versione 15.0.5 e quindi spegnerlo.
6. Avviare il secondo nodo e ripetere gli stessi passaggi.
7. Solo quando entrambi i sistemi funzionano singolarmente e hanno la stessa versione, ricreare il cluster secondo la documentazione del produttore.

Questa procedura impedisce che una configurazione difettosa venga nuovamente replicata tra i nodi durante l'aggiornamento.

## Fonti

1. [Documentazione SEPPmail – «Cluster / Alta disponibilità»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipi di cluster e replica della configurazione su tutti i nodi.
2. [Documentazione SEPPmail – «Amministrazione»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): sequenza di aggiornamento nel cluster (frontend prima del backend) e requisito di versioni identiche.
3. [HIN Mailgateway: Backup & Disaster Recovery nel cluster](/blog/hin-mailgateway-backup-disaster-recovery): approfondimento sulla replica del cluster, sul backup e sul ripristino.
