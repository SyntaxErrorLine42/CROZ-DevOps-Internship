# OpenShift Cluster Instalacija

## DNS konfiguracija

### **CLUSTER 1**

| Uloga     | DNS                                 | IP ADRESA |
|-----------|-------------------------------------|-----------|
| Master 0  | master0.op1os.lan.croz.net          |10.0.16.18 |
| Worker 0  | worker0.op1os.lan.croz.net          |10.0.16.19 |
| Worker 1  | worker1.op1os.lan.croz.net          |10.0.16.20 |

---

### **CLUSTER 2**

| Uloga     | DNS                                 | IP ADRESA |
|-----------|-------------------------------------|-----------|
| Master 0  | master0.op2os.lan.croz.net          |10.0.16.21 |
| Master 1  | master1.op2os.lan.croz.net          |10.0.16.22 |
| Master 2  | master2.op2os.lan.croz.net          |10.0.16.23 |
| Worker 0  | worker0.op2os.lan.croz.net          |10.0.16.24 |
| Worker 1  | worker1.op2os.lan.croz.net          |10.0.16.25 |

---

### **STORAGE**

| Uloga     | DNS                                 | IP ADRESA |
|-----------|-------------------------------------|-----------|
| Storage   | storage.opos.lan.croz.net           |10.0.16.26 |

---

### **KUBERNETES API & ROUTES**

| Opis                              | Vrijednost                          | IP ADRESA |
|-----------------------------------|-------------------------------------|-----------|
| Kubernetes API (Cluster 1)        | api.op1os.lan.croz.net              |10.0.16.26 |
| Interni Kubernetes API (Cluster 1)| api-int.op1os.lan.croz.net          |10.0.16.26 |
| Routes (Cluster 1)                | *.apps.op1os.lan.croz.net           |10.0.16.26 |
| Kubernetes API (Cluster 2)        | api.op2os.lan.croz.net              |10.0.16.29 |
| Interni Kubernetes API (Cluster 2)| api-int.op2os.lan.croz.net          |10.0.16.29 |
| Routes (Cluster 2)                | *.apps.op2os.lan.croz.net           |10.0.16.30 |


## Preuzimanje alata

Preuzeti su sljedeći alati sa [OpenShift downloads](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/:)

    $ openshift-install

    $ openshift-baremetal-install

    $ oc CLI alat

Instalacija je pokrenuta lokalno sa pripremljenog direktorija (agent-installer).

## Generiranje SSH ključa

Potrebno je generirati SSH ključ za pristup klasteru    :

    $ ssh-keygen -t ed25519 -N '' -f <path>/<file_name>


## Agent-based bare-metal instalacija za Cluster 2
### install-config.yaml

Ova datoteka definira osnovne parametre za instalaciju OpenShift klastera
- **pullSecret** se preuzima s https://cloud.redhat.com/openshift/install/pull-secret
- **sshKey** sadrži ključ koji je ranije generiran

Install-config:

    apiVersion: v1
    baseDomain: lan.croz.net
    compute: 
    - name: worker
    replicas: 2
    architecture: amd64
    controlPlane: 
    name: master
    replicas: 3
    architecture: amd64
    metadata:
    name: op2os 
    networking:
    clusterNetwork:
        - cidr: 10.128.0.0/14
        hostPrefix: 23
    machineNetwork:
        - cidr: 10.0.16.0/23
    serviceNetwork:
        - 172.22.0.0/16
    networkType: OVNKubernetes
    platform:
    baremetal:
        apiVIPs:
        - 10.0.16.29
        ingressVIPs:
        - 10.0.16.30
    fips: false 
    pullSecret: 'Pull secret' 
    sshKey: 'ssh-rsa ... op2os' 


### agent-config.yaml

Ova datoteka definira fizičke čvorove koji sudjeluju u instalaciji.
- **rendezvousIP** je adresa jednog od master čvorova koji služi za koordinaciju tijekom instalacije

