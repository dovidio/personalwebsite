---
author: Umberto D'Ovidio
date: "2017-11-19T00:00:00Z"
description: Java 9 permette di modularizzare i propri progetti in modo mai visto
  prima. Scopriamo come
title: Modularità in java 9
visible: 1
---

Una delle introduzioni più interessanti nella nuova versione di Java è senza dubbio quella dei moduli. I moduli permettono di descrivere le relazioni e le dipendenze all'interno del codice della tua applicazione. 
<!--more-->
Il vantaggio principale è una migliore organizzazione di progetti complessi e la creazione di un runtime Java che contiene solo le Api richieste dalla tua applicazione. Questa "cura dimagrante" aiuta tutti gli sviluppatori che scrivono codice per ambienti con risorse limitate, ad esempio per sensori smart.

### Introduzione ai moduli

Essenzialmente un modulo è un raggruppamento di packages e risorse che possono essere richiamate con il nome del modulo.
La *module declaration* (dichiarazione del modulo) specifica il nome del modulo e la relazione che con gli altri moduli. Essa è contenuta in un file chiamato **module-info.java**. Questo file viene poi compilato in un file *class* e viene chiamato come il *module descriptor* (descrittore del modulo).
La descrizione del modulo inizia con la keyword **module**. 


### Un esempio di modulo

Fondamentalmente, un modulo può fare due cose:
- specificare una relazione di dipendenza da altri moduli attraverso la keyword **requires**
- specificare quali **packages** rende disponibili agli altri moduli attraverso la keyword **exports**

