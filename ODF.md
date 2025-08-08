#### Zadatak

Potrebno je iskoristiti deveti laptop za ceph storage cluster koji će služiti kao Persistent Volume za prvi i drugi cluster koji će se preko ODF operatora spajati na Ceph.

---
Za instalaciju single node Ceph clustera koristimo dnf paket cephadm jer je moderniji i bolje prilagođen za single node clustere. Pratimo korake s https://www.redhat.com/en/blog/ceph-cluster-single-machine. 

Prvo dodajemo najnoviji paket za skidanje cephadm:

    $ subscription-manager repos --enable=rhceph-8-tools-for-rhel-9-x86_64-rpm
    $ sudo dnf update

Zatim skidamo sve potrebne pakete:

    $ sudo dnf install podman cephadm ceph-common ceph-base -y

Instalaciju Ceph clustera pokrećemo s:

```
$ sudo cephadm bootstrap \
--cluster-network 10.0.16.0/23 \
--mon-ip 10.0.16.26 \
--registry-url registry.redhat.io \
--registry-username '<naš_username>' \
--registry-password '<naša_lozinka>' \
--dashboard-password-noupdate \
--initial-dashboard-user admin \
--initial-dashboard-password ceph \
--allow-fqdn-hostname \
--single-host-defaults
```

--single-host-defaults je tag koji je prilagođen za single node Ceph clustere i postavi ove specifikacije:

```
global/osd_crush_chooseleaf_type = 0
global/osd_pool_default_size = 2
mgr/mgr_standby_modules = False
```
One određuju da se replike stvaraju unatoč tome što će biti na istom hostu i da nema zamjenskih managera koji će preusmjerivati promet u cluster.

Sada možemo vidjeti stanje Ceph clustera sa -s tagom:

```
$ ceph -s
  cluster:
    id:     d6e665a8-7392-11f0-9742-3c18a057a79b
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 2
 
  services:
    mon: 1 daemons, quorum storage (age 17h)
    mgr: storage.rkbwdt(active, since 18h), standbys: storage.fdimse
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
```

Vidimo da dobivamo HEALTH_WARN zbog nedostatka OSD-a. OSD je Object Storage Daemon i njegova je uloga spremanje i upravljanje našim podatcima.

Da bi dodali OSD potreban nam je čisti SSD kojeg smo zatražili od internog IT-a.