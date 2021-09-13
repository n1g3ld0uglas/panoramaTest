# Calico Enterprise - Panorama Firewall Integration

Panorama is a management interface for configuring Palo Alto Networks Next-Generation Firewall (NGFW). <br/>
It supports Device-Groups for easier management of multiple firewall deployments.

![Panorama-Diagram](https://user-images.githubusercontent.com/82048393/133112762-49429e0c-51cf-4bc2-9d02-a3fd2396a5fd.png)

For this discussion, we will be using ```Panorama v8.1``` (specifically v8.1.2) which supports XML based programming interface.

## User Configuration
First, we need a user with API access to Panorama. <br/>
While admin user or any other super-user role user can be used, we strongly advise in favor of using a separate account with only API access.

### API admin role
* Create an admin-role with access to only XML API enabled. <br/>
We suggest disabling Web UI/Command Line access.

* For network-security enforcements, Palo Alto Firewalls use Zone-based policies. <br/>
Zones are network segmentation constructs.
* To define communication policies between zones, Panorama supports Security Rules. <br/>
Security rules allow an admin to create a rule specifying Source and Destination zones associated with a Service (protocol, port) definition and an Action (allow/deny) to take in case of match. <br/>
* These rules enforce stateful policies between the traffic zones. <br/>
* In addition to zones, a rule can also include IP addresses (with CIDR) and a group of IP addresses defined as address-groups. <br/>
However, IP address based rules are beyond the scope of this implementation.

<p align="center">
<img width="623" alt="Screenshot 2021-09-13 at 16 42 35" src="https://user-images.githubusercontent.com/82048393/133114786-9b23c7af-e0fc-4700-95b6-508711483ffc.png">
</p>

### API admin user

Next, create a user using the admin role created above. <br/>
Note the username/password to be used during configuration on the Calico Enterprise side.

<p align="center">
<img width="623" alt="Screenshot 2021-09-13 at 16 44 57" src="https://user-images.githubusercontent.com/82048393/133115085-3d328b89-32d2-4a02-9795-bf2c19563866.png">
</p>

### (Optionally) - create API key

Create API key for Panorama configuration to be used in XML-API. This step is optional because the Calico Enterprise Firewall Integration module can work with username/password or APIKey. As there is no APIKey revocation interface in Panorama, invalidating API Key, in case of a compromise, can mean deleting the user altogether. We leave it to the security posture of given deployment to use either username/password or API Key. <br/>

```
$ curl -k -X POST 'https://<panorama-ip-adress-hostname>/api/?type=keygen&user=<username>&password=<password>'

<response status = 'success'><result><key>...API-Key...</key></result></response>%
```

In the following sections, we will see how to configure Panorama and Calico Enterprise to set up a seamless synchronization of firewall security rules into Calico Enterprise network policies.

<p align="center">
![structure](https://user-images.githubusercontent.com/82048393/133113711-3ad8cca4-3f85-4b0e-a61a-1a9e66cd2023.png)
</p>

## Zone Rules
Security admin can come up with customized zone definitions and policies suitable for their deployment. <br/>
Let’s look at a sample policy definition in Panorama. <br/>
For our example, we are adding a Security Pre-Rule to block connection from zone dmz to backend when both source and destination port is 8080 under test1 device-group.

### Rule name
Add a rule-name. Other fields are optional.

<p align="center">
<img width="780" alt="Screenshot 2021-09-13 at 17 41 12" src="https://user-images.githubusercontent.com/82048393/133123584-8ab75fa8-191a-4632-a8af-93d529fa7b23.png">
</p>

### Source zone
Click Add on the bottom-left to add zone(s), dmz.

<p align="center">
<img width="780" alt="Screenshot 2021-09-13 at 17 42 37" src="https://user-images.githubusercontent.com/82048393/133123725-ae39cabc-b420-4679-ac51-02f05f8b7894.png">
</p>

  
## Create a namespace
```
apiVersion: v1
kind: Namespace
metadata:
  name: panorama-integration
```

## Create a configMap
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

## Create a secret
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

## RBAC Config

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

## Firewall Integration Manifest

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

## 2nd Deployment

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

## Final ConfigMap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: tigera-firewall-controller
  namespace: panorama-integration
data:
  tigera.firewall.addressSelection: node
  tigera.firewall.policy.selector: projectcalico.org/tier == 'panorama'
```
