---
author: Umberto D'Ovidio
date: "2016-07-20T00:00:00Z"
description: Spesso Java ed Android richiedono di scrivere infinite linee di codice
  che alla fine della fiera servono a poco. Butterknife rende più semplice effettuare
  alcune semplici operazioni che tradizionalmente richiedono molto lavoro, come il
  creare un onClickListener.
title: Butterknife, l'arma contro il boilerplate code
---

Nella programmazione, il boilerplate code o boilerplate si riferisce a sezioni di codice che devono essere incluse in molti posti senza cambiare molto. E’ un termine che si utilizza molto quando si parla di linguaggi di programmazione considerati verbosi, nei quali il programmatore deve scrivere molto codice senza produrre molto.
<!--more-->

Se hai mai programmato un’app Android con Android Studio ti sarai reso conto che purtroppo il boilerplate code è molto presente. Una semplice applicazione infatti può richiedere molto codice senza fare poi molto. E’ qui che ci viene incontro una semplice libreria creata da [Jake Wharton](http://jakewharton.com/) chiamata [Butter Knife](https://github.com/JakeWharton/butterknife).

Per utilizzarla, assicurati di aggiungere le seguenti linee di codice ai tuoi build.gradle files. Configura il tuo build.gradle file a livello di progetto per utilizzare il plugin ‘android-apt’ e successivamente usalo a livello di modulo e aggiungi butterknife alle dependencies.
dependencies

#### build.gradle progetto

{{< highlight java >}}
buildscript {
  repositories {
    mavenCentral()
   }
  dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
  }
}
{{< / highlight >}}

#### build.gradle modulo

{{< highlight java >}}
apply plugin: 'android-apt'

android {
  ...
}

 {
  compile 'com.jakewharton:butterknife:8.2.1'
  apt 'com.jakewharton:butterknife-compiler:8.2.1'
}

{{< / highlight >}}

Ora il nostro lavoro sarà più semplice. Infatti, se abbiamo molti oggetti di tipo View nella nostra applicazione, dovremmo perdere tempo e chiamare findViewById per ogni View che vogliamo utilizzare dinamicamente. Per esempio

#### Senza butterknife

{{< highlight java >}}
class ExampleActivity extends Activity {
  TextView tv1;
  TextView tv2;
  TextView tv3;
  .............
  TextView  tv10;


  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    tv1 = findViewById(R.id.tv1);
    ............................
    ............................
    tv10 = findViewById(R.id.tv10);
    // finalmente possiamo iniziare a lavorare sui nostri fields
  }
}
{{< / highlight >}}

Con Butterknife ci basterà referenziare i field tramite il comando @BindView, e successivamente all’interno della classe onCreate chiamare ButterKnife.bind(this). Ecco come cambia l’esempio precedente.

#### Con ButterKnife

{{< highlight java >}}
class ExampleActivity extends Activity {
 @BindView(R.id.tv1) TextView tv1;
 @BindView(R.id.tv2) TextView tv2;
 @BindView(R.id.tv3) TextView tv3;
 .............
 TextView  tv10;


 @Override public void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   setContentView(R.layout.simple_activity);
   ButterKnife.bind(this);
   // Pronti per lavorare sui nostri fields
 }
}
{{< / highlight >}}


Per quanto riguarda gli onClickListeners, ButterKnife ci semplifica ancora la vita. Ecco come possiamo crearne uno in modo semplice.

#### OnClickListener

{{< highlight java >}}

@OnClick(R.id.submit)
public void saluta(Button button) {
  button.setText("Ciao!");
}
{{< / highlight >}}


Ci sono poi altre funzionalità che potrebbero essere d’aiuto. Se sei interessato consulta la [documentazione ufficiale](http://jakewharton.github.io/butterknife/).
