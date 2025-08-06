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

  apiVersion: argoproj.io/v1alpha1
  kind: EventBus
  metadata:
    name: default
    namespace: argo-events
  spec:
    nats:
      native:
        replicas: 1


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
2. Operator Prometheusa za webhook adresu gleda secret pod imenon alertmanager-main, a kad stvorimo alertmanager file, on stvori secrete pod imenima poput alertmanager-main-secret-0, dakle, trebamo kreirati vlastiti secret imena alertmanager-main.