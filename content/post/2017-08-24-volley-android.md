---
author: Umberto D'Ovidio
date: "2017-08-24T00:00:00Z"
description: Come effettuare richieste http su Android con Volley
title: Utilizzare Volley per eseguire richieste http
---

**Volley** è una libreria HTTP creata da Google che permette di fare operazioni di networking in Android in maniera semplice e veloce. 

<!--more-->

Supporta nativamente stringhe, Json e immagini.
Il punto di forza rispetto a tutte le altre librerie in Android è sicuramente il supporto di Google che la utilizza attivamente nelle sue app.  Uno dei punti deboli è la scarsa documentazione, per cui spesso bisogna andare a leggersi il [codice sorgente](https://github.com/google/volley) per capire come ottenere determinati risultati. 

### Una semplice richiesta http in volley

Prima di tutto, se vuoi utilizzare Volley nel tuo progetto Android, dovrai aggiungerla alle dipendenze nel file build.gradle:

{{< highlight gradle >}}
dependencies {
    ...
    compile 'com.android.volley:volley:1.0.0'
}
{{< / highlight >}}

Volley utilizza una coda per gestire le richieste http. È possibile creare l'oggetto *RequestQueue* utilizzando il metodo statico *newRequestQueue()* della classe *Volley*. Una volta ottenuta una coda di richieste, possiamo creare la nostra richiesta e aggiungerla alla coda. Volley supporta di default 3 tipi di richieste: 
- *StringRequest*
- *JsonRequest*
- *InputStreamRequest* 

E' possibile poi creare un tipo di richiesta personalizzato estendendo l'oggetto *Request* e implementando i metodi *parseNetworkResponse()* e *deliverResponse()*. Per ora vedremo un semplice esempio che utilizza la *StringRequest*.

{{< highlight java >}} 
public class VolleyTutorial extends AppCompatActivity {

    private static final String TAG_RESPONSE = "VOLLEYRESPONSE";
    private static final String TAG_ERROR = "VOLLEYERROR";
    private static final String TAG_REQUEST = "VOLLEYREQUEST";
    private Button sendRequestButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_volley_tutorial);
        sendRequestButton = (Button) findViewById(R.id.request_btn);

        // Instantiate the RequestQueue.
        final RequestQueue queue = Volley.newRequestQueue(this);
        final String url = "http://www.google.com";

        sendRequestButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                faiRichiesta1(queue, url);
                faiRichiesta2(queue, url);
            }
        });
    }

    public void faiRichiesta1(final RequestQueue queue, final String url) {
        StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
                new Response.Listener<String>() {
                    @Override
                    public void onResponse(String response) {
                        // Display the first 500 characters of the response string.
                        Log.d(TAG_RESPONSE, "La risposta 2 è: " + response.substring(0, 500));
                    }
                }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                Log.e(TAG_ERROR, "non ha funzionato: " + error.toString());
            }
        });
        queue.add(stringRequest);
        Log.d(TAG_REQUEST, "aggiunta richiesta 2 alla coda");
    }

    public void faiRichiesta2(final RequestQueue queue, final String url) {
        StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
                new Response.Listener<String>() {
                    @Override
                    public void onResponse(String response) {
                        // Display the first 500 characters of the response string.
                        Log.d(TAG_RESPONSE, "La risposta 2 è: " + response.substring(0, 500));
                    }
                }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                Log.e(TAG_ERROR, "non ha funzionato: " + error.toString());
            }
        });
        queue.add(stringRequest);
        Log.d(TAG_REQUEST, "aggiunta richiesta 2 alla coda");
    }
    
}
{{< / highlight >}}

{{< highlight xml >}} 
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="io.dovid.volleytutorial.VolleyTutorial">

    <Button
        android:id="@+id/request_btn"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="8dp"
        android:layout_marginLeft="8dp"
        android:layout_marginRight="8dp"
        android:layout_marginTop="8dp"
        android:text="@string/send_request"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
</android.support.constraint.ConstraintLayout>
{{< / highlight >}}

