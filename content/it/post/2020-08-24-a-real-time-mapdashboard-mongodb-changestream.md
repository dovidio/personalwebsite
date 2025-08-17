---
title: "Una dashboard che si aggiorna in tempo reale con i Mongodb Change Streams"
date: 2020-08-24T16:00:00+02:00
draft: false
tags: ["mongodb", "golang", "angular", "mapbox"]
---

Mongodb offre, dalla versione 3.6, i [Change Streams](https://www.mongodb.com/blog/post/an-introduction-to-change-streams), una nuova funzionalita' che consente di iscriversi ad una collezione singola, un database o un deployment, e ricevere nuovi dati in real-time.

Ho creato una [piccola app full-stack](https://github.com/dovidio/realtimedashboard) per esplorare questa funzionalita. L'app mostra (finti) downloads di ipotetiche app mobile in una mappa del mondo in tempo reale.
Le tecnologie che ho utilizzato sono le seguenti:
- MongoDB per la persistenza con i change streams per la parte in tempo reale
- Backend server in Go, con il mongo-driver come sola dipendenza
- App frontend in Angular e Mapbox GL JS per la visualizzazione della mappa
Il server backend manda i cambiamenti all'app frontend utilizzando una WebSocket e la libreria [go WebSockets](https://godoc.org/golang.org/x/net/websocket).
Nelle prossime sezioni, descrivo in dettaglio ciascuna parte dell'applicazione.

## Configurazione dei MongoDb Change Streams
I Change Streams in Mongodb sono disponibili con la versione 3.6 e solo utilizzando i replica sets o i clusters sharded. Per maggiori informazione vai [qui](https://docs.mongodb.com/manual/changeStreams/). Per la mia applicazione ho deciso di utilizzare la funzionalita' replica sets. Per far cio', ho fatto partire l'immagine docker Mongodb, ed ho eseguito il seguente comando:
```bash
docker exec mongodb mongo --eval "rs.initiate({_id : 'rs0', members: [{ _id : 0, host : \"mongodb:27017\" }]});rs.slaveOk(); db.getMongo().setReadPref('nearest');db.getMongo().setSlaveOk();"
```

## Il server
Come accennato in precedenza, il server e' scritto in Go e ha pochissime dipendenze. Oltre a servire la risorsa **AppDownload**, si occupa anche di generare dati random per il testing. Le parti piu' importanti ed interessanti del server sono il database watcher e il WebSocket handler.

### Il Database watcher
La parte piu' interessante dell'applicazione e' contenuta nel file `watcher.go`. Il file contiene un'interfaccia chiamata `DatabaseWatchHandler` che contiente tre metodi: register observer, unregister observer, and watch app downloads.

```golang
type DatabaseWatchHandler interface {
	RegisterObserver(Observer) uuid.UUID
	UnregisterObserver(u uuid.UUID)
	WatchAppDownloads()
}
```

`WatchAppDownloads()` si occupa di isriversi ai **Change Stream** per la collezione di **AppDownload**.
Ogni volta che la collezione viene modificata, la modifica viene mandata a tutti gli observer, che vengono gestiti con i metodi `RegisterObserver()` e `UnregisterObserver()`.

``` golang
func (m *MongoDbWatchHandler) WatchAppDownloads() {
	for {
		if err := m.WatchAppDownloadsImpl(); err != nil {
			fmt.Println(err)
			time.Sleep(1 * time.Second)
		}
	}
}
```
L'implementazione di `WatchAppDownloads()` e' abbastanza immediata. Chiama il metodo `WatchAppDownloadsImpl()`, dove avviene la vera magia, e se trova un errore pausa per un secondo prima di riconnettersi.

``` golang
func (m *MongoDbWatchHandler) WatchAppDownloadsImpl() error {
	collection := m.db.Collection("appdownloads")
	var pipeline = mongo.Pipeline{
		{{"$project", bson.D{{"operationType", 0}, {"ns", 0}, {"documentKey", 0}, {"clusterTime", 0}}}},
	}
	ctx := context.Background()
	streamCur, err := collection.Watch(ctx, pipeline, options.ChangeStream().SetFullDocument(options.UpdateLookup))
	if err != nil {
		return fmt.Errorf("Error getting streaming cursor: %v", err)
	}

	for streamCur.Next(ctx) {
		var result bson.M
		streamCur.Decode(&result)

		fullDocument, found := result["fullDocument"]
		if !found {
			return fmt.Errorf("Cannnot find full document: %v", err)
		}

		appDownload, err := extractAppDownload(fullDocument)
		if err != nil {
			fmt.Print(err)
		}

		m.mut.Lock()
		for _, observer := range m.observers {
			observer.OnNewAppDownload(appDownload)
		}
		m.mut.Unlock()
	}

	return errors.New("streaming cursor finished")
}
```

Qua e' dove utilizziamo la funzionalita' change stream di mongo.
Utilizzando il metodo `Watch()` su una collezione, ci si iscrive a tutti i cambiamenti su di essa. Nel nostro caso, siamo interessati alla collezione appdownloads. Il metodo [Watch()](https://godoc.org/go.mongodb.org/mongo-driver/mongo#Client.Watch) ha bisogno di uncontext, una [mongodb pipeline](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/), e alcune [ChangeStreamOptions](https://godoc.org/go.mongodb.org/mongo-driver/mongo/options#ChangeStreamOptions). La nostra pipeline utilizza l'operatore [$project](https://docs.mongodb.com/manual/reference/operator/aggregation/project/#pipe._S_project), specificando alcuni field che vogliamo esclusi dal payload. Per quanto riguarda le change stream options, vogliamo usare `UpdateLookup`. Questo significa che, oltre a ricevere il delta dei cambiamenti, riceviamo l'intero documento. Una volta che abbiamo uno stream cursor, possiamo iterare su di esso utilizzando il metodo `Next()`. All'interno del for loop, il payload viene convertito in un dto che viene poi mandato a tutti gli observers.

### Il WebSocket handler
```golang
func (w *WebsocketHandler) Websocket(ws *websocket.Conn) {
	myID := w.databaseWatcher.RegisterObserver(&websocketObserver{ws: ws})
	for {
		var msg message
		if err := websocket.JSON.Receive(ws, &msg); err != nil {
			log.Printf("Error while reading message: %v", err)
			break
		}

		log.Printf("received message %s\n", msg.Data)
	}
	w.databaseWatcher.UnregisterObserver(myID)
}
```
Il WebSocket handler e' molto semplice. Il metodo `Websocket` viene chiamato ogni volta che i client aprono una nuova connessione WebSocket con il server. In questo metodo, ci occupiamo di registrare un nuovo observer nel databaseWatcher in modo da venire avvisati ad ogni cambiamento nella collezione appdownloads.

```golang
func (w *websocketObserver) OnNewAppDownload(a AppDownload) {
	if err := websocket.JSON.Send(w.ws, a); err != nil {
		log.Printf("Error while trying to send update to websocket: %v", err)
	}
}
```

L'observer implementa il metodo `OnNewAppDownload`, che si occupa semplicemente di mandare il dto ai client con il metodo `WebSocket.JSON.Send`.

### Putting everything together
Tutta la logica di configurazione risiede in `main.go`.
```golang
func main() {
	db := db.NewClient().Database("appdownloads")
	repository := appdownload.NewMongoRepository(db)
	dbWatcher := appdownload.NewMongoWatchHandler(db, make(map[uuid.UUID]appdownload.Observer, 0))

	handler := appdownload.NewHandler(repository)
	wsHandler := appdownload.NewWebsocketHandler(dbWatcher)

	http.Handle("/appdownloads", cors.Middleware(http.HandlerFunc(handler.Handle)))
	http.Handle("/appdownloadssocket", websocket.Handler(wsHandler.Websocket))

	go dbWatcher.WatchAppDownloads()

	if period := shouldGenerateData(); period > 0 {
		quit := make(chan struct{})
		go appdownload.GenerateData(time.Duration(period)*time.Millisecond, repository, quit)
	}

	err := http.ListenAndServe("0.0.0.0:8080", nil)
	if err != nil {
		log.Fatal(err)
	}
}
```
Come puoi vedere non c'e' molto. Ci limitiamo ad istanziare un watcher, un WebSocket handler e un altro handler per il REST. In aggiunta, facciamo partire un generatore di dati a seconda di alcune variabili ambientali.

## Il client
Il client implementa una logica piuttosto semplice. Quando l'applicazione viene caricata, scarica tutti gli app downloads. In aggiunta, si iscrive alla socket appdownloads per venire notificato di ogni seguente cambiamento.

### StatisticsService
Il servizio espone 4 observables differenti.
- **appDownloads**: contiene i dati in formato GeoJSON, utilizzato dalla libreria Mapbox
- **ByCountry**: gli app downloads aggregati per paese
- **ByTimeOfDay**: gli app downloads aggregati per periodo del giorno (mattina, pomeriggio, sera o notte)
- **ByApp**: gli app downloads aggregati per nome dell'app
La cosa interessante di questo servizio e' che encapsula tutta la logica riguardante richieste http e connessioni WebSocket. I componenti che utilizzano questo servizio si limitano a iscriversi agli observables offerti, senza sapere come i dati sono ottenuti.

### Codice Mapbox-specifico
```typescript
initializeMap(): void {
    mapboxgl.accessToken = environment.mapbox.accessToken;
    this.map = new mapboxgl.Map({
      container: 'map',
      style: this.style,
      zoom: 4,
      center: [this.lng, this.lat]
    });
    // Add map controls
    this.map.addControl(new mapboxgl.NavigationControl());

    this.map.on('load', (event) => {

      this.map.addSource('appdownloads', {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: []
        },
        cluster: true,
        clusterMaxZoom: 14, // Max zoom to cluster points on
        clusterRadius: 50 // Radius of each cluster when clustering points (defaults to 50)
      });

      this.source = this.map.getSource('appdownloads');

      this.sub = this.statsService.appDownloads$.subscribe((apps) => {
        let data = new FeatureCollection(apps);
        this.source.setData(data);
      });

      this.map.addLayer({
        id: 'appdownloads',
        source: 'appdownloads',
        type: 'circle',
        filter: ['has', 'point_count'],
        paint: {
          'circle-color': [
            'step',
            ['get', 'point_count'],
            '#51bbd6',
            100,
            '#f1f075',
            750
          ],
          'circle-radius': [
            'step',
            ['get', 'point_count'],
            20,
            100,
            30,
            750,
            40
          ]
        }
      });

      this.map.addLayer({
        id: 'appdownloads-count',
        type: 'symbol',
        source: 'appdownloads',
        filter: ['has', 'point_count'],
        layout: {
          'text-field': '{point_count_abbreviated}',
          'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
          'text-size': 12
        }
      });

      this.map.addLayer({
        id: 'unclustered-point',
        type: 'circle',
        source: 'appdownloads',
        filter: ['!', ['has', 'point_count']],
        paint: {
          'circle-color': '#11b4da',
          'circle-radius': 4,
          'circle-stroke-width': 1,
          'circle-stroke-color': '#fff'
        }
      });

    })
  }
```

Questo metodo viene chiamato da all'interno di  `ngOnInit()`. Inizializziamo una mappa Mapbox, passando un token e uno style (definiti altrove). Dopo aver aggiunto i controlli (per navigaizone e zoom), aggiungiamo del codice di configurazione una volta che la mappa viene caricata. Per prima cosa, creiamo una data source del tipo [GeoJSON](https://en.wikipedia.org/wiki/GeoJSON). Questo e' il modo piu' efficiente per aggiungere markers alla mappa. Ci iscriviamo poi a appDownload$ di `StatisticService`. Ogni volta che **$appDownloads** emette, chiamiamo `dataSource.setData` con i dati piu' recenti.
Definiamo poi 3 layers:
- Un primo layer, con id 'appdownloads', definisce i colori e la grandezza dei nostri cerchi.
- Un secondo layer, con id 'appdownloads-count', definisce il font per il conteggio degli app download visualizzato all'interno dei cerchi
- L'ultimo layer, con id 'unclustered-point', definisce il colore dei cerchi per i punti singoli

### Eseguire l'app
L'app puo' essere eseguita con il comando `run.sh`, che utilizza docker-compose per inizializzare i vari container che costituiscono l'applicazione

## Conclusioni e sviluppi futuri
Al giorno d'oggi ci aspettiamo che le nostre app vengano aggiornate in tempo reale.
In questo post, ho esplorato un possibile stack che supporta questo caso d'uso, utilizzando Mongodb, Go e Angular. Abbiamo visto come utilizzare la funzionalita; di change steam di mongo e come aggiornare i client con le WebSocket. Altri database e prodotti facilitano questo tipo di caso d'uso (e.g. [Firebase](https://firebase.google.com/)), ma spesso offrono meno flessibilita' e ti costringono nella loro piattaforma. Da notare che il codice presentato non e' da intendersi come pronto per entrare in produzione ma piu' come uno spunto iniziale per ragionare sul problema.
Alcune idee per sviluppi futuri:
- Rendere l'app piu' appetibile, magari aggiungendo grafici :)
- Aggiungere un endpoint dove si possono creare nuovi app download
- Migliorare la qualita' dei dati simulati
- Disaccoppiare il database watcher dal WebSocket handler utiizzando un event bus o una libreria che supporta il reactive programming
- Rendere l'app cloud-native e farla girare su Kubernetes