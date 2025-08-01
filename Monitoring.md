# Monitoring sa Prometheusom

## Aplikacija
Za potrebe testiranja napravljena je jednostavna aplikacija koja poslužuje **current_connections** na **/metrics** endpoint, a čija se vrijednost može mijenjati kroz **/change_metric** endpoint:

```
const express = require('express');
const app = express();

let currentConnections = 150;

app.get('/metrics', (req, res) => {
  res.set('Content-Type', 'text/plain');
  res.send(`current_connections ${currentConnections}
`);
});

app.get('/change_metric', (req, res) => {
  const val = Number(req.query.value);
  currentConnections = val;
  res.redirect('/metrics');
});

const port = process.env.PORT || 8080;
app.listen(port);
```
Aplikacija se zatim deploya u isti namespace u koji će se instalirati Prometheus.

## Instalacija
Prometheus se može instalirati kao operator preko OpenShift konzole (*Console -> Operators -> OperatorHub -> Prometheus Community -> Install*).

Nakon toga potrebno je napraviti novu Prometheus instancu (*Installed Operators -> Prometheus -> Create Instance*) i zadati joj ime. 
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  creationTimestamp: '2025-08-01T11:53:38Z'
  generation: 1
  managedFields: ...
  name: prometheus
  namespace: prometheus
  resourceVersion: '2986428'
  uid: 08b8f10f-8da9-48d0-9844-f23a20716bee
spec:
  evaluationInterval: 30s
  serviceAccountName: prometheus-k8s
  serviceMonitorSelector: {}
  alerting:
    alertmanagers:
      - name: alertmanager-main
        namespace: monitoring
        port: web
  portName: web
  probeSelector: {}
  podMonitorSelector: {}
  scrapeInterval: 30s
  replicas: 2
  ruleSelector: {}
status:
  availableReplicas: 2
  conditions:
    - lastTransitionTime: '2025-08-01T12:14:30Z'
      status: 'True'
      type: Available
    - lastTransitionTime: '2025-08-01T11:53:38Z'
      status: 'True'
      type: Reconciled
  paused: false
  replicas: 2
  shardStatuses:
    - availableReplicas: 2
      replicas: 2
      shardID: '0'
      unavailableReplicas: 0
      updatedReplicas: 2
  unavailableReplicas: 0
  updatedReplicas: 2
```
Kada je ovo postavljeno kreira se servis, i može se exportati ruta za **Prometheus UI** kroz koji se mogu pratiti sve metrike i alertovi koje Prometheus prati.

Osim Prometheus instance potrebno je napraviti instancu ServiceMonitora koji služi za definiranje onoga što želimo da Prometheus scrape-a. Potrebno ga je usmjeriti na **/metrics** endpoint aplikacije.

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  creationTimestamp: '2025-08-01T11:54:53Z'
  generation: 1
  managedFields: ...
  name: prometheus-sm
  namespace: prometheus
  resourceVersion: '2979997'
  uid: 460d7b22-d421-47b6-af8c-593de6d706c7
spec:
  endpoints:
    - interval: 30s
      path: /metrics
      port: http
  selector:
    matchLabels:
      operated-prometheus: 'true'
```

Idući korak je postavljanje AlertManagera kojim se definira uvjet za koji Prometheus šalje alertove, u ovom primjeru alertovi se šalju kada je vrijednost **current_connections** veća od 1000:

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  creationTimestamp: '2025-08-01T11:57:35Z'
  generation: 1
  managedFields: ...
  name: metrics-rule
  namespace: prometheus
  resourceVersion: '2980820'
  uid: 56cef101-1d52-4179-b7b7-19c1e7516f8c
spec:
  groups:
    - name: metrics.rules
      rules:
        - alert: HighConnections
          annotations:
            description: More than 1000 connections are active.
            summary: High number of connections detected
          expr: current_connections > 1000
          for: 30s
          labels:
            severity: critical
```

Alertovi su vidljivi u Prometheus UI.