Agent-config:

    apiVersion: v1alpha1
    kind: AgentConfig
    rendezvousIP: 10.0.16.21
    hosts:
    - hostname: master0
        role: master
        interfaces:
        - name: enp0s13f0u2
        macAddress: 60:7D:09:37:5F:D4
        networkConfig:
        interfaces:
            - name: enp0s13f0u2
            type: ethernet
            state: up
            mac-address: 60:7D:09:37:5F:D4
            ipv4:
                enabled: true
                address:
                - ip: 10.0.16.21
                    prefix-length: 23
                dhcp: false
            ipv6:
                enabled: false
        dns-resolver:
            config:
            server:
                - 10.0.10.2
                - 10.0.10.3
        routes:
            config:
            - destination: 0.0.0.0/0
                next-hop-address: 10.0.16.1
                next-hop-interface: enp0s13f0u2
                table-id: 254

    - hostname: master1
        role: master
        interfaces:
        - name: enp0s13f0u2
        macAddress: 60:6D:3C:ED:91:50
        networkConfig:
        interfaces:
            - name: enp0s13f0u2
            type: ethernet
            state: up
            mac-address: 60:6D:3C:ED:91:50
            ipv4:
                enabled: true
                address:
                - ip: 10.0.16.22
                    prefix-length: 23
                dhcp: false
            ipv6:
                enabled: false
        dns-resolver:
            config:
            server:
                - 10.0.10.2
                - 10.0.10.3
        routes:
            config:
            - destination: 0.0.0.0/0
                next-hop-address: 10.0.16.1
                next-hop-interface: enp0s13f0u2
                table-id: 254

    - hostname: master2
        role: master
        interfaces:
        - name: enp0s13f0u2
        macAddress: 60:6D:3C:ED:90:C8
        networkConfig:
        interfaces:
            - name: enp0s13f0u2
            type: ethernet
            state: up
            mac-address: 60:6D:3C:ED:90:C8
            ipv4:
                enabled: true
                address:
                - ip: 10.0.16.23
                    prefix-length: 23
                dhcp: false
            ipv6:
                enabled: false
        dns-resolver:
            config:
            server:
                - 10.0.10.2
                - 10.0.10.3
        routes:
            config:
            - destination: 0.0.0.0/0
                next-hop-address: 10.0.16.1
                next-hop-interface: enp0s13f0u2
                table-id: 254

    - hostname: worker0
        role: worker
        interfaces:
        - name: enp0s13f0u2
        macAddress: 60:6D:3C:ED:58:82
        networkConfig:
        interfaces:
            - name: enp0s13f0u2
            type: ethernet
            state: up
            mac-address: 60:6D:3C:ED:58:82
            ipv4:
                enabled: true
                address:
                - ip: 10.0.16.24
                    prefix-length: 23
                dhcp: false
            ipv6:
                enabled: false
        dns-resolver:
            config:
            server:
                - 10.0.10.2
                - 10.0.10.3
        routes:
            config:
            - destination: 0.0.0.0/0
                next-hop-address: 10.0.16.1
                next-hop-interface: enp0s13f0u2
                table-id: 254

    - hostname: worker1
        role: worker
        interfaces:
        - name: enp0s13f0u2
        macAddress: 60:7D:09:37:5F:B0
        networkConfig:
        interfaces:
            - name: enp0s13f0u2
            type: ethernet
            state: up
            mac-address: 60:7D:09:37:5F:B0
            ipv4:
                enabled: true
                address:
                - ip: 10.0.16.25
                    prefix-length: 23
                dhcp: false
            ipv6:
                enabled: false
        dns-resolver:
            config:
            server:
                - 10.0.10.2
                - 10.0.10.3
        routes:
            config:
            - destination: 0.0.0.0/0
                next-hop-address: 10.0.16.1
                next-hop-interface: enp0s13f0u2
                table-id: 254

### Kreiranje Agent ISO image-a

Naredba:

    ./openshift-baremetal-install --dir install-dir agent create image

generira ove datoteke:
- **agent.x86_64.iso**
- **auth** direktorij sa pristupnim podacima

### Podizanje čvorova

Svi čvorovi se podižu preko istog **agent.x86_64.iso** koji sadrži sve potrebne podatke i agent instalaciju.

Instalacija se može pratiti preko CLI:

    ./openshift-install --dir install-dir wait-for install-complete --log-level=info

## Pristup klasteru nakon instalacije

Postavljanje **kubeconfig**:
    
    export KUBECONFIG=install-dir/auth/kubeconfig

Provjera klastera:

    $ oc get nodes
    $ oc get co
    $ oc get pods -A

Pristup OpenShift Web Console:

- URL: https://console-openshift-console.apps.my-cluster.example.com
- Korisničko ime: **kubeadmin**
- Lozinka: sadržaj datoteke **install-dir/auth/kubeadmin-password**

### Instalacija na platformi none

