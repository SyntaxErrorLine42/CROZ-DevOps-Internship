### Motivacija

U jednom od prethodnih zadataka smo radili premještaj podova sa jednog na drugi node. To je odlično kada npr previše resursa trošimo na jednom nodeu i želimo izbalansirati resurse. Sada nam je cilj ne samo prebaciti taj jedan pod, već sve podove izvaditi iz jednog nodea i isključiti node. Na primjer to je jako korisno kada ne želimo plaćati node da samo stoji idle nakon što prebacimo sve podove s jednog nodea na ostale. Dakle ovo radimo u slučaju kada ```$ oc drain <node>``` nije dovoljan, već želimo i izdvojiti node iz clustera.

### Preduvjeti

Za ostvarivanje remote power on opcije koristimo WakeOnLan protokol. On funckionira tako da se nodeu koji je u sleep/hibernate modeu pošalje ```Magic packet``` preko LAN-a koji ga upali. Za početak moramo provjeriti je li uopće dostupan WOL protokol za naš mrežni interface. Prvo moramo saznati koji je naš mrežni interface, poznajući našu ip adresu i mac adresu pozovemo ```$ ip a``` i identificiramo ime interfacea, što je u našem slučaju ```enp0s13f0u2```. Sada pozovemo tool ```ethtool``` koji je već instaliran na RHEL OS-ovima. Pozovimo:

```
$ ethtool enp0s13f0u2 | grep Wake

Wake-on: d
```

```d``` znači disabled, mi ga moramo namjestiti na ```g``` što znači da Wake On Lan reagira na magic pakete, to radimo komandom:
```
$ ethtool -s enp0s13f0u2 wol g
```
i provjerimo:
```
$ ethtool enp0s13f0u2 | grep Wake

Wake-on: g
```

Slijedeći korak je rebootati i otići u BIOS i otići u SETTINGS->NETWORK SETTINGS->WAKE ON LAN i odabrati ```ENABLE```.

Sada imamo sve uvijete za provesti WOL protokol. Sada možemo suspendati node s ```$ sudo systemctl suspend``` te odemo na bastion laptop i vidimo da je node ```not ready``` s komandom ```$ oc get nodes```.

Sada na laptopu NA ISTOJ MREŽI skinemo paket ```wakeonlan``` koji je dostupan i u apt repo te jednostavno pozovemo:
```
$ wakeonlan <MAC_ADRESA>
Sending magic packet to 255.255.255.255:9 with <MAC_ADRESA>
```
ispod jedne minute vidjet ćemo da je node ready.

### Zadatak

Želimo ugasiti jednog workera izvan radnog vremena, dakle uzet ćemo workera 1 i ugasiti izvan 9h-17h radnog vremena te će ostati samo worker 0.

Za to ćemo koristiti KEDA ScalingJob. Ideja će biti slična kao i premještaj podova.

Prva stvar koja nam treba je image koji sadržaje oboje `wakeonlan` i `kubectl` komande, to se jednostavno napravi sa sljedećim Dockerfileom kojeg pushamo na Docker:

```
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    curl \
    wget \
    iputils-ping \
    wakeonlan \
    && rm -rf /var/lib/apt/lists/*

RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    && install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl \
    && rm kubectl

CMD [ "bash" ]
```
Također trebamo napraviti Service Account sa cluster roleom jer samo tako može zapravo suspendati node:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-admin
  namespace: keda-test
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-admin-binding
subjects:
- kind: ServiceAccount
  name: node-admin
  namespace: keda-test
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io


```

Sada možemo narpaviti probni pod za testiranje komandi. Ideja je imati dva Joba, jedan će izvršavati `kubectl drain` i `oc debug` u node i onda `suspend`. Drugi će raditi `wakeonlan` i zatim `kubectl uncordon`. 
Problem: za slanje wakeonlan paketa MORAMO biti u istoj mreži, a podovi su po defaultu u vlastitoj izloranoj mreži, stoga je potrebno postaviti `hostNetwork: true`. WOL paket je UDP paket na port 9 koji se broadcasta na sve mac adrese i u payloadu sadrži ciljanu MAC adresu.

Napravimo prvo jednostavni test pod da provjerimo radi li ova ideja:
```
apiVersion: v1
kind: Pod
metadata:
  name: tester
  namespace: keda-test
spec:
  serviceAccountName: node-admin
  hostNetwork: true
  restartPolicy: Never
  containers:
    - name: wake-uncordon
      image: lvolarevic/kubewithwol:latest
      securityContext:
        privileged: true
      command: ["/bin/sh", "-c"]
      args:
      - |
        wakeonlan 60:7d:09:37:5f:b0
        kubectl uncordon worker1
```
Applyajmo ga i pogledajmo logove nakon completeona:
```
$ oc logs tester

Sending magic packet to 255.255.255.255:9 with <MAC_ADRESA>
node/worker1 already uncordoned
```
Odlično, možemo i na wiresharku ako filtriramo pakete po `udp.port = 9` vidjeti da se WOL šalje te se računalo budi iz suspenzije. Možemo pristupiti clusteru s kubectl jer dobiva kubeconfig context od clustera u kojem se nalazi i wol se može spojit jer nije na izloranoj mreži.

Idemo provjeriti i drugi Job, napravimo jedan tester također:
```
apiVersion: v1
kind: Pod
metadata:
  name: tester
  namespace: keda-test
