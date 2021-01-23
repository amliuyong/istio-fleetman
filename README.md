https://istio.io/latest/docs/setup/getting-started/#download

## Install
```
curl -L https://istio.io/downloadIstio | sh -

cd istio-1.8.2/bin
PWD=$(pwd)
export PATH=$PWD/:$PATH
```
```
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
```
## Istio

### Canary release

#### add `version` for diff Deployments 

```yaml

apiVersion: v1
kind: Service
metadata:
  name: fleetman-staff-service
spec:
  selector:
    app: staff-service
  ports:
    - name: http
      port: 8080
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: staff-service
spec:
  selector:
    matchLabels:
      app: staff-service
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: staff-service
        version: safe
    spec:
      containers:
      - image: richardchesterwood/istio-fleetman-staff-service:6-old
        name: staff-service        
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: staff-service-risky-version
spec:
  selector:
    matchLabels:
      app: staff-service
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: staff-service
        version: risky
    spec:
      containers:
      - name: staff-service
        image: richardchesterwood/istio-fleetman-staff-service:6-new
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always
        ports:
        - containerPort: 8080

```

#### use VirtualService and DestinationRule do canary release
```yaml

kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-staff-service-vs  # "just" a name for this virtualservice
  namespace: default
spec:
  hosts:
    - fleetman-staff-service.default.svc.cluster.local  # The Service DNS (ie the regular K8S Service) name that we're applying routing rules to.
  http:
    - route:
        - destination:
            host: fleetman-staff-service.default.svc.cluster.local # The Target DNS name
            subset: safe-group  # The name defined in the DestinationRule
          weight: 90
        - destination:
            host: fleetman-staff-service.default.svc.cluster.local # The Target DNS name
            subset: risky-group  # The name defined in the DestinationRule
          weight: 10


---

kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-staff-service-canary-release # This can be anything you like.
  namespace: default
spec:
  host: fleetman-staff-service # Service
  subsets:
    - labels:   # SELECTOR.
        version: safe # find pods with label "safe"
      name: safe-group
    - labels:
        version: risky
      name: risky-group

```

```
# virtual service
kubectl get vs
 
# Destination Rule
kubectl get dr

```


## Uninstall

```
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```
