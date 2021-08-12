# Health Checks
By default, k8s will restart a container if it crashes. 
It uses `liveness` and `readiness` probes to identify the healthy containers to send traffic to, as well as the unhealthy containers to kill and recreate.

## Liveness Probe
`liveness` probes are used to know when a pod is alive or dead.

### configure the `liveness probe` in yaml
Configure the `livenessProbe` for each container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-app
spec:
  containers:
  - name: liveness
    image: brentley/ecsdemo-nodejs
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
```

`periodSeconds: 5` lets `kubelet` check the health every 5 seconds.
`initialDelaySeconds: 5` tells `kubelet` to wait for 5 seconds before doing the first probe.

- we can check if nodes have been restarted by seeing the `RESTARTS` column
  ```sh
  $ kubectl get pod liveness-app

  NAME           READY     STATUS    RESTARTS   AGE
  liveness-app   1/1       Running   0          11s
  ```

- we can also see event history to check for any probe failures or restarts
  ```sh
  $ kubectl describe pod liveness-app

    Events:
    Type    Reason                 Age    From                                        Message
    ----    ------                 ----   ----                                        -------
    Normal  Scheduled              38s    default-scheduler                           Successfully assigned  liveness-app to ip-192-168-18-63.ec2.internal
    Normal  SuccessfulMountVolume  38s    kubelet,  ip-192-168-18-63.ec2.internal     MountVolume.SetUp  succeeded for volume   "default-token-8bmt2"
    Normal  Pulling                37s    kubelet,  ip-192-168-18-63.ec2.internal     pulling image "brentley/ ecsdemo-nodejs"
    Normal  Pulled                 37s    kubelet,  ip-192-168-18-63.ec2.internal     Successfully pulled image  "brentley/ecsdemo-nodejs"
    Normal  Created                37s    kubelet,  ip-192-168-18-63.ec2.internal     Created container
    Normal  Started                37s    kubelet,  ip-192-168-18-63.ec2.internal     Started container
  ```

### To check container logs
- To check the status of the container health checks
  ```sh
  $ kubectl logs liveness-app
  ```
- We can also check the container logs of the previous inistantiation
  ```sh
  $ kubectl logs liveness-app --previous
  ```

## Readiness Probe
`readiness` probes are used to know when a pod is ready to serve traffic.

### Configure the `readiness probe` in yaml
Configure the `readinessProbe` for each container.
Here we use a `touch` command to create a file `/tmp/healthy` when creating the container, and using a `cat` command to check when the container is ready.

```yaml
# readiness-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-deployment
  template:
    metadata:
      labels:
        app: readiness-deployment
    spec:
      containers:
      - name: readiness-deployment
        image: alpine
        command: ["sh", "-c", "touch /tmp/healthy && sleep 86400"]
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 3
```

To check the pods' readiness
```sh
$ kubectl get pods -l app=readiness-deployment

  NAME                                    READY     STATUS    RESTARTS   AGE
  readiness-deployment-7869b5d679-922mx   1/1       Running   0          31s
  readiness-deployment-7869b5d679-vd55d   1/1       Running   0          31s
  readiness-deployment-7869b5d679-vxb6g   1/1       Running   0          31s
```

To get an overview
```
$ kubectl describe deployment readiness-deployment | grep Replicas:

  Replicas:               3 desired | 3 updated | 3 total | 2 available | 1 unavailable
``` 