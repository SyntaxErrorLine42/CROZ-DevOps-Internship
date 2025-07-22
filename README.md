### **CLUSTER 1**

| Uloga     | DNS                                 | IP ADRESA |
|-----------|-------------------------------------|-----------|
| Master 0  | master0.op1os.lan.croz.net          |           |
| Worker 0  | worker0.op1os.lan.croz.net          |           |
| Worker 1  | worker1.op1os.lan.croz.net          |           |

---

### **CLUSTER 2**

| Uloga     | DNS                                 | IP ADRESA |
|-----------|-------------------------------------|-----------|
| Master 0  | master0.op2os.lan.croz.net          |           |
| Master 1  | master1.op2os.lan.croz.net          |           |
| Master 2  | master2.op2os.lan.croz.net          |           |
| Worker 0  | worker0.op2os.lan.croz.net          |           |
| Worker 1  | worker1.op2os.lan.croz.net          |           |

---

### **STORAGE**

| Uloga     | DNS                                 | IP ADRESA |
|-----------|-------------------------------------|-----------|
| Storage   | storage.opos.lan.croz.net           |           |

---

### **KUBERNETES API & ROUTES**

| Opis                              | Vrijednost                          | IP ADRESA |
|-----------------------------------|-------------------------------------|-----------|
| Kubernetes API (Cluster 1)        | api.op1os.lan.croz.net              |           |
| Interni Kubernetes API (Cluster 1)| api-int.op1os.lan.croz.net          |           |
| Routes (Cluster 1)                | *.apps.op1os.lan.croz.net           |           |
| Kubernetes API (Cluster 2)        | api.op2os.lan.croz.net              |           |
| Interni Kubernetes API (Cluster 2)| api-int.op2os.lan.croz.net          |           |
| Routes (Cluster 2)                | *.apps.op2os.lan.croz.net           |           |

