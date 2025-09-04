Zadatak je sljedeći: Prometheus scrapea metriku od current_connections i ima postavljen alert kada je "broj konekcija" veći od 1000. Trebamo namjestiti alert manager da šalje webhook do Argo Eventsa koji pokreće Argo Workflow ispisa loga.

Instalacija Argo Workflowsa:

    $ oc create namespace argo
    $ oc apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.7.0/install.yaml

    $ oc expose svc argo-server -n argo
    $ oc get route argo-server -n argo
    ili
    $ oc -n argo port-forward service/argo-server 2746:2746

Zatim trebamo otići u 

    $ oc edit deployment argo-server

te dodati:

    spec:
    containers:
    - name: argo-server
        image: argoproj/argocli:v3.7.0
        args:
        - server
        - --auth-mode=server   # <-- ovu liniju
  
Sada napokon možemo pristupiti Argo UI.

Dalje je potrebno stvoriti potrebna security dopuštenja za argo: napravimo rbac-argo.yaml i upišemo:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: argo-workflow
    rules:
    - apiGroups: ["argoproj.io"]
        resources: ["workflows", "workflowtemplates", "cronworkflows", "workflowtaskresults"]
        verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    - apiGroups: [""]
        resources: ["pods", "pods/log", "persistentvolumeclaims", "configmaps", "secrets", "serviceaccounts"]
        verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: argo-workflows-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: argo-workflow
    subjects:
    - kind: ServiceAccount
      name: default
      namespace: argo
    - kind: ServiceAccount
      name: default
      namespace: argo-events

te applyamo:

    $ oc apply -f rbac-argo.yaml

Napravimo jednostavni workflow za ispis poruke, odemo u Workflows -> Create Workflows:

    apiVersion: argoproj.io/v1alpha1
    kind: Workflow
    metadata:
      generateName: hello-world-
    spec:
      entrypoint: hello
      templates:
        - name: hello
        container:
            image: alpine:3.18
            command: [sh, -c]
            args: ["echo 'hello world'"]

te odaberemo "Create". Workflow bi se trebao stvoriti bez problema zbog prethodnog Role Bindinga.

Sada kreiramo novi namespace argo-events i skidamo:

    $ oc new-project argo-events
    $ oc apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml

te dodajemo anyuid SCC za argo events:

    $ oc adm policy add-scc-to-user anyuid -z argo-events-sa -n argo-events

i restartamo deployment:

    $ oc -n argo-events rollout restart deployment controller-manager

Onda kreiramo jednostavni event bus u fileu bus-event.yaml i applyamo ga:
```
  apiVersion: argoproj.io/v1alpha1
  kind: EventBus
  metadata:
    name: default
    namespace: argo-events
  spec:
    nats:
      native:
        replicas: 1
```

    $ oc apply -f bus-event.yaml

Odemo u Event Sources -> Create i pasteamo

    apiVersion: argoproj.io/v1alpha1
    kind: EventSource
    metadata:
      name: webhook-source
      namespace: argo-events
    spec:
      eventBusName: default
      service:
        ports:
        - port: 12000
          targetPort: 12000
    webhook:
        hello:
          endpoint: /hello
          method: POST
          port: '12000'

Odemo u Sensors i odaberemo Create i pasteamo

    apiVersion: argoproj.io/v1alpha1
    kind: Sensor
    metadata:
      name: webhook-sensor
      namespace: argo-events
    spec:
      dependencies:
        - name: hello-dep
          eventSourceName: webhook-source
          eventName: hello
      triggers:
        - template:
            name: trigger-workflow
            argoWorkflow:
              group: argoproj.io
              version: v1alpha1
              resource: workflows
              operation: submit
              source:
                resource:
                  apiVersion: argoproj.io/v1alpha1
                  kind: Workflow
                  metadata:
                    generateName: hello-from-webhook-
                  spec:
                    entrypoint: hello
                    templates:
                      - name: hello
                        container:
                          image: alpine:3.18
                          command: ["sh", "-c"]
                          args: ["echo 'Hello from Webhook!'"]

Sada trebamo exposati server koji prima webhook:

    $ oc expose service/webhook-source-eventsource-svc

i dobijemo route:

    webhook-source-eventsource-svc-argo-events.apps.op1os.lan.croz.net

Sada iz terminala možemo pozvati:

    $ curl -X POST \
    -H "Content-Type: application/json" \
    -d '{"message": "from alertmanager"}' \
    http://webhook-source-eventsource-svc-argo-events.apps.op1os.lan.croz.net/hello

