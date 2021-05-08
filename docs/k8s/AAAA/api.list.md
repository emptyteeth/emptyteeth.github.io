# api list

>1.18.3

## api versions

>kubectl api-versions

```text
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
crd.projectcalico.org/v1
discovery.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
metrics.k8s.io/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
traefik.containo.us/v1alpha1
v1
```

## api resources

### namespaced

>kubectl api-resources --namespaced=true -o wide

```text
NAME                        SHORTNAMES   APIGROUP                    NAMESPACED   KIND                       VERBS
bindings                                                             true         Binding                    [create]
configmaps                  cm                                       true         ConfigMap                  [create delete deletecollection get list patch update watch]
endpoints                   ep                                       true         Endpoints                  [create delete deletecollection get list patch update watch]
events                      ev                                       true         Event                      [create delete deletecollection get list patch update watch]
limitranges                 limits                                   true         LimitRange                 [create delete deletecollection get list patch update watch]
persistentvolumeclaims      pvc                                      true         PersistentVolumeClaim      [create delete deletecollection get list patch update watch]
pods                        po                                       true         Pod                        [create delete deletecollection get list patch update watch]
podtemplates                                                         true         PodTemplate                [create delete deletecollection get list patch update watch]
replicationcontrollers      rc                                       true         ReplicationController      [create delete deletecollection get list patch update watch]
resourcequotas              quota                                    true         ResourceQuota              [create delete deletecollection get list patch update watch]
secrets                                                              true         Secret                     [create delete deletecollection get list patch update watch]
serviceaccounts             sa                                       true         ServiceAccount             [create delete deletecollection get list patch update watch `impersonate`]
services                    svc                                      true         Service                    [create delete get list patch update watch]
controllerrevisions                      apps                        true         ControllerRevision         [create delete deletecollection get list patch update watch]
daemonsets                  ds           apps                        true         DaemonSet                  [create delete deletecollection get list patch update watch]
deployments                 deploy       apps                        true         Deployment                 [create delete deletecollection get list patch update watch]
replicasets                 rs           apps                        true         ReplicaSet                 [create delete deletecollection get list patch update watch]
statefulsets                sts          apps                        true         StatefulSet                [create delete deletecollection get list patch update watch]
localsubjectaccessreviews                authorization.k8s.io        true         LocalSubjectAccessReview   [create]
horizontalpodautoscalers    hpa          autoscaling                 true         HorizontalPodAutoscaler    [create delete deletecollection get list patch update watch]
cronjobs                    cj           batch                       true         CronJob                    [create delete deletecollection get list patch update watch]
jobs                                     batch                       true         Job                        [create delete deletecollection get list patch update watch]
leases                                   coordination.k8s.io         true         Lease                      [create delete deletecollection get list patch update watch]
endpointslices                           discovery.k8s.io            true         EndpointSlice              [create delete deletecollection get list patch update watch]
events                      ev           events.k8s.io               true         Event                      [create delete deletecollection get list patch update watch]
ingresses                   ing          extensions                  true         Ingress                    [create delete deletecollection get list patch update watch]
pods                                     metrics.k8s.io              true         PodMetrics                 [get list]
ingresses                   ing          networking.k8s.io           true         Ingress                    [create delete deletecollection get list patch update watch]
networkpolicies             netpol       networking.k8s.io           true         NetworkPolicy              [create delete deletecollection get list patch update watch]
poddisruptionbudgets        pdb          policy                      true         PodDisruptionBudget        [create delete deletecollection get list patch update watch]
rolebindings                             rbac.authorization.k8s.io   true         RoleBinding                [create delete deletecollection get list patch update watch]
roles                                    rbac.authorization.k8s.io   true         Role                       [create delete deletecollection get list patch update watch `escalate` `bind`]
```

---

### cluster wide

>kubectl api-resources --namespaced=false -o wide

```text
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND                             VERBS
componentstatuses                 cs                                          false        ComponentStatus                  [get list]
namespaces                        ns                                          false        Namespace                        [create delete get list patch update watch]
nodes                             no                                          false        Node                             [create delete deletecollection get list patch update watch]
persistentvolumes                 pv                                          false        PersistentVolume                 [create delete deletecollection get list patch update watch]
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration     [create delete deletecollection get list patch update watch]
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration   [create delete deletecollection get list patch update watch]
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition         [create delete deletecollection get list patch update watch]
apiservices                                    apiregistration.k8s.io         false        APIService                       [create delete deletecollection get list patch update watch]
tokenreviews                                   authentication.k8s.io          false        TokenReview                      [create]
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview          [create]
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview           [create]
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview              [create]
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest        [create delete deletecollection get list patch update watch]
nodes                                          metrics.k8s.io                 false        NodeMetrics                      [get list]
ingressclasses                                 networking.k8s.io              false        IngressClass                     [create delete deletecollection get list patch update watch]
runtimeclasses                                 node.k8s.io                    false        RuntimeClass                     [create delete deletecollection get list patch update watch]
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy                [create delete deletecollection get list patch update watch `use`]
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding               [create delete deletecollection get list patch update watch]
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole                      [create delete deletecollection get list patch update watch `escalate` `bind`]
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass                    [create delete deletecollection get list patch update watch]
csidrivers                                     storage.k8s.io                 false        CSIDriver                        [create delete deletecollection get list patch update watch]
csinodes                                       storage.k8s.io                 false        CSINode                          [create delete deletecollection get list patch update watch]
storageclasses                    sc           storage.k8s.io                 false        StorageClass                     [create delete deletecollection get list patch update watch]
volumeattachments                              storage.k8s.io                 false        VolumeAttachment                 [create delete deletecollection get list patch update watch]
```
