---
author: Umberto D'Ovidio
date: "2018-09-24T00:00:00Z"
description: Come testare i tuoi componenti Android con JUnit
title: Testing su Android Parte 1 - JUnit e Unit Testing
visible: 1
---

Stanco di provare le tue app Android manualmente per verificarne il funzionamento? Inizia a scrivere test!
<!--more-->

Android offre svariate possibilità per testare la propria app. Si va dallo [Unit Testing](), dove ogni componente della tua app viene testato minuziosamente, al testing della UI e all'automatizzazione di quest'ultimo. L'argomento è molto vasto per cui per essere esaustivo ho deciso di dividerlo in differenti episodi.

I test in android sono basati su [JUnit](http://junit.org/junit4/), e possono essere effettuati sia localmente che nel dispositivo Android. 
Nel primo caso i nostri test si trovano in *nome-modulo/src/test/java* e vengono effettuati in una *JVM* locale. Questo tipo di test sono ideali quando vogliamo testare metodi e classi che non dipendono dal framework Android. 

Nel secondo caso invece i test si trovano in *nome-modulo/src/androidTest/java* e vengono effettuati nell'emulatore o nel tuo dispositivo Android all'interno di un *APK* che viene eseguita nello stesso processo della tua applicazione, in modo tale da poter richiamare i metodi della tua app principale e automatizzare l'interazione con l'interfaccia grafica. 
Possiamo poi differenziare i test in base all'area di attività che ricoprono. In tal caso abbiamo i **test unitari (unit test)** che si occupano di collaudare classi e funzioni, **integration test**, atti a verificare l'azione congiunta di più unità e i **test funzionali** che simulano le possibili interazioni dell'utente con la nostra app. 

### Scrivere Test Cases JUnit

Nella terminologia JUnit, "test case" è una classe Java che contiene un insieme di test da eseguire. Ogni classe Java in Android può essere utilizzata come test case, a condizione che abbia un costruttore pubblico senza parametri e benga decorata con l'annotazione *@RunWith(AndroidJUnit4.class)*.
Esempio:

{{< highlight java >}}
@RunWith(AndroidJUnit4.class) public class TestJUnit {
  // codice di test
}
{{< / highlight >}}

All'interno della nostra classe avremo dunque i test veri e propri, annotati con *@Test*

{{< highlight java >}}
@RunWith(AndroidJUnit4.class) public class TestJUnit {
  
  @Test
  public void testVeroEProprio() {
      Assert.assertEquals("la matematica non è un'opinione", 2 + 2, 4);
  }
}
{{< / highlight >}}

Tramite la classe *Assert* di JUnit possiamo verificare che il risultato delle nostre unità sia uguale a quello aspettato. Per farlo basta chiamare la funzione *assertEquals* dandogli come parametri il risultato della nostra unità e il risultato che ci aspettiamo. Ci sono moltissime altre funzioni che possiamo utilizzare per controllare il risultato delle nostre unità di computazione, per tanto ti invito a dare un'occhiata alla [documentazione ufficiale](https://developer.android.com/reference/junit/framework/Assert.html). 

### Preparare e ripulire i nostri test

Il test case JUnit può anche contenere dei metodi che permettono di preparare tutte le risorse di cui abbiamo bisogno per i nostri test e di chiudere le risorse che abbiamo creato per il nostro unit test. Ciò è particolarmente utile quando ad esempio facciamo dei test che richiedano l'utilizzo di un database. 
Questi metodi vengono annotati rispettivamente con *@Before* e *@After*. Il metodo *@Before* viene chiamato prima di ogni test e il metodo *@After* viene chiamato alla fine di ogni test.
E' possibile usare anche le funzioni *@BeforeClass* e *@AfterClass* che vengono chiamate solo una volta rispettivamente prima e dopo aver effettuato tutti i test. 

### Testare componenti Android

JUnit offre il concetto di "rules" che permettono di testare alcuni scenari particolari. Ad esempio possiamo far si che nei nostri test non sia necessario richiedere permessi runtime in Android Marshmallow o superiori utilizzando la [GrantPermissionRule](https://developer.android.com/reference/android/support/test/rule/GrantPermissionRule.html). Questo permette di creare dei test automatici dell'interfaccia grafica evitando che il dialogo di richiesta permessi la blocchi. 
Un'altra *Rule* molto utile è l'[ActivityTestRule](), che lancia la classe che forniamo come argomento prima che vengano eseguiti i test e la termina in seguito. Ecco un semplice esempio: 

{{< highlight java >}}
@RunWith(AndroidJUnit4.class) public class TestJUnit {
  
  @Rule  
  public ActivityTestRule mActivityRule = new ActivityTestRule<>(
            MainActivity.class);

  @Before
  public void init() {
      TextView tv = (TextView) findViewById(R.id.tv);
  }
  
  @Test
  public void testaTextView() {
      Assert.assertEquals("", tv.getText(), "Ciao");
  }
}
{{< / highlight >}}

### Configurazione Gradle

Per eseguire i nostri unit test abbiamo bisogno anche di aggiungere le opportune dipendenze a Gradle. Per aggiungere dipendenze che vengono utilizzate in fase di test, specifichiamo una *androidTestCompile*. Ad esempio avremo:

{{< highlight xml >}}
  dependencies {
    androidTestCompile 'com.android.support.test:runner:0.5'
    androidTestCompile 'com.android.support.test:rules:0.5'
  }
{{< / highlight >}}

Inoltre dobbiamo specificare un *testApplicationId* per far si che i nostri test non vadano ad intaccare le altre build nel nostro emulatore o nel nostro dispositivo. Come convenzione si usa come nome il nome del package di default e si aggiunge *.test* alla fine. 

### Esempio

Ho preparato una [piccola app](https://github.com/Cyborg101/dovidioTutorials/tree/master/BasicUnitTesting) che mostra un esempio di unit testing. L'app chiede un numero e ne calcola il fattoriale. Ho creato una classe nel package *test* che ho chiamato *EsempioUnitTest* che è il nostro test case e contiene alcuni metodi di test. 
<img src="{{ site.url }}/assets/unit_test_screenshot.png" alt="Rappresentazione dello stack" style="margin: 2rem auto;"/>

Per eseguire tutti i metodi è sufficiente premere il pulsante verde vicino e chedere che la classe venga eseguita. Verranno dunque eseguiti tutti gli unit test e vedrai una barra che diventa verde.