I u Argo UI odaberemo Event Flow i vidimo da se stvorio novi pod željenim logom kojeg možemo očitati u "Logs"

Zadnje nam je ostalo namjestiti AlertManager da šalje post zahtjeve na našu rutu.
Ovdje sam naišao na veliki problem zbog Prometheus default imenovanja: evo dvije greške koje je bitno razmotriti:

1. Instanca Prometheus operatora koju smo stvorili na pocetku (dokumentacija za instalaciju Prometheusa) po defaultu ima krivo ime endpointa za slanje alertova, naime default ime servicea na kojem sluša alertmanager je "alertmanager-operated", a u yaml-u od instance Prometheusa je "alertmanager-main" što je ime alertmanagera ali ne servicea što nama i treba, dakle potreno je otiću u konzolu, odabrati instancu prometheusa i promjeniti "name: alertmanager-main" u "name: alertmanager-operated"
2. Ako koristimo default imenovanje secreta od strane alertmanagera, zvat će se alertmanager-<imealertmanagera>-generated, ali default configSecret koji odabire secret koji ce se koristiti je <imealertmanagera>-alertmanager. Dakle, ako kod stvaranja default instance alertmanagera, u pitanju ienovanja stvori default secret jednog imena i secret selector drugog imena. Kad hoveramo preko configSecret vidimo: ConfigSecret is the name of a Kubernetes Secret in the same namespace as the Alertmanager object, which contains the configuration for this Alertmanager instance. If empty, it defaults to `alertmanager-<alertmanager-name>`. The Alertmanager configuration should be available under the `alertmanager.yaml` key. Additional keys from the original secret are copied to the generated secret and mounted into the ` etc/alertmanager/config` directory in the `alertmanager` container. If either the secret or the `alertmanager.yaml` key is missing, the operator provisions a minimal Alertmanager configuration with one empty receiver (effectively dropping alert notifications).

Stvaranje alertmanagera redom ide ovako: prvo stvorimo alertmanager-config.yaml:

    kind: AlertmanagerConfig
    apiVersion: monitoring.coreos.com/v1alpha1
    metadata:
      name: alertmanager-config
      namespace: js-prom
      labels:
        alertmanagerConfig: "main"
    spec:
      receivers:
        - name: argo-webhook
          webhookConfigs:
            - url: 'http://webhook-source-eventsource-svc-argo-events.apps.op1os.lan.croz.net/hello'
      route:
        groupInterval: 10s
        groupWait: 0s
        receiver: argo-webhook
        repeatInterval: 1h

tu definiramo url za webhook i ostale informacije o konfiguraciji. Onda stvaramo alertmanager.yaml kojim selektiramo u konfiguraciju na osnovu cega ce se stvoriti secret koji je operatoru bitan za razaznavanje kome slati alert:

    apiVersion: monitoring.coreos.com/v1
    kind: Alertmanager
    metadata:
      name: alertmanager
      namespace: js-prom
    spec:
      replicas: 2
      alertmanagerConfigSelector:
        matchLabels:
          alertmanagerConfig: main

Bitno je da ispravno labelamo config i onda ispravno matchLabelamo taj label.
Stvorit će se secret koji možemo vidjet s 

    $ oc get secrets

i vidjet ćemo nešto tipa:

    alertmanager-<ime_alertmanagera>-generated

Uzmemo to ime, odemo u alert manager i maknemo dio koda koji selektira config i dodatmo novu liniju koda koji odabire secret:

    apiVersion: monitoring.coreos.com/v1
    kind: Alertmanager
    metadata:
      name: alertmanager
      namespace: js-prom
    spec:
      replicas: 2
      configSecret: alertmanager-<ime_alertmanagera>-generated

te nakon što reapplyamo sve je postavljeno.

## Kronologija

1. Prometheus scrapea target current_connections
2. PrometheusRule vidi da je target iznad 1000, prekršen je rule, stvara se alert
3. Prometheus šalje alert do AlertManagera
4. Alert manager šalje HTTP POST request na webhook endpoint koji smo zadali
5. Argo Events vidi aktivnost na webhooku, šalje event na Event Bus koji funkcionira kao queue
6. Argo Sensor osluškuje Event Bus, kad vidi novi event pokreće novi Workflow
7. Zadani workflow ispisuje echo poruku koju možemo vidjeti u "Logs" unutar "Workflows"