Klaster s jednim masterom i dva workera se ne može podići na baremetal platformi zbog ograničenja openshift-install komande, stoga je platforma promjenjena na none. Razlika platformi none i baremetal je u tome da baremetal s explicitno zadanim IP adresama stvara API i Ingress VIPs na koje se prima promet. U platform none se ne stvaraju virtualne IP adrese vec se čitav promet mapira na rendezvous IP, stoga je potreban LoadBalancer poput HaProxy-a.

## /etc/haproxy/haproxy.cfg

    global
      log         127.0.0.1 local2
    global
      log         127.0.0.1 local2
      pidfile     /var/run/haproxy.pid
      maxconn     4000
      daemon
    defaults
      mode                    http
      log                     global
      option                  dontlognull
      option http-server-close
      option                  redispatch
      retries                 3
      timeout http-request    10s
      timeout queue           1m
      timeout connect         10s
      timeout client          1m
      timeout server          1m
      timeout http-keep-alive 10s
      timeout check           10s
      maxconn                 3000

    listen api-server-6443
      bind 10.0.16.26:6443
      mode tcp
      option  httpchk GET /readyz HTTP/1.0
      option  log-health-checks
      balance roundrobin
      server master0 master0.op1os.lan.croz.net:6443 weight 1 verify none check check-ssl inter 10s fall 2 rise 3

    listen machine-config-server-22623
      bind 10.0.16.26:22623
      mode tcp
      server master0 master0.op1os.lan.croz.net:22623 check inter 1s

    listen ingress-router-443
      bind 10.0.16.26:443
      mode tcp
      balance source
      server worker0 worker0.op1os.lan.croz.net:443 check inter 1s
      server worker1 worker1.op1os.lan.croz.net:443 check inter 1s

    listen ingress-router-80
      bind 10.0.16.26:80
      mode tcp
      balance source
      server worker0 worker0.op1os.lan.croz.net:80 check inter 1s
      server worker1 worker1.op1os.lan.croz.net:80 check inter 1s

## agent-config.yaml
    apiVersion: v1alpha1
    kind: AgentConfig
    metadata:
    name: op1os
    rendezvousIP: 10.0.16.18
    hosts:
    - hostname: master0
        role: master
        interfaces:
        - name: enp0s13f0u2
            macAddress: 60:7d:09:37:5f:bb
        networkConfig:
        interfaces:
            - name: enp0s13f0u2
            type: ethernet
            state: up
            mac-address: 60:7d:09:37:5f:bb
            ipv4:
                enabled: true
                address:
                - ip: 10.0.16.18
                    prefix-length: 23
                dhcp: false
        dns-resolver:
            config:
            server:
                - 10.0.10.2
                - 10.0.10.3
        routes:
            config:
            - destination: 0.0.0.0/0
                next-hop-address: 10.0.16.1
                next-hop-interface: enp0s13f0u2
                table-id: 254

## install-config.yaml

    apiVersion: v1
    baseDomain: lan.croz.net
    compute:
    - architecture: amd64
        hyperthreading: Enabled
        name: worker
        replicas: 0
    controlPlane:
    architecture: amd64
    hyperthreading: Enabled
    name: master
    replicas: 1
    metadata:
    name: op1os
    networking:
    clusterNetwork:
        - cidr: 10.128.0.0/14
        hostPrefix: 23
    machineNetwork:
        - cidr: 10.0.16.0/23
    networkType: OVNKubernetes
    serviceNetwork:
        - 172.30.0.0/16
    platform:
        none:
    pullSecret: ...
    sshKey: ...

---
Ostatak instalacije prati instalaciju klastera na baremetal platformi.
---
### HaProxy

Nakon što smo namjestili /etc/haproxy/haproxy.cfg, pozvali smo komandu: 

    $ sudo haproxy -c -f /etc/haproxy/haproxy.cfg

Dobivamo potvrdu:

    Configuration file is valid

te pozivamo:

    $ sudo haproxy -f /etc/haproxy/haproxy.cfg

Stanje s portovima vidimo s 

    $ sudo ss -ltnp | grep haproxy

i dobivamo ispis 

    LISTEN 0      3000      10.0.16.26:80         0.0.0.0:*    users:(("haproxy",pid=32800,fd=7))                        
    LISTEN 0      3000      10.0.16.26:443        0.0.0.0:*    users:(("haproxy",pid=32800,fd=6))                        
    LISTEN 0      3000      10.0.16.26:22623      0.0.0.0:*    users:(("haproxy",pid=32800,fd=5))                        
    LISTEN 0      3000      10.0.16.26:6443       0.0.0.0:*    users:(("haproxy",pid=32800,fd=4))
