# K3S Private Registry Setup

This document describes how to create a K3S private registry that can be pushed to from a local Docker Desktop.

## Setup Private Registry

- Create `registry.yaml` file:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-registry
  namespace: kube-system
spec:
  replicas: 1
  selector:
    app: kube-registry
  template:
    metadata:
      name: kube-registry
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
          env:
            - name: REGISTRY_HTTP_ADDR
              value: :5000
            - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
              value: /var/lib/registry
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          volumeMounts:
            - name: registry
              mountPath: /var/lib/registry
          ports:
            - containerPort: 5000
      nodeSelector:
        node-role.kubernetes.io/master: "true"
      volumes:
        - name: registry
          hostPath:
            path: /var/lib/registry
---
apiVersion: v1
kind: Service
metadata:
  name: kube-registry
  namespace: kube-system
spec:
  selector:
    app: kube-registry
  ports:
    - port: 5000
      targetPort: 5000
  type: LoadBalancer
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

- Create the `registry.yaml` file:

```bash
sudo nano /etc/rancher/k3s/registries.yaml
```

- Copy in the following:

```yaml
mirrors:
  "rpi-master:5000":
    endpoint:
      - "http://192.168.3.10:5000"
```

- Restart the cluster:

```bash
sudo systemctl restart k3s
```

## Configure local Desktop Docker

Finally, configure your local Docker client by adding:

"insecure-registries": ["<hostname>:5000"]

![Desker Desktop Private Registry](./images/docker-desktop-private-registry.png)

## Test Pushing from Docker Desktop to Cluster

- Find a local image to push to the cluster:

```bash
docker images
```

```console
REPOSITORY               TAG       IMAGE ID       CREATED          SIZE
go-api                   v1        81bebf4de0fd   20 minutes ago   16.5MB
```

- Login to K3S private repository:

```bash
docker login rpi-master:5000
```

- Tag the image:

````bash
docker tag 81bebf4de0fd rpi-master:5000/go-api:v1```
````

- Push to the private repository:

```bash
docker push rpi-master:5000/go-api:v1
```

- Create a simple manifest with replication controller and service:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: go-api
  namespace: default
spec:
  replicas: 10
  selector:
    app: go-api
  template:
    metadata:
      name: go-api
      labels:
        app: go-api
    spec:
      containers:
        - name: go-api-container
          image: rpi-master:5000/go-api:v1
          resources:
            limits:
              cpu: 100m
              memory: 200Mi
          ports:
            - containerPort: 9001
---
apiVersion: v1
kind: Service
metadata:
  name: go-api
  namespace: default
spec:
  selector:
    app: go-api
  ports:
    - port: 9001
      targetPort: 80
  type: LoadBalancer
```

Note the container reference explicitly state the K3S private repository.

- Apply it to the cluster:

```bash
kubectl apply -f go-api.yaml
```

- Check the results:

```bash
kubectl get pods
```

```console
NAME           READY   STATUS    RESTARTS   AGE
go-api-2bdqq   1/1     Running   0          20s
go-api-2k7j9   1/1     Running   0          20s
go-api-7br2g   1/1     Running   0          20s
go-api-94jz2   1/1     Running   0          20s
go-api-999wk   1/1     Running   0          20s
go-api-dtwgj   1/1     Running   0          20s
go-api-fws4l   1/1     Running   0          20s
go-api-lmgv6   1/1     Running   0          20s
go-api-lxppk   1/1     Running   0          20s
go-api-vrn7s   1/1     Running   0          20s
```

- Remove it from the cluster:

```bash
kubectl delete -f go-api.yaml
```

## References

[Adding a Private Docker Registry to your RPi Kubernetes Cluster](https://medium.com/@chris.allmark/adding-a-private-docker-registry-to-your-rpi-kubernetes-cluster-3b549cc33c4f)
[How to Use Your Own Registry with Docker Desktop](https://www.docker.com/blog/how-to-use-your-own-registry-2/)

## Notes

## Navigation

- [Next](./?)
- [Index](./README.md)
