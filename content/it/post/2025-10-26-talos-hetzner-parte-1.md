---
title: "Kubernetes su Hetzner con Talos: Parte 1 - Bootstrap dei Nodi Control Plane"
date: 2025-10-26T08:00:00+02:00
draft: false
tags: ["kubernetes", "talos", "hetzner"]
---

## Panoramica

Questo è il primo articolo della serie su come configurare un cluster Kubernetes su Hetzner Cloud utilizzando Talos. In questo articolo, ci concentreremo sul bootstrap di 3 nodi control plane che possono anche eseguire workload.

Il motivi che mi hanno spinto ad addentrarmi in questo viaggio sono i seguenti:
1. **Apprendimento** - Imparare Kubernetes costruendo un cluster da zero
2. **Self-hosting** - Avere un cluster dove eseguire le app che uso quotidianamente (come [Navidrome](https://www.navidrome.org/) per la musica)
3. **Progetti personali** - Un cluster dove effettuare il deployment dei miei progetti e sperimentare. 

Se non hai requisiti simili, questa serie probabilmente non fa per te. Ma se sei un hobbista che vuole imparare e hostare i propri servizi a basso costo, continua a leggere.
Tieni inoltre presente che non amministro cluster Kubernetes per lavoro - questo è un progetto per hobby. Se stai cercando una guida enterprise-level da un professionale amministratore Kubernetes, questa potrebbe non essere la serie giusta per te.
Potresti anche voler leggere l'[eccellente serie di Mathias Pius](https://datavirke.dk/posts/bare-metal-kubernetes-part-1-talos-on-hetzner/) che ho consultato frequentemente quando rimanevo bloccato, così come il tutorial ufficiale di Talos per Hetzner.

## Perché Talos + Hetzner?

Talos OS è una distribuzione Linux minimale e hardenizzata progettata specificamente per Kubernetes. Combinata con l'infrastruttura cloud economica di Hetzner, ottieni un cluster sicuro e gestibile a una frazione del costo dei principali provider cloud.
Per mantenere i costi ulteriormente bassi, useremo una configurazione a tre nodi, dove ogni nodo esegue sia il control plane che i workload effettivi.
Il compromesso è una leggermente minore isolazione tra control plane e workload, ma per molti casi d'uso, questo è perfettamente accettabile. Il costo totale stimato e' meno di 20 euro al mese.

## Prerequisiti

- Account Hetzner Cloud e token API con permessi di lettura/scrittura. Imposta la variabile d'ambiente `HCLOUD_TOKEN` con il token API. Inoltre, assicurati di avere installato lo [strumento hcloud di Hetzner](https://github.com/hetznercloud/cli)
- [talosctl](https://docs.siderolabs.com/talos/v1.11/getting-started/talosctl) installato
- Comprensione di base dei concetti di Kubernetes. Per principianti assoluti puoi comunque seguire, ma suggerisco di integrare questo tutorial con l'eccellente libro [Kubernetes Up & Running](https://www.amazon.com/Kubernetes-Running-Dive-Future-Infrastructure/dp/1492046531)
- [packer](https://developer.hashicorp.com/packer/install) per creare lo snapshot iniziale dell'immagine.

## Passo 1: Creare l'Immagine Talos con Packer

Per prima cosa, abbiamo bisogno di un'immagine Talos utilizzabile in seguito da Hetzner. Useremo Packer per buildare e caricare questa immagine.

```hcl
packer {
  required_plugins {
    hcloud = {
      source  = "github.com/hetznercloud/hcloud"
      version = "~> 1"
    }
  }
  }

  variable "talos_version" {
  type    = string
  default = "v1.11.3"
  }

  variable "arch" {
  type    = string
  default = "amd64"
  }

  variable "server_type" {
  type    = string
  default = "cx23"
  }

  variable "server_location" {
  type    = string
  default = "nbg1"
  }

  locals {
  image = "https://factory.talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/${var.talos_version}/hcloud-${var.arch}.raw.xz"
  }

  source "hcloud" "talos" {
  rescue       = "linux64"
  image        = "debian-11"
  location     = "${var.server_location}"
  server_type  = "${var.server_type}"
  ssh_username = "root"

  snapshot_name   = "talos system disk - ${var.arch} - ${var.talos_version}"
  snapshot_labels = {
    type    = "infra",
    os      = "talos",
    version = "${var.talos_version}",
    arch    = "${var.arch}",
  }
  }

  build {
  sources = ["source.hcloud.talos"]

  provisioner "shell" {
    inline = [
      "apt-get install -y wget",
      "wget -O /tmp/talos.raw.xz ${local.image}",
      "xz -d -c /tmp/talos.raw.xz | dd of=/dev/sda && sync",
    ]
  }
  }
```

Builda l'immagine:
```bash
# Naviga nella tua directory infrastructure (adatta il percorso secondo necessità)
cd infrastructure
packer build hcloud.pkr.hcl
```

Questo comando farà alcune cose:
1. Avvia un server Debian su Hetzner
2. Abilita la modalità rescue e riavvia il server
3. Scarica l'immagine Talos dalla factory Talos per sovrascrivere l'immagine Debian
4. Spegne il server e crea lo snapshot
5. Carica lo snapshot su Hetzner

L'intero processo mi ha richiesto 7 minuti, quindi sentiti libero di fare una pausa mentre l'automazione compie il suo dovere. 
Una volta terminato, tieni traccia dell'ID dello snapshot dall'output di Packer - lo useremo nei passaggi seguenti.
```bash
export SNAPSHOT_ID=<id dello snapshot>
```

## Passo 2: Configurare il Load Balancer

Useremo un load balancer per l'endpoint dell'API Kubernetes, così come per l'ingress.
Questo non è strettamente necessario all'inizio e potresti risparmiare $5 al mese non usandolo,
ma ho deciso di usarlo dato che mi servirà comunque per i miei progetti personali.

```bash
# Crea load balancer
hcloud load-balancer create \
  --name talos-control-plane \
  --network-zone eu-central \
  --type lb11 \
  --label 'type=controlplane'

# Aggiungi servizio per Kubernetes API
hcloud load-balancer add-service \
  --name talos-control-plane \
  --protocol tcp \
  --listen-port 6443 \
  --destination-port 6443

# Aggiungi target usando label
hcloud load-balancer add-target talos-control-plane --label-selector 'type=controlplane'
```
Prima creiamo il load balancer. Dopodiché esponiamo la porta `6443` e usiamo la stessa porta come destinazione. Creiamo un target group usando il selettore di label, così che ogni volta che avviamo un nuovo nodo con quella label, verrà automaticamente aggiunto al gruppo.

Ora possiamo trovare l'IP del nostro load balancer appena creato con il seguente comando
```bash
hcloud loadbalancer list
```

Aggiungiamolo a una variabile d'ambiente, ci servirà nei passaggi seguenti
```bash
export LB_IP=<ip del tuo load balancer>
```

## Passo 3: Generare la Configurazione Talos

Ora siamo pronti per generare la configurazione talos, che useremo per connetterci al cluster e per avviare nuovi nodi.

```bash
# Genera configurazione cluster
talosctl gen config talos-k8s-hcloud-tutorial https://$LB_IP:6443 \
  --with-examples=false \
  --with-docs=false
```

Questo crea diversi file, inclusi `controlplane.yaml`, `worker.yaml`, e `talosconfig`.

## Passo 4: Patch della Configurazione per l'Integrazione con Hetzner

Applicheremo alcune modifiche alla configurazione. Per farlo, useremo delle patch.
Creiamo una cartella patches dove possiamo tenere traccia di tutte le patch.
Inizieremo patchando il cluster per permettere externalCloudProviders. Questo sarà richiesto per usare l'[hetzner cloud controller manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager).


```yaml
# patches/external_cloud_providers.yaml
cluster:
  externalCloudProvider:
    enabled: true
```

Successivamente, creeremo una patch per permettere ai workload di essere schedulati sui nodi control plane.
Questo impedirà ai nodi di essere taintati, che fermerebbe lo scheduling dei workload.
```yaml
# patches/allow_controlplane_workloads.yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

Applica la patch:
```bash
talosctl machineconfig patch controlplane.yaml --patch @patches/allow_controlplane_workloads.yaml --patch @patches/external_cloud_providers.yaml -o controlplane.yaml
```

## Passo 5: Deploy dei Nodi Control Plane

Ora creiamo i nostri 3 nodi control plane usando la configurazione patchata:

```bash
# Crea tre nodi control plane
for i in 1 2 3; do
  hcloud server create \
    --name talos-control-plane-$i \
    --image $SNAPSHOT_ID \
    --type cx23 \
    --location nbg1 \
    --label 'type=controlplane' \
    --user-data-from-file controlplane.yaml
done
```

Per il mio cluster sto usando le istanze più piccole disponibili dato che sono più che sufficienti per le mie necessità.

## Passo 6: Bootstrap del Cluster
Troviamo l'IP del primo nodo usando `hcloud server list | grep talos-control-plane` ed esportiamolo come variabile d'ambiente:

```bash
export FIRST_NODE_IP=<ip del tuo nodo control-plane>
```

Ora configuriamo il client Talos per connettersi a questo nodo impostando sia l'endpoint che il nodo:

```bash
talosctl --talosconfig talosconfig config endpoint $FIRST_NODE_IP
talosctl --talosconfig talosconfig config node $FIRST_NODE_IP
```

Siamo pronti per fare il bootstrap di [etcd](https://etcd.io) sul nostro primo nodo control plane

```bash
talosctl --talosconfig talosconfig bootstrap
```

Dovresti quindi essere in grado di vedere tutti e tre i nodi con il seguente comando:
```bash
talosctl --talosconfig talosconfig get members
```

Possiamo quindi recuperare il kubeconfig e usarlo con kubectl

```bash
talosctl --talosconfig talosconfig kubeconfig .
export KUBECONFIG=./kubeconfig
kubectl get nodes -o wide
```

Verifica che tutto funzioni:

```bash
# Controlla stato nodi
kubectl get nodes -o wide
```

Possiamo dare un'occhiata alla salute del cluster usando il comando talosctl dashboard

```bash
talosctl --talosconfig talosconfig dashboard
```

## Passo 7: Hetzner Cloud Controller Manager

Configurare l'Hetzner cloud controller manager è stato abbastanza semplice, ho appena seguito il suo [quickstart.md ufficiale](https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/docs/guides/quickstart.md).

Crea un secret contenente il tuo token API Hetzner:
```bash
kubectl -n kube-system create secret generic hcloud --from-literal=token=<hcloud API token>
```

Aggiungi il repository helm
```bash
helm repo add hcloud https://charts.hetzner.cloud
helm repo update hcloud
```

Installa il chart
```bash
helm install hccm hcloud/hcloud-cloud-controller-manager -n kube-system
```

Dovresti essere in grado di vedere hcloud-cloud-controller-manager nei deployments
```bash
kubectl get deployments -n kube-system
```

## Prossimi passi?

Nelle prossime parti di questa serie, tratteremo:
- [Configurare ArgoCD](/post/2025-10-27-talos-hetzner-parte-2/) (Parte 2)
- Configurare lo storage con Longhorn
- Deploy di [Navidrome](https://www.navidrome.org/) per lo streaming musicale

## Riepilogo

Abbiamo bootstrapato con successo un cluster Kubernetes a 3 nodi usando Talos su Hetzner Cloud. I nostri nodi control plane possono anche eseguire workload, ottenendo una configurazione economica e altamente disponibile.
