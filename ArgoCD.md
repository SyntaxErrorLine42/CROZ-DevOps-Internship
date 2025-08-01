# ArgoCD instalacija

Za početak unosimo ove dvije komande:

    $ kubectl create namespace argocd
    $ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

te i onda:

    $ oc expose svc/argocd-sample-server -n argocd

Dobijemo rute:

    argocd-sample-server-openshift-operators.apps.op1os.lan.croz.net

    argocd-sample-server-openshift-operators.apps.op2os.lan.croz.net

s kojim pristupamo na ArgoCD sučelje.

Spajamo naš git repo na kojem se nalaze deployment.yaml i service.yaml preko Settings -> Repositories -> Add repo te ispunimo sve potrebne informacije (link, credentials, Auto-Sync).

Nakon toga idemo na Applications -> New app, te ispunjavamo informacije od kojih su najbitniji odabiri cluster URL-a u kojem je deployana sama instanca argo CD operatora https://kubernetes.default.svc i našeg namespacea u kojem se nalazi naša aplikacija. Zatim stisnemo CREATE.

Nakon prvog synca će se pojaviti error u slučaju da je aplikacija u različitom namespacu od same instance argo CD-a. To se rješava ovako 

    oc label namespace <namespace_od_aplikacije> argocd.argoproj.io/managed-by=<namespace_od_operatora>

Ako smo posavili auto-sync sa self healingom i auto pruneom, svaki naš push na github bi se odmah trebao vidjeti u deploymentu aplikacije.