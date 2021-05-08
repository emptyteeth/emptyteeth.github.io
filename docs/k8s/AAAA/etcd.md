# etcd

## 目录结构

```shell
etcdctl get / --prefix --keys-only
etcdhelper ls

/registry/apiextensions.k8s.io/customresourcedefinitions/ingressroutes.traefik.containo.us
/registry/apiregistration.k8s.io/apiservices/v1.apps
/registry/clusterrolebindings/cluster-admin
/registry/clusterroles/cluster-admin
/registry/configmaps/kube-system/coredns
/registry/controllerrevisions/kube-system/kube-proxy-554798858b
/registry/controllerrevisions/kube-system/kube-proxy-5cbdd7f5df
/registry/csinodes/master02
/registry/csinodes/master03
/registry/daemonsets/kube-system/kube-proxy
/registry/deployments/kube-system/coredns
/registry/endpointslices/kube-system/metrics-server-wr874
/registry/horizontalpodautoscalers/default/echo
/registry/ingress/prometheus/prometheus-server
/registry/ingressclasses/traefik
/registry/leases/kube-node-lease/master02
/registry/leases/kube-node-lease/master03
/registry/leases/kube-system/kube-controller-manager
/registry/leases/kube-system/kube-scheduler
/registry/masterleases/10.11.1.22
/registry/minions/master02
/registry/minions/master03
/registry/namespaces/default
/registry/namespaces/kube-node-lease
/registry/namespaces/kube-public
/registry/namespaces/kube-system
/registry/persistentvolumeclaims/cicd/gitea-data-gitea-0
/registry/persistentvolumes/pvc-3d6a2c38-5115-485a-8b88-dd0d7513ae1f
/registry/poddisruptionbudgets/kube-system/metrics-server
/registry/pods/kube-system/coredns-f9fd979d6-bdwnc
/registry/podsecuritypolicy/psp.flannel.unprivileged
/registry/priorityclasses/system-cluster-critical
/registry/priorityclasses/system-node-critical
/registry/ranges/serviceips
/registry/ranges/servicenodeports
/registry/replicasets/kube-system/coredns-65955cb4d4
/registry/replicasets/kube-system/coredns-66bff467f8
/registry/rolebindings/kube-system/kube-proxy
/registry/roles/kube-system/kube-proxy
/registry/secrets/kube-system/coredns-token-s7sp2
/registry/serviceaccounts/kube-system/coredns
/registry/services/endpoints/default/kubernetes
/registry/services/specs/default/kubernetes
/registry/statefulsets/default/postgresql
/registry/storageclasses/glusterfs
/registry/traefik.containo.us/ingressroutes/traefik/authelia
/registry/traefik.containo.us/middlewares/traefik/authelia
compact_rev_key
```

```shell
etcdhelper dump

  {
    "key": "/registry/namespaces/default",
    "value": "{\"kind\":\"Namespace\",\"apiVersion\":\"v1\",\"metadata\":{\"name\":\"default\",\"uid\":\"87be29f2-0050-4f6f-a036-7ce3ec5af191\",\"creationTimestamp\":\"2020-07-10T21:06:53Z\",\"managedFields\":[{\"manager\":\"kube-apiserver\",\"operation\":\"Update\",\"apiVersion\":\"v1\",\"time\":\"2020-07-10T21:06:53Z\",\"fieldsType\":\"FieldsV1\",\"fieldsV1\":{\"f:status\":{\"f:phase\":{}}}}]},\"spec\":{\"finalizers\":[\"kubernetes\"]},\"status\":{\"phase\":\"Active\"}}\n",
    "create_revision": 152,
    "mod_revision": 152,
    "version": 1
  }

etcdhelper get /registry/namespaces/default
etcdctl get /registry/namespaces/default |auger decode

/v1, Kind=Namespace
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "default",
    "uid": "87be29f2-0050-4f6f-a036-7ce3ec5af191",
    "creationTimestamp": "2020-07-10T21:06:53Z",
    "managedFields": [
      {
        "manager": "kube-apiserver",
        "operation": "Update",
        "apiVersion": "v1",
        "time": "2020-07-10T21:06:53Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {"f:status":{"f:phase":{}}}
      }
    ]
  },
  "spec": {
    "finalizers": [
      "kubernetes"
    ]
  },
  "status": {
    "phase": "Active"
  }
```
