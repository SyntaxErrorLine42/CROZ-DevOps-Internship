# Platform: external

Kod OpenShift instalacije imamo izbor za platformu. Neke od platforma su: GCP, AZURE, NONE, BAREMETAL...
Baremetal i none koristimo kada smo sami zadužni za mašine na kojima će se vrtiti server (baremetal kada želimo virtualne IP adrese, none kada želimo loadbalancer). Pitanje je koja je razlika između odabiranja nekog cloud providera i externala. Ako odaberemo npr platform: gcp i pozovemo

    $ openshift-install create cluster

openshift-installer ce primjetiti platform gcp i provjeriti tvoj terminal session jesi li prijavljen na svoj gcloud te ako jesi, gcloud će preuzeti instalaciju openshift clustera. Google cloud će se pobrinuti za apsolutno sve. Stvorit će se manisfests i ignition fileovi s obzirom na install-config.yaml te mi nećemo trebati postavljati IP adrese, DNS servere, loadbalancere... Dakle, pristup platform: <ime_cloud_providera> je praktično out of the box rješenje.

Čemu služi external? External je uglavnom ista stvar kao i prijašnja platforma samo što mi kao admini clustera imamo više kontrole nad instalacijom. Kada openshift-install naiđe na platform: external, on zna da će neki cloud bit zadužan za održavanje servera, tj. mi netrebamo pružati fizičke mašine. Ovdje ne pozivamo instalaciju već:

    $ openshift-install create manifests
    $ openshift-install create ignition-configs

Dalje kada dobijemo manifests i ignition fileove, mi ih pružamo GCP-u preko gcloud komande (ovo je samo dan primjer za GCP, svaki drugi cloud ima svoje terminal komande ali uglavnom je ista schema). Dakle, mi u cloudu postavljamo sve sami, zadajemo IP adrese, LB, DNS servere, volume mount point...

Ako koristimo external: {}, openshift-installer apsolutno ne zna za koji cloud radi fileove tako da smo mi dužni pripremiti sve.
Ali ako koristimo nešto kao:

    platform:
        external:
            platformName: <ime_cloud_providera> 
            cloudClusterManager: external

onda OpenShift installer ipak zna o kojem cloudu je riječ i tu mu možemo provideati neke aspekte clustera koje ne želimo konfigurirati sami (to je specifično za svaki cloud). CCM možemo postaviti na external, time signaliziramo installeru da je ipak u pitanju vanjski kontroler clustera.

### installer/pkg/types/external/platform.go

    package external

    // CloudControllerManager describes the type of cloud controller manager to be enabled.
    type CloudControllerManager string

    const (
        // CloudControllerManagerTypeExternal specifies that an external cloud provider is to be configured.
        CloudControllerManagerTypeExternal = "External"

        // CloudControllerManagerTypeNone specifies that no cloud provider is to be configured.
        CloudControllerManagerTypeNone = ""
    )

    // Platform stores configuration related to external cloud providers.
    type Platform struct {
        // PlatformName holds the arbitrary string representing the infrastructure provider name, expected to be set at the installation time.
        // This field is solely for informational and reporting purposes and is not expected to be used for decision-making.
        // +kubebuilder:default:="Unknown"
        // +default="Unknown"
        // +kubebuilder:validation:XValidation:rule="oldSelf == 'Unknown' || self == oldSelf",message="platform name cannot be changed once set"
        // +optional
        PlatformName string `json:"platformName,omitempty"`

        // CloudControllerManager when set to external, this property will enable an external cloud provider.
        // +kubebuilder:default:=""
        // +default=""
        // +kubebuilder:validation:Enum="";External
        // +optional
        CloudControllerManager CloudControllerManager `json:"cloudControllerManager,omitempty"`
    }

Ukratko: za platform: <ime_cloud_providera> cloud radi sve umjesto nas (IPI), za external: {} mi radimo sve umjesto clouda te cloud služi samo kao zamjena za naše fizičke mašine (UPI), a također u external: možemo dodati i u kojem cloudu stvaramo cluster te aspekte koje može automatski predati na odgovornost clouda. Također jedna korist od platforme external je da možemo koristiti manje poznati cloud koji nije automatski podržavan u OpenShiftu.

## Reference: 
https://issues.redhat.com/browse/OCPSTRAT-516

https://github.com/openshift/enhancements/blob/master/enhancements/cloud-integration/infrastructure-external-platform-type.md

https://github.com/openshift/installer/blob/main/pkg/types/external/platform.go

https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_on_gcp/installing-gcp-user-infra

https://docs.providers.openshift.org/platform-external/

https://docs.okd.io/4.18/installing/installing_oci/installing-oci-agent-based-installer.html