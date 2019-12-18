# px-secure
Securing your data with [Portworx](https://docs.portworx.com)

# Overview
The following describes how to setup Portworx security with a single token. In this model, the goal is to authenticate users by Kubernetes, then have a Kubernetes to Portworx token used by all users. This model protects the storage system from unwanted access from outside Kubernetes.

## Download a spec
* [Example Portworx 2.2.x Config](https://install.portworx.com/2.2?mc=false&kbver=1.14.6&b=true&f=true&c=px-secure-cluster-fa553474-582e-44be-8379-414c4d0cbb72&stork=true&lh=true&st=k8s)

## Generating secrets
* [Create secrets](https://2.1.docs.portworx.com/concepts/authorization/pre-install/#self-signing-tokens) for [environment variables](https://2.1.docs.portworx.com/concepts/authorization/install/#environment-variables)

```
PORTWORX_AUTH_SYSTEM_KEY=$(cat /dev/urandom | base64 | fold -w 64 | head -n 1)
PORTWORX_AUTH_STORK_KEY=$(cat /dev/urandom | base64 | fold -w 64 | head -n 1)
PORTWORX_AUTH_SHARED_SECRET=$(cat /dev/urandom | base64 | fold -w 64 | head -n 1)
```

* Create a secret with the information

```
kubectl -n kube-system create secret generic pxkeys \
   --from-literal=system-secret=$PORTWORX_AUTH_SYSTEM_KEY \
   --from-literal=stork-secret=$PORTWORX_AUTH_STORK_KEY \
   --from-literal=shared-secret=$PORTWORX_AUTH_SHARED_SECRET
```

Test: Retreiving a key from the secret:

```
kubectl -n kube-system get secret pxkeys -o json | jq -r '.data."shared-secret"' | base64 -d
```

## Add to Portworx yaml
* Add to Portworx config yaml file as shown in [documentation](https://2.1.docs.portworx.com/portworx-install-with-kubernetes/operate-and-maintain-on-kubernetes/authorization/enable/#example)
* Checklist
  1. Stork shared key needs to be added to stork and to Portworx as shown in the documentation.
  1. System key needs to be added to Portworx
  1. Shared key needs to be added to Portworx
  1. The Token issuer value needs to be added. The issuer is a string value which must identify the token generator. The token generator must set this value in the token itself under the `iss` claim. In this example, the issuer is set to `portworx.com`

Example diff

```diff
170c170
<              "-x", "kubernetes"]
---
>              "-x", "kubernetes", "-jwt_issuer", "portworx.com"]
178c178,192
<             
---
>             - name: "PORTWORX_AUTH_JWT_SHAREDSECRET"
>               valueFrom:
>                 secretKeyRef:
>                   name: pxkeys
>                   key: shared-secret
>             - name: "PORTWORX_AUTH_SYSTEM_KEY"
>               valueFrom:
>                 secretKeyRef:
>                   name: pxkeys
>                   key: system-secret
>             - name: "PORTWORX_AUTH_STORK_KEY"
>               valueFrom:
>                 secretKeyRef:
>                   name: pxkeys
>                   key: stork-secret
641a656,660
>         - name: "PX_SHARED_SECRET"
>           valueFrom:
>             secretKeyRef:
>               name: pxkeys
>               key: stork-secret
```

## Generate token
First create a user configuration yaml and save it to a file. In this example, we will save the following to a file called `user.yaml`:

```yaml
name: Kubernetes User
email: info@portworx.com
sub: kubernetes/info@portworx.com
roles: ["system.user"]
groups: ["kubernetes"]
```

Now you can create a token. Notice in the example below that we have set the issuer to match the setting in Portworx to `portworx.com` and have set the duration of the token to one year:

```
/opt/pwx/bin/pxctl auth token generate \
  --auth-config=user.yaml \
  --issuer=portworx.com \
  --shared-secret=$PORTWORX_AUTH_SHARED_SECRET \
  --token-duration=1y
```

You can then save the token in a Kubernetes secret. In the example below, it saves it as `portworx/px-k8s-user`:

```
kubectl -n portworx create secret generic px-k8s-user --from-literal=auth-token=ey..
```

## Storage Class
Create a storage class to use secret:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: px-storage
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "1"
  openstorage.io/auth-secret-name: px-k8s-user
  openstorage.io/auth-secret-namespace: portworx
allowVolumeExpansion: true
```

## Example application
PVC:

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-data
  annotations:
    volume.beta.kubernetes.io/storage-class: px-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

MySQL:
```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mysql
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
        version: "1"
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-data
```