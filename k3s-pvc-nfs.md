# K3S Persistent Storage Setup

This document describes how store the K3S private registry we created earlier onto our NFS server and us
will be stored on the SD card. In a later chapter we will move this to our NFS server.

## TBC

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

no_root_squash, because many of the docker images we use run as root in the container. Without this, those images will not be able to write to the file system

using CIDR format here to limit access to these exports to the IPs our cluster Pis use.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
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
    namespace: default
    name: registry-pvc
---
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: docker-registry-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```



sudo exportfs -ar

## References

## Notes

## Navigation

- [Next](./)
- [Index](./README.md)
