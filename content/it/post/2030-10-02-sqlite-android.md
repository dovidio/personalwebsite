---
author: Umberto D'Ovidio
date: "2030-10-02T00:00:00Z"
description: Come salvare i dati della nostra applicazione usando SQLite
title: Utilizzare SQLite in Android
---

La maggior parte delle applicazioni Android ha bisogno di salvare dei dati in qualche modo. Ci sono diverse soluzioni possibili,
che dipendono in gran parte dal tipo di problema. Oggi vedremo come utilizzare un database SQLite in Android.

<!--more-->

Android include nel suo runtime una versione di [SQLite](https://www.sqlite.org/), un database basato su SQL leggero ed efficiente, ideale per dispositivi che non dispongono di molta potenza di calcolo.
Per creare, cancellare, modificare ed inserire record in un database SQLite, Android include una propria API simile a JDBC.

In questo tutorial vedremo come utilizzare questa API servendoci di una semplice app come esempio. La nostra app è disponibile su [github](), e permette di creare una lista di "cose da fare", che verranno salvate nel database e saranno quindi visibili anche dopo aver chiuso e riaperto l'app. 


### Contratto

Per prima cosa abbiamo definito un contratto, ovvero una classe contenente delle costanti che definiscono i  nomi delle nostre tabelle e delle colonne che esse contengono. La [documentazione ufficiale](https://developer.android.com/training/basics/data-storage/databases.html) di Android suggerisce di creare una classe innestata per ciascuna tabella contenuta nel nostro database. Siccome nel nostro esempio abbiamo una sola tabella, avremmo solo una classe innestata.

{{< highlight java >}}
public final class ToDoContract {

    private ToDoContract() {} // impediamo che qualcuno istanzi questa classe

    public static class ToDo implements BaseColumns {
        public static final String TABLE_NAME = "todo";
        public static final String COLUMN_NAME_TITLE = "title";
        public static final String COLUMN_NAME_DESCRIPTION = "description";
    }
}
{{< / highlight >}}

### DatabaseHelper

Sucessivamente avremmo bisogno di una classe che estende *SQLiteOpenHelper*. In questa classe implementeremo i metodi *onCreate* e *onUpgrade* che verranno chiamati rispettivamente quando creiamo per la prima volta il database e quando installiamo la nostra app con una versione del database più recente. 

{{< highlight java >}}
public class DatabaseHelper extends SQLiteOpenHelper {

    private static DatabaseHelper sInstance;
    private static final String DATABASE_NAME = "todo.db";
    private static final int DATABASE_VERSION = 1;

    // in questo modo evitiamo che venga istanziato senza seguire il singleton pattern
    private DatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    private static final String SQL_CREATE_TABLE =
            "CREATE TABLE " + ToDoContract.ToDo.TABLE_NAME + " (" +
            ToDoContract.ToDo._ID + " INTEGER PRIMARY KEY," +
            ToDoContract.ToDo.COLUMN_NAME_TITLE + " TEXT NOT NULL," +
            ToDoContract.ToDo.COLUMN_NAME_DESCRIPTION + " TEXT NOT NULL)";

    private static final String SQL_DELETE_TABLE = "DROP TABLE IF EXISTS " + ToDoContract.ToDo.TABLE_NAME;

    public static synchronized DatabaseHelper getInstance(Context context) {
        if (sInstance == null) {
            sInstance = new DatabaseHelper(context.getApplicationContext());
        }
        return sInstance;
    }

    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL(SQL_CREATE_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {
        sqLiteDatabase.execSQL(SQL_DELETE_TABLE);
        onCreate(sqLiteDatabase);
    }
}
{{< / highlight >}}

Ci serviremo della nostra classe *DatabaseHelper()* ogni volta che abbiamo bisogno di un database sul quale effettuare delle operazioni. Basterà chiamare il metodo *getWritableDatabase()*. Per istanziare il nostro DatabaseHelper ci basterà chiamare il metodo *getInstance*.

Abbiamo già visto come cancellare e creare un database, ora vedremo come inserire record, fare un update sui nostri record e infine come cancellarli. 

### Inserimento di un record

Per quanto riguarda l'inserimento ci serviremo del metodo *insert()* chiamato su un database scrivibile che abbiamo richiesto al nostro *DatabaseHelper*. Questo metodo richiede tre parametri: il nome della nostra tabella,
il nome di una colonna per il "*null column hack*" e un oggetto *ContentValues*. Per creare una nuova riga, SQLite ha bisogno che venga definito un valore per almeno una colonna. 
La classe *ContentValues* è una sorta di mappa chiave-valore, dove la chiave è rappresentata dal nome della colonna della nostra tabella e il valore è il valore che vogliamo inserire. Per inserire dei valori, dovremo quindi istanziare un oggetto *ContentValues* ed inserire i valori attraverso il metodo *put()*. Una volta che abbiamo inserito tutti i valori voluti, chiamiamo *insert()* passandogli quest'istanza di *ContentValues*.
Nel caso in cui nel nostro insert passiamo un oggetto *ContentValues* vuoto, allora non sarebbe possibile creare una nuova riga. Se invece passiamo anche il nome di una colonna per il *null hack* allora la riga verrà creata con successo e verrà inserito il valore NULL nella colonna che abbiamo fornito. Per maggiori informazioni consulta la [documentazione ufficiale](https://developer.android.com/reference/android/database/sqlite/SQLiteDatabase.html#insert(java.lang.String, java.lang.String, android.content.ContentValues)).
Vediamo un esempio pratico.

{{< highlight java >}}
        DatabaseHelper databaseHelper = DatabaseHelper.getInstance(context);
        SQLiteDatabase writeDatabase = databaseHelper.getWritableDatabase();
        ContentValues cv = new ContentValues();

        cv.put(ToDoContract.ToDo.COLUMN_NAME_TITLE, "Bucato");
        cv.put(ToDoContract.ToDo.COLUMN_NAME_DESCRIPTION, "Stendere il bucato");
        writeDatabase.insert(ToDoContract.ToDo.TABLE_NAME, null, cv);


        cv.put(ToDoContract.ToDo.COLUMN_NAME_TITLE, "Spesa");
        cv.put(ToDoContract.ToDo.COLUMN_NAME_DESCRIPTION, "Fare la spesa");
        writeDatabase.insert(ToDoContract.ToDo.TABLE_NAME, null, cv);

        databaseHelper.close();
        writeDatabase.close();
{{< / highlight >}}


### Update di un record 

Per quanto riguarda l'update dei nostri record, ci serviremo del metodo *update()* chiamato sempre sul nostro database. 
I parametri richiesti da questo metodo sono il nome della tabella, un oggetto *ContentValues* che rappresenta le colonne e i valori di rimpiazzo da usare, una clausola *WHERE* che specifica su quali record fare l'update e una lista opzionale di parametri che si può utilizzare nel caso in cui la nostra clausola *WHERE* contenga dei punti di domanda.
Di seguito abbiamo un esempio pratico:

{{< highlight java >}}
        DatabaseHelper databaseHelper = DatabaseHelper.getInstance(context);
        SQLiteDatabase writeDatabase = databaseHelper.getWritableDatabase();
        ContentValues cv = new ContentValues();

        cv.put(ToDoContract.ToDo.COLUMN_NAME_DESCRIPTION, "Fare la spesa settimanale");

        writeDatabase.update(
                ToDoContract.ToDo.TABLE_NAME,
                cv,
                ToDoContract.ToDo.COLUMN_NAME_TITLE + " = ?",
                new String[] {"Spesa"});

        databaseHelper.close();
        writeDatabase.close();
{{< / highlight >}}


### Cancellazione record 

Vediamo infine come cancellare un record. Avrai intuito che possiamo utilizzare un metodo *delete()* chiamato sul nostro database. Esso richiede il nome della tabella, una clausola *WHERE* e una lista di parametri nel caso in cui la clausola *WHERE* contenga il punto di domanda. Ecco un esempio pratico: 

{{< highlight java >}}
        DatabaseHelper databaseHelper = DatabaseHelper.getInstance(context);
        SQLiteDatabase writeDatabase = databaseHelper.getWritableDatabase();

        writeDatabase.delete(ToDoContract.ToDo.TABLE_NAME,
                ToDoContract.ToDo.COLUMN_NAME_TITLE + " = ?",
                new String[] {"Bucato"});

        databaseHelper.close();
        writeDatabase.close();
{{< / highlight >}}

Volendo si possono effettuare le operazioni di inserimento, update e cancellazione utilizzando la sintassi SQL e passando il comando SQL alla funzione *execSQL()*.

### Interrogazione database

Per interrogare un database sfruttiamo il metodo *rawQuery*, che chiamiamo sempre sul nostro oggetto Database. I parametri richiesti sono una stringa con la quale effettuiamo un *SELECT* sul database e un parametro opzionale rappresentato da una lista di stringhe dove metteremo i nomi delle colonne che andranno a rimpiazzare i punti di domanda presenti all'interno della nostra query (se ce ne sono).
Il risultato di questo metodo è un *Cursor*, ovvero un oggetto che ci permette di scorrere i risultati ottenuti dalla query. 
Per scorrere i risultati ci basterà chiamare il metodo *moveToNext()* su questo oggetto. Questo farà puntare il nostro *Cursor* alla prossima riga e ci ritorna un booleano che indica se ci sono altre righe da scorrere o meno. Basterà dunque un semplice loop del tipo *while(cursor.moveToNext()) { // fai quello che devi fare }*. 
Possiamo poi ottenere i valori delle colonne per la riga corrente usando dei metodi del tipo get. Abbiamo ad esempio *getInt(int i)*,
*getString(int i)*, *getBoolean(int i)* etc... Il parametro richiesto da questi metodi è un intero che specifica il numero della colonna del nostro risultato. Per ottenere questo intero è sufficiente chiamare il metodo *getColumnIndexOrThrow(String s)* passandogli il nome della colonna che ci interessa come parametro. Se questa non è presente, verrà sollevata un'eccezione. 
Di seguito abbiamo un esempio di interrogazione su tutti i record presenti all'interno della nostra unica tabella, che vengono poi stampati sulla console. 

{{< highlight java >}}
        DatabaseHelper databaseHelper = DatabaseHelper.getInstance(context);
        SQLiteDatabase writeDatabase = databaseHelper.getWritableDatabase();

        String query = "SELECT * FROM " + ToDoContract.ToDo.TABLE_NAME;

        Cursor result = writeDatabase.rawQuery(query, null);

        String out = "N\tTITLE\tDESCRIPITON\n";
        
        while (result.moveToNext()) {
            int id = result.getInt(result.getColumnIndexOrThrow(ToDoContract.ToDo._ID));
            String title = result.getString(result.getColumnIndexOrThrow(ToDoContract.ToDo.COLUMN_NAME_TITLE));
            String description = result.getString(result.getColumnIndexOrThrow(ToDoContract.ToDo.COLUMN_NAME_DESCRIPTION));
            out += id + "\t" + title + "\t" + description + "\n";
        }

        System.out.println(out);
        
        databaseHelper.close();
        writeDatabase.close();
        result.close();
{{< / highlight >}}

### Mostrare la nostra lista con il CursorAdapter






