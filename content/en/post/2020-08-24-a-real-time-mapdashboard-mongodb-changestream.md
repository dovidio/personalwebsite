---
title: "A real-time world dashboard using Mongodb Change Streams"
date: 2020-08-24T16:00:00+02:00
draft: false
tags: ["mongodb", "golang", "angular", "mapbox"]
---

Since version 3.6, MongoDB offers [Change Streams](https://www.mongodb.com/blog/post/an-introduction-to-change-streams), a new feature that allows user to subscribe to all data changes in a single collection, a database, or an entire deployment.

To explore this feature, I have written a [small full-stack application](https://github.com/dovidio/realtimedashboard) that displays (fake) app downloads on a world map in real-time.
The technologies used are the following:
- MongoDB for the persistence using the change streams feature for real-time updates
- Go backend server, with pretty much no dependencies besides the Mongodb driver
- An Angular frontend application using Mapbox GL JS for the map visualization
The backend server pushes changes to the frontend application using a web socket using [go WebSockets](https://godoc.org/golang.org/x/net/websocket)
In the following sections, I will describe in detail each part of the application.

## MongoDb Change Streams configuration
In order to use Change Streams in Mongodb, you need at least version 3.6 and use either replica sets or sharded clusters. More info can be found [here](https://docs.mongodb.com/manual/changeStreams/). For my application I have decided to use the replica sets functionality. To do so, once I have spinned up the Mongodb image, I execute the following command:
```bash
docker exec mongodb mongo --eval "rs.initiate({_id : 'rs0', members: [{ _id : 0, host : \"mongodb:27017\" }]});rs.slaveOk(); db.getMongo().setReadPref('nearest');db.getMongo().setSlaveOk();"
```

## The server

As stated before, the server is written in Go and has minimal dependencies. Besides serving the *AppDownload* resource, it also generates fake data for testing purposes. The most important and interesting parts of the server are the database watcher and the WebSocket handler.

### The Database watcher

The most interesting part of the app is `watcher.go`. Here I have created an interface called `DatabaseWatchHandler` that contains three methods: register observer, unregister observer, and watch app downloads.

```golang
type DatabaseWatchHandler interface {
	RegisterObserver(Observer) uuid.UUID
	UnregisterObserver(u uuid.UUID)
	WatchAppDownloads()
}
```

The `WatchAppDownloads()` takes care of subscribing to a **Change Stream** on the **AppDownload** collection.
Every time it gets notified with new data, it will push this data to the list of observers, that is handled
using the `RegisterObserver()` and `UnregisterObserver()` method.

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
The implementation of the `WatchAppDownloads()` looks quite straightforward. It calls the method `WatchAppDownloadsImpl()`, where the real magic resided, and if an error arise it sleeps for a second, log the error message and reconnect again.
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

This is where we are using the change stream features of mongo.
This can be used by calling the `Watch()` method on a collection. In our case, we are interested in the appdownloads collection. The [Watch()](https://godoc.org/go.mongodb.org/mongo-driver/mongo#Client.Watch) method requires a context, a [mongodb pipeline](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/), and some [ChangeStreamOptions](https://godoc.org/go.mongodb.org/mongo-driver/mongo/options#ChangeStreamOptions). Our pipeline uses a [$project](https://docs.mongodb.com/manual/reference/operator/aggregation/project/#pipe._S_project) operator, where we specify some of the fields we want to remove from the payload. When it comes to the change stream options, we want to use the `UpdateLookup`. This means that we will get, in addition to the delta of changes, the full document copy. Once we have the stream cursor, we can iterate through the changes using the `Next()` method. Inside this loop, I'm converting the document payload in the app-specific dto and sending this update to all the observers.

### The WebSocket handler
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
The WebSocket handler is pretty minimal. The `Websocket` method is called every time a client tries to open
a WebSocket connection with the server. In this method, we take care of registering a new observer to the databaseWatcher so that we are informed every time we have new changes in the appdownloads collection.

```golang
func (w *websocketObserver) OnNewAppDownload(a AppDownload) {
	if err := websocket.JSON.Send(w.ws, a); err != nil {
		log.Printf("Error while trying to send update to websocket: %v", err)
	}
}
```

The observer implements `OnNewAppDownload`, which is simply pushing new data to the client using the WebSocket.JSON.Send method.

### Putting everything together
All the wiring logic is done in `main.go`.
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
As you can see there is not too much magic into it. We are just instantiating the watcher, the WebSocket handler, and another rest handler. Additionally, we might start an app download generator for testing purposes.

## The client
The client follows a pretty simple logic. When loading the page it initially fetches all app downloads. Additionally, it subscribes to the appdownloads socket to get notified about new app downloads.

### StatisticsService
This service exposes 4 different observables:
- **appDownloads**: holds a json containing data in GeoJson format, which is used by the mabox API
- **ByCountry**: app downloads aggregated by country
- **ByTimeOfDay**: app downloads aggregated by time of day (morning, afternoon, evening, night)
- **ByApp**: app downloads aggregated by app name
The interesting thing about this service is that it abstracts away the WebSocket and HTTP logic from other components. Is simply exposes and observables of app downloads and the component using the service doesn't need to know if it got those from a REST API or a WebSocket.

### Mapbox specific code
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

This method is called in `ngOnInit()`. We are initializing a Mapbox map, passing a specific style, and a token (those are defined elsewhere). After adding map controls (for navigation and zooming), we add some configuration code when the map is loaded. First, we create a sort of data source of type [GeoJSON](https://en.wikipedia.org/wiki/GeoJSON). This is the most efficient way for adding markers. Then we subscribe to our `StatisticService`. Every time **$appDownloads** emits, we set this recent data to the source.
Now we define 3 layers:
- A first layer, with id 'appdownloads', defines the circles' color and radius.
- A second layer, with id 'appdownloads-count', defines the font to be rendered inside the circles
- The last layer, with id 'unclustered-point', defines the color of the circle for single points in the map

### Running the app
The app is packaged into docker containers and can be run with the `run.sh` script, which uses docker-compose to spin up the different components used by the application

## Conclusions and possible improvements
These days we expect our applications to have real-time updates. In this post, I have explored how we could create a simple real-time application using Mongodb, Go and Angular. We have seen how mongo offers us change streams and how to connect those to web sockets. Other databases and offerings make it easier to create real-time applications (e.g. [Firebase](https://firebase.google.com/)), but oftentimes they lock you on one platform and make it hard to migrate later on. Note that this is in no way production-ready code, but it can serve as a good starting point to think about the challenges and the design of real-time applications.
Here are some ideas for expanding the application in the future:
- Make it pretty, add more charting capabilities :)
- Add a post endpoint where user can add new app downloads
- Improve the quality of the simulated data
- Decouple the database watcher from the WebSocket handler by using an event bus or some reactive library
- Make the app cloud-native and ready to be run in Kubernetes