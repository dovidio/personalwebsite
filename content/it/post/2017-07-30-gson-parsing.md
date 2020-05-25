---
author: Umberto D'Ovidio
date: "2017-07-30T00:00:00Z"
description: Gson è una libreria che permette di velocizzare il parsing json
image: /images/gson.png
title: Semplificare il parsing Json con Gson
---
Se nella tua vita di programmatore Java ti trovi spesso ad effettuare serializzazione e deserializzazione Json, dovresti prendere in considerazione l'utilizzo di una libreria fatta apposta per velocizzare questi task. [Gson](https://github.com/google/gson) è una libreria Java creata da Google che permette di convertire un oggetto Java in un oggetto Json e viceversa in modo veloce ed efficiente. Ci sono molte librerie Java capaci di fare ciò, ma Gson è una fra le poche che non richiede l'uso di annotazioni ne tantomeno del codice sorgente delle classi che vogliamo convertire in Json.
Contiene due metodi chiamati *fromJson()* e *toJson()* che sono l'equivalente dei metodi Java *toString()* e *fromString()*.
<!--more-->

## Deserializzazione Json (da Json a JAVA)
In questo tutorial darò un esempio di *deserializzazione* Json, ovvero spiegherò come a partire da una risposta Json che ci manda un server possiamo costruire e popolare un oggetto Java. Per farlo ci serviremo dell'Api di Github.
La nostra app sarà in grado di cercare repository su Github e stamperà a schermo i primi 30 risultati. Se vuoi scaricare il codice sorgente, puoi trovarlo a questo [link](https://github.com/Cyborg101/dovidioTutorials/tree/master/gsonparsing).

Prima di tutto aggiungiamo la dipendenza al pom.xml di Maven.

{{< highlight xml >}}
<dependency>
  <groupId>com.google.code.gson</groupId>
  <artifactId>gson</artifactId>
  <version>2.8.0</version>
  <scope>compile</scope>
  </dependency>
</dependencies>
{{< / highlight >}}

Creeremo poi dei semplici oggetti Java che verranno popolati a partire dal Json che riceviamo in risposta da Github. 
Mi sono limitato a creare oggetti con pochi campi anche se l'Api Github ne ritorna molti di più. Gson non si fà problemi e popolerà i campi che abbiamo nel nostro oggetto Java ignorando le chiavi Json che non hanno una corrispondenza con i campi della nostra classe Java. E' importante però assicurarsi che il nome dei nostri campi corrisponda alle chiavi che riceviamo nel Json.

Bene, procediamo dunque alla creazione della classe **Repository**, che conterrà i campi fullname (nome della repository), html_link (link alla repository), watchers (numero delle persone che stanno tenendo d'occhio questa repository) e forks.
Implementiamo poi il metodo **toString()** che utilizzeremo per stampare a schermo questo oggetto.

{{< highlight java >}}
package gsonparsing;

class Repository {
    String full_name;
    String html_url;
    int forks;
    int watchers;

    @Override
    public String toString() {
        String out = "name: " + full_name + "\n";
        out += "url: " + html_url + '\n'; 
        out += "number of watchers: " + watchers + '\n';
        out += "number of forks: " + forks;
        return out;
    }
}
{{< / highlight >}}

Dato che riceveremo, verosimilmente, più di una repository in risposta, creiamo anche una classe che conterrà queste repository, e la chiamiamo, molto fantasiosamente, **Repositories**.

{{< highlight java >}}
package gsonparsing;
import java.util.List;

public class Repositories {
    List<Repository> items;

    @Override
    public String toString() {
        String out = "Number of repositories: " + items.size();
        for (Repository r : items) {
            out += "{\n" + r.toString() + "\n}\n";
        }
        return out;
    }
}
{{< / highlight >}}

Ora che abbiamo queste due classi, creiamo una classe principale che si occuperà di mandare una richiesta a Github e di utilizzare Gson per creare un'istanza di Repositories. Per ricercare le repositories che ci interessano basterà fornire come argomento una chiave di ricerca quando avviamo il programma.

{{< highlight java >}}
package gsonparsing;
import java.io.*;
import java.net.*;
import com.google.gson.Gson;


public class GithubSearch {

    private static final String GITHUB_BASE_URL = "https://api.github.com/search/repositories";
    private static final String QUERY_CHAR = "q";
    private static final String SORT = "sort";
    private static final String BYSTARS = "stars";

    public static void main(String[] args) {
        if (args.length != 1) {
            String response = "Attenzione! \n";
            response += "Utilizzo: java GithubSearch <nomedellaRepository>";
            System.out.println(response);
            System.exit(1);
        }

        String query = args[0];
        String requestUrl = buildRequestUrlString(query);

        try {
            String result = getHTML(requestUrl);
            System.out.println(result);
        } catch (Exception e) {
            System.out.println("Errore nella richiesta");
            System.out.println(e.toString());
        }


    }

    private static String buildRequestUrlString(String query) {
        return GITHUB_BASE_URL + "?q=" + query + '&' + SORT + '=' + BYSTARS;
    }

    public static String getHTML(String urlToRead) throws Exception {
        StringBuilder result = new StringBuilder();
        URL url = new URL(urlToRead);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("GET");
        BufferedReader rd = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        Gson gson = new Gson();
        Repositories repositories = null;
        repositories = gson.fromJson(rd, Repositories.class);        
        return repositories.toString();
   }
}
{{< / highlight >}}

Come puoi vedere, Gson si occupa di fare tutto il lavoro. Per deserializzare il Json abbiamo semplicemente chiamato il metodo *fromJson()* sull'oggetto Gson, e gli abbiamo passato il *BufferedReader* con la nostra risposta e la nostra classe Repositories. 

