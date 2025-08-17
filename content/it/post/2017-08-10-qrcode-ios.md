---
author: Umberto D'Ovidio
date: "2017-08-10T00:00:00Z"
description: Come creare un'app che permette di legger i QR code in iOS usando Swift?
title: Leggere i QR Code con Swift
---
La scorsa settimana ho scritto un breve [tutorial](http://dovid.io/2017/08/03/qrcode-android.html) su come includere la funzionalità di lettura dei qr code nella tua app Android. Per par condicio, oggi vedremo come farlo in iOS. Sfrutteremo la libreria che ci viene messa a disposizione da iOS chiamata [AVFoundation](https://developer.apple.com/av-foundation/).
<!--more-->


La nostra app sarà in grado di leggere un QR code e aprirà un web browser nel caso in cui l'informazione letta dal QR code sia un url valido.

Ci serviremo di due classi: una per la lettura del QR code e una per aprire il web browser.

### Lettura QR Code

{{< highlight swift >}}
import AVFoundation
class ViewController: UIViewController, AVCaptureMetadataOutputObjectsDelegate {
    var captureSession:AVCaptureSession?
    var videoPreviewLayer:AVCaptureVideoPreviewLayer?
    var qrCodeFrameView:UIView?
{{< / highlight >}}

Per prima cosa, la nostra classe deve implementare il protocollo **AVCaptureMetadataOutputObjectsDelegate**, che si trova nella libreria **AVFoundation**. 

{{< highlight swift >}}
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let captureDevice = AVCaptureDevice.defaultDevice(withMediaType: AVMediaTypeVideo)
        
        do {
            let input = try AVCaptureDeviceInput(device: captureDevice)
            captureSession = AVCaptureSession()
            captureSession?.addInput(input)
            
            let captureMetadataOutput = AVCaptureMetadataOutput()
            captureSession?.addOutput(captureMetadataOutput)
            
            captureMetadataOutput.setMetadataObjectsDelegate(self, queue: DispatchQueue.main)
            
            captureMetadataOutput.metadataObjectTypes = [AVMetadataObjectTypeQRCode]
            
            videoPreviewLayer = AVCaptureVideoPreviewLayer(session: captureSession)
            videoPreviewLayer?.videoGravity = AVLayerVideoGravityResizeAspectFill
            videoPreviewLayer?.frame = view.layer.bounds
            view.layer.addSublayer(videoPreviewLayer!)
            
            captureSession?.startRunning()
            
            // oggetto che utilizziamo per evidenziare il QR code
            qrCodeFrameView = UIView()
            
            if let qrCodeFrameView = qrCodeFrameView {
                qrCodeFrameView.layer.borderColor = UIColor.green.cgColor
                qrCodeFrameView.layer.borderWidth = 2
                view.addSubview(qrCodeFrameView)
                view.bringSubview(toFront: qrCodeFrameView)
            }
            
        } catch {
            print(error)
            return
        }
        
    }
{{< / highlight >}}


Attenzione a non dimenticarsi di importare la libreria! Per leggere i QR Code ci serviremo di cinque oggetti: **AVCaptureDevice**, **AVCaptureDeviceInput**, **AVCaptureSession**, **AVCaptureMetadataOutput**, **AVCapturePreviewLayer**.
Il primo rappresenta il nostro dispositivo fisico e le proprietà associate ad esso. Il nostro dispositivo genera input che viene fornito all'oggetto **AVCaptureSession**. La nostra sessione di cattura poi viene fornita all'oggetto **AVCaptureMetadataOutput** al quale forniamo i tipi di metadati che stiamo cercando (ovvero il QR code) e inoltre settiamo la nostra classe come **Delegate** di questo oggetto. 
Dobbiamo inoltre mostrare il contenuto della sessione, ovvero la cattura della fotocamera utilizzando l'oggetto **AVCaptureVideoPreviewLayer**.

{{< highlight swift >}}
    func captureOutput(_ captureOutput: AVCaptureOutput!, didOutputMetadataObjects metadataObjects: [Any]!, from connection: AVCaptureConnection!) {
        
        if metadataObjects == nil || metadataObjects.count == 0 {
            qrCodeFrameView?.frame = CGRect.zero
            return
        }
        
        let metadataObj = metadataObjects[0] as! AVMetadataMachineReadableCodeObject
        if metadataObj.type == AVMetadataObjectTypeQRCode {
            
            let qrCodeObject = videoPreviewLayer?.transformedMetadataObject(for: metadataObj)
            qrCodeFrameView?.frame = qrCodeObject!.bounds
            
            if let urlString  = metadataObj.stringValue {
                // se è un url valido allora vai alla prossima classe
                if isValidUrl(urlString: urlString) {
                    let webController = WebViewController()
                    webController.urlString = urlString
                    self.navigationController?.pushViewController(WebViewController(), animated: true)
                } else {
                    let alert = UIAlertController(title: "Attenzione!", message: "url non valido", preferredStyle: .alert)
                    let alertAction = UIAlertAction(title: "ok", style: .default, handler: nil)
                    alert.addAction(alertAction)
                    alert.show(self, sender: nil)
                }
            }
        }
    }

    func isValidUrl (urlString: String?) -> Bool {
        if let urlString = urlString {
            if let url = NSURL(string: urlString) {
                return UIApplication.shared.canOpenURL(url as URL)
            }
        }
        return false
    }
{{< / highlight >}}

Implementiamo dunque il protocollo **AVCaptureMetadataOutputObjectsDelegate** e lo facciamo implementando la funzione **captureOutput()**.
All'interno di questa funzione ci assicuriamo che l'array di metadati non sia nullo e abbia almeno un elemento al suo interno, altrimenti non andiamo oltre.
Prendiamo poi il primo elemento dell'array e facciamo il cast ad un **AVMetadataMachineReadableCodeObject** per controllare che esso sia effettivamente un QR code. In caso positivo, lo sfruttiamo per mostrare un rettangolo che indica che il QR code è stato letto con successo, e procediamo a mostrare l'url in una WebView.

### Apertura Url

{{< highlight swift >}}
import UIKit
import WebKit

class WebViewController: UIViewController, WKUIDelegate {

    var urlString = ""
    var webView: WKWebView!
    
    override func loadView() {
        let webConfiguration = WKWebViewConfiguration()
        webView = WKWebView(frame: .zero, configuration: webConfiguration)
        webView.uiDelegate = self
        view = webView
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        let myURL = URL(string: urlString)
        let request = URLRequest(url: myURL!)
        webView.load(request)
    }
}
{{< / highlight >}}

