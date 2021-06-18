# panoramaTest


Create a namespace
```
apiVersion: v1
kind: Namespace
metadata:
  name: panorama-integration
```

Create a configMap
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: fw-config
  namespace: panorama-integration
data:
  fw_hostname: ip-10-0-0-87.ec2.internal
  fw_devicegroup: CalicoEnterprise
```  

Create a secret
```
kind: Secret
apiVersion: v1
type: Opaque
metadata:
  name: fw-secret-config
  namespace: panorama-integration
data:
  fw_username: ----
  fw_password: ----
```

RBAC Config

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: firewall-integration-controller
  namespace: calico-monitoring
---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: firewall-integration-controller
rules:
  - apiGroups:
      - "projectcalico.org"
      - ""
    resources:
      - tiers
      - globalnetworkpolicies
      - tier.globalnetworkpolicies
      - pods
    verbs: ["*"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: firewall-integration-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: firewall-integration-controller
subjects:
- kind: ServiceAccount
  name: firewall-integration-controller
  namespace: calico-monitoring
---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: firewall-integration-controller
  namespace: calico-monitoring
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs: ["*"]
---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: firewall-integration-controller
  namespace: calico-monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: firewall-integration-controller
subjects:
- kind: ServiceAccount
  name: firewall-integration-controller
  namespace: calico-monitoring
```

Firewall Integration Manifest

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fw-integ
  namespace: calico-monitoring
  labels:
    k8s-app: firewall-integration-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: firewall-integration-controller
  template:
    metadata:
      labels:
        k8s-app: firewall-integration-controller
    spec:
      serviceAccountName: firewall-integration-controller
      imagePullSecrets:
      - name: cnx-pull-secret
      containers:
      - name: fw-integ
        image: quay.io/tigera/firewall-integration:v3.4.1-pre.2
        imagePullPolicy: Always
        command: ["/controller"]
        env:
        - name: FW_HOSTNAME
          valueFrom:
            configMapKeyRef:
              name: fw-config
              key: fw_hostname
        - name: FW_USERNAME
          valueFrom:
            secretKeyRef:
              name: fw-secret-config
              key: fw_username
        - name: FW_PASSWORD
          valueFrom:
            secretKeyRef:
              name: fw-secret-config
              key: fw_password
        - name: FW_DEVGROUP
          valueFrom:
            configMapKeyRef:
              name: fw-config
              key: fw_devicegroup
        - name: FW_POLL_INTERVAL
          value: "1m"
        - name: Calico Enterprise_TIER_PREFIX
          value: fw
        - name: Calico Enterprise_NETWORK_PREFIX
          value: panw
        - name: Calico Enterprise_TIER_ORDER
          value: "101"
        - name: Calico Enterprise_PASS_TO_NEXT_TIER
          value: "true"
```
