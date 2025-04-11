# INGRESSGATEWAY - Private Servcie Connect

![Architecture](./images/PSC.png "Private Service Connect Architecture")


## PLAY GKE CLUSTER (Service Producer)
### Create a subnet for Private Service Connect 
```shell
# You cannot use the same subnet in multiple service attachment configurations.
gcloud compute networks subnets create psc-istio-ingressgateway-private \
    --project gcp-play \
    --network play-vpc-network \
    --region europe-west3 \
    --range 10.10.0.0/16 \
    --purpose PRIVATE_SERVICE_CONNECT
```
### Deploy Istio ingressgateway helm chart that creates internal Loadbalancer
```shell
# values.yaml
service:
  externalTrafficPolicy: Local
  annotations:
    cloud.google.com/neg: '{"ingress":true}'
    networking.gke.io/load-balancer-type: Internal
```

### Deploy a `whereami` service
```shell
kubectl apply -f ingressgateway-whereami.yaml
```

### Create ServiceAttachment
```shell
kubectl apply -f ingressgateway-serviceattachment.yaml
```

## STAGE GKE CLUSTER (Service Consumer)
### craete an ENDPOINT IP address at the consumer side for each PSC connection
**NOTE:** This enpoint IP address receives the traffic and route it to published service on Service Producer side via PSC subnet on Producer side 
```shell
gcloud compute addresses create endpoint-istio-ingressgateway-private-ip \
    --project=gcp-stage \
    --region=europe-west3 \
    --subnet=stage-kube-subnetwork \
    --purpose=GCE_ENDPOINT
```

### create a forwarding rule
```shell
gcloud compute forwarding-rules create psc-istio-ingressgateway-private \
    --project=gcp-stage \
    --region=europe-west3 \
    --network=stage-vpc-network \
    --subnet=stage-kube-subnetwork \
    --address=endpoint-istio-ingressgateway-private-ip \
    --target-service-attachment=projects/gcp-play/regions/europe-west3/serviceAttachments/istio-ingressgateway-private
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

## References
* [PSC Example](https://codelabs.developers.google.com/cloudnet-psc-ilb-gke#0)