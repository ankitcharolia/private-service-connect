# INGRESSGATEWAY - Private Servcie Connect

![Architecture](./images/PSC.png "Private Service Connect Architecture")


## PLAY GKE CLUSTER (Service Producer)
### Create a subnet for Private Service Connect 
```shell
gcloud compute networks subnets create psc-istio-ingressgateway-private \
    --project gcp-play \
    --network play-vpc \
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
```shell
apiVersion: networking.gke.io/v1
kind: ServiceAttachment
metadata:
 name: istio-ingressgateway-private
 namespace: istio-ingress-private
spec:
 connectionPreference: ACCEPT_MANUAL
 # psc subnet name
 natSubnets:
 - psc-istio-ingressgateway-private
 proxyProtocol: false
 resourceRef:
   kind: Service
   name: istio-ingressgateway-private
```

## STAGE GKE CLUSTER (Service Consumer)
### craete an ENDPOINT IP address at the consumer side for each PSC connection
**NOTE:** This enpoint IP address receives the traffic and route it to published service on Service Producer side via PSC subnet on Producer side 
```shell
   gcloud compute addresses create endpoint-istio-ingressgateway-private-ip \
       --project=gcp-stage \
       --region=europe-west3 \
       --subnet=k8s-subnetwork \
       --purpose=GCE_ENDPOINT
```

### create a forwarding rule
```shell
   gcloud compute forwarding-rules create psc-k8s-dashboard \
       --project=gcp-stage \
       --region=europe-west3 \
       --network=stage-vpc \
       --subnet=k8s-subnetwork \
       --address=endpoint-istio-ingressgateway-private-ip \
       --target-service-attachment=projects/gcp-stage/regions/europe-west3/serviceAttachments/istio-ingressgateway-private
```

### 

* [PSC Example](https://codelabs.developers.google.com/cloudnet-psc-ilb-gke#0)