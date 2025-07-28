# Instalacija i spajanje NFS poslužitelja


## Instalacija i postavljanje NFS servera:

Instalacija:
```
sudo dnf install nfs-utils
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server
```
Izrada i i postavljanje prava pristupa za direktorij koji se poslužuje
```
sudo mkdir -p /srv/nfs/shared
sudo chown -R nobody:nobody /srv/nfs/shared
sudo chmod 755 /srv/nfs/shared
```

Izmjena /etc/exports da sadrži direktorij koji želimo dijeliti:

```
/srv/nfs/shared <ip_adresa_cidr>(rw,sync,no_subtree_check)
```

Primjena NFS podešavanja:

```
sudo exportfs -rav
sudo exportfs -v
```

Konfiguracija firewalla:

```
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
```

Omogućavanje pristupa SELinuxu ako je aktivan, provjera blokira li pristup:

```
sudo setsebool -P nfs_exports_all_rw 1
```

Testiranje pristupa sa klijenta:

```
sudo mount -t nfs <server_ip>:/srv/nfs/shared /mnt
```

<<<<<<< HEAD
## Spajanje na server iz klasteraj preko NFS provisionera
=======
## Spajanje na server iz klastera preko NFS provisionera
>>>>>>> 242f5e5 (Instalacija i postavljanje NFS)


Napraviti novi projekt samo za nfs i prebaciti se na njega

```
oc new-project nfs-provisoner
```

Dati pristup za volumene i odgovarajući SCC:

```
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-subdir-external-provisioner -n nfs-provisioner
oc adm policy add-scc-to-user anyuid -z nfs-subdir-external-provisioner -n nfs-provisioner

```

Napraviti novi helm repo i instalirati ga preko [uputa](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner):

```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/exported/path
```

Ovako postavljan provisioner radi samo u odabranom namespaceu.

