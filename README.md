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
