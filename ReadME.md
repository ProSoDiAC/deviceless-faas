# 🔁 Serverless Event-Driven Deployment with Nuclio & RabbitMQ

Questo progetto mostra un'architettura serverless che sfrutta Nuclio, RabbitMQ e dispositivi IoT per attivare automaticamente il deploy di funzioni in risposta ad eventi asincroni.

## 📐 Architettura

1. Un sistema cloud (S4T) genera eventi relativi alla creazione di risorse (es. plugin).
2. Gli eventi vengono pubblicati su un exchange RabbitMQ.
3. Una funzione Nuclio con trigger RabbitMQ riceve l'evento.
4. La funzione effettua una chiamata HTTP GET verso una board remota.
5. La board, tramite un miniserver Node.js, esegue il deploy di una funzione Nuclio localmente.

📌 Diagramma dettagliato 
![FaaS4T](docs/FaaS4T.PNG "FaaS4T")

## ⚙️ Requisiti

- DockerHub account
- Cluster [K3s](https://github.com/MDSLab/Stack4Things_k3s_deployment?tab=readme-ov-file#k3s-installation) attivo 
- [Helm](https://github.com/MDSLab/Stack4Things_k3s_deployment?tab=readme-ov-file#helm-installation)
- [kubectl](https://github.com/MDSLab/Stack4Things_k3s_deployment?tab=readme-ov-file#k3s-installation)
- nuctl (CLI Nuclio)
- Node.js (installato sulla board)

---

## 🚀 Setup

### 1. Installazione di Nuclio su K3s

Eseguire i seguenti comandi:

```bash
curl -s https://api.github.com/repos/nuclio/nuclio/releases/latest \
  | grep -i "browser_download_url.*nuctl.*$(uname)" \
  | cut -d : -f 2,3 | tr -d \" | wget -O nuctl -qi - && chmod +x nuctl

docker login

kubectl create namespace nuclio

kubectl --namespace nuclio create secret docker-registry registry-credentials \
  --docker-username <username> \
  --docker-password <password> \
  --docker-server <registry-url> \
  --docker-email <email>

helm repo add nuclio https://nuclio.github.io/nuclio/charts

helm install nuclio \
  --set registry.secretName=registry-credentials \
  --set registry.pushPullUrl=index.docker.io/<tuo-username> \
  --namespace nuclio \
  nuclio/nuclio
```

Dopo l'installazione, è possibile verificare che tutti i pod siano in esecuzione con il comando:
```
kubectl get pods -n nuclio
helm get values nuclio -n nuclio
```

Inoltre, è possibile verificare il corretto registry URL:
```
helm get values nuclio -n nuclio
```
---

### 2. Deploy della Funzione Nuclio con Trigger RabbitMQ

La funzione scritta in Go si trova in functions/fn1.go. È configurata per attivarsi tramite RabbitMQ con le seguenti impostazioni:

- Exchange: openstack
- Queue: iotronic.conductor_manager
- URL AMQP: es. amqp://user:pass@rabbitmq-host:5672


La funzione:

- Logga il corpo dell'evento ricevuto
- Effettua una GET verso http://\<board-ip>:8080/fn1
- Salva localmente gli eventi su /tmp/events.json
- Restituisce l'elenco degli eventi come JSON

---

### 3. Miniserver sulla Board (Node.js)

La board esegue un semplice miniserver che ascolta su /fn1 ed esegue un comando di deploy Nuclio.

File: [board/server.js](https://github.com/fabiooraziomirto/FaaS4Things/blob/main/board/server.js)

Esecuzione:

```bash
node server.js
```

Comando interno:

```bash
./nuctl deploy \
  -p https://raw.githubusercontent.com/nuclio/nuclio/master/hack/examples/golang/helloworld/helloworld.go \
  --registry localhost:5000 \
  helloworld \
  --run-registry localhost:5000 \
  --platform local
```

Assicurati che nuctl sia disponibile sulla board e che un registry Docker locale sia in esecuzione.

---

## 📂 Struttura della Repository

```text
.
├── functions/
│   ├── fn1.go                   # Funzione Nuclio con trigger RabbitMQ (architettura event-driven)
│   ├── complexity/               # Esperimento: complessità O(n)/O(n²)/O(n³) su lettura file
│   │   ├── complexity_1.go / .py # O(n)
│   │   ├── complexity_2.go / .py # O(n²)
│   │   └── complexity_3.go       # O(n³)
│   └── benchmarks/               # Esperimento: complessità O(n)/O(n²) su operazioni array
│       ├── sum.go, monitoring_on.py     # O(n) - somma array
│       └── bubblesort.go, monitoring_on2.py # O(n²) - bubble sort
├── board/
│   └── server.js                # Miniserver Node.js sulla board
├── deploy/
│   └── nuclio-trigger.yaml      # (opzionale) Configurazione YAML Nuclio
├── docs/
│   └── FaaS4T.png               # Diagramma architettura
├── README.md
└── ...
```

---

## 🧩 Tecnologie Utilizzate

- Nuclio (serverless framework)
- RabbitMQ (event streaming)
- Node.js (miniserver sulla board)
- K3s (lightweight Kubernetes)
- Docker (per containerizzazione)

---


📌 Ultimo aggiornamento: Aprile 2025


