# metalLB

# 1. Installation
- https://metallb.universe.tf/installation/
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
```

# 2. Configuration
```
$ kubectl get nodes -o wide
NAME      STATUS   ROLES                  AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master    Ready    control-plane,master   12d   v1.22.3   192.168.122.199   <none>        Ubuntu 20.04.3 LTS   5.11.0-38-generic   docker://20.10.10
worker1   Ready    node                   12d   v1.22.3   192.168.122.134   <none>        Ubuntu 20.04.3 LTS   5.11.0-38-generic   docker://20.10.10
worker2   Ready    node                   12d   v1.22.3   192.168.122.214   <none>        Ubuntu 20.04.3 LTS   5.11.0-38-generic   docker://20.10.10
```
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.122.220-192.168.122.250
EOF
```
```
$ kubectl describe configmap config -n metallb-system
Name:         config
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>

Data
====
config:
----
address-pools:
- name: default
  protocol: layer2
  addresses:
  - 192.168.122.220-192.168.122.250


BinaryData
====

Events:  <none>
```

# 3. Apply yamlfile with LoadBalancer
```
apiVersion: v1
kind: Service
metadata:
  name: facerecognizer-srv
  labels:
    run: facerecognizer-srv    <------ if you might use kiali, then the label should be "app" not "run".
spec:
  #type: NodePort
  type: LoadBalancer           <------ Changing Here
  ports:
  - port: 5000
    targetPort: 5000
    protocol: TCP
    name: http
    #nodePort: 30002
  selector:
    run: facerecognizer-test
```
```
$ kubectl get services
NAME                                                    TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)          AGE
facerecognizer-srv                                      LoadBalancer   10.97.104.96     192.168.122.221   5000:30002/TCP   14h
gpu-operator                                            ClusterIP      10.107.20.212    <none>            8080/TCP         12d
gpu-operator-1635474989-node-feature-discovery-master   ClusterIP      10.101.66.193    <none>            8080/TCP         12d
kubernetes                                              ClusterIP      10.96.0.1        <none>            443/TCP          12d
```

# 4. istio Gateway
```
$ kubectl apply -f - << EOF 
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: facerecognizer-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: facerecognizer-vsrv
spec:
  hosts:
  - "*"
  gateways:
  - facerecognizer-gateway
  http:
  - match:
    - uri:
        prefix: "" 
    route:
    - destination:
        port:
          number: 5000
        host: facerecognizer-srv
EOF
```
```
$ kubectl get gateway
NAME                     AGE
facerecognizer-gateway   12s

$ kubectl get virtualservices
NAME                  GATEWAYS                     HOSTS   AGE
facerecognizer-vsrv   ["facerecognizer-gateway"]   ["*"]   20s
```
```
$ curl 192.168.122.220
<html>
  <head>
    <title>Wellcome</title>
  </head>
  <body>
    <h1>Wellcome</h1>
    <a href="./stream">Face Recognizer</a> <br>
    <a href="./nvidia-smi">nvidia-smi</a> <br>
  </body>
</html>

$ curl 192.168.122.220/nvidia-smi
Thu Nov 11 13:30:03 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.57.02    Driver Version: 470.57.02    CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P1000        On   | 00000000:06:00.0 Off |                  N/A |
| 34%   33C    P8    N/A /  N/A |    667MiB /  4040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```
