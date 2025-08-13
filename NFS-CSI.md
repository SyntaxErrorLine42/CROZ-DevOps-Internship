NFS CSI omogućuje clusteru da koristi NFS za provisioning persistent volumena. Requirement je imati postavljen NFS server, naš se nalazi na /nfs/exports na storage.opos.lan.croz.net (10.0.16.26/23). Instalacija NFS-CSI je jako jednostavna: https://github.com/kubernetes-csi/csi-driver-nfs/tree/master

## Instalacija
```
$ curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.11.0/deploy/install-driver.sh | bash -s v4.11.0 --
```

S ovim ćemo napraviti deploymente u namespaceu ```kube-system```:

```
NAME                                       READY   STATUS    RESTARTS   AGE     IP             NODE
csi-nfs-controller-56bfddd689-dh5tk       4/4     Running   0          35s     10.240.0.19    k8s-agentpool-22533604-0
csi-nfs-node-cvgbs                        3/3     Running   0          35s     10.240.0.35    k8s-agentpool-22533604-1
csi-nfs-node-dr4s4                        3/3     Running   0          35s     10.240.0.4     k8s-agentpool-22533604-0
```

## Setup

Zatim radimo storage class, napravimo file i applyamo ga:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.0.16.26
  share: /nfs/exports
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=4.1
```

Zatim stvaramo pvc file i applyamo ga:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi

```

Zatim koristimo jednostavni Pod koji koristi zadani pvc:
```
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test
spec:
  containers:
    - name: app
      image: busybox
      command: [ "sleep", "3600" ]
      volumeMounts:
        - name: nfs-vol
          mountPath: /data
  volumes:
    - name: nfs-vol
      persistentVolumeClaim:
        claimName: nfs-pvc

```

Ako odemo u pod s debugom vidimo da je ispravno mountan /data. Možemo npr. touchat test.txt i vidimo da se na storageu stvorila mapa `pvc-<id>` te se u njoj nalazi test.txt. Ako izbrišemo pod i ponovno ga applyamo, file će ostat sačuvan.

## Volume Snapshot

Prvo stvaramo VolumeSnapshotClass:

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: nfs-snapshot-class
driver: nfs.csi.k8s.io
deletionPolicy: Delete

```

Zatim radimo VolumeSnapshot:
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: nfs-snapshot
spec:
  volumeSnapshotClassName: nfs-snapshot-class
  source:
    persistentVolumeClaimName: nfs-pvc

```

I sada vidimo:
```
oc get vs
NAME           READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS        SNAPSHOTCONTENT
nfs-snapshot   true         nfs-pvc                             121           nfs-snapshot-class   snapcontent-418be3cb-8d8c-4ab7-9e6b-59c9df1e778b
```

Sada smo napravili snapshot našeg trenutnog stanja u PV-u. Sada možemo iskoristiti taj snapshot i napraviti novi PVC koji sadrži stanje prvog PVC-a. Napravimo novi pvc koji referencira taj snapshot:
```
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-restore
spec:
  containers:
    - name: app
      image: busybox
      command: [ "sleep", "3600" ]
      volumeMounts:
        - name: nfs-vol
          mountPath: /data
  volumes:
    - name: nfs-vol
      persistentVolumeClaim:
        claimName: nfs-pvc-restore
```

I sada kada debugamo u pod nfs-test-restore vidimo da se naš test.txt file nalazi i u novom PV-u.

Trenutno stanje na NFS serveru:
```
[root@api exports]# ls                                               
op1os  op2os  pvc-1b78bdd5-df08-4e1b-9adf-09a5282aeaf2  pvc-7bee8dc7-507a-40f2-a6fa-eddfde52c88d  snapshot-418be3cb-8d8c-4ab7-9e6b-59c9df1e778b
[root@api exports]# ls pvc-1b78bdd5-df08-4e1b-9adf-09a5282aeaf2/     
test
[root@api exports]# ls pvc-7bee8dc7-507a-40f2-a6fa-eddfde52c88d/     
test
[root@api exports]# ls snapshot-418be3cb-8d8c-4ab7-9e6b-59c9df1e778b/
pvc-7bee8dc7-507a-40f2-a6fa-eddfde52c88d.tar.gz
```