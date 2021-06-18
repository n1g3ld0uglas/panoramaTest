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

2nd Deployemtn

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tigera-firewall-controller
  namespace: panorama-integration
  labels:
    k8s-app: tigera-firewall-controller
spec:
  replicas: 1
  strategy:
     type: Recreate
  selector:
    matchLabels:
      k8s-app: tigera-firewall-controller
  template:
    metadata:
      name: tigera-firewall-controller
      namespace: panorama-integration
      labels:
        k8s-app: tigera-firewall-controller
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: tigera-firewall-controller
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      imagePullSecrets:
        - name: tigera-pull-secret
      containers:
        - name: tigera-firewall-controlller
          image: quay.io/tigera/firewall-integration:v3.4.1-pre.2
          imagePullPolicy: IfNotPresent
          env:
            - name: ENABLED_CONTROLLERS
              value: "panorama"
            - name: LOG_LEVEL
              value: "debug"
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
              value: "15s"
            - name: TSEE_TIER_PREFIX
              value: panorama
            - name: TSEE_NETWORK_PREFIX
              value: pan
            - name: TSEE_TIER_ORDER
              value: "250"
            - name: TSEE_PASS_TO_NEXT_TIER
              value: "true"
            - name: FW_INSECURE_SKIP_VERIFY
              value: "true"
            - name: FW_POLICY_SELECTOR_EXPRESSION
              valueFrom:
                configMapKeyRef:
                  name: tigera-firewall-controller
                  key: tigera.firewall.policy.selector
            - name: FW_ADDRESS_SELECTION
              valueFrom:
                configMapKeyRef:
                  name: tigera-firewall-controller
                  key: tigera.firewall.addressSelection
            - name: FW_FORTIGATE_CONFIG_PATH
              value: "/etc/tigera/fortigate-configs.yaml"
            - name: FW_FORTIMGR_CONFIG_PATH
              value: "/etc/tigera/fortimgr-configs.yaml"
          volumeMounts:
            - name: firewall-configuration
              mountPath: /etc/tigera/
      volumes:
        - name: firewall-configuration
          configMap:
            defaultMode: 420
            items:
              - key: tigera.firewall.fortigate
                path: fortigate-configs.yaml
              - key: tigera.firewall.fortimgr
                path: fortimgr-configs.yaml
            name: tigera-firewall-controller-configs
```
