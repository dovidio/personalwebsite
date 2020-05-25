---
author: Umberto D'Ovidio
date: "2017-08-17T00:00:00Z"
description: Cosa sono gli Intent ed Intent Filter in Android e come funzionano?
title: Passare da un Activity all'altra con gli Intent
---

Ogni sviluppatore Android che si rispetti deve conoscere molto bene l'utilizzo degli **Intent**, questo perchè sono una delle classi più usate in assoluto. Se li hai già utilizzati ma non hai mai approfondito il discorso, allora sei nel posto giusto.
<!--more-->

La documentazione ufficiale [Android](https://developer.android.com/reference/android/content/Intent.html) definisce un **Intent** come una descrizione astratta di un'operazione da eseguire. 
Possiamo vedere un Intent come qualcosa di simile ad una richiesta HTTP. Specifica il nome di una risorsa (che in Android viene chiamata *data*) e un'azione che si vuole svolgere (che chiameremo *action*). 
Mentre in HTTP le azioni sono verbi come *GET*, *PUT*, *DELETE*, negli Intent queste sono costanti come *ACTION_VIEW* o *ACTION_EDIT*.
Nella tua classe Android puoi dunque creare un Intent nel seguente modo:

{{< highlight java >}}
Intent intent = new Intent(ACTION_VIEW, "http://dovid.io");
{{< / highlight >}}

chiamando poi *startActivity(intent)* Android andrà alla ricerca dell'Activity più adatta per effettuare quel determinato tipo di azione su quel determinato tipo di risorsa (in questo caso aprirebbe un web browser).
Oltre all'*action* e ai *data*, ci sono altri tipi di informazione che puoi passare ad un Intent: 
- Categoria (*category*): fornisce informazione aggiuntiva sull'azione da eseguire. Ad esempio *CATEGORY_LAUNCHER* significa che dovrebbe apparire come applicazione in primo piano.
- Tipo (*type*): specifica il tipo [MIME](https://it.wikipedia.org/wiki/Multipurpose_Internet_Mail_Extensions) dei *data* che passiamo. Normalmente questo campo non è necessario perchè il tipo di MIME viene estrapolato automaticamente da Android, tuttavia ci sono dei casi in cui risulta utile fornire un tipo esplicito.
- Component (*component*): specifica il nome di una classe componente da usare per l'Intent. Di solito anche questa informazione viene estrapolata dalle altre informazioni (*action* e *data*) che vengono assegnate ad un componente in grado di gestirle. Se però settiamo questo attributo, allora non ci sarà l'estrapolazione automatica ma verrà usato il componente fornito. 
- Extras: informazioni aggiuntive che vogliamo passare al componente che verrà eseguito.  

### Tipi di Intent

Basandoci su quanto detto fino adesso, possiamo dire che ci sono due tipi di Intent: quelli espliciti, dove forniamo noi il componente che verrà creato, e quelli impliciti, dove a partire da *type*, *category* e *action* viene cercato il componente più adatto. E' possibile che ci siano più componenti che possono gestire la richiesta, in tal caso verrà chiesto all'utente specificamente quale componente scegliere. 

### Intent Filter

Tutti i componenti che vogliono essere inizializzati tramite Intent devono dichiarare un **Intent Filter**, in modo che Android sappia cosa viene supportato dal componente. Per far ciò è sufficiente aggiungere un *<intent-filter>* all'interno di un <activity> nel nostro file **AndroidManifest.xml**.  Ad esempio, quando creiamo un nuovo progetto in Android Studio, alla classe principale viene aggiunto automaticamente il seguente Intent Filter:

{{< highlight xml >}}
<intent-filter>
    <action android:name="android.intent.action.MAIN" />    
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
{{< / highlight >}}

E' possibile avere anche più di un *action* nel nostro intent filter. È possibile poi specificare il tipo di *data* usando: 

{{< highlight xml >}}
<data android:mimeType="text/plain">
{{< / highlight >}}

In questo caso il tipo di dato è *text/plain*, ma è ovviamente possibile scegliere altri tipi. 

### Come ottenere i dati passati con l'Intent

Supponiamo di avere un Activity che dichiara nel suo manifesto un intent-filter contenente un' *action* del tipo *android.intent.action.VIEW* e un tipo di dato *text/plain*. Questo permette dunque alla nostra Activity di essere scelta quando qualcuno lancia un Intent con l'azione e il tipo corrispondente. Come facciamo dunque ad ottenere i dati che ci vengono mandati? Anzitutto ricordiamo che per includere dei dati in un Intent, dobbiamo invocare il metodo *putExtra()* passando come argomenti un identificatore dell'informazione che stiamo passando  e l'informazione vera e propria. 
Quando siamo poi nella nostra Activity che gestisce l'Intent, chiamiamo il metodo *getStringExtra()* passandogli un'identificatore per l'informazione che ci interessa. Per evitare di utilizzare diversi identificatori per lo stesso tipo di informazione, Android fornisce già alcune stringhe costanti per determinati tipi di informazione, e consiglio di usarle sempre qualora possibile.

### Esempio pratico

Facciamo ora un esempio pratico creando due semplici applicazioni. All'interno della prima creiamo un Intent per leggere del testo, mentre nella seconda applicazione settiamo un intent-filter che dichiari che la nostra Activity è in grado di leggere testo. 
Entrambe le app sono disponibili su Github, la prima a [questo](https://github.com/Cyborg101/dovidioTutorials/tree/master/DelegoLettura) indirizzo, la seguente a quest'[altro](https://github.com/Cyborg101/dovidioTutorials/tree/master/Leggo).

Come puoi vedere le app sono entrambe costituite da una sola classe. 
Nella prima app, che ho chiamato **DelegoLettura**, abbiamo una semplice schermata con un pulsante. Quando il pulsante viene premuto, verrà chiesto di scegliere un'app per leggere del testo. La seconda app, chiamata **Leggo**, si occuperà invece di leggere il testo.

Diamo ora un'occhiata al codice sorgente di entrambe le app, a cominciare da **DelegoLettura**.

{{< highlight java >}}
package io.dovid.delegolettura;

import android.content.Intent;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;


/* Author: Umberto D'Ovidio
 * Website: http://dovid.io
 * Email: umberto.dovidio@gmail.com
 * Tutorial page: http://dovid.io/guida-intent-android.html
 */
public class MainActivity extends AppCompatActivity {

    private Button delegaButton;
    private final String lorem = "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Aenean ut gravida lorem.";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        delegaButton = (Button) findViewById(R.id.delega_button);

        delegaButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent();
                intent.setAction(Intent.ACTION_VIEW);
                intent.setType("text/plain");
                intent.putExtra(Intent.EXTRA_TEXT, lorem);
                startActivity(Intent.createChooser(intent, "Scegli un'app per leggere il testo"));
            }
        });
    }
}
{{< / highlight >}}

