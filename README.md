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
<img width="623" alt="Screenshot 2021-09-13 at 16 44 57" src="https://user-images.githubusercontent.com/82048393/133113711-3ad8cca4-3f85-4b0e-a61a-1a9e66cd2023.png">
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

### Destination zone
Make sure the top drop-down shows Select and then click Add to add destination zone(s), backend.

<p align="center">
<img width="777" alt="Screenshot 2021-09-13 at 17 48 32" src="https://user-images.githubusercontent.com/82048393/133124535-afce3598-bc0f-48e8-bca5-db343408b8af.png">
</p>

### Service
Navigate to ```Object``` > ```Services``` to add a service definition under the same device-group, test1. <br/>
NOTE: The source-port is optional and the protocol is TCP, by default. Set source/destination port to 8080.

<p align="center">
<img width="776" alt="Screenshot 2021-09-13 at 17 50 33" src="https://user-images.githubusercontent.com/82048393/133124765-d9c7be6c-45b7-4787-b4c7-e82f3a54e5ff.png">
</p>

Make sure that the drop-down shows select under Service/URL category in rule definition before adding the newly created service.

<p align="center">
<img width="780" alt="Screenshot 2021-09-13 at 17 51 36" src="https://user-images.githubusercontent.com/82048393/133124890-74dbdc74-249f-4bac-ba43-c0f35f96ec0c.png">
</p>

### Action
Set action to ```Block``` under Action Settings.

<p align="center">
<img width="780" alt="Screenshot 2021-09-13 at 17 54 51" src="https://user-images.githubusercontent.com/82048393/133125365-cb9f99f0-c408-411c-92ef-f3c52077448b.png">
</p>

All other settings in the policy rule definition are optional and are beyond the scope of this integration module.

# Calico Enterprise Configuration

Firewall Integration module on Calico Enterprise runs as a kubernetes deployment that compiles and periodically synchronizes security policy rules from Panorama to Calico Enterprise network policies. The module supports configuration options to specify Panorama and Calico Enterprise specific settings.

  
# Panorama configuarion
```
apiVersion: v1
kind: Namespace
metadata:
  name: calico-monitoring
```

## Hostname & Device-group

Hostname can be specified as an IP Address or FQDN. Firewall Integration currently supports only 1 Device-group at a time. <br/> 
Corresponding configuration is FW_HOSTNAME & FW_DEVGROUP, respectively. <br/> 
Both the settings are mandatory and can be specified as a ConfigMap, as mentioned below:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: fw-config
  namespace: calico-monitoring
data:
  fw_hostname: <Panorama-Device-IP>
  fw_devicegroup: <Panorama-Device-Group>
```  

## Username/Password or API Key

While credentials (username/password, APIKey) can be specified as ConfigMap. <br/>
Standard k8s practise dictates using Secret resource to specify such configuration. <br/>
Firewall Integration relies on underlying kubernetes setup for security-at-rest for these configuration items. <br/>
These values are interpreted as FW_USERNAME, FW_PASSWORD & FW_APIKEY configuration options in the Firewall Integration module. <br/>
Here is an example of Secret config with all required items.

```
kind: Secret
apiVersion: v1
type: Opaque
metadata:
  name: fw-secret-config
  namespace: calico-monitoring
data:
  fw_username: NDBndNDlnd== (fake username)
  fw_password: NDBndNDlndNDc3N3b3ND (fake pw)
