Zadatak: ako je broj konekcija pre visok duže od 3 minute, premjestiti aplikaciju na drugi node, u protivnom staviti aplikaciju na prvi node.

# Rješenje

Prvo stvaramo novi Prometheus Rule
    
    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: prometheus-switch-rules
      namespace: js-prom
    spec:
      groups:
        - name: metric.rules
          rules:
            - alert: HighConnectionsSwitch1
              annotations:
                description: Premještamo na worker 1
                summary: Premještamo na worker 1 zbog pre velikog broja konekcija
              expr: current_connections >= 1000
              for: 3m
              labels:
                severity: critical
            - alert: LowConnectionsSwitch0
              annotations:
                description: Premještamo na worker 0
                summary: Premještamo na worker 0 zbog pre velikog broja konekcija
              expr: current_connections < 1000
              for: 3m
              labels:
                severity: critical

Zatim stvaramo novi alertmanagerConfig:

    apiVersion: monitoring.coreos.com/v1alpha1
    kind: AlertmanagerConfig
    metadata:
      name: alertmanager-config-switch
      namespace: js-prom
      labels:
        alertmanagerConfig: switch
    spec:
      receivers:
        - name: move-to-worker0
          webhookConfigs:
            - url: 'http://webhook-switch-eventsource-svc-argo-events.apps.op1os.lan.croz.net/low'
    
        - name: move-to-worker1
          webhookConfigs:
            - url: 'http://webhook-switch-eventsource-svc-argo-events.apps.op1os.lan.croz.net/high'
    
      route:
        receiver: move-to-worker0
        routes:
          - matchers:
              - name: alertname
                value: HighConnectionsSwitch1
                matchType: =
            receiver: move-to-worker1
    
          - matchers:
              - name: alertname
                value: LowConnectionsSwitch0
                matchType: =
            receiver: move-to-worker0

Stvaramo alertmanager metodom objašnjenom u dokumentaciji za Argo Workflows, dakle prvo želimo stvoriti secrete:

    apiVersion: monitoring.coreos.com/v1
    kind: Alertmanager
    metadata:
      name: alertmanager-switch
      namespace: js-prom
    spec:
      replicas: 2
      alertmanagerConfigSelector:
        matchLabels:
          alertmanagerConfig: switch

Zatim je stvoren secret pod imenom alertmanager-alertmanager-switch-generated, te mijenjamo alertmanager da selektira njega:

    apiVersion: monitoring.coreos.com/v1
    kind: Alertmanager
    metadata:
      name: alertmanager-switch
      namespace: js-prom
    spec:
      replicas: 2
    #  alertmanagerConfigSelector:
    #    matchLabels:
    #      alertmanagerConfig: switch
      configSecret: alertmanager-alertmanager-switch-generated

Zatim radimo novi Argo Source webhook:

    apiVersion: argoproj.io/v1alpha1
    kind: EventSource
    metadata:
      name: webhook-switch
      namespace: argo-events
    spec:
      eventBusName: default
      service:
        ports:
          - port: 12001
            targetPort: 12001
      webhook:
        high:
          endpoint: /high
          method: POST
          port: "12001"
        low:
          endpoint: /low
          method: POST
          port: "12001"

Koristili smo stari Event Bus iz prethodnog zadatka pod imenom default. Svi alertovi imaju svoje zasebno ime pa neće biti konflikta.

Također moramo exposati ovaj service pomoću:

    $ oc expose service/webhook-switch-eventsource-svc

Sada radimo dva senzora, prvi će respondati na event low, drugi na high

    apiVersion: argoproj.io/v1alpha1
    kind: Sensor
    metadata:
      name: switch0
      namespace: argo-events
      labels:
        example: "true"
    spec:
      dependencies:
        - name: dependency-1
          eventSourceName: webhook-switch
          eventName: low
      triggers:
        - template:
            name: workflow-trigger-low
            k8s:
              group: argoproj.io
              version: v1alpha1
              resource: workflows
              operation: create
              source:
                resource:
                  apiVersion: argoproj.io/v1alpha1
                  kind: Workflow
                  metadata:
                    generateName: workflow-from-sensor-
                  spec:
                    entrypoint: main
                    templates:
                      - name: main
                        container:
                          image: alpine:3.18
                          command: [sh, -c]
                          args:
                            - >-
                              kubectl patch deployment js-prom-deploy -n js-prom
                              --type=merge
                              -p={"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"worker0.op1os.lan.croz.net"}}}}}
    
    ---
    apiVersion: argoproj.io/v1alpha1
    kind: Sensor
    metadata:
      name: switch1
      namespace: argo-events
      labels:
        example: "true"
    spec:
      dependencies:
        - name: dependency-1
          eventSourceName: webhook-switch
          eventName: high
      triggers:
        - template:
            name: workflow-trigger-high
            k8s:
              group: argoproj.io
              version: v1alpha1
              resource: workflows
              operation: create
              source:
                resource:
                  apiVersion: argoproj.io/v1alpha1
                  kind: Workflow
                  metadata:
                    generateName: workflow-from-sensor-
                  spec:
                    entrypoint: main
                    templates:
                      - name: main
                        container:
                          image: bitnami/kubectl
                          command: [sh, -c]
                          args:
                            - >-
                              kubectl patch deployment js-prom-deploy -n js-prom
                              --type=merge
                              -p={"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"worker1.op1os.lan.croz.net"}}}}}

Jako je bitno odabrati sliku bitnami/kubectl jer ona ima kubectl komandu.

Također moramo dopustiti novom podu da čita i writea na deploymente:

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
      - apiGroups: ["apps"]                     # <-- add this block
        resources: ["deployments"]
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
        namespace: js-prom
      - kind: ServiceAccount
        name: default
        namespace: argo-events

To je to:)
    