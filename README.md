## Task Helm and Istio

### Helm Debug

we need to make some changes on configuration files for deployemnt to can properly work as shown on the below:


* File: **values.yaml**
we make swapping for port and target port.

```
image:
  name: hello-world
  tag: latest
  registry: adamgolab

service:
  port: 80
  targetPort: 8000

deployment:
  replicas: 1
  depRegion: de
  default: en
  ```
  
  * File: **configMap.yaml**

we add else if and make correcly configuration based for each path (de,es,fr).

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: {{ .Values.image.name }}-{{ .Values.deployment.depRegion }}
data:
  # Configuration values can be set as key-value properties
{{ if eq .Values.deployment.depRegion "de"}}
  de: Freund
{{ else if eq .Values.deployment.depRegion "es"}}
  es: Amigo
{{ else if eq .Values.deployment.depRegion "fr"}}
  fr: Ami
{{ else }}
  en: World
{{ end }}

```

  * File: **service.yaml**

we make this change to make match **Selector** for deployment.

    Before: svc-name: {{ .Values.image.name }}-{{.Values.deployment.default}}
    After:  svc-name: {{ .Values.image.name }}-{{.Values.deployment.depRegion}}
    
```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.image.name }}-{{.Values.deployment.depRegion}}
  labels:
    svc-name: {{ .Values.image.name }}-{{.Values.deployment.depRegion}}
spec:
  type: ClusterIP
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    protocol: TCP
  selector:
    svc-name: {{ .Values.image.name }}-{{.Values.deployment.depRegion}}
```

then, we will make deploy helm charts for 3 application.

```

helm upgrade -n interview --install --values 1-helm-debug/helm/values.yaml interview-task-de 1-helm-debug/helm/ **--set deployment.depRegion=de**

helm upgrade -n interview --install --values 1-helm-debug/helm/values.yaml interview-task-es 1-helm-debug/helm/ **--set deployment.depRegion=es**

helm upgrade -n interview --install --values 1-helm-debug/helm/values.yaml interview-task-fr 1-helm-debug/helm/ **--set deployment.depRegion=fr**
```

### Deploy and Configure for Nginx Ingress Controller

The Nginx Ingress Controller is a Kubernetes add-on that allows you to use Nginx as a load balancer for your applications running on a Kubernetes cluster. The Nginx Ingress Controller works by listening for incoming traffic and routing it to the appropriate service based on the hostname and path specified in the request.


- **Install Nginx Ingress Controller
```
# add new repo ingress-nginx/ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add stable https://charts.helm.sh/stable
helm repo update

# install
helm install nginx-ingress-controller ingress-nginx/ingress-nginx
```

- **Create Ingress resource for L7 load balancing by http hosts & paths for 3 application.


- nginx.ingress.kubernetes.io/rewrite-target: /
The nginx.ingress.kubernetes.io/rewrite-target annotation is used to specify the target URL that the Nginx Ingress Controller should rewrite the request to. The / value indicates that the request should be rewritten to the root path of the target service.


```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  creationTimestamp: "2022-12-25T11:57:08Z"
  generation: 15
  name: xyz-x
  namespace: interview
  resourceVersion: "1134556"
  uid: 77ae7bf8-95ea-4e31-92a9-cdb93304733e
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: hello-world-de
            port:
              number: 80
        path: /de
        pathType: Prefix
  - http:
      paths:
      - backend:
          service:
            name: hello-world-fr
            port:
              number: 80
        path: /fr
        pathType: Prefix
  - http:
      paths:
      - backend:
          service:
            name: hello-world-es
            port:
              number: 80
        path: /es
        pathType: Prefix
status:
  loadBalancer:
    ingress:
    - hostname: a97dbbaff8e114fd386aa4ba3f365cd3-2121781278.us-east-1.elb.amazonaws.com
