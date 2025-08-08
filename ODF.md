#### Zadatak

Potrebno je iskoristiti deveti laptop za ceph storage cluster koji će služiti kao Persistent Volume za prvi i drugi cluster koji će se preko ODF operatora spajati na Ceph.

---
## Instalacija Ceph clustera

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

S komandom 'lsblk' dobijemo ime novog SSD-a i pozovemo komandu za stvaranje OSD-a:

    $ sudo ceph orch daemon add osd storage.opos.lan.croz.net:/dev/nvme0n1


Također, smanjli smo minimalnu velicinu osd_pool_default_size na 1 s komandom:

    $ ceph config set global osd_pool_default_size 1

Trenutno stanje je ovakvo:

```
ceph -s
  cluster:
    id:     d6e665a8-7392-11f0-9742-3c18a057a79b
    health: HEALTH_WARN
            1 stray host(s) with 1 daemon(s) not managed by cephadm
            1 pool(s) have no replicas configured
 
  services:
    mon: 1 daemons, quorum storage (age 25m)
    mgr: storage.rkbwdt(active, since 25m), standbys: storage.fdimse
    osd: 1 osds: 1 up (since 8m), 1 in (since 8m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 577 KiB
    usage:   8.6 MiB used, 954 GiB / 954 GiB avail
    pgs:     1 active+clean
```

HEALTH_WARN zanemajujemo zbog toga što imamo samo jedan disk pa neće biti duplikata.

Sada kreiramo novi pool imena RBD s:

    $ sudo ceph osd pool create rbd

    $ sudo ceph osd pool application enable rbd rbd

Sada možemo vidjeti sa 

    $ ceph osd pool ls

listu naših poolova:

    .mgr
    rbd

.mgr je unutarnji pool koji se automatski kreira i njega ne koristimo, a rbd je naš pool u kojem možemo definirati naše "virtualne diskove" koje možemo dodjeljivati aplikacijama na korištenje.

Npr. možemo stvoriti dva virtualna diska s komandama:

```
$ sudo rbd create mysql --size 1G

$ sudo rbd create mongodb --size 2G
```
I onda izlistamo diskove unutar poola:

    $ sudo rbd list

i možemo vidjeti:

    mysql
    mongodb

To je samo primjer, a sada ćemo stvoriti 2 poola za naše clustere:

```
$ sudo ceph osd pool create cluster1
$ sudo ceph osd pool application enable cluster1 rbd

$ sudo ceph osd pool create cluster2
$ sudo ceph osd pool application enable cluster2 rbd
```
## ODF
Sada ćemo skinuti ODF operator s operatorHub-a te ćemo u menu dobiti novu opciju Storage->Data Foundation->StorageSystem->Create StorageSystem.

Odaberemo 'Full development' i opciju 'External'.

Zatim ćemo dobiti python skriptu koju ćemo morati pokrenuti na laptopu s Ceph clusterom. Skinemo je i prebacimo na Ceph laptop:

    $ scp ceph-external-cluster-details-exporter.py storage@10.0.16.26:~

Zatim pozovemo:

```
$ python3 ceph-external-cluster-details-exporter.py --rbd-data-pool-name cluster1
$ python3 ceph-external-cluster-details-exporter.py --rbd-data-pool-name cluster2
```

te dobivamo odgovarajući JSON metadata kojeg lijepimo u StorageSystem creator.