spec:
  serviceAccountName: node-admin
  hostNetwork: true
  restartPolicy: Never
  containers:
    - name: wake-uncordon
      image: lvolarevic/kubewithwol:latest
      securityContext:
        privileged: true
      command: ["/bin/sh", "-c"]
      args:
      - |
        kubectl debug node/worker1 --profile='sysadmin' --image lvolarevic/kubewithwol:latest -- chroot /host systemctl suspend
        kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data --force

```
Jako je bitno da je profil sysadmin jer inaće u logovima vidimo da chroot /host nije permitted. Također kod korištenja kubectl komande potrebno je specificirati image koji se koristi u debug nodeu (to nije potrebno u oc komandi, ali mi ovdje koristimo kubectl).
Kada pogledamo logse od ovog testera vidmio:
```
$ oc logs tester

Creating debugging pod node-debugger-worker1-h97tg with container debugger on node worker1.
node/worker1 already cordoned
```
NOTE: koristio sam primjere gdje je node već cordoned i u prethodnom logs uncordoned radi smanjivanja texta, najbitnije je da se vidi da kubectl komanda radi.

Idemo sada dovršiti zadatak. Cilj nam je izvan radnog vremena ugasiti node, a za vrijeme radnog vremena upaliti. Već smo koristili `type: cron` u dokumentaciji KEDA.md tako da trebamo samo promijeniti trigger.

Radimo dva ScalingJob-a (Ovdje koristimo KEDU, mogli smo i običan CronJob, ispada isto na kraju):
```
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: cordon
  namespace: keda-test # Bitno radi service accounta
#  annotations:
#    autoscaling.keda.sh/paused: "true"
spec:
  jobTargetRef:
    template:
      spec:
        serviceAccountName: node-admin
        hostNetwork: true
        containers:
          - name: wake-uncordon
            image: lvolarevic/kubewithwol:latest
            securityContext:
              privileged: true
            command: ["/bin/sh", "-c"]
            args:
            - |
              kubectl debug node/worker1 --profile='sysadmin' --image lvolarevic/kubewithwol:latest -- chroot /host systemctl suspend
              kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data --force
        restartPolicy: Never
  pollingInterval: 30
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  maxReplicaCount: 1
  triggers:
  - type: cron
    metadata:
      start: "0 17 * * *"
      end: "1 17 * * *"
      timezone: "Europe/Zagreb"
      pollingInterval: "30"
      desiredReplicas: "1"
```
```
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: uncordon
  namespace: keda-test # Bitno radi service accounta
#  annotations:
#    autoscaling.keda.sh/paused: "true"
spec:
  jobTargetRef:
    template:
      spec:
        serviceAccountName: node-admin
        hostNetwork: true
        containers:
          - name: wake-uncordon
            image: lvolarevic/kubewithwol:latest
            securityContext:
              privileged: true
            command: ["/bin/sh", "-c"]
            args:
            - |
              wakeonlan 60:7d:09:37:5f:b0
              kubectl uncordon worker1
        restartPolicy: Never
  pollingInterval: 30
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  maxReplicaCount: 1
  triggers:
  - type: cron
    metadata:
      start: "0 9 * * *"
      end: "1 9 * * *"
      timezone: "Europe/Zagreb"
      pollingInterval: "30"
      desiredReplicas: "1"
```
EXTRA: Također možemo i namjestiti trigger kao prometheus metrics slično što smo radili u KEDA dokumentaciji koristeći jednostavnu JavaScript aplikaciju koja exposa svoje metrike. U tom slučaju Jobovi će izgledati ovako:
```
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: cordon
  namespace: keda-test # Bitno radi service accounta
  annotations:
    autoscaling.keda.sh/paused: "true"
spec:
  jobTargetRef:
    template:
      spec:
        serviceAccountName: node-admin
        hostNetwork: true
        containers:
          - name: wake-uncordon
            image: lvolarevic/kubewithwol:latest
            securityContext:
              privileged: true
            command: ["/bin/sh", "-c"]
            args:
            - |
              kubectl debug node/worker1 --profile='sysadmin' --image lvolarevic/kubewithwol:latest -- chroot /host systemctl suspend
              kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data --force
        restartPolicy: Never
  pollingInterval: 30
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  maxReplicaCount: 1
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-operated-prometheus.apps.op2os.lan.croz.net
      metricName: current_connections_over
      threshold: "1000"
      query: current_connections
```
```
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: uncordon
  namespace: keda-test # Bitno radi service account
  annotations:
    autoscaling.keda.sh/paused: "true"
spec:
  jobTargetRef:
    template:
      spec:
        serviceAccountName: node-admin
        hostNetwork: true
        containers:
          - name: wake-uncordon
            image: lvolarevic/kubewithwol:latest
            securityContext:
              privileged: true
            command: ["/bin/sh", "-c"]
            args:
            - |
              wakeonlan 60:7d:09:37:5f:b0
              kubectl uncordon worker1
        restartPolicy: Never
  pollingInterval: 30
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  maxReplicaCount: 1
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-operated-prometheus.apps.op2os.lan.croz.net
      metricName: current_connections_below
      threshold: "1"
      query: 'current_connections < 1000'
```