```

### Verify results and outputs

```
PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl get pod -n interview
NAME                              READY   STATUS    RESTARTS   AGE
hello-world-de-768ff66b8-27nlc    1/1     Running   0          78m
hello-world-es-6c77bd9c9b-t9stm   1/1     Running   0          78m
hello-world-fr-77bc6cbb9b-wwb8w   1/1     Running   0          78m

PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl get deploy -n interview
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-world-de   1/1     1            1           78m
hello-world-es   1/1     1            1           78m
hello-world-fr   1/1     1            1           79m

PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl get svc -n interview   
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
hello-world-de   ClusterIP   172.20.63.246    <none>        80/TCP    78m
hello-world-es   ClusterIP   172.20.214.248   <none>        80/TCP    78m
hello-world-fr   ClusterIP   172.20.70.6      <none>        80/TCP    79m

PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl get ingress -n interview
NAME     CLASS    HOSTS   ADDRESS                                                                   PORTS   AGE
xyz-x    nginx    *       a97dbbaff8e114fd386aa4ba3f365cd3-2121781278.us-east-1.elb.amazonaws.com   80      37h  
  ```


```
PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl logs hello-world-de-768ff66b8-27nlc -n interview

Node app is running at localhost: 8000


PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl exec hello-world-de-768ff66b8-27nlc -n interview -it -- bash
root@hello-world-de-768ff66b8-27nlc:/app# curl localhost
curl: (7) Failed to connect to localhost port 80: Connection refused
root@hello-world-de-768ff66b8-27nlc:/app# curl localhost:8000
Hello Freund
```

###  Ingress Details

```
PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl describe ingress xyz-x -n interview
Name:             xyz-x
Labels:           <none>
Namespace:        interview
Address:          a97dbbaff8e114fd386aa4ba3f365cd3-2121781278.us-east-1.elb.amazonaws.com
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /de   hello-world-de:80 (10.0.2.159:8000)
  *
              /fr   hello-world-fr:80 (10.0.1.58:8000)
  *
              /es   hello-world-es:80 (10.0.1.235:8000)
Annotations:  kubernetes.io/ingress.class: nginx
              nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    45m (x13 over 114m)  nginx-ingress-controller  Scheduled for sync
