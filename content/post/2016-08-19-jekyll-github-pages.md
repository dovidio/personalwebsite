---
author: Umberto D'Ovidio
date: "2016-08-19T00:00:00Z"
description: Come pubblicare il proprio sito Jekyll su github-pages gratis. In questo
  post scoprirai com'è semplice e veloce pubblicare il tuo sito su github pages.
image: /images/jekyll-logo.png
title: Come pubblicare il proprio sito jekyll su Github Pages gratis
---

In un [post recente]({{site.url}}/2016/08/15/jekyll.html) ho introdotto Jekyll e i
suoi vantaggi rispetto a Wordpress. Brevemente, Jekyll è più sicuro, veloce e consente
di poter ospitare gratuitamente il tuo sito su [github pages](https://pages.github.com/). Oggi vedremo i passi da compiere per poter ospitare il tuo sito su github pages.
<!--more-->

Prima di tutto, consiglio di installare [Bundler](http://bundler.io/) per gestire le dipendenze
delle Ruby Gems. Per installare Bundler, devi assicurarti di aver già installato Ruby.
Per far ciò, apri un terminale, e digita il seguente comando per verificare di aver effettivamente installato Ruby.

{{< highlight bash >}}
$ ruby --version
{{< / highlight >}}

Se ruby non è installato, o se la versione è più bassa della 2.0, [installalo](https://www.ruby-lang.org/en/downloads/).
Poi installa Bundler:
{{< highlight bash >}}
$ gem install bundler
{{< / highlight >}}

Ovviamente dovrai poi aver installato [git](https://git-scm.com/downloads).
Inizializza la tua repository git:
{{< highlight bash >}}
$ git init il-nome-del-mio-sito-jekyll
{{< / highlight >}}
passa alla cartella appena creata
{{< highlight bash >}}
$ cd il-nome-del-mio-sito-jekyll
{{< / highlight >}}

Crea un nuovo file chiamato Gemfile e copia incolla le seguenti righe:
{{< highlight bash >}}
source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins
{{< / highlight >}}

Installa jekyll:
{{< highlight bash >}}
$ bundle install
{{< / highlight >}}

A questo punto, se hai un sistema Mac potresti incorrere in un errore. Infatti, una delle gem da installare è nokogiri, che richiede accesso a libxml2. Purtroppo nelle nuove versioni di mac, questa libreria è inacessibile. Per ovviare al problema, si può installare nokogiri con i seguenti comandi:

1. Nel caso di MacOs Sierra
{{< highlight bash >}}
$ gem install nokogiri -- --use-system-libraries=true --with-xml2-include=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.12.sdk/usr/include/libxml2/
{{< / highlight >}}

2. Con El Capitan
{{< highlight bash >}}
gem install nokogiri -- --use-system-libraries=true --with-xml2-include=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk/usr/include/libxml2/
{{< / highlight >}}

A questo punto puoi riprovare ad eseguire 'bundle install'.

Se non hai ancora creato il tuo sito jekyll, puoi creare un sito di prova con il seguente
comando:
{{< highlight bash >}}
$ bundle exec jekyll new . --force
{{< / highlight >}}
Successivamente modifica il tuo Gemfile eliminando la seguente riga:
{{< highlight bash >}}
"jekyll", "3.2.1"
{{< / highlight >}}

Rimuovi il simbolo # per togliere il commento alla seguente linea:
{{< highlight bash >}}
gem "github-pages", group :jekyll_plugins
{{< / highlight >}}

Puoi modificare il sito editando i vari file creati. Ad esempio, se vuoi modificare header e footer, basta modificare header.html e footer.html contenuti nella cartella &#95;includes.
Se invece vuoi scrivere un post, crea un nuovo file sotto alla cartella &#95;posts.

Per visualizzare il tuo sito, prima di pubblicarlo, esegui il seguente comando:

{{< highlight bash >}}
$ bundle exec jekyll serve
{{< / highlight >}}

Pe
Se vuoi che jekyll aggiorni il sito ogni volta che compi delle modifiche (che devono essere ovviamente salvate), ti basterà
aggiungere al comando precedente la parola **watch**
{{< highlight bash >}}
$ bundle exec jekyll serve watch
{{< / highlight >}}
Se invece vuoi che il tuo sito venga costruito rispetto ad un file di configurazione specifico,
che chiamerai, come esempio, &#95;config&#95;dev.yml, dai il seguente comando:
{{< highlight bash >}}
$ bundle exec jekyll serve watch --config _config_dev.yml
{{< / highlight >}}

Una volta che sei soddisfatto del tuo sito e vuoi pubblicarlo, inizia creando una repository su [github](https://github.com) chiamata username.github.io (dove per username si intende il tuo nome utente su github). Aggiungi questa repository come origin con il seguente comando:
{{< highlight bash >}}
$   git remote add origin https://github.com/user/username.github.io.git
{{< / highlight >}}
Avendo cura di utilizzare il nome specifico della tua repository.
Ora basta aggiungere tutti i file alla *staging area* di git, fare un *commit* ed un *push* ed il tuo sito sarà visibile all'indirizzo https://username.github.io

### Risorse utili:
- [Sito ufficiale Jekyll](https://jekyllrb.com/)
- [Documentazione ufficiale](https://jekyllrb.com/docs/home/)
- [Configurare Jekyll localmente](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)
- [Installare jekyll su Windows](http://jekyll.tips/jekyll-casts/install-jekyll-on-windows/)
- [Installare jekyll su MacOs](http://jekyll.tips/jekyll-casts/install-jekyll-on-os-x/)
- [Installare jekyll su Linux](http://jekyll.tips/jekyll-casts/install-jekyll-on-linux/)
