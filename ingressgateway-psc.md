# INGRESSGATEWAY - Private Servcie Connect

![Architecture](./images/PSC.png "Private Service Connect Architecture")

## PLAY GKE CLUSTER (Service Producer)
### Create a subnet for Private Service Connect 
```shell
# You cannot use the same subnet in multiple service attachment configurations.
# find the correct IP cidr range here: https://mxtoolbox.com/subnetcalculator.aspx
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

### Create ServiceAttachment, Gateway and Virtualservice
```shell
kubectl apply -f ingressgateway-play.yaml
```

### setup istio-ingress private gateway
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: istio-gateway
  namespace: istio-ingress-private
spec:
  selector:
    app: istio-ingressgateway-private
    istio: ingressgateway-private
  servers:
  - hosts:
    - play.fff.private
    - '*.play.fff.private'
    port:
      name: http
      number: 80
      protocol: HTTP
```

### Create a VirtualService in PLAY for the whereami service:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: whereami
  namespace: default
spec:
  hosts:
  - "whereami.play.fff.private"
  - "whereami.play.fff.services"
  gateways:
  - istio-ingress-private/istio-gateway
  - istio-ingress/istio-gateway
  http:
  - route:
    - destination:
        host: whereami
        port:
          number: 80
```

## STAGE GKE CLUSTER (Service Consumer)
### craete an ENDPOINT IP address at the consumer side for each PSC connection
**NOTE:** This enpoint IP address receives the traffic and route it to published service on Service Producer side via PSC subnet on Producer side 
```shell
gcloud compute addresses create play-istio-ingressgateway-private-endpoint-ip \
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
    --address=play-istio-ingressgateway-private-endpoint-ip \
    --target-service-attachment=projects/gcp-play/regions/europe-west3/serviceAttachments/istio-ingressgateway-private
```

### Create a Kubernetes Service to represent the PSC endpoint to PLAY's Istio ingress gateway
```yaml
apiVersion: v1
kind: Service
metadata:
  name: play-istio-ingressgateway-private
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
  name: play-istio-ingressgateway-private
  namespace: default
subsets:
- addresses:
  - ip: 10.0.0.75  # Replace with the istio-ingressgateway-private-endpoint-ip value
  ports:
  - port: 80
    name: http
  - port: 443
    name: https
```


## References
* [PSC Example](https://codelabs.developers.google.com/cloudnet-psc-ilb-gke#0)