Questa activity ha un bottone che una volta cliccato manda due richieste http a google e logga la risposta nella console di android studio. Creiamo una StringRequest con il costruttore che chiede il metodo della richiesta (GET, POST, PUT, DELETE etc...), una stringa che rappresenta un url, un oggetto Response.Listener ed un Error.Listener. Abbiamo creato entrambi i listener inline creando due classi anonime. 

Nella prima classe dobbiamo implementare il metodo *onResponse(String response)* che verrà chiamato quando riceviamo una risposta valida. Nella seconda classe invece implementiamo il metodo *onErrorResponse(VolleyError error)* che viene chiamato qualora ci sia un errore nella richiesta o nella risposta http.

Data la natura asincrona della libreria, la seconda richiesta http che facciamo viene mandata prima di ricevere la risposta dalla prima richiesta. Ci sono però dei casi in cui vogliamo eseguire una richiesta http solo un'altra richiesta ha ricevuto una risposta. Per esempio facciamo finta di voler fare una richiesta a google, una a yahoo e una a bing. Inoltre vogliamo che la richiesta a yahoo venga fatta dopo aver ricevuto risposta da google, e la richiesta bing deve essere fatta dopo aver ricevuto risposta da yahoo. Potremmo fare così:

{{< highlight java >}}
        final String googleUrl = "http://www.google.com";
        final String yahooUrl = "http://yahoo.com/";
        final String bingUrl = "http://www.bing.com";

        StringRequest googleRequest = new StringRequest(Request.Method.GET, googleUrl, new Response.Listener<String>() {
            @Override
            public void onResponse(String response) {
                Log.d(TAG_RESPONSE, "risposta google: " + response.toString());
                StringRequest yahooRequest = new StringRequest(Request.Method.GET, yahooUrl, new Response.Listener<String>() {
                    @Override
                    public void onResponse(String response) {
                        Log.d(TAG_RESPONSE, "risposta yahoo: " + response.toString());
                        StringRequest bingRequest = new StringRequest(Request.Method.GET, bingUrl, new Response.Listener<String>() {
                            @Override
                            public void onResponse(String response) {
                                Log.d(TAG_RESPONSE, "risposta bing: " + response.toString());
                            }
                        }, new Response.ErrorListener() {
                            @Override
                            public void onErrorResponse(VolleyError error) {
                                error.printStackTrace();
                            }
                        });
                        queue.add(bingRequest);
                        Log.d(TAG_REQUEST, "aggiunta richiesta bing");
                    }
                }, new Response.ErrorListener() {
                    @Override
                    public void onErrorResponse(VolleyError error) {
                        error.printStackTrace();
                    }
                });
                queue.add(yahooRequest);
                Log.d(TAG_REQUEST, "aggiunta richiesta yahoo");
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                error.printStackTrace();
            }
        });
        Log.d(TAG_REQUEST, "aggiunta richiesta google");
        queue.add(googleRequest);
{{< / highlight >}}

Questo tipo di soluzione molto illeggibile quindi è fuori discussione. Dobbiamo trovare un altro modo.

### Enter the Future

Volley contiene una classe chiamata *RequestFuture* che può essere considerato come il risultato della richiesta asincrona. Possiamo passare al costruttore di *StringRequest* un oggetto *RequestFuture*, dicendo così a volley di mettere il risultato della richiesta http all'interno dell'oggetto *RequestFuture*. Per ottenere il risultato della richiesta sarà dunque sufficiente chiamare il metodo *get()* sull'oggetto future. Se il nostro oggetto non ha ancora ricevuto risposta, allora bloccherà il thread sulla quale è eseguito.

Questo è estremamente utile nel problema che abbiamo presentato prima. Proviamo ora a riscrivere il codice di prima con questo nuovo concetto.

{{< highlight java >}} 
public class VolleyTutorial extends AppCompatActivity {

