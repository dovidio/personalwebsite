---
title: "Kubernetes su Hetzner con Talos: Parte 3 - Configurazione di Longhorn Storage"
date: 2025-10-28T08:00:00+02:00
draft: false
tags: ["kubernetes", "talos", "hetzner", "longhorn", "storage"]
---

## Panoramica

Nelle [parti precedenti di questa serie](/post/2025-10-27-talos-hetzner-parte-2/), abbiamo avviato un cluster Kubernetes e installato Argo CD. Ora è il momento di aggiungere storage persistente con Longhorn. Longhorn è un sistema di storage a blocchi distribuito leggero, affidabile e facile da usare per Kubernetes. È perfetto per il nostro cluster self-hosted in quanto fornisce replica dei dati, capacità di backup e una bella interfaccia web per la gestione.

Se non hai seguito le parti precedenti, assicurati di avere:
- Un cluster Talos Kubernetes funzionante ([Parte 1](/post/2025-10-26-talos-hetzner-parte-1/))
- Argo CD installato e configurato ([Parte 2](/post/2025-10-27-talos-hetzner-parte-2/))

## Perché Longhorn?

Per un cluster self-hosted, Longhorn offre diversi vantaggi:

- **Costruito per Kubernetes** - Storage nativo Kubernetes che si integra perfettamente
- **Resilienza dei Dati** - Replica automatica tra nodi previene la perdita di dati
- **Gestione Facile** - Interfaccia web per monitoraggio e gestione dello storage
- **Supporto Backup** - Capacità di backup integrate verso storage esterno
- **Nessuna Dipendenza Esterna** - Utilizza lo storage locale sui tuoi nodi

## Prerequisiti

- Cluster Talos Kubernetes dalla Parte 1
- Argo CD installato dalla Parte 2
- `talosctl` configurato per accedere al tuo cluster
- `kubectl` configurato per accedere al tuo cluster

## Passo 1: Installare le Estensioni Talos Richieste

Longhorn richiede estensioni di sistema specifiche per funzionare correttamente. Dobbiamo aggiungere `iscsi-tools` e `util-linux-tools` a ogni nodo.

Prima, generiamo un'immagine Talos personalizzata con le estensioni richieste:

```bash
curl -X POST --data-binary @- https://factory.talos.dev/schematics <<EOF
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
      - siderolabs/util-linux-tools
EOF
```

Questo restituirà un ID schematico che useremo per l'aggiornamento:

```json
{"id":"613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245"}
```

Salva questo ID schematico come variabile d'ambiente:

```bash
export SCHEMATIC_ID="613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245"
```

## Passo 2: Aggiornare Ogni Nodo con le Estensioni

Ora dobbiamo aggiornare ogni nodo per includere le estensioni richieste. Lo faremo un nodo alla volta:

```bash
# Ottieni l'elenco dei tuoi nodi
kubectl get nodes -o wide

# Aggiorna ogni nodo (sostituisci con i tuoi IP effettivi dei nodi)
export NODE1_IP="your-node-1-ip"
export NODE2_IP="your-node-2-ip" 
export NODE3_IP="your-node-3-ip"

# Aggiorna nodo 1
talosctl upgrade \
  --nodes $NODE1_IP \
  --image factory.talos.dev/installer/$SCHEMATIC_ID:v1.11.3

# Aspetta che il nodo 1 torni online, poi aggiorna il nodo 2
talosctl upgrade \
  --nodes $NODE2_IP \
  --image factory.talos.dev/installer/$SCHEMATIC_ID:v1.11.3

# Aspetta che il nodo 2 torni online, poi aggiorna il nodo 3
talosctl upgrade \
  --nodes $NODE3_IP \
  --image factory.talos.dev/installer/$SCHEMATIC_ID:v1.11.3
```

Ogni nodo si riavvierà durante l'aggiornamento. Aspetta che ogni nodo torni online prima di aggiornare il successivo.

## Passo 3: Verificare l'Installazione delle Estensioni

Dopo che tutti i nodi sono stati aggiornati, verifica che le estensioni siano installate:

```bash
# Controlla le estensioni su ogni nodo
talosctl get extensions -n $NODE1_IP
talosctl get extensions -n $NODE2_IP
talosctl get extensions -n $NODE3_IP
```

Dovresti vedere `iscsi-tools` e `util-linux-tools` elencati nell'output.

## Passo 4: Creare il Namespace Longhorn

