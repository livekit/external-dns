# Cluster Nodes as Source

This tutorial describes how to configure ExternalDNS to use the cluster nodes as source.
Using nodes (`--source=node`) as source is possible to synchronize a DNS zone with the nodes of a cluster.

The node source adds an `A` record per each node `externalIP` (if not found, any IPv4 `internalIP` is used instead).
It also adds an `AAAA` record per each node IPv6 `internalIP`.
The TTL of the records can be set with the `external-dns.alpha.kubernetes.io/ttl` node annotation.
The FQDN template provides more than 100+ functions, documented [here](https://go-task.github.io/slim-sprig/). For instance, it includes a function to replace all `.` with `-` in the node name, which can be useful with cloud providers that include dots in the node name. There are two additional functions available on top of the standard sprig functions: `isIPv4` and `isIPv6`. The functions can be used to test a string for being an IPv4 or IPv6 address.

Nodes marked as **Unschedulable** as per [core/v1/NodeSpec](https://pkg.go.dev/k8s.io/api@v0.31.1/core/v1#NodeSpec) are excluded.
This avoid exposing Unhealthy, NotReady or SchedulingDisabled (cordon) nodes.

## Manifest (for cluster without RBAC enabled)

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.15.1
        args:
        - --source=node # will use nodes as source
        - --provider=aws
        - --zone-name-filter=external-dns-test.my-org.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        - --domain-filter=external-dns-test.my-org.com
        - --aws-zone-type=public
        - --registry=txt
        - --fqdn-template={{.Name | replace "." "-"}}.external-dns-test.my-org.com
        - --txt-owner-id=my-identifier
        - --policy=sync
        - --log-level=debug
```

## Manifest (for cluster with RBAC enabled)

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: ["route.openshift.io"]
  resources: ["routes"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.15.1
        args:
        - --source=node # will use nodes as source
        - --provider=aws
        - --zone-name-filter=external-dns-test.my-org.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        - --domain-filter=external-dns-test.my-org.com
        - --aws-zone-type=public
        - --registry=txt
        - --fqdn-template={{.Name | replace "." "-"}}.external-dns-test.my-org.com
        - --txt-owner-id=my-identifier
        - --policy=sync
        - --log-level=debug
```
