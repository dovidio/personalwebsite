---
title: "Un anno di progetti"
date: 2026-10-25T08:00:00+02:00
draft: false
description: "Sette progetti costruiti tra un cambio di pannolino e una poppata, e l'anno in cui l'AI è finalmente diventata brava."
tags: ["projects", "ai", "fatherhood"]
translationKey: "a-year-of-projects"
---

E niente, è passato un anno dall'ultimo post sul blog.
Non sgridatemi, ho una buona scusa. Il giorno dopo aver scritto [l'ultimo post](/it/post/2025-10-28-talos-hetzner-parte-3/), sono diventato papà.
Ero preparato a sacrificare tempo per i miei hobby, ma non pensavo che sarebbe stato così duro.
A quanto pare ci vuole davvero un villaggio!

Ma smettiamola di cantare le lodi della gioia della paternità, questo è pur sempre una specie di blog tecnico (o almeno ci prova).
Quindi a quanto pare l'AI è [finalmente diventata brava](https://x.com/dhh/status/2007504187568074843). È passata dal generare un'app di time tracking con la brillante strategia di `setInterval(() => secondsTracked += 1, 1000)` al generare codice coerente, sempre più veloce di quanto un umano possa produrne e discutibilmente più corretto (se il tuo dominio era nel training set).

Anche se abbiamo avuto meno tempo da dedicare ai progetti, è stato un anno produttivo per me e mia moglie.
Ed è il momento di celebrare tutti quei piccoli progetti che hanno riempito il nostro tempo tra un cambio di pannolino e una poppata.

[Labora](https://getlabora.app): questo l'ha ideato e sviluppato interamente mia moglie con l'assistenza dell'AI. Aiuta a tenere traccia delle contrazioni e dà indicazioni su quando è il momento di correre in ospedale. La sua semplicità e la sua UX hanno aiutato più di 3.000 mamme a mettere al mondo dei bambini.

[Innuce](https://innuce.app): un'estensione Chrome e una web app per riassumere i video di YouTube. L'ha avviata mia moglie, e io l'ho preso in mano durante le vacanze di Natale, in un episodio di [psicosi da AI](https://lucumr.pocoo.org/2026/1/18/agent-psychosis/). Pensavamo di farci un'azienda :D
Siamo abbastanza orgogliosi del risultato, e continuiamo a usarlo nella nostra vita quotidiana, ma non ha generato granché di trazione. Non ha aiutato il fatto che YouTube stesso abbia creato una funzione di riassunto sul suo sito. Comunque prima o poi lo aprirò e lo tirerò giù.
Ho imparato molto da questo. Un paio di cose:
1. Lo sviluppo di estensioni Chrome.
2. CloudFormation e le cosette AWS (Cognito, Lambda, CloudWatch, DynamoDB)
3. Ruby on Rails, dove ho deciso di spostarmi dopo essermi bruciato con le cosette AWS
4. La bellezza e il business dei proxy rotanti, il client pooling
5. SSE, WebSocket e il collegamento alle API degli agenti (usando OpenRouter all'inizio e passando a Mistral alla fine)

[Zenith](https://dovidio.github.io/zenith/): uno dei più recenti. L'idea è che, visto che scriviamo meno codice (o niente), dovremmo ripensare gli editor di codice e ottimizzarli per la lettura, non per la scrittura. Di fatto Zenith è un navigatore di codice in sola lettura.

[Colino](https://getcolino.umberto.xyz): aggregatore di notizie da terminale, fatto a metà. È stato uno dei primi progetti in cui ho usato pesantemente l'AI, e un po' si vede. Non è ancora molto coerente in termini di funzionalità. Un giorno lo riprenderò. Ma in pratica puoi aggregare notizie via RSS, leggerle dal terminale e persino fartele riassumere dall'AI.

[Kita-lugano](https://dovidio.github.io/kita-lugano/): dovevo capire quali asili nido c'erano a Lugano e non trovavo un posto unico dove filtrarli. Quindi eccolo qui. Spero faccia risparmiare un po' di tempo a qualche genitore impegnato.

[Snappacci](https://github.com/dovidio/snappacci): una libreria store-and-forward per immagini e blob in Java, che sfrutta la nuova API di memoria nativa. Non è ancora stata usata in produzione, ma è stata una buona scusa per imparare qualcosa sulla verifica formale e sulla fattibilità di usare TLA+ nell'era degli agenti.

[TimeTrackete](https://dovidio.github.io/time-tracking/): app iOS nativa per tracciare il tempo usando un singolo file come registro di eventi. Ci ho messo un paio d'ore a farla ed è quella che uso per tracciare le ore al lavoro. La cosa più bella è che include azioni Shortcut che si possono associare a un doppio tap, così non serve nemmeno aprire l'app per iniziare o fermare il conteggio.