Crea un namespace dedicato per Longhorn con i permessi di sicurezza necessari:

```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system pod-security.kubernetes.io/enforce=privileged
kubectl label namespace longhorn-system pod-security.kubernetes.io/audit=privileged
kubectl label namespace longhorn-system pod-security.kubernetes.io/warn=privileged
```

Queste etichette sono richieste perché Longhorn ha bisogno di accesso privilegiato per gestire lo storage sui nodi.

## Passo 5: Deployare Longhorn tramite Argo CD

Ora usiamo Argo CD per deployare Longhorn. Crea un file chiamato `longhorn-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argocd
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
  project: default
  sources:
    - chart: longhorn
      repoURL: https://charts.longhorn.io/
      targetRevision: v1.10.0
      helm:
        values: |
          preUpgradeChecker:
            jobEnabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: longhorn-system
```

Applica l'applicazione usando kubectl:

```bash
kubectl apply -f longhorn-app.yaml
```

## Passo 6: Monitorare il Deployment

Puoi monitorare il progresso del deployment nella dashboard di Argo CD o tramite la CLI:

```bash
# Controlla lo stato dell'applicazione Argo CD
argocd app get longhorn

# Osserva i pod Longhorn che si avviano
kubectl get pods -n longhorn-system -w
```

Il deployment potrebbe richiedere alcuni minuti. Potresti dover attivare una sincronizzazione in Argo CD se alcune risorse appaiono degradate inizialmente - questo è normale dato che alcuni componenti aspettano che altri siano pronti.

## Passo 7: Accedere alla Dashboard Longhorn

Una volta che Longhorn è deployato, puoi accedere alla sua interfaccia web usando il port forwarding:

```bash
kubectl port-forward svc/longhorn-frontend -n longhorn-system 8081:80
```

Poi apri il tuo browser a `http://localhost:8081` per vedere la dashboard Longhorn. Vedrai tutti i tuoi nodi e il loro storage disponibile.

Per comodità, puoi creare un alias:

```bash
alias longhorn_dashboard="kubectl port-forward svc/longhorn-frontend -n longhorn-system 8081:80"
```

## Passo 8 (Opzionale): Aggiungere Storage Esterno

Se hai volumi di storage aggiuntivi in Hetzner che vuoi che Longhorn utilizzi, ecco come configurarli:

### Attaccare il Volume
Prima, attacca il volume a uno dei tuoi nodi tramite la Console Hetzner Cloud.

### Configurare Talos per Montare il Volume
Crea un file patch chiamato `longhorn-storage-patch.yaml` per esporre il volume come user volume:

```yaml
# longhorn-storage-patch.yaml
machine:
  disks:
    - device: /dev/sdb  # Sostituisci con il tuo device volume
      partitions:
        - mountpoint: /var/mnt/longhorn-extra-storage
```

Se il volume è già formattato, potresti dover prima pulirlo usando talosctl:

```bash
talosctl wipe disk sdb -n $NODE_IP
```

Poi applica la patch usando talosctl e verifica:

```bash
talosctl machineconfig patch controlplane.yaml --patch @longhorn-storage-patch.yaml -o controlplane-updated.yaml
# Applica la configurazione aggiornata al nodo specifico
talosctl get volumestatus -n $NODE_IP
```

### Aggiungere Disco nell'Interfaccia Longhorn
Una volta che il volume è montato, vai alla dashboard Longhorn, naviga su "Node" -> seleziona il tuo nodo -> "Edit Disks" -> "Add Disk" e aggiungi il nuovo percorso storage.

## Cosa Succede Dopo?

Ora che abbiamo configurato lo storage persistente, siamo pronti per deployare applicazioni stateful! Nella prossima parte di questa serie, deploieremo Navidrome per lo streaming musicale, che dimostrerà come usare lo storage Longhorn per applicazioni reali.

## Riassunto

Abbiamo installato con successo lo storage distribuito Longhorn sul nostro cluster Talos Kubernetes. La nostra configurazione ora include:

- Nodi Talos con le estensioni storage richieste
- Longhorn deployato tramite Argo CD per la gestione GitOps
- Una dashboard web per il monitoraggio dello storage
- Storage persistente pronto per applicazioni stateful

Con Longhorn in posizione, ora abbiamo una soluzione storage production-ready che può gestire replica dei dati, backup e recovery per le nostre applicazioni self-hosted.