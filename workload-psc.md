# WORKLOAD - Private Servcie Connect

![Private Service Connect Architecture](./images/PSC.png "Private Service Connect Architecture")


## PLAY GKE CLUSTER (Service Producer)
### Create a subnet for Private Service Connect 
```shell
gcloud compute networks subnets create psc-wherami-subnetwork \
    --project gcp-test \
    --network test-vpc \
    --region europe-west3 \
    --range 10.10.10.8/30 \
    --purpose PRIVATE_SERVICE_CONNECT
```

### Deploy a `whereami` service
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whereami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whereami
  template:
    metadata:
      labels:
        app: whereami
    spec:
      containers:
      - name: whereami
        image: gcr.io/google-samples/whereami:v1.2.24
        ports:
          - name: http
            containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 1
---
apiVersion: v1
kind: Service
metadata:
  name: whereami
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: whereami
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

### Create ServiceAttachment
```yaml
apiVersion: networking.gke.io/v1
kind: ServiceAttachment
metadata:
 name: whereami
 namespace: default
spec:
 connectionPreference: ACCEPT_MANUAL
 # psc subnet name
 natSubnets:
 - psc-wherami-subnetwork
 proxyProtocol: false
 resourceRef:
   kind: Service
   name: whereami
```

## STAGE GKE CLUSTER (Service Consumer)
### craete an ENDPOINT IP address at the consumer side for each PSC connection
**NOTE:** This enpoint IP address receives the traffic and route it to published service on Service Producer side via PSC subnet on Producer side 
```shell
   gcloud compute addresses create endpoint-wherami-ip \
       --project=gcp-stage \
       --region=europe-west3 \
       --subnet=kube-subnetwork \
       --purpose=GCE_ENDPOINT
```

### create a forwarding rule
```shell
   gcloud compute forwarding-rules create psc-whereami \
       --project=gcp-stage \
       --region=europe-west3 \
       --network=stage-vpc \
       --subnet=kube-subnetwork \
       --address=endpoint-wherami-ip \
       --target-service-attachment=projects/gcp-stage/regions/europe-west3/serviceAttachments/whereami
```

### create virtualservice to access a workload running in PLAY using STAGE CLUSTER virtualservcie
### Create a Kubernetes Service to represent the PSC endpoint
```yaml
apiVersion: v1
kind: Service
metadata:
  name: play-whereami
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
---
apiVersion: v1
kind: Endpoints
metadata:
  name: play-whereami
  namespace: default
subsets:
- addresses:
  - ip: ENDPOINT_IP_ADDRESS  # Replace with the actual endpoint-wherami-ip value
  ports:
  - port: 80
    name: http
```

### Create an Istio VirtualService to route traffic
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: play-whereami
  namespace: default
spec:
  hosts:
  - "whereami.example.com"
  gateways:
  - istio-ingress-private/gateway
  http:
  - route:
    - destination:
        host: play-whereami  # Must match the service name above
        port:
          number: 80
```

### Create a Kubernetes Service to represent the PSC endpoint to PLAY's Istio ingress gateway
```yaml
apiVersion: v1
kind: Service
metadata:
  name: play-istio-gateway
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
---
apiVersion: v1
kind: Endpoints
metadata:
  name: play-istio-gateway
  namespace: default
subsets:
- addresses:
  - ip: ENDPOINT_IP_ADDRESS  # Replace with the endpoint-istio-ingressgateway-private-ip value
  ports:
  - port: 80
    name: http
  - port: 443
    name: https
```

### Create VirtualServices to route traffic to different services in PLAY cluster
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: play-whereami
  namespace: default
spec:
  hosts:
  - "whereami.example.com"  # Host used to access the whereami service
  gateways:
  - istio-ingress-private/gateway  # The gateway in your STAGE cluster
  http:
  - route:
    - destination:
        host: play-istio-gateway  # The service we created to represent the PSC endpoint
        port:
          number: 80
```

### Example of accessing another service in PLAY cluster
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: play-another-service
  namespace: default
spec:
  hosts:
  - "another-service.example.com"  # Different hostname for a different service
  gateways:
  - istio-ingress-private/gateway
  http:
  - route:
    - destination:
        host: play-istio-gateway
        port:
          number: 80
```

* [PSC Example](https://codelabs.developers.google.com/cloudnet-psc-ilb-gke#0)