---
author: Umberto D'Ovidio
date: "2017-08-03T00:00:00Z"
description: Come creare un'app che permette di legger i QR code in Android?
feature-img: assets/qrcode_android_featured.png
feature-img-lg: assets/qrcode_android_featured_lg.png
feature-img-md: assets/qrcode_android_featured_md.png
feature-img-sm: assets/qrcode_android_featured_sm.png
feature-img-xs: assets/qrcode_android_featured_xs.png
image: /images/qrcode.png
title: Leggere i QR Code in Android
---

Come si integra la funzionalità di lettura dei QR code in un'app Android? Scopriamolo insieme!
<!--more-->

Ho preparato un semplice tutorial che fornisce un ottimo esempio di come affrontare il problema. La nostra app sarà in grado di leggere i QR code e di aprire un browser se  l'informazione codificata nel QR code è un url valido. Puoi trovare il codice sorgente su [Github](https://github.com/Cyborg101/dovidioTutorials/tree/master/QrCodeReader-android).

La funzionalità di lettura del QR code non è integrata nella sdk android, ma bisogna, se non vogliamo fare tutto da soli, importare una libreria. 
La libreria più famosa e storica è senza dubbio [Zxing](https://github.com/zxing/zxing) (Zebra crossing). Noi ci serviremo di un'altra [libreria](https://github.com/dm77/barcodescanner) basata su zxing che rende tutto più semplice. 

Iniziamo dunque a darci dentro col codice! Per prima cosa, apriamo Android studio e aggiungiamo la libreria nel file  build.gradle a livello app.

{{< highlight gradle >}}
compile 'me.dm7.barcodescanner:zxing:1.9.4'
{{< / highlight >}}

Nota che non consiglio di usare l'ultima versione (che al momento è la 1.9.5) perchè sembra non funzionare in portrait mode, almeno non nel mio galaxy s5 mini.


Dobbiamo poi chiedere l'autorizzazione per accedere alla fotocamera. Per farlo aggiungiamo la seguente riga al file **Android Manifest.xml**.

{{< highlight xml >}}
<uses-permission android:name="android.permission.CAMERA" />
{{< / highlight >}}

A partire da Android 6.0 (Marshmallow) le autorizzazioni vengono richieste a runtime, le richiederemo dunque nella nostra Activity Java.

Creiamo dunque una empty Activity chiamata QrCodeReader, e aggiungiamoci il seguente codice:

{{< highlight gradle >}}
package io.dovid.qrcodereader;

import android.Manifest;
import android.app.Activity;
import android.app.AlertDialog;
import android.content.DialogInterface;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Bundle;
import android.support.v4.app.ActivityCompat;
import android.support.v4.content.ContextCompat;
import android.webkit.URLUtil;
import com.google.zxing.Result;


import me.dm7.barcodescanner.zxing.ZXingScannerView;


public class QrCodeReader extends Activity implements ZXingScannerView.ResultHandler {
    private ZXingScannerView mScannerView;
    private static final int PERMESSO_FOTOCAMERA = 4725;

    @Override
    public void onCreate(Bundle state) {
        super.onCreate(state);
        // Richiediamo i permessi a runtime (per tutti i dispositivi >= Android 6.0)
        if (ContextCompat.checkSelfPermission(QrCodeReader.this, Manifest.permission.CAMERA)
                != PackageManager.PERMISSION_GRANTED) {
            // richiediamo il permesso di utilizzare la fotocamera
            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.CAMERA},
                    1);
        }

        mScannerView = new ZXingScannerView(this);   // Inizializziamo la nostra scanner view
        setContentView(mScannerView); // settiamo la nostra scannerview come content view
    }

    @Override
    public void onResume() {
        super.onResume();
        mScannerView.setResultHandler(this);
        mScannerView.startCamera();
    }

    @Override
    public void onPause() {
        super.onPause();
        mScannerView.stopCamera();
    }

    @Override
    public void handleResult(Result rawResult) {
        String url = rawResult.getText();
        System.out.println(url);
        if (!URLUtil.isValidUrl(url)) {
            AlertDialog dialog = new AlertDialog.Builder(this).
                    setTitle("Attenzione").
                    setMessage("Il qr code non contiene un url!").
                    setPositiveButton("Qr code", new DialogInterface.OnClickListener() {
                        @Override
                        public void onClick(DialogInterface dialogInterface, int i) {
                            dialogInterface.cancel();
                        }
                    }).
                    show();
        } else {
            openBrowser(url);
        }

        mScannerView.resumeCameraPreview(this);
    }

    private void openBrowser(String url) {
        Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
        startActivity(intent);
    }
}
{{< / highlight >}}


Qualche commento sul codice. Prima di tutto non utilizziamo un file xml come layout in quanto usiamo la nostra ScannerView a tutto schermo. Una volta che abbiamo creato la nostra ScannerView e settato la nostra ContentView, abbiamo un QR code reader pronto per essere utilizzato! Non solo, legge anche numerosi formati di codice a barre! Infatti è possibile specificare quali formati ci interessano attraverso il metodo setFormats() chiamato sull'oggetto ScannerView. 
Nota che abbiamo inoltre incluso il codice che chiede a runtime il permesso di accedere alla fotocamera.

Una volta che lo scanner ha effettuato la lettura, chiama il metodo handleResult dove ci assicuriamo che il contenuto del QR code è un url che visualizziamo a browser. Se così non fosse avvertiamo l'utente con un alert dialog. 