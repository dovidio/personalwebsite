---
title: "Kubernetes su Hetzner con Talos: Parte 2 - Installazione di Argo CD"
date: 2025-10-27T08:00:00+02:00
draft: false
tags: ["kubernetes", "talos", "hetzner", "argocd"]
---

## Panoramica

Nella [parte precedente di questa serie](/post/2025-10-26-talos-hetzner-parte-1/), abbiamo avviato un cluster Kubernetes a 3 nodi su Hetzner Cloud utilizzando Talos OS. Ora installeremo Argo CD, uno strumento dichiarativo di continuous delivery GitOps per Kubernetes. Argo CD ci permette di gestire le nostre applicazioni utilizzando la configurazione come codice, il che significa che possiamo versionare i nostri deployment e avere una chiara traccia di controllo di ciò che è in esecuzione nel nostro cluster.

Se non hai ancora seguito la Parte 1, torna indietro e configura prima il tuo cluster. Questo tutorial presuppone che tu abbia un cluster Kubernetes funzionante e kubectl configurato per connettersi ad esso.

## Perché Argo CD?

Per un cluster self-hosted che esegue progetti personali e applicazioni secondarie, Argo CD offre diversi vantaggi:

- **Workflow GitOps** - Il tuo repository Git diventa l'unica fonte di verità per lo stato del cluster
- **Rollback facili** - Se qualcosa si rompe, puoi rapidamente tornare a uno stato precedente funzionante
- **Visibilità** - L'interfaccia web ti dà una visione chiara di tutte le tue applicazioni e del loro stato di sincronizzazione
- **Automazione** - Argo CD può sincronizzare automaticamente le tue applicazioni quando fai push delle modifiche su Git

## Prerequisiti

- Un cluster Kubernetes funzionante (dalla Parte 1)
- kubectl configurato per accedere al tuo cluster
- Homebrew (per installare la CLI di Argo CD su macOS)

## Passo 1: Creare il Namespace di Argo CD

Prima, creiamo un namespace dedicato per Argo CD:

```bash
kubectl create namespace argocd
```

Mantenere Argo CD nel suo namespace aiuta con l'organizzazione e rende più facile la gestione delle risorse.

## Passo 2: Installare Argo CD in Modalità Alta Disponibilità

Installeremo Argo CD utilizzando il manifest ufficiale per l'alta disponibilità. Dato che abbiamo un cluster a 3 nodi, eseguire Argo CD in modalità HA ha senso per l'affidabilità:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

Questo deploierà tutti i componenti necessari inclusi l'API server, il repo server, l'application controller e Redis per il caching. L'installazione potrebbe richiedere un minuto o due mentre scarica le immagini e avvia i pod.

Puoi osservare i pod che si avviano con:

```bash
kubectl get pods -n argocd -w
```

Aspetta finché tutti i pod sono nello stato `Running` prima di procedere.

## Passo 3: Installare la CLI di Argo CD

La CLI di Argo CD rende facile interagire con Argo CD dal tuo terminale. Installala usando Homebrew:

```bash
brew install argocd
```

## Passo 4: Recuperare la Password Admin Iniziale

Argo CD crea una password admin iniziale durante l'installazione. Recuperala usando la CLI:

```bash
argocd admin initial-password -n argocd
```

In alternativa, puoi ottenerla direttamente dal secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Tieni questa password a portata di mano - ne avrai bisogno nel prossimo passo.

## Passo 5: Accedere alla Dashboard di Argo CD

Il modo più semplice per accedere all'interfaccia web di Argo CD è utilizzare il comando admin dashboard:

```bash
argocd admin dashboard -n argocd
```

Questo comando gestirà automaticamente il port forwarding e aprirà il tuo browser all'interfaccia di Argo CD. Vedrai un avviso di certificato dato che stiamo usando un certificato auto-firmato - è previsto per ora.

Puoi effettuare il login con:
- Username: `admin`
- Password: la password dal Passo 4

## Passo 6: Login tramite CLI

Ora facciamo il login a Argo CD dalla CLI:

```bash
argocd login localhost:8080
```

Quando richiesto, usa:
- Username: `admin`
- Password: la password dal Passo 5

Potresti vedere un avviso sul certificato - digita `y` per procedere.

## Passo 7: Cambiare la Password Admin

Per sicurezza, cambia la password admin predefinita con qualcosa che ricorderai:

```bash
argocd account update-password
```

Segui i prompt per impostare una nuova password.

## Passo 8: Test con l'Applicazione Guestbook

Facciamo il deploy di un'applicazione di test per verificare che tutto funzioni. Prima, passa al namespace argocd di default per risparmiare un po' di digitazione:

```bash
kubectl config set-context --current --namespace=argocd
```

Ora crea l'applicazione guestbook:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Questo dice ad Argo CD di tracciare l'applicazione guestbook dal repository di esempio e deploiarla nel namespace default.

Sincronizza l'applicazione per effettivamente deploiarla:

```bash
argocd app sync guestbook
```

Dovresti vedere l'output che mostra Argo CD che deploia le risorse guestbook. Controlla lo stato:

```bash
argocd app get guestbook
```

Puoi anche visualizzare l'applicazione nell'interfaccia web a `https://localhost:8080` - vedrai una bella visualizzazione di tutte le risorse Kubernetes che compongono l'app guestbook.

## Passo 9: Accedere all'Interfaccia Senza Port Forwarding

Il metodo port-forward funziona bene per l'accesso temporaneo, ma richiede di tenere aperta una finestra del terminale. Per l'uso regolare, puoi accedere alla dashboard di Argo CD usando:

```bash
argocd admin dashboard -n argocd
```

Questo comando aprirà automaticamente il tuo browser e gestirà il port forwarding per te.

Se vuoi esporre Argo CD al mondo esterno permanentemente, la documentazione ufficiale di Argo CD ha guide dettagliate sulla configurazione di ingress con vari ingress controller. Per il mio caso d'uso personale, non ho bisogno di accesso esterno dato che posso usare il comando admin dashboard ogni volta che ho bisogno di controllare i miei deployment.

## Cosa Succede Dopo?

Ora che abbiamo Argo CD in esecuzione, abbiamo una solida base per gestire le applicazioni nel nostro cluster. Nelle prossime parti di questa serie, copriremo:

- Configurazione dello storage con Longhorn
- Deploy di applicazioni reali come Navidrome per lo streaming musicale
- Gestione di quelle applicazioni con Argo CD

## Riassunto

Abbiamo installato con successo Argo CD in modalità alta disponibilità sul nostro cluster Kubernetes Talos. Abbiamo verificato l'installazione deployando l'applicazione di test guestbook e ora possiamo accedere all'interfaccia web per visualizzare i nostri deployment. Con Argo CD in posizione, siamo pronti per iniziare a gestire le nostre applicazioni usando i principi GitOps.