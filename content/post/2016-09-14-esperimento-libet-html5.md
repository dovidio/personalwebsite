---
author: Umberto D'Ovidio
date: "2016-09-14T00:00:00Z"
description: Si può sfidare il libero arbitrio dal proprio browser? Un team di scienziati
  ha ricreato il famoso esperimento di Libet usando tecnologie HTML5.
image: /images/libet.png
title: Sfidare il libero arbitrio dal proprio browser?
---

[L'esperimento di Libet](https://it.wikipedia.org/wiki/Benjamin_Libet) è senza dubbio uno dei più famosi nella storia della psicologia, e scatena da anni ampi dibattiti. 
La procedura è variata nel corso degli anni, tuttavia si può cercare di sintetizzare la procedura in 6 punti.
<!--more-->
1. All'inizio di ciascun trial, un messaggio avverte i partecipanti di prepararsi per lo stimolo
2. Dopo un intervallo variabile, viene mostrato un orologio
3. Invece delle lancette dell'orologio, un puntino inizia a girare intorno al centro dell'orologio ad una velocità costante
4. Durante il primo giro, i partecipanti non devono fare nulla (infatti questa parte serve solo ad abituarli alla velocità del puntino)
5. Successivamente, al secondo giro, li viene chiesto di premere un tasto quando desiderato.
6. Una volta terminato il secondo giro, viene mostrato un orologio senza alcun puntino o lancetta.
Il compito dei partecipanti è quello di indicare in quale posizione si trovava il puntino nel momento in cui hanno **deciso** di premere il tasto. Spesso viene chiesto anche quando il tasto è stato **premuto**. In altri esperimenti vengono anche inclusi alcuni stimoli uditivi che possono essere riprodotti **durante** la pressione del tasto o con un piccolo **ritardo**.
Spesso viene monitorata l'attività cerebrale tramite [elettroencefalogramma](https://it.wikipedia.org/wiki/Elettroencefalografia).

I risultati mostrano che l'attività cerebrale (nello specifico quello che viene chiamato [readiness potential](https://en.wikipedia.org/wiki/Bereitschaftspotential) precede di alcune centinaia di millisecondi il momento in cui i soggetti riferiscono di aver deciso di premere il tasto, a seguire la pressione del tasto e il suono. Secondo alcune interpretazioni, questi risultati dimostrano che l'azione cosciente altro non è che un sottoprodotto dell'attività cerebrale. In parole povere, il nostro cervello decide prima di noi.

<img src="{{ site.url }}/assets/libet1.jpg" alt="Risultati esperimento libet" />

Indipendentemente dal dibattito scatenato e dalle possibili interpretazioni, l'esperimento in questione viene ancora svolto in moltissimi laboratori e fornisce validi risultati ed informazioni su come percepiamo i suoni e sul nostro senso di agentività. Creare una versione standardizzata dell'esperimento utilizzando le tecnologie HTML5 potrebbe facilitare la replicabilità dell'esperimento e fornire risultati più consistenti, indipendentemente dall'hardware e dal sistema operativo utilizzato.

E proprio per questo, è stata sviluppata una [versione](http://www.nature.com/articles/srep32689) dell'esperimento di Libet da alcuni ricercatori spagnoli che funziona su browser.
La tecnologia sviluppata è stata chiamata [LabClock Web](https://github.com/txipi/Labclock-Web), ed adotta alcuni accorgimenti interessanti ed al passo coi tempi. Per far sì che i risultati siano consistenti a dispetto dell'enorme variabilità di hardware e software, vengono fatti alcuni test preliminari. Attraverso la libreria [Modernizr](https://modernizr.com/) viene controllato che il dispositivo utilizzato sia effettivamente in grado di riprodurre contenuti HTML5. Inoltre LabClock Web è abbastanza intelligente da controllare che la risoluzione dello schermo sia sufficiente per visualizzare gli stimoli in modo soddisfacente.  
Lo stimolo principale, ovvero il puntino ruotante, viene animato tramite CSS.
Per quanto riguarda gli stimoli uditivi, essi sono gestiti mediante la [Web Audio API](https://developer.mozilla.org/it/docs/Web/API/Web_Audio_API), e vengono salvati in buffers all'inizio dell'esperimento per evitare eventuali ritardi.
Le risposte dei partecipanti vengono salvate attraverso i [DOM events](http://www.w3schools.com/js/js_htmldom_events.asp). Una volta che l'esperimento viene completato, i risultati vengono mandati attraverso una connessione HTTP post attraverso AJAX.
Inoltre, i risultati vengono salvati anche localmente nel browser.

I test condotti dai ricercatori dimostrano un'ottima precisione nella presentazione degli stimoli, che rimane invariata utilizzando diversi sistemi operativi. Inoltre, gli esperimenti condotti su soggetti, confermano i risultati delle ricerche precedenti, sia effettuando l'esperimento con soggetti in laboratorio che online.

Questa ricerca dimostra come le nuove tecnologie HTML5 siano pronte per essere utilizzate dalle ricerche di psicologia e neuroscienze. Ma quali sono i vantaggi?
- Gli esperimenti potranno essere condotti con lo stesso software indipendentemente dal sistema operativo utilizzato
- Si potrà risparmiare sui costi degli esperimenti, infatti nel momento in cui rendere disponibile il codice sorgente diventa prassi comune, i laboratori non dovranno più pagare per costosissimi software.
- I risultati potranno essere inviati immediatamente via HTTP post, facilitando così la collaborazione internazionale.
- Poter presentare un esperimento online consentirà di trovare più facilmente partecipanti agli esperimenti e di risolvere il sempre presente problema della **sample size**.
