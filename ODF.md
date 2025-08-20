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

    $ sudo ceph orch daemon add osd <ime_hosta>:/dev/<ime_diska>



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

### Postavljanje RGW Servisa

RGW Servis služi kao API za upravljanje S3 podacima na Cephu.

Upute za [postavljanje sa service specifikacijom](https://docs.redhat.com/en/documentation/red_hat_ceph_storage/6/html/object_gateway_guide/deployment#deploying-the-ceph-object-gateway-using-the-service-specification_rgw), kreira se datoteka ***radosgw.yaml***:

```
service_type: rgw
service_id: default
placement:
  hosts:
  - storage
  count_per_host: 1
spec:
  rgw_realm: test_realm
  rgw_zone: test_zone
  rgw_zonegroup: test_zonegroup
  rgw_frontend_port: 3333
networks:
  -  10.0.16.0/23
```

Prije primjene provjeriti postoje li odgovarajući (*default*) realm, realm zone i realm zone group instance, pa ih kreirati ako ne postoje:

```
radosgw-admin realm create --rgw-realm=test_realm --default

radosgw-admin zonegroup create --rgw-zonegroup=test_zonegroup --rgw-realm=test_realm --default --master

radosgw-admin zone create --rgw-zonegroup=test_zonegroup --rgw-zone=test_zone --default

radosgw-admin period update --rgw-realm=test_realm --commit
```

Konačno, *radosgw.yaml* se mounta u storage host i deploya:
```
sudo cephadm shell --mount radosgw.yml:/var/lib/ceph/radosgw/radosgw.yml
[ceph: root@host01 /]# ceph orch apply -i /var/lib/ceph/radosgw/radosgw.yml
```

Kao provjera može se pozvati:
```
ceph orch ls
ceph orch ps --daemon_type=rgw
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

Zatim odabiremo 'CREATE' te pustimo da se instalira. Nakon instalacije valjanost storagea možemo vidjeti s komandom:

    $ oc get storagecluster -n openshift-storage

te vidimo:

```
NAME                          AGE     PHASE   EXTERNAL   CREATED AT             VERSION
ocs-external-storagecluster   6m38s   Ready   true       2025-08-08T10:13:15Z   4.19.0
```


## Dodatci:

U slučaju reinstalacije CEPH OSD-a potrebno je formatirati kompletni disk:
```
$ cephadm ceph-volume lvm zap --destroy /dev/<ime_diska>

$ wipefs -af /dev/<ime_diska>

$ cephadm ceph-volume lvm zap --destroy /dev/<ime_diska>

$ partprobe /dev/<ime_diska>

$ reboot
```

Zatim:

```
$ ceph fsid  # ovo kopiramo i stavimo u sljedeću naredbu
$ cephadm rm-cluster --force --zap-osds --fsid <fsid>
```

Također, kod deinstalacije systemStoragea (kind se zapravo zove systemCluster) potrebno je otići u yaml i promijeniti mode deinstalacije iz ```graceful``` u ```forced``` i također treba izbrisati sve unutar ```finalizers:``` polja.