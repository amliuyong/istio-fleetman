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

## Install Kiali and the other addons 
```
cd istio-1.8.2

kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system


istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger

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
#### you can also use headers to route the service for `dark release`

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-staff-service
  namespace: default
spec:
  hosts:
    - fleetman-staff-service
  http:
    - match:
        - headers:
            x-my-header:
              exact: canary
      route:
        - destination:
            host: fleetman-staff-service
            subset: risky-group
    - route:
        - destination:
            host: fleetman-staff-service
            subset: safe-group
```

### Stickiness

set `trafficPolicy.loadBalancer.consistentHash` in DestinationRule


- httpHeaderName
- httpCookie	
- useSourceIp	 
- httpQueryParameterName	
- minimumRingSize

https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings-ConsistentHashLB

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: a-set-of-routing-rules-we-can-call-this-anything  # "just" a name for this virtualservice
  namespace: default
spec:
  hosts:
    - fleetman-staff-service.default.svc.cluster.local  # The Service DNS (ie the regular K8S Service) name that we're applying routing rules to.
  http:
    - route:
        - destination:
            host: fleetman-staff-service.default.svc.cluster.local # The Target DNS name
            subset: all-staff-service-pods  # The name defined in the DestinationRule
          # weight: 100 not needed if there's only one.

---

kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: grouping-rules-for-our-photograph-canary-release # This can be anything you like.
  namespace: default
spec:
  host: fleetman-staff-service # Service
  trafficPolicy:  #  https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings-ConsistentHashLB
    loadBalancer:
      consistentHash:
        httpHeaderName: "x-myval"
  subsets:
    - labels:   # SELECTOR.
        app: staff-service # find pods with label "safe"
      name: all-staff-service-pods


```
#### Propagate Headers in Spring use Interceptor
https://github.com/amliuyong/istio-fleetman/blob/master/istio-fleetman-api-gateway/src/main/java/com/virtualpairprogrammers/api/config/PropagateHeadersInterceptor.java

```java

@Component
public class PropagateHeadersInterceptor implements RequestInterceptor {
	
	private @Autowired HttpServletRequest request;

	public void apply(RequestTemplate template) {
		try
		{
			Enumeration<String> e = request.getHeaderNames();
			while (e.hasMoreElements())
			{
				String header = e.nextElement();
				if (header.startsWith("x-"))
				{
					String value = request.getHeader(header);
					template.header(header, value);
				}
			}
		}
		catch (IllegalStateException e) {}
	}
}

```

## Ingress Gateway

#### service `fleetman-webapp`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: webapp
        version: original
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/istio-fleetman-webapp-angular:6
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always

---


apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-experimental
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: webapp
        version: experimental
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/istio-fleetman-webapp-angular:6-experimental
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always

---

apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either other services
  # or maybe external users to connect to (eg browser)
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
  type: ClusterIP

```
#### Gateway for `fleetman-webapp`
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
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
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*" # Copy the value in the gateway hosts - usually a Domain Name
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
        - destination:
            host: fleetman-webapp
            subset: original
          weight: 90
        - destination:
            host: fleetman-webapp
            subset: experimental
          weight: 10
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  host: fleetman-webapp
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental

```

```
cd '_course_files/3 Gateways Solution'

kubectl apply -f 5-application-no-istio.yaml

kubectl apply -f 6-istio-rules.yaml


kubectl get svc -A | grep istio-ingressgateway
istio-system   istio-ingressgateway         LoadBalancer   10.97.27.66      localhost     15021:30064/TCP,80:31580/TCP,443:31458/TCP,31400:31581/TCP,15443:30582/TCP   9h

http://localhost

```
## Gateway - prefix routing
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: webapp
        version: original
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/istio-fleetman-webapp-angular:6
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-experimental
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: webapp
        version: experimental
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/istio-fleetman-webapp-angular:6-experimental
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either other services
  # or maybe external users to connect to (eg browser)
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080
  type: NodePort

```
```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
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

---

kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*"
  gateways:
    - ingress-gateway-configuration
  http:
    - match:
      - uri:  # IF
          prefix: "/experimental"
      - uri:  # OR
          prefix: "/canary"
      route: # THEN
      - destination:
          host: fleetman-webapp
          subset: experimental
    - match:
      - uri :
          prefix: "/"
      route:
      - destination:
          host: fleetman-webapp
          subset: original

---

kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  host: fleetman-webapp
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental


```
```
cd '_course_files/3 Gateways Solution - Prefix Routing'

kubectl apply -f 5-application-no-istio.yaml
kubectl apply -f 6-istio-rules.yaml

# test
curl -s localhost/canary  | grep title
curl -s localhost  | grep title

```

## Gateway - Subdomain Routing
```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.fleetman.com"   # Domain name of the external website
    - "fleetman.com"

---

kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "fleetman.com"
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
      - destination:
          host: fleetman-webapp
          subset: original

---

kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp-experiment
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "experimental.fleetman.com"
  gateways:
    - ingress-gateway-configuration
  http:
      - route:
        - destination:
            host: fleetman-webapp
            subset: experimental

---

kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  host: fleetman-webapp
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental

```
etc/hosts
```
127.0.0.1       fleetman.com
127.0.0.1       experimental.fleetman.com
```

```
cd '_course_files/3 Gateways Solution - Subdomain Routing'

kubectl apply -f 5-application-no-istio.yaml
kubectl apply -f 6-istio-rules.yaml

http://experimental.fleetman.com/
http://fleetman.com/

```

## Gateway - Headers Routing
```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
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

---

kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*"
  gateways:
    - ingress-gateway-configuration
  http:
    - match:
      - headers:  # IF
          my-header:
            exact: canary
      route: # THEN
      - destination:
          host: fleetman-webapp
          subset: experimental
    - route: # CATCH ALL
      - destination:
          host: fleetman-webapp
          subset: original
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  host: fleetman-webapp
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental
      
```
```
cd '_course_files/4 Dark Releases'

kubectl apply -f 5-application-no-istio.yaml
kubectl apply -f 6-istio-rules.yaml


curl -s -H 'my-header: canary' localhost  | grep title
curl -s  localhost | grep title

```

## Fault Injection

```yaml

kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-vehicle-telemetry
  namespace: default
spec:
  hosts:
    - fleetman-vehicle-telemetry
  http:
    - fault:
        abort:
          httpStatus: 503
          percentage:
            value: 100
      route:
        - destination:
            host: fleetman-vehicle-telemetry


```

## Circuit Breaking

https://github.com/Netflix/Hystrix

####  ingress-gateway for webapp
```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
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
---
# All traffic routed to the fleetman-webapp service
# No DestinationRule needed as we aren't doing any subsets, load balancing or outlier detection.
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*"
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
      - destination:
          host: fleetman-webapp
```

#### config circuit-breaker for `fleetman-staff-service`
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker-for-the-entire-default-namespace
spec:
  host: "fleetman-staff-service.default.svc.cluster.local" # This is the name of the k8s service that we're configuring

  trafficPolicy:
    outlierDetection: # Circuit Breakers HAVE TO BE SWITCHED ON
      maxEjectionPercent: 100
      consecutive5xxErrors: 2
      interval: 10s
      baseEjectionTime: 30s


```

```
cd '_course_files/6 Circuit Breaking'

kubectl apply -f 5-application-no-istio.yaml
kubectl apply -f 6-gateway.yaml 
kubectl apply -f 7-circuit-breaking.yaml

```













## Uninstall

```
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```


