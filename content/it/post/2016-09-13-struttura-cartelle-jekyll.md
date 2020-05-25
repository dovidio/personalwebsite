---
author: Umberto D'Ovidio
date: "2016-09-13T00:00:00Z"
description: Per poter utilizzare al meglio Jekyll, è importante conoscere la sua
  struttura di cartelle e file.
image: /images/jekyll-logo.png
title: Struttura delle cartelle di Jekyll
---


Per utilizzare Jekyll con successo, è utile conoscere la struttura delle sue cartelle. In questo articolo scopriremo i file e le cartelle più importanti.
<!--more-->

## &#95;config.yml
Questo file viene utilizzato per specificare alcune variabili importanti per il tuo sito, tipo l’url, la descrizione o il titolo. Si possono inoltre configurare collections o defaults, di cui parlerò in qualche post futuro. Inoltre si possono specificare alcune variabili runtime che vogliamo che vengano eseguite sempre.

## &#95;drafts
Questa cartella contiene tutti i post non pubblicati e senza data. Se hai intenzione di scrivere post lunghi, che richiedono tempo e revisioni, questa è la cartella adatta dove metterli.

## &#95;includes
Nella cartella includes mettiamo le parti del sito che sono presenti in più pagine, ad esempio l’header o il footer, o un form per iscriversi ad una newsletter.

## &#95;layouts
In questa cartella abbiamo i templates che racchiudono il contenuto dei nostri post. Ad esempio, possiamo creare un file chiamato default che includerà header e footer. Quando poi creiamo una pagina, possiamo utilizzare il YAML Front Matter e specificare l’opzione layout: default, che risulterà dunque nell’utilizzare il template chiamato default.html.

## &#95;posts
Qui abbiamo i nostri post, scritti con la sintassi markdown.

## &#95;data
Contiene file YAML, JSON e CVS. I dati all’interno di questi file possono essere utilizzati in tutto il tuo sito Jekyll.

## &#95;site
Una volta che Jekyll ha costruito il nostro sito, ci saranno varie pagine e cartelle. Quindi questa è la cartella dove c’è il sito vero e proprio, pronto per essere pubblicato.

## .jekyll-metadata
Questo file viene utilizzato da Jekyll quando si utilizza l’opzione di incremental build.

## Altri files e cartelle
Ovviamente il tuo sito può essere composto da altri file e cartelle, come la cartella css. Questi files e cartelle vengono semplicemente copiate ed incollate nella cartella &#95;site.

### Risorse utili
Se vuoi saperne di più, ecco un paio di risorse utili:

#### - [Documentazione ufficiale di Jekyll](https://jekyllrb.com/docs/home/)

#### - [Jekyll tips](http://jekyll.tips/)