```

## Firewall Polling Interval (time)

Configuration setting FW_POLL_INTERVAL specifies the interval to read security rules from Panorama and create policies in Calico Enterprise. <br/>
The default value for this configuration is 10 minutes. We suggest using a value that doesn’t overwhelm Panorama connectivity while keeping the policy enforcement in check and timely.


# RBAC Configuarion

Firewall Integration honors standard Kubernetes RBAC configuration. <br/> 
This means in order to work with Calico Enterprise resources, a service-account and corresponding role, bindings are needed:

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

# Calico Enterprise Configuration:
Firewall Integration provides Calico Enterprise specific configurations. <br/>
Following is a list of options with brief description:

## Calico Enterprise_TIER_PREFIX (string):
Panorama policies are read and compiled into global network policies. <br/>
All these global network policies are stored in a single tier. <br/>
This tier name is derived from Prefix setting and device-group name. <br/> 
It’s not a required setting and defaults to fw.

## Calico Enterprise_NETWORK_PREFIX (string)
Panorama security rules are compiled into per-zone level global network policies for each zone mentioned in the rules. <br/>
Name of such a global network policy is derived from Network prefix and zone-name. It’s not a required setting and defaults to ```panw```.

## Calico Enterprise_TIER_ORDER (number)
Each network policy tier in Calico Enterprise has a corresponding order. <br/>
This setting allows you to specify the order of the policy tier. <br/>
It defaults to 101 or immediately after allow-cnx which is a default first tier in Calico Enterprise.

## Calico Enterprise_PASS_TO_NEXT_TIER (bool)
All zone specific rules are contained to the tier, populated and synchronized by Firewall Integration module. <br/> 
If Calico admin plans to add rules beyond this tier, the integration tier has to add a pass rule to handover policy decisions to the next tier. <br/>
This configuration is off by default, meaning Zone based network policies are container to the integration tier.


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

## etcd/KDD Datastore Support
Firewall Integration module supports both etcd and KDD mode of Kubernetes deployment. </br>
When using a seperate etcd as Kubernetes data-store, you can specify corresponding configuration using Calico Enterprise standard configuration options.

# Understanding Security Rules to Policy Translation
Let’s look at an example to understand how zone-based security-rules in Panorama translate into Calico Enterprise Global Network Policies.<br/>
<br/>
If security-rules on Panorama are configured like the following:


<p align="center">
<img width="683" alt="Screenshot 2021-09-13 at 21 24 18" src="https://user-images.githubusercontent.com/82048393/133151391-26cb9be9-463d-46aa-b03b-4c8f506889a9.png">
</p>

Firewall Integration module reads and compresses into zone specific logic. <br/>
This may mean that the rules for a particular zone spread across multiple security rules will be compressed together into one policy during compilation.<br/>
<br/>
The compilation process helps identify rules in terms of Ingress/Egress policy with respect to a zone which further simplifies translation into global network policy.

<p align="center">
<img width="645" alt="Screenshot 2021-09-13 at 21 30 32" src="https://user-images.githubusercontent.com/82048393/133152159-6192bb2e-ddb3-4c8b-aad6-6fb8c30d4d5c.png">
</p>

Policies are read in order, starting with --
* Shared Pre-Rules
* Device-Group Pre-Rules
* Shared Post-Rules
* Device-Group Post-Rules
* Predefined Rules

This is in confirmation to the Panorama rules order applicable to Firewall Integration use-case.<br/>
<br/>
Notice rule (4) & (7) in Panorama rules captured above. They are zones specific protocol, source/destination port rule definition. Take rule (4) for an example, it specifies source-zone dmz is allowed connection to destination-zone ‘interface’ over TCP port 443 only. Firewall integration reads service definition and adds this information to the compiled policy accordingly with respect to the Egress and Ingress flow for given zones.<br/>
<br/>
Predefined rules come as a factory installed on Panorama. These rules cannot be deleted but can be overridden. Firewall integration reads these inter-zone and intra-zone rules and adjusts network policies on Calico Enterprise accordingly.<br/>
<br/>
Here is a snapshot of the F/W tier on Calico Enterprise for above captured Panorama rules.

<p align="center">
<img width="617" alt="Screenshot 2021-09-13 at 21 34 57" src="https://user-images.githubusercontent.com/82048393/133152667-8b9eb6db-6d3b-4c88-a92c-4c77268cf44b.png">
</p>

Note the ‘Firewall Tier’, fw-test1 derived from Calico Enterprise_TIER_PREFIX and device-group configured. All the rules in the policy assumes that there are workloads in the kubernetes labeled with ‘zone’ key with values matching appropriate zone names, e.g. a workload with ‘zone = dmz’ label will follow Ingress/Egress rules in global network policy ‘panw-zone-dmz’. <br/>
<br/>
The last 3 policies in the ‘Firewall Tier’ are created for any workloads matching ‘zone’ labels but without a corresponding entry in Panorama security rules. Adding these rules are important to enforce Predefined policies in Panorama.<br/>
<br/>
Let’s look inside a policy to see how the rules look like:

<p align="center">
<img width="622" alt="Screenshot 2021-09-13 at 21 37 16" src="https://user-images.githubusercontent.com/82048393/133152953-da5483e5-91ba-4191-a89b-d63a972c3e3c.png">
</p>

# Important Points

1. Panorama supports multi-level (up to 4) device-groups with common Shared hierarchy. <br/>
Firewall Integration assumes the device-group configured is level 1 only. <br/>
In other words, Firewall Integration will read security rules in Shared and given device-group alone.

2. Firewall Integration module doesn’t authenticate Panorama certificate identity. <br/>
This is an issue with vendor libraries being used. <br/>
If your deployment requires certificate authentication, please reach out to us for a custom solution.

3. If you make a change to Panorama configuration beyond security-rules, services or to security-rules, services belonging to a device-group not configured for Firewall Integration, the integration module will still treat it as a material configuration change. <br/>
This is because a limitation in Panorama API doesn’t allow notifying for change in specific configuration object or device-group. <br/>
This means, Firewall Integration module will be forced to re-read security-rules, services configuration for configured device-group whenever there is any change in Panorama configuration. <br/>
Limitation in Panorama API forces Firewall Integration to check for any new configuration within or beyond device-group of interest.

4. Network policy tier created and maintained by Firewall Integration module can be modified by a user with Network Admin role or a role allowed to write policies. <br/> However, these changes will be overwritten by GlobalNetworkPolicies by Firewall Integration after syncing every polling interval. <br/>
For most practical deployment, we advise to add RBAC tiered policies limiting editing of the tier to Firewall Integration module only.


# Panorama Overview:
Panorama provides centralised management and reporting functionality for multiple firewalls within an environment. <br/>
Accessible from a very similar GUI to a standalone firewall, there are a few key differences on a panorama versus a firewall, notably the addition of a tab called panorama.

<p align="center">
<img width="622" alt="Screenshot 2021-09-13 at 21 57 12" src="https://user-images.githubusercontent.com/82048393/133155460-224acd8e-ac2b-421b-a05c-9da7953ff57c.png">
</p>


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
