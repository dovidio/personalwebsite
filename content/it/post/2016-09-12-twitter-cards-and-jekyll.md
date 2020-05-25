---
author: Umberto D'Ovidio
date: "2016-09-12T00:00:00Z"
description: In questo articolo spiego come supportare le cards twitter con il tuo
  sito Jekyll. In questo potrai decidere come appare il tuo sito quando viene condiviso
  su twitter.
image: /images/twitter.png
title: Supportare le cards Twitter con Jekyll
---

Ho appena modificato il mio sito, aggiungendo il supporto per le twitter cards. Ho utilizzato la [Summary Card](https://dev.twitter.com/cards/types/summary), in quanto credo sia la più appropriata per il mio sito. La card apparirà sotto il tweet da 140 caratteri.
<!--more-->
Eccone un esempio.
{{< highlight html >}}
<meta name="twitter:card" content="summary" />
<meta name="twitter:site" content="@site_username" />
<meta name="twitter:title" content="Title" />
<meta name="twitter:url" content="URL">
<meta name="twitter:description" content="Fino a 200 caratteri" />
<meta name="twitter:image" content="link/immagine.jpg" />
{{< / highlight >}}

Ovviamente tutti questi meta tag dovranno apparire nella head del sito. Una buona idea da seguire è quella di creare un file chiamato twitter-cards.html e di includerlo nel file &#95;layouts/default.html.
I primi 3 meta tag sono uguali per ogni singola pagina e post del sito. Gli altri 4 cambiano invece a seconda del post che viene linkato e dovremmo quindi fare ricorso al linguaggio liquid per richiamare la variabile desiderata.

## Titolo
Ogni post del mio blog ha il suo titolo, quindi l'idea è di ottenere il titolo se presente e di utilizzare il titolo del sito in caso contrario.
{{< highlight html >}}
{% if page.title %}
  <meta name="twitter:title" content="{{ page.title }}">
{% else %}
  <meta name="twitter:title" content="{{ site.title }}">
{% endif %}
{{< / highlight >}}

## Url
Per quanto riguarda l'url, utilizzo il **site.url** e quello relativo alla pagina, ovvero il **page.url**
{{< highlight html >}}
{% if page.url %}
  <meta name="twitter:url" content="{{ site.url }}{{ page.url }}">
{% endif %}
{{< / highlight >}}

## Descrizione
Per la descrizione utilizzo la *description* dei posts, e se non è presente, utilizzo la *site.description*
{{< highlight html >}}
{% if page.description %}
  <meta name="twitter:description" content="{{ page.description }}">
{% else %}
  <meta name="twitter:description" content="{{ site.description }}">
{% endif %}
{{< / highlight >}}

## Validator
Inoltre puoi vedere come apparirebbero le tue twitter-cards attraverso il [card validator](https://cards-dev.twitter.com/validator). E' obbligatorio validare almeno una card del tuo sito, in modo tale che il tuo dominio sia messo nella **whitelist**.