```

### Nginx Ingress Logs

PS C:\terraform-on-aws-eks-main\Task\Istio-Task\crp2-tech-challenge-master> kubectl logs nginx-ingress-controller-ingress-nginx-controller-55f87544znh26 -n nginx-ingress-controller -f

10.0.1.254 - - [27/Dec/2022:01:19:24 +0000] "GET /de HTTP/2.0" 200 12 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36" 538 0.002 [interview-hello-world-de-80] [] 10.0.2.159:8000 12 0.003 200 32b2bc16cd2c017750af6cb2864aaa1a
10.0.1.254 - - [27/Dec/2022:01:19:43 +0000] "GET /es HTTP/2.0" 200 11 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36" 60 0.001 [interview-hello-world-es-80] [] 10.0.1.235:8000 11 0.001 200 92e42d728e4a5cbbcccc05a46ed09dd0
10.0.1.254 - - [27/Dec/2022:01:19:49 +0000] "GET /fr HTTP/2.0" 200 9 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36" 23 0.002 [interview-hello-world-fr-80] [] 10.0.1.58:8000 9 0.001 200 1f03ea86caafd7b45e3a2407ba1ad0e3


Output
```bash
# visit this from browser
a97dbbaff8e114fd386aa4ba3f365cd3-2121781278.us-east-1.elb.amazonaws.com/de
Hello Freund
a97dbbaff8e114fd386aa4ba3f365cd3-2121781278.us-east-1.elb.amazonaws.com/es
Hello Amigo
a97dbbaff8e114fd386aa4ba3f365cd3-2121781278.us-east-1.elb.amazonaws.com/fr
Hello Ami
```


Part 2:

## Istio
## 1.1 What is Service Mesh
Ref: https://istio.io/docs/concepts/what-is-istio/#what-is-a-service-mesh


__Istio Service Mesh__ is a network connectivity (i.e. mesh) within Kubernetes cluster created by __Envoy proxy__ containers, be it a standalone or a sidecar proxy:



Another huge benefit of Istio is the default in-cluster mutual TLS.

Without istio, say if using __Ingress controller__, you can configure _TLS termnation at ingress controller pod, like this:


With __istio__, connections among pods in the cluster behind the what-used-to-be ingress controller (i.e. Istio Gateway) can be mutual TLS, without changing app code:




## 1.2 Istio Service Mesh Architecture
Ref: https://istio.io/latest/docs/ops/deployment/architecture/

- service-mesh implementations comes with a __control plane__(istiod) and a __data plane__(a standalone edge Envoy proxy and sidecar Envoy proxies)
- __data plane__ is composed of a set of intelligent proxies (__Envoy__) deployed as sidecars.  These proxies mediate and control all network communication between microservices. They also collect and report telemetry on all mesh traffic.
- __control plane__ lives outside of the request path and is used to administer and control the behavior of the data plane




Data plane:
- __Envoy proxy__: a high-performance proxy developed in C++ to mediate all inbound and outbound traffic for all services in the service mesh. Envoy proxies are the only Istio components that interact with data plane traffic.
    - Traffic control 
        - a different load balancing policy to traffic for a particular subset of service instances
        - Staged rollouts with %-based traffic split
        - HTTP/2 and gRPC proxies
        - Istio Resources
            - Virtual services
            - Destination rules
            - Gateways
            - Service entries
            - Sidecars
    - Network resiliency
        - Fault injection
        - Retries
        - Circuit breakers
        - Failovers
        - Health checks
    - Security and Authentication
        - rate limiting
        - TLS termination


Control Plane: 
- __Istiod__ (consists of __pilot, galley, and citadel__)
    - Dynamic service discovery: in order to direct traffic within your mesh, Istio needs to know where all your endpoints are
    - strong service-to-service and end-user authentication with built-in identity and credential management
    - Pilot - the core data-plane config (xDS) server
    - Galley - configuration watching, validation, forwarding
    - Citadel - certificate signing, secret generation, integration with CAs, etc
    - Telemetry - a “mixer” component responsible for aggregating and syndicating telemetry to various backends
    - Policy - a request-path “mixer” component responsible for enforcing policy



## installing istioctl

$curl -L https://istio.io/downloadIstio| sh-

istioctl verify-install

istioctl install --set profile=demo -y

istioctl verify-install

istioctl analyze


## Deploy applications to the istio service mesh 


1. Expose this service to outside of the cluster by istio gateway ingress controller.
2. When query http://<address>/eu should load balancing round robind shows belows output
  
"Hello Freund"
  
"Hello Amigo"
  
"Hello Ami"
  
   
```
  Kubectl label ns interview istio-injection=enabled
  Kubectl label ns nginx-ingress-controller istio-injection=enabled
  kubectl rollout restart deploy nginx-ingress-controller-ingress-nginx-controller -n nginx-ingress-controller

  kubectl rollout restart deploy hello-world-de -n interview
  kubectl rollout restart deploy hello-world-es -n interview
  kubectl rollout restart deploy hello-world-fr -n intervie
   
```
 
```
 apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: interview-task-de
    meta.helm.sh/release-namespace: interview
  labels:
    app.kubernetes.io/managed-by: Helm
    svc-name: hello-world-eu
  namespace: interview
  name: hello-world-eu
spec:
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    svc-name: hello-world-eu
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
  ```
  
  
  
  ```
  apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: hello-word-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"   # Domain name of the external website
```


```
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: hello-word-vs
  namespace: interview
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*" # Copy the value in the gateway hosts - usually a Domain Name
  gateways:
    - hello-word-gateway
  http:
  - match:     # This is a LIST
    - uri: 
        exact: /eu   # prefix: /static
    route: # THEN
    - destination:
        host: hello-world-eu
 	port:
	  number: 8000	
```

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: hello-word-destination-rule
spec:
  host: hello-world-eu.interview.svc.cluster.local 
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  ```
  
  
  