Per rendere chiaro il funzionamento dei moduli svilupperemo insieme una semplice applicazione che se ne serve. Il codice sorgente è disponibile su [Github](https://github.com/Cyborg101/dovidioTutorials/tree/master/Modulo)
Per prima cosa creiamo una cartella del progetto, che chiameremo **Modulo**.
All'interno di questa cartella creiamo un'altra cartella chiamata **src**, che conterrà tutto il codice sorgente della nostra applicazione.
Entriamo in *src* e creiamo due cartelle: **modulopricipale** e **utils**. Entriamo all'interno della prima cartella e creiamo un'altra cartella chiamata **moduloprincipale**. All'interno di questa cartella creiamo un'altra cartella che chiameremo **appesempio**. 
Ora entriamo nella cartella **utils**, creiamo una cartella **utils** al suo interno e all'interno di quest'ultima creiamo una cartella **stringutils**.
Alla fine di questa fase il tuo albero di cartelle dovrebbe essere uguale all'immagine mostrata sotto.

<img src="{{ site.url }}/assets/modules-directory-structure.png" alt="Struttura dell'albero delle cartelle dell'applicazione" style="display: block;width: 500px;margin:2em auto"/>

L'applicazione avrà un semplice modulo principale che utilizzerà un altro modulo dove sono specificate alcune funzioni di utilità per le stringhe.
Entriamo dunque nella cartella utils/utils/stringutils e creiamo una classe chiamata **StringFormatter.java**.
Il codice della classe è il seguente

{{< highlight java >}}
package utils.stringutils;

public class StringFormatter {
	
	public static String capitalize(String s) {
		if (s.length() == 1) {
			return String.valueOf(Character.toUpperCase(s.charAt(0)));
		} 
		if (s.length() > 1) {
			return Character.toUpperCase(s.charAt(0)) + s.substring(1);	
		} 
		return "";
	}

	public static String toUpperCase(String s) {
		StringBuilder sb = new StringBuilder();
		for (char c : s.toCharArray()) {
			sb.append(Character.toUpperCase(c));
		}
		return sb.toString();
	}
}
{{< / highlight >}}

La classe fornisce due semplici metodi. Il primo permette di rendere maiuscola la prima lettera di una parola. Ad esempio, chiamando *capitalize("umberto")*
ci verrà ritornata la stringa "Umberto". La seconda classe invece trasforma una parola in maiuscolo, quindi chiamando *toUpperCase("umberto")* ci verrà restituita la stringa "UMBERTO".

Ora che abbiamo creato queste utili funzioni, è ora di utilizzarle. Creiamo quindi una classe chiamata **Main.java** nella cartella src/moduloprincipale/moduloprincipale/appesempio.

Il codice della classe è il seguente:

{{< highlight java >}} 
package moduloprincipale.appesempio;

import utils.stringutils.StringFormatter;

public class Main {
	
	public static void main(String[] args) {
		System.out.println("Siamo nella classe principale");

		String umberto = "umberto";

		System.out.println("capitalizziamo la stringa " + umberto + ": " + StringFormatter.capitalize(umberto));

		System.out.println("Ora invece rendiamola maiuscola: " + StringFormatter.toUpperCase(umberto));
	}
}
{{< / highlight >}}

Creiamo ora i file **module-info.java**, ovvero i file che specificano la relazione tra i moduli.
Entriamo nella cartella src/utils e creiamo il file module-info.java.
Il codice del file sarà il seguente: 


{{< highlight java >}}
module utils {
	exports utils.stringutils;
}
{{< / highlight >}}

La descrizione del nostro modulo informa Java che vogliamo che il package utils.stringutils venga reso disponibile a tutti i moduli. Se vogliamo renderlo disponibile solo ad un modulo particolare possiamo specificare il nome del modulo dopo la parola chiave **to**. Ad esempio, se vogliamo che il nostro package sia disponibile solo per il *moduloprincipale*, scriveremo così:

{{< highlight java >}}
module utils {
	exports utils.stringutils to moduloprincipale;
}
{{< / highlight >}}

Creiamo ora il file module-info.java all'interno della cartella src/moduloprincipale e scriviamo il seguente codice.

{{< highlight java >}} 
module moduloprincipale {
	requires utils;
}
{{< / highlight >}}

Questo significa che il nostro modulo richiede il pacchetto utils, e potremmo servircene all'interno della nostra classe.

Per compilare tutto possiamo utilizzare il vecchio ma pur sempre abile **javac**, che si è aggiornato per l'occasione con delle nuove opzioni. 
Andiamo nella directory principale, ovvero **Modulo**, e compiliamo tutto con il seguente comando

{{< highlight bash >}}
javac -d moduli --module-source-path src src/moduloprincipale/moduloprincipale/appesempio/Main.java 
{{< / highlight >}}
Questo comando dice di compilare tuttte le classi all'interno di una directory chiamata moduli. L'opzione **--module-source-path** compila automaticamente i file nell'albero sotto la cartella specificata, in questo caso src. 
Per eseguire la nostra applicazione utilizziamo il seguente comando

{{< highlight bash >}}
java --module-path appmodules -m moduloprincipale/moduloprincipale.appesempio.Main
{{< / highlight >}}

Il risultato è il seguente:

<img src="{{ site.url }}/assets/modules-screenshot-compilation.png" alt="Screenshot compilazione ed esecuzione del programma" style="display: block;width: 500px;margin:2em auto"/>


La modularità e senz'altro una caratteristica importante di Java 9 e l'argomento è molto vasto. Se vuoi saperne di più, ecco qualche libro utile:
- <a target="_blank" href="https://www.amazon.it/gp/product/1491954167/ref=as_li_tl?ie=UTF8&camp=3414&creative=21718&creativeASIN=1491954167&linkCode=as2&tag=dovidio-21&linkId=ffaee695b7cf55c8bc199136678f8c32">Java 9 Modularity: Patterns and Practices for Developing Maintainable Applications</a><img src="//ir-it.amazon-adsystem.com/e/ir?t=dovidio-21&l=am2&o=29&a=1491954167" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />
- <a target="_blank" href="https://www.amazon.it/gp/product/1259589331/ref=as_li_tl?ie=UTF8&camp=3414&creative=21718&creativeASIN=1259589331&linkCode=as2&tag=dovidio-21&linkId=afd53cc4e2a0ff8381d6a1cee45374d4">Java: The Complete Reference</a><img src="//ir-it.amazon-adsystem.com/e/ir?t=dovidio-21&l=am2&o=29&a=1259589331" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />
- <a target="_blank" href="https://www.amazon.it/gp/product/B071S84XCK/ref=as_li_tl?ie=UTF8&camp=3414&creative=21718&creativeASIN=B071S84XCK&linkCode=as2&tag=dovidio-21&linkId=0dc4ea184967f8909567dbadbe727adc">Java 9 for Programmers (Deitel Developer Series)</a><img src="//ir-it.amazon-adsystem.com/e/ir?t=dovidio-21&l=am2&o=29&a=B071S84XCK" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />

Video su java 9 e i moduli:
<iframe width="560" height="315" style="display: block;width: 500px;margin:2em auto" src="https://www.youtube.com/embed/wGDoYkB9pd4" frameborder="0" allowfullscreen></iframe>
