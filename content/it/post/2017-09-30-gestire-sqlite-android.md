---
author: Umberto D'Ovidio
date: "2017-09-30T00:00:00Z"
description: Ecco la migliore pratiche che ho scoperto sulla gestione di sqlite in
  Android.
feature-img: assets/default.png
feature-img-lg: assets/default.png
feature-img-md: assets/default.png
feature-img-sm: assets/default.png
feature-img-xs: assets/default.png
image: /images/sqlite_promo_og.png
title: Come gestire SQLite su Android senza diventare pazzo
---

Se vuoi salvare dati nella tua app Android, quasi sicuramente utilizzerai il database SQLite. Purtroppo però la documentazione ufficiale su come gestirlo lascia molto a desiderare. In questo post scopriremo alcune tecniche che possono sembrare controintuitive, ma ti risparmieranno un sacco di grattacapi.

<!--more-->

Recentemente mi sono trovato a sviluppare un'app Android piuttosto complessa che richiede la sincronizzazione con un server. I dati dell'app vengono infatti salvate in un database remoto, in modo che quando l'utente rientra con il suo account può sempre ritrovare i dati aggiornati indipendentemente dal dispositivo utilizzato. Quando il dispositivo è connesso a internet non c'è bisogno di particolari accorgimenti in quanto tutti i dati che vengono prodotti localmente vengono mandati al server quasi in contemporanea. Ma cosa succede se l'utente è offline?

In questo caso l'app registra le azioni effettuate dall'utente all'interno di un database SQLite locale, e quando torna online effettua delle operazioni di sincronizzazione con il database remoto. Per rendere l'esperienza d'uso più fluida possibile ho deciso di permettere all'utente di continuare ad utilizzare l'app normalmente mentre le operazioni di sincronizzazione vengono effettuate in un altro thread. E' qui che ho iniziato a scontrarmi con alcuni problemi di SQLite.

Android offre diverse classi che permettono di interagire con SQLite in maniera semplice. Fra queste abbiamo [SQLiteOpenHelper](https://developer.android.com/reference/android/database/sqlite/SQLiteOpenHelper.html), una classe che aiuta nel processo di creazione del database e nella gestione delle versioni. 
Per creare un database SQLite, la documentazione ufficiale suggerisce di estendere questa classe ed eseguire il codice di creazione del database all'interno del metodo [onCreate(SQLiteDatabase db)](https://developer.android.com/reference/android/database/sqlite/SQLiteOpenHelper.html#onCreate(android.database.sqlite.SQLiteDatabase)).
Facciamo un esempio pratico di seguito:

{{< highlight java >}}
public class DatabaseHelper extends SQLiteOpenHelper {

    public DatabaseHelper(Context context) {
        super(context.getApplicationContext(), "db", null, 1);

    }

    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL("CREATE TABLE TODO (ID INTEGER PRIMARY KEY, NAME TEXT NOT NULL)");
    }
}
{{< / highlight >}}

