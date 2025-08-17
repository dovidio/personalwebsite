---
author: Umberto D'Ovidio
date: "2017-09-19T00:00:00Z"
description: MultiTimer è un'applicazione Android che permette di gestire timer multipli
feature-img: assets/multitimer.png
feature-img-lg: assets/multitimer_lg.png
feature-img-md: assets/multitimer_md.png
feature-img-sm: assets/multitimer_sm.png
feature-img-xs: assets/multitimer_xs.png
image: /images/multitimer.png
title: Gestisci multipli timer con MultiTimer
---

Recentemente ho pubblicato per la prima volta un'app nel Google Play Store con il mio account sviluppatore. Vuoi sapere di cosa si tratta?

<!--more-->

[MultiTimer](https://play.google.com/store/apps/details?id=io.dovid.multitimer.paid) è un'app che ho sviluppato recentemente e che ho deciso di pubblicare nel Google Play Store.
Consente di creare, modificare e gestire timer multipli, in modo da non trovarsi a risettare il timer ogni volta che scade. 
Tra le varie funzionalità c'è la possibilità di scegliere diversi temi, scegliere la suoneria del timer e decidere se ricevere una notifica o meno. 

<img src="{{ site.url }}/assets/multitimer.png" alt="MultiTimer app" style="margin: 2rem 2em;"/>

L'app di per sè è molto semplice ma mi ha permesso di scoprire e migliorare alcuni aspetti della programmazione in Android e presenta alcune particolarità che la rendono molto affidabile. Ad esempio, tutti i timer vengono salvati in un database, così che puoi tranquillamente abbandonare l'app quando vuoi e al tuo ritorno li ritrovi ancora lì. Non solo, se chiudi l'app verrai comunque avvisato alla scadenza dei timer nel caso in cui erano già partiti al momento della chiusura dell'app. 

Circa il 60% del codice è stato scritto in Kotlin, il nuovo linguaggio di programmazione creato da IntelliJ e ora ufficialmente supportato da Google per lo sviluppo Android. Devo ammettere che non ho avuto molti problemi ad utilizzarlo in quanto è molto simile a Swift e permette comunque di utilizzare tutti i metodi e le librerie che utilizzo spesso in Java. 
Secondo i creatori di questo linguaggio, Kotlin dovrebbe permettere un più rapido sviluppo ed essere più sicuro di Java in quanto permette di evitare al 95% casi di NullPointerException. 

Dai un'occhiata a MultiTimer sul [Play Store](https://play.google.com/store/apps/details?id=io.dovid.multitimer.paid), è gratis!