    private static final String TAG_RESPONSE = "VOLLEYRESPONSE";
    private static final String TAG_ERROR = "VOLLEYERROR";
    private static final String TAG_REQUEST = "VOLLEYREQUEST";
    private Button sendRequestButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_volley_tutorial);
        sendRequestButton = (Button) findViewById(R.id.request_btn);

        sendRequestButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                String[] urls = new String[] {
                        "http://www.google.com",
                        "http://yahoo.com/",
                        "http://www.bing.com"
                };
                new SearchEngineTast().execute(urls);
            }
        });
    }


    private class SearchEngineTast extends AsyncTask<String, Void, Void> {

        @Override
        protected Void doInBackground(String... strings) {
            if (strings.length < 3) {
                return  null;
            }

            RequestQueue queue = Volley.newRequestQueue(VolleyTutorial.this);

            final String googleUrl = strings[0];
            final String yahooUrl = strings[1];
            final String bingUrl = strings[2];

            RequestFuture<String> future = RequestFuture.newFuture();
            Response.ErrorListener errorListener = new Response.ErrorListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    error.printStackTrace();
                }
            };

            StringRequest googleRequest = new StringRequest(Request.Method.GET, googleUrl, future, errorListener);
            StringRequest yahooRequest = new StringRequest(Request.Method.GET, yahooUrl, future, errorListener);
            StringRequest bingRequest = new StringRequest(Request.Method.GET, bingUrl, future, errorListener);

            try {

                queue.add(googleRequest);
                Log.d(TAG_REQUEST, "addedd google request");
                String googleResponse = future.get(5, TimeUnit.SECONDS);
                Log.d(TAG_RESPONSE, "google response: " + googleResponse);

                queue.add(yahooRequest);
                Log.d(TAG_REQUEST, "addedd yahoo request");
                String yahooResponse = future.get(5, TimeUnit.SECONDS);
                Log.d(TAG_RESPONSE, "yahoo response: " + yahooResponse);

                queue.add(bingRequest);
                Log.d(TAG_REQUEST, "addedd bing request");
                String bingResponse = future.get(5, TimeUnit.SECONDS);
                Log.d(TAG_RESPONSE, "bing response: " + bingResponse);

            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                e.printStackTrace();
            }

            return null;
        }
    }

}
{{< / highlight >}}

Questa volta tutto il codice relativo alle richieste http è stato messo all'interno di un *AsyncTask*, questo perchè il metodo *get* della classe *RequestFuture* è bloccante e non vogliamo bloccare il main thread.
Inoltre al metodo get passiamo anche un intervallo di tempo limite oltre il quale non siamo disposti ad aspettare. 

### Creare la propria RequestQueue

Abbiamo già visto come creare una *RequestQueue* servendoci del metodo *newRequestQueue*. Secondo la documentazione android, la nostra *RequestQueue* dovrebbe essere un *Singleton*, ovvero non dovrebbero esserci più di una istanza all'interno della nostra applicazione. Per ottenere ciò possiamo servirci di una classe che incapsula la nostra *RequestQueue*. Inoltre per istanziare la nostra *RequestQueue* ci serviremo dell'*application context* invece che del *context* relativo ad una singola *activity*, questo per evitare che la coda venga ricreata ogni volta che viene ricreata l'*activity*.

Ecco un esempio: 
{{< highlight java >}} 
public class RequestSingleton {
    private static RequestSingleton mInstance;
    private RequestQueue mRequestQueue;
    private static Context mCtx;

    private RequestSingleton(Context context) {
        mCtx = context;
        mRequestQueue = getRequestQueue();
    }

    public static synchronized RequestSingleton getInstance(Context context) {
        if (mInstance == null) {
            mInstance = new RequestSingleton(context);
        }
        return mInstance;
    }

    public RequestQueue getRequestQueue() {
        if (mRequestQueue == null) {
            // getApplicationContext() is key, it keeps you from leaking the
            // Activity or BroadcastReceiver if someone passes one in.
            mRequestQueue = Volley.newRequestQueue(mCtx.getApplicationContext());
        }
        return mRequestQueue;
    }

    public <T> void addToRequestQueue(Request<T> req) {
        getRequestQueue().add(req);
    }

}
{{< / highlight >}}

Per aggiungere una *Request* alla nostra coda, ci basterà dunque chiamare l'istanza della coda e chiamare il metodo *addToRequestQueue()*

{{< highlight java >}}
RequestSingleton.getInstance(this).addToRequestQueue(request);
{{< / highlight >}}
