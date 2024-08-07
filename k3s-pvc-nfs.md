# K3S Persistent Storage Setup

This document describes how to store the K3S private registry we created earlier on our NFS server instead of on the local SD card. It creates a Persistent Volume (PV) and Persistent Volume Claim (PVC) in K3S.

## Create and Export NFS folder

```bash
sudo mkdir -p /mnt/nfs/registry
```

```bash
sudo chown -R parrisg /mnt/nfs/registry
```

```bash
sudo nano /etc/exports
```

```console
/mnt/nfs/registry 192.168.3.0/27(rw,sync,no_subtree_check,no_root_squash)
```

Use a network subnet address in CIDR format to limit access to these exports to the cluster only.

`sync` means the server replies to requests only after the changes have been committed to stable storage.

`no_subtree_check` disables subtree checking and improves reliability in some circumstances.

`no_root_squash` is used because many of the docker images we use run as root in the container. Without this, those images will not be able to write to the file system.

- Reload the NFS server:

```bash
sudo exportfs -ar
```

- Create a persistent volume and persistent volume claim [registry-pv.yaml](./scripts/registry-pv.yaml):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kube-registry-pv
  namespace: kube-system
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /mnt/nfs/registry
    server: 192.168.3.10
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    name: kube-registry-pvc
    namespace: kube-system
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kube-registry-pvc
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Setup Private Registry

- Create a deployment and service [registry.yaml](./scripts/registry-on-pv.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    app: kube-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-registry
  template:
    metadata:
      labels:
        app: kube-registry
    spec:
      containers:
        - name: registry
          image: registry:2
          resources:
            limits:
              cpu: 100m
              memory: 200Mi
          ports:
            - containerPort: 5000
          env:
            - name: REGISTRY_AUTH
              value: htpasswd
            - name: REGISTRY_AUTH_HTPASSWD_REALM
              value: Kube Registry
            - name: REGISTRY_AUTH_HTPASSWD_PATH
              value: /auth/htpasswd
            - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
              value: /mnt/nfs/registry
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          volumeMounts:
            - name: storage
              mountPath: /mnt/nfs/registry
            - name: htpasswd
              mountPath: /auth
              readOnly: true
      nodeSelector:
        node-role.kubernetes.io/master: "true"
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: kube-registry-pvc
        - name: htpasswd
          secret:
            secretName: kube-registry-htpasswd
---
apiVersion: v1
kind: Service
metadata:
  name: kube-registry-service
  namespace: kube-system
spec:
  selector:
    app: kube-registry
  ports:
    - protocol: TCP
      port: 5000
```

## References

## Notes

### Get list of images in remote registry

```bash
curl -X GET -u username:password http://rpi-master:5000/v2/_catalog     
```

### Get tags for the `go-api` repositry

```bash
curl -X GET -u username:password http://rpi-master:5000/v2/go-api/tags/list
```

## Navigation

- [Next](./)
- [Index](./README.md)