In questo esempio abbiamo creato un database con una tabella chiamata TODO (che in inglese sta per "cose da fare") e abbiamo creato due colonne, una per l'id e una per il nome.
Il database viene creato la prima volta che istanziamo la classe DatabaseHelper. Non solo, la classe DatabaseHelper ci ritorna degli oggetti SQLiteDatabase attraverso i suoi metodi [getReadableDatabase()](https://developer.android.com/reference/android/database/sqlite/SQLiteOpenHelper.html#getReadableDatabase()) e [getWritableDatabase()](https://developer.android.com/reference/android/database/sqlite/SQLiteOpenHelper.html#getWritableDatabase()) attraverso il quale possiamo svolgere operazioni di lettura e scrittura sul database. 
Questi oggetti possono essere chiusi una volta effettuata l'operazione di scrittura/lettura tramite il metodo **close()**.

Secondo la documentazione ufficiale, una volta ottenuta una connessione al database, questa connessione può rimanere aperta fin tanto che la tua app ne abbia bisogno, e viene suggerito di chiuderla nel metodo **onDestroy()**. 

Nel mio caso ogni **Activity** che aveva bisogno di accedere al database aveva una sua istanza dell'oggetto **DatabaseHelper**, che veniva chiuso nel metodo **onDestroy()**.
Tutto funzionava a meraviglia finchè non ho inserito il codice di sincronizzazione all'interno di un altro Thread. Infatti dopo aver implementato la sincronizzazione ho iniziato a riscontrare alcuni crash fastidiosi e piuttosto imprevedibili.

L'app crashava infatti con un errore di questo tipo: 

{{< highlight bash >}}
android.database.sqlite.SQLiteDatabaseLockedException: database is locked (code 5): , while compiling: PRAGMA journal_mode
{{< / highlight >}}

Dopo un pò di ricerca e debugging ho scoperto la cause del problema. Nel codice di sincronizzazione, ad un certo punto, venivano inserite alcune azioni utilizzando una transazione. Durante questa fase della sincronizzazione una delle mie Activity cercava di accedere al database per leggere alcuni dati e fare un refresh dell'interfaccia grafica. Il database era però già occupato e dunque veniva restituito questo errore. 

A questo punto ho iniziato un'estenuante ricerca Google, dove ho letto tutto ed il contrario di tutto sull'argomento. Fortunatamente ad un certo punto ho trovato questo [post](https://stackoverflow.com/questions/2493331/what-are-the-best-practices-for-sqlite-on-android#answer-3689883) su Stack Overflow. In breve viene spiegato che ogni oggetto del tipo **SQLiteOpenHelper** ha una singola connessione al database, e che le operazioni effettuate sul database tramite questo oggetto vengono serializzate. Se però vengono create più connessioni e due di queste cercano di scrivere nel database contemporaneamente, una di queste fallisce. Come fare dunque? Semplice, basta essere sicuri che ci sia una sola connessione ogni volta. 
Per far utilizziamo il [Singleton Pattern](https://it.wikipedia.org/wiki/Singleton) e accediamo alla nostra singola connessione attraverso un metodo synchronized. Il codice di prima diventa così: 

{{< highlight java >}}
public class DatabaseHelper extends SQLiteOpenHelper {

    private static DatabaseHelper db;

    // il costruttore è privato così siamo sicuri che nessuno possa creare più di una connessione
    private DatabaseHelper(Context context) {
        super(context.getApplicationContext(), "db", null, 1);

    }

    public static synchronized DatabaseHelper getInstance(Context context) {
        if (db == null) {
            db = new DatabaseHelper(context);
        }
        return db;
    }

    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL("CREATE TABLE TODO (ID INTEGER PRIMARY KEY, NAME TEXT NOT NULL)");
    }

    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {

    }
}
{{< / highlight >}}

Ogni volta che hai bisogno di un oggetto **DatabaseHelper** basterà dunque richiamarlo nel seguente modo:

{{< highlight java >}}
DatabaseHelper databaseHelper = DatabaseHelper.getInstance(context);
{{< / highlight >}}

Una volta implementato questo cambiamento, ho però iniziato a riscontrare un altro tipo di eccezione:
{{< highlight java >}}
java.lang.IllegalStateException: Cannot perform this operation because the connection pool has been closed.
{{< / highlight >}}

Il problema è che tutto il mio codice che effettuava operazioni su database era strutturato nel seguente modo: 

{{< highlight java >}}
SQLiteDatabase writeDatabase = null;
try {
    writeDatabase = databaseHelper.getWritabaleDatabase();

    // fai qualche operazione sul database

} catch (SQLiteException e) {
    // cattura l'eccezione e facci qualcosa
} finally {
    if (writeDatabase != null && writeDatabase.isOpen()) {
        writeDatabase.close();
    }
}
{{< / highlight >}}

Poteva dunque succedere che nel codice di sincronizzazione avevo chiuso l'ultima connessione al database, mentre nel codice all'interno delle mie activity stavo cercando di effettuare una query. Ho quindi deciso di fare un drastico cambio al codice eliminando tutte gli statement del tipo **writeDatabase.close()**. 

*"Aspetta, mi stai dicendo che non chiudi mai la connessione al database?"*

Si, è così. Nel caso in cui accedi al database dallo sempre dallo stesso thread invece non dovresti avere questi problemi. 

*"Ma non si rischiano memory leaks tenendo delle connessioni aperte?"*

No, perchè la connessione che rimane aperta è solo una. L'importante è chiudere sempre gli oggetti del tipo Cursor. Vedi [questa domanda su Stack Overflow](https://stackoverflow.com/questions/7211941/never-close-android-sqlite-connection)

*"E per quanto riguarde l'errore "Leak found ..." che vedo spesso su LogCat?"*

Quell'errore appare quando apri un database senza chiuderlo ed in seguito ne apri un altro.

Per avere una prova empirica che tutto quello che ho detto funziona, ho creato una [piccola app](https://github.com/Cyborg101/dovidioTutorials/tree/master/DatabaseLocking). L'app in questione ha 4 bottoni. Quando viene premuto uno di questi bottoni, vengono creati e mandati in esecuzione 10 thread diversi. Ciascuno di questi thread aggiunge 10 record nel database. 
Ci sono 4 pulsanti e ognuno di questi fa cose diverse. Le possibilità sono: 
1. Viene utilizzato un singolo databaseHelper comune a tutti i thread e vengono chiuse le connessioni al termine della scrittura.
2. Vengono utilizzati diversi databaseHelper, uno per thread, e vengono chiuse le connessioni al termine della scrittura.
3. Viene utilizzato un singolo databaseHelper comune a tutti i thread e non vengono chiuse le connessioni al termine della scrittura.
4. Vengono utilizzati diversi databaseHelper, uno per thread, e non vengono chiuse le connessioni al termine della scrittura.

Provando e riprovando l'app ho notato che l'unico pulsante che non porta al crash dell'app è quello che utilizza un solo databaseHelper e non chiude le connessioni. 
I pulsanti del tipo 2 e 4 infatti riportano un errore di database lock. Il pulsante 1 invece è particolare. Sul mio dispositivo fisico (Samsung s5 mini) fa terminare l'app senza loggare nessun errore. Sull'emulatore invece viene riportato un errore di database lock.

Spero di esserti stato d'aiuto con questo post, se hai qualche osservazione o precisazione non esitare a lasciare un commento.



