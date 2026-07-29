---
slug: "hin-mailgateway-15-0-5-risolvere-l-errore-di-accesso-dopo-l-aggiornamento-del-cluster"
title: "HIN Mailgateway 15.0.5: risolvere l’errore di accesso dopo l’aggiornamento del cluster"
navTitle: "Errore di accesso 15.0.5"
description: "Dopo l’aggiornamento di un cluster HIN Mailgateway alla versione 15.0.5, l’accesso smette di funzionare su entrambi i nodi dopo pochi minuti. Questa procedura consente di ripristinare le appliance in modo controllato."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min. di lettura"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
url: "https://rafaelpfister.ch/it/blog/hin-mailgateway-15-0-5-risolvere-l-errore-di-accesso-dopo-l-aggiornamento-del-cluster"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 3bf0ad28c6b9b80f5644d7281912c3966fd7d0632665afcc13e055bda963e5c2
translatedAt: 2026-07-29T12:29:38.945Z
---

# HIN Mailgateway 15.0.5: risolvere l’errore di accesso dopo l’aggiornamento del cluster

Durante l’aggiornamento di un HIN Mailgateway dalla versione 14.1.4.2 alla 15.0.5, un errore nella replica del cluster può bloccare l’accesso su entrambe le appliance. I sistemi singoli non sono interessati. Il produttore conosce il problema e prevede una correzione in una versione successiva.

## Sintomi

Subito dopo l’aggiornamento, l’interfaccia web è ancora accessibile. Circa dieci minuti più tardi, l’accesso fallisce su entrambi i nodi del cluster. Il fatto che l’errore si manifesti con ritardo e su entrambi i sistemi indica come causa la configurazione del cluster replicata.

## Ripristino

I seguenti passaggi modificano la configurazione del cluster. Prima di procedere, devono essere disponibili backup aggiornati e l’identificatore del cluster.

1. Ripristinare gli snapshot creati contemporaneamente di entrambi i nodi del cluster.
2. Dopo il ripristino, lasciare spento un nodo.
3. Sul nodo in esecuzione, scaricare prima l’identificatore del cluster e poi sciogliere il cluster.
4. Attenzione: dopo lo scioglimento, l’appliance si riavvia immediatamente e senza ulteriore richiesta di conferma.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Aggiornare il primo nodo alla versione 15.0.5, quindi spegnerlo.
6. Avviare il secondo nodo e ripetere gli stessi passaggi.
7. Solo quando entrambi i sistemi funzionano singolarmente e hanno la stessa versione, ricreare il cluster secondo la documentazione del produttore.

Questa procedura impedisce che una configurazione difettosa venga nuovamente replicata tra i nodi durante l’aggiornamento.

## Fonti

1. [Documentazione SEPPmail – «Cluster / Alta disponibilità»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipi di cluster e replica della configurazione su tutti i nodi.
2. [Documentazione SEPPmail – «Amministrazione»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): sequenza di aggiornamento nel cluster (frontend prima del backend) e requisito di versioni identiche.
3. [HIN Mailgateway: Backup e Disaster Recovery nel cluster](/blog/hin-mailgateway-backup-disaster-recovery): approfondimento su replica del cluster, backup e ripristino.
