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
| Kubernetes API (Cluster 1)        | api.op1os.lan.croz.net              |10.0.16.27 |
| Interni Kubernetes API (Cluster 1)| api-int.op1os.lan.croz.net          |10.0.16.27 |
| Routes (Cluster 1)                | *.apps.op1os.lan.croz.net           |10.0.16.28 |
| Kubernetes API (Cluster 2)        | api.op2os.lan.croz.net              |10.0.16.29 |
| Interni Kubernetes API (Cluster 2)| api-int.op2os.lan.croz.net          |10.0.16.29 |
| Routes (Cluster 2)                | *.apps.op2os.lan.croz.net           |10.0.16.30 |

