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

Prije primjene potrebno je napraviti odgovarajuće realm, realm zone i realm zone group instance. Bitno je da zonegroup bude master jer će se inače period buniti da nema master zonegroup:

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

Ceph JAKO ne voli biti na single node clusteru, i defualt postavke su za minimalno 2 node-a (size 2). Te defualt postavke zajedno sa veličinom PGova po poolu je potrebno smanjiti na minimum jer će se Ceph inače preplaviti PGovima kod spajanja na ODF:
```
#!/bin/bash

# Omogući postavljanje pool size na 1
sudo ceph config set mon mon_allow_pool_size_one true
sudo ceph config get mon mon_allow_pool_size_one

# Dobavi listu svih poolova
pools=$(sudo ceph osd pool ls)

for pool in $pools; do
    echo "Processing pool: $pool"

    # Isključi autoscaler odmah na početku
    sudo ceph osd pool set "$pool" pg_autoscale_mode off

    # Postavljanje replica size i min_size na 1
    sudo ceph osd pool set "$pool" size 1 --yes-i-really-mean-it
    sudo ceph osd pool set "$pool" min_size 1

    # Postavljanje pg_num i pgp_num na 8
    sudo ceph osd pool set "$pool" pg_num 8
    sudo ceph osd pool set "$pool" pgp_num 8
done
```
Nakon ove skripte se Ceph više neće žaliti na *degraded data redundancy*, već će se samo žaliti na *no replication configured*, te se neće događati kvar zbog previše PGova po OSDu jer se broj PGova na svakom poolu smanjuje sa 32 na 8 (maksimum po OSDu je 250). Jako je bitno i da je *pg_autoscale_mode* ***off*** jer će se PGovi inače automatski skalirati i doći će do PG overheada.

Kao provjera može se pozvati:
```
ceph orch ls
ceph orch ps --daemon_type=rgw
```

### Testiranje S3

Potrebno je instalirati i podesiti AWS CLI:

```
sudo dnf install awscli -y
aws --version
```

Zatim se konfigurira CLI sa pristupnim ključevima:

```
aws configure
# AWS Access Key ID [None]: <your-access-key>
# AWS Secret Access Key [None]: <your-secret-key>
# Default region name [None]: us-east-1
# Default output format [None]: json
```
Ovi se pristupni podaci dobiju iz *radosgw-admin user info* naredbe, za pristup se koristi korisnik *dashboard* (može se koristiti bilo koji korisnik), a pristupni ključevi su u dijelu *keys*, region polje treba biti us-east-1 će RGW inače odbiti pristup:
```
[storage@storage ~]$ sudo radosgw-admin user info --uid=dashboard
{
    "user_id": "dashboard",
    "display_name": "Ceph Dashboard",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "dashboard",
            "access_key": "3TI3WLMUTPS0MGKRBYNE",
            "secret_key": "dvOaw1fMe2rwHq5DHXvoc6vI9jicCi9AezQmGFwI",
            "active": true,
            "create_date": "2025-08-20T13:35:02.120554Z"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "system": true,
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": [],
    "account_id": "",
    "path": "/",
    "create_date": "2025-08-20T13:35:02.120484Z",
    "tags": [],
    "group_ids": []
}
```

Kada smo konfigurirali AWS pristup, možemo kreirati bucket za testiranje:

```
aws --endpoint-url http://storage.opos.lan.croz.net s3 mb s3://test-bucket
aws --endpoint-url http://storage.opos.lan.croz.net s3 ls
```

I provjeriti radi li S3 ispravno:
```
$ echo "hello ceph" > test.txt
$ aws --endpoint-url http://storage.opos.lan.croz.net s3 cp test.txt s3://test-bucket/
2025-08-21 08:49:08 test-bucket
$ aws --endpoint-url http://storage.opos.lan.croz.net s3 ls s3://test-bucket/
2025-08-21 08:57:36         11 test.txt
$ aws --endpoint-url http://storage.opos.lan.croz.net s3 cp s3://test-bucket/test.txt downloaded.txt
download: s3://test-bucket/test.txt to ./downloaded.txt 
$ cat downloaded.txt 
hello ceph
```

*NAPOMENA: Domena storage.opos.lan.croz.net odnosi se na storage računalo na kojem se nalazi i haproxy za cluster 1, zbog kojeg je default http port 80 rezerviran. Da bi se to zaobišlo haproxy konfiguracija je promijenjena dodatkom ACL pravila koji zahtjeve sa storage.opos.lan.croz.net hostname prosljeđuje na RGW poslužitelj na portu 3333, a sve ostale zahtjeve prosljeđuje normalno prema cluster 1.


## ODF
Sada ćemo skinuti ODF operator s operatorHub-a te ćemo u menu dobiti novu opciju Storage->Data Foundation->StorageSystem->Create StorageSystem.

Odaberemo 'Full development' i opciju 'External'.

Zatim ćemo dobiti python skriptu koju ćemo morati pokrenuti na laptopu s Ceph clusterom. Skinemo je i prebacimo na Ceph laptop:

    $ scp ceph-external-cluster-details-exporter.py storage@10.0.16.26:~

Na Cephu kreiramo nove usere, po jedan za svaki cluster, sa svim dozvolama:

```
ceph auth add client.odf.cluster1user mon 'allow *' osd 'allow *' mgr 'allow *'
ceph auth add client.odf.cluster2user mon 'allow *' osd 'allow *' mgr 'allow *'
```

Zatim pozovemo python skriptu koja za parametre treba ime RBD data poola, RGW endpoint i usera:

```
python3 ceph-external-cluster-details-exporter.py --rbd-data-pool-name cluster2 --rgw-endpoint 10.0.16.26:3333 --run-as-user client.odf.cluster2user
python3 ceph-external-cluster-details-exporter.py --rbd-data-pool-name cluster1 --rgw-endpoint 10.0.16.26:3333 --run-as-user client.odf.cluster1user
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