Come puoi vedere, il codice che crea l'Intent e lancia una nuova Activity viene eseguito solo quando il bottone viene premuto. Abbiamo utilizzato l'identificatore di default Intent.EXTRA_TEXT e passiamo alla nuova activity una stringa con un testo. In questo caso abbiamo forzato l'utente a scegliere un'app per completare l'azione, questo perchè è possibile che ci sia un'app di default per leggere il testo e dunque l'utente non avrebbe la possibilità di scegliere la nostra app **Leggo** per effettuare la lettura. 

Ora passiamo alla classe **Leggo**. 

{{< highlight xml >}} 
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="io.dovid.leggo">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            <intent-filter android:label="@string/app_name">
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data android:mimeType="text/plain"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
{{< / highlight >}}

Nella nostra classe principale abbiamo due intent-filter: il primo indica che questa è la classe principale all'interno dell'app ed è quella che viene creata quando facciamo tap sulla launcher icon, mentre nel secondo intent filter dichiariamo che questa classe può essere chiamata da altre classi ed è in grado di leggere testo non formattato. Quando premiamo  il bottone nella prima app, apparirà la seconda app fra le possibili scelte per completare l'azione (ricordati di installarla perchè ciò avvenga). 
Ora esaminiamo il codice della classe principale.

{{< highlight java >}}
package io.dovid.leggo;

import android.content.Intent;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.TextView;
import android.widget.Toast;

/* Author: Umberto D'Ovidio
 * Website: http://dovid.io
 * Email: umberto.dovidio@gmail.com
 * Tutorial page: http://dovid.io/guida-intent-android
 */
public class MainActivity extends AppCompatActivity {

    private TextView testoLetto;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        testoLetto = (TextView) findViewById(R.id.text);
        String messaggio = getIntent().getStringExtra(Intent.EXTRA_TEXT);

        if (messaggio != null) {
            testoLetto.setText(messaggio);
        } else {
            testoLetto.setVisibility(View.GONE);
            Toast.makeText(this, "Non è stato fornito alcun testo", Toast.LENGTH_LONG).show();
        }
    }
}
{{< / highlight >}}

In modo molto semplice cerchiamo l'informazione data dall'Intent con un *getStringExtra* al quale passiamo un identificatore. Se l'informazione è presente la mostriamo a schermo, altrimenti mostriamo un messaggio di errore.
