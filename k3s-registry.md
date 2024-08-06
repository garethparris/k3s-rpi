# K3S Private Registry Setup

This document describes how to create a K3S private registry that can be pushed to from a local Docker Desktop. This private registry will be stored on the SD card. In a later chapter we will move this to our NFS server.

## Create Credentials to store in K3S

- We need to create a `htpasswd` file to store a username and password, and store it in K3S secrets.

- Install `apache2-utils`:

```bash
sudo apt install -y apache2-utils
```

- Create a `htpasswd` file for a specific user:

```bash
htpasswd -Bc htpasswd parrisg
```

- Store the `htpasswd` file as secret in K3S:

```bash
sudo kubectl create secret generic kube-registry-htpasswd --from-file ./htpasswd -n kube-system
```

- Optionally, delete the local `htpasswd` file:

```bash
rm htpasswd
```

## Setup Private Registry

- Create a deployment and service [registry.yaml](./scripts/registry-on-sdcard.yaml):

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
              value: /var/lib/registry
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          volumeMounts:
            - name: storage
              mountPath: /var/lib/registry
            - name: htpasswd
              mountPath: /auth
              readOnly: true
      nodeSelector:
        node-role.kubernetes.io/master: "true"
      volumes:
        - name: storage
          hostPath:
            path: /var/lib/registry
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

- Apply to the cluster:

```bash
kubectl apply -f registry.yaml
```

- Add a `/etc/rancher/k3s/registries.yaml` file on each node in the cluster for K3S to resolve the repository to an IP address using the following template:

```yaml
mirrors:
  "<hostname>:5000":
    endpoint:
      - "http://<ip-address>:5000"
```

- You may need to create a folder on each of the nodes:

```bash
sudo mkdir /etc/rancher/k3s
```

- Create the `registries.yaml` file:

```bash
sudo nano /etc/rancher/k3s/registries.yaml
```

- Copy in the following:

```yaml
mirrors:
  "rpi-master:5000":
    endpoint:
      - "http://192.168.3.10:5000"
configs:
  "192.168.3.10:5000":
    auth:
      username: username
      password: password
```

- Reboot:

```bash
sudo reboot
```

## Configure Desktop Docker

- Add the private registry details to Docker Desktop under `Settings/Docker Engine` in this format:

```json
"insecure-registries": ["<hostname>:5000"]
```

- It should look something like this:

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "insecure-registries": ["rpi-master:5000"]
}
```

- Restart Docker Desktop

![Desker Desktop Private Registry](./images/docker-desktop-private-registry.png)

## Test Push from Docker Desktop to Private Registry

- When Docker Desktop has restarted, find a local container image to push to the cluster:

```bash
docker images
```

```console
REPOSITORY               TAG       IMAGE ID       CREATED          SIZE
go-api                   v1.0      81bebf4de0fd   20 minutes ago   16.5MB
```

- Login to the K3S private repository, using the credentials you created for `htpasswd` above:

```bash
docker login rpi-master:5000
```

- Tag the image:

```bash
docker tag 81bebf4de0fd rpi-master:5000/go-api:v1.0
```

- Push to the private repository:

```bash
docker push rpi-master:5000/go-api:v1.0
```

- Create a simple manifest to generate 10 replicas with a service [go-api.yaml](./scripts/go-api.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-api
  namespace: default
  labels:
    app: go-api
spec:
  replicas: 10
  selector:
    matchLabels:
      app: go-api
  template:
    metadata:
      labels:
        app: go-api
    spec:
      containers:
        - name: go-api-container
          image: rpi-master:5000/go-api:v1.0
          resources:
            limits:
              cpu: 100m
              memory: 200Mi
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: go-api-service
  namespace: default
spec:
  selector:
    app: go-api
  ports:
    - protocol: TCP
      port: 8080
```

Note the container image explicitly states the K3S private repository `image: pi-master:5000/go-api:v1`.

- Apply it to the cluster:

```bash
kubectl apply -f go-api.yaml
```

- Check the results:

```bash
kubectl get pods -o wide
```

```console
NAME           READY   STATUS    RESTARTS   AGE     IP           NODE
go-api-4prcb   1/1     Running   0          5m10s   10.42.2.15   rpi-worker-02
go-api-96dvr   1/1     Running   0          5m10s   10.42.2.16   rpi-worker-02
go-api-df745   1/1     Running   0          5m10s   10.42.1.11   rpi-worker-01
go-api-jblvk   1/1     Running   0          5m10s   10.42.1.10   rpi-worker-01
go-api-jkxvg   1/1     Running   0          5m10s   10.42.3.14   rpi-worker-03
go-api-k696j   1/1     Running   0          5m10s   10.42.0.55   rpi-master
go-api-lfpdl   1/1     Running   0          5m10s   10.42.4.11   rpi-worker-04
go-api-llnpr   1/1     Running   0          5m10s   10.42.0.54   rpi-master
go-api-ngkvn   1/1     Running   0          5m10s   10.42.3.13   rpi-worker-03
go-api-nxjrq   1/1     Running   0          5m10s   10.42.4.12   rpi-worker-04
```

- Tidy up by removing it from the cluster:

```bash
kubectl delete -f go-api.yaml
```

## References

[Adding a Private Docker Registry to your RPi Kubernetes Cluster](https://medium.com/@chris.allmark/adding-a-private-docker-registry-to-your-rpi-kubernetes-cluster-3b549cc33c4f)

[How to Use Your Own Registry with Docker Desktop](https://www.docker.com/blog/how-to-use-your-own-registry-2/)

[Private Registry Configuration](https://docs.k3s.io/installation/private-registry)

## Notes

### Delete secret from K3S

```bash
sudo kubectl delete secret kube-registry-htpasswd -n kube-system
```

### Logout of registry

```bash
docker logout rpi-master:5000
```

## Navigation

- [Next](./k3s-pvc-nfs.md)
- [Index](./README.md)
