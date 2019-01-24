# Setup

Deploy the two services to the K8S cluster.

```bash
kubectl apply -f ServiceMesh-GreeterWeather/k8s/greeter-weather-svc.yaml
kubectl apply -f WeatherProxy/k8s/weather-proxy-svc.yaml
```

The yaml files create the Service and Deployment resources in the Kubernetes environment. To verify that the services are deployed and running:

```bash
$ kubectl get po
NAME                                         READY   STATUS    RESTARTS   AGE
greetandweather-service-v1-94964667f-bcbdz   2/2     Running   0          3h6m
weather-proxy-service-v1-954db49dd-z9ltj     2/2     Running   0          3h5m
```

In addition to the verification that the services are running also make sure that the “Ready” shows 2/2 for each service. There needs to be 2 containers in each pod; one is the actual service and the second container is the Envoy proxy – commonly referred to as the sidecar.

Before the services are accessible you need to deploy some Istio resources; the ingress gateway, virtualservice, destination rule, and a ServiceEntry to add the external endpoint in the IPTable.

Creates the ingress gateway that will front our services

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/greeter-weather-gateway.yaml
```

Creates the Istio resource of kind virtualservice

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/greeter-weather-virtualservice.yaml
```

Creates the Istio resource of kind DestinationRule

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/dest-rule
```

Creates an egress gateway. This allows services that are external to the service mesh be accessible. Services that are external to the service mesh are not known to the Envoy proxies; therefore, when a request is made from the service the Envoy will not know how to route to the requested endpoint. The ServiceEntry places an entry in the IPTable. The proxy will then know how to route the request to the external endpoint.

```bash
$ kubectl apply -f ServiceMesh-GreeterWeather/istio/weather-serviceEntry.yaml
```

Let’s verify that all resources are created, deployed, and operational.

```bash
$ kubectl get gateway
NAME      AGE
gateway   3h

$ kubectl get svc -n istio-system
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
grafana                  ClusterIP      10.100.34.184   <none>        3000/TCP
istio-citadel            ClusterIP      10.109.206.80   <none>        8060/TCP,9093/TCP
istio-egressgateway      ClusterIP      10.98.123.129   <none>        80/TCP,443/TCP
istio-galley             ClusterIP      10.99.119.75    <none>        443/TCP,9093/TCP
istio-ingressgateway     LoadBalancer   10.96.92.55     localhost     80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:31112/TCP,8060:32671/TCP,853:32565/TCP,15030:32584/TCP,15031:30500/TCP   3h34m
istio-pilot              ClusterIP      10.106.87.120   <none>        15010/TCP,15011/TCP,8080/TCP,9093/TCP
istio-policy             ClusterIP      10.111.167.26   <none>        9091/TCP,15004/TCP,9093/TCP
istio-sidecar-injector   ClusterIP      10.96.242.5     <none>        443/TCP
istio-telemetry          ClusterIP      10.106.72.214   <none>        9091/TCP,15004/TCP,9093/TCP,42422/TCP
jaeger-agent             ClusterIP      None            <none>        5775/UDP,6831/UDP,6832/UDP
jaeger-collector         ClusterIP      10.104.234.26   <none>        14267/TCP,14268/TCP
jaeger-query             ClusterIP      10.97.171.131   <none>        16686/TCP
prometheus               ClusterIP      10.98.222.19    <none>        9090/TCP
servicegraph             ClusterIP      10.103.20.163   <none>        8088/TCP
tracing                  ClusterIP      10.109.10.118   <none>        80/TCP
zipkin                   ClusterIP      10.99.46.113    <none>        9411/TCP
```

The key component in this list is the istio-ingressgateway. You will notice that it is a type of LoadBalancer and its external IP is localhost, for this deployment. The reason it is localhost in this deployment is because I am using the local Kubernetes cluster in Docker-CE. On the hand, if this was a true external Kubernetes cluster, such as one deployed on Oracle’s OKE, you would be provided a valid external IP address.

There are two services that we will be exercising. The services can be invoked through curl using one of the two URLs.

`http://<ingress-gateway-IP>/greet`: Sends a generic hello message back to the requester
`http://<ingress-gateway-IP>/weather/92022/units/imperial`: Returns the current weather conditions for the US city with the zip code of 92022.

# mTLS Authentication

With all of the key components deployed let’s now enforce mTLS between the services. When you invoke with the uri of `/greet` the greeter-weather service handles the entire request. However, when invoking the uri `/weather/92022/units/imperial` the initial service accepts the request and then routes the request to the weather-proxy service. We want to enable mTLS between these two services.

Kubernetes has two types of clients; humans and Pods – your applications. Since Kubernetes does not store any resource of type human / user, all user information must be kept in an external system, such as a LDAP server.

To enable mTLS authentication we will use the default service accounts for the authentication.

Enable mTLS between the two services.

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/mtls-to-weatherproxy.yaml
```

Setup mTLS from the greeter-weather to the weather-proxy service. This informs the client-side proxy to initiate a mTLS communication with the server-side proxy.

Invoking the uri `/weather/92022/units/imperial` will result in a 503 error. The reason being you have only set up one side of the communication. The server-side proxy needs to be configured with a DestinationRule that sets a trafficPolicy for tls.

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/mtls-to-weatherproxy-destrule.yaml
```

Exercising the request again will result in a valid response from the weather-proxy service.

# Authorization

This next section we will discuss authorization.

For this exercise to demonstrate authorization we will use the Kubernetes service accounts.

First, let’s remove the Services and Deployments that were done previously.

```bash
kubectl delete -f ServiceMesh-GreeterWeather/k8s/greeter-weather-svc.yaml
kubectl delete -f ServiceMesh-GreeterWeather/istio/greeter-weather-virtualservice.yaml
kubectl delete -f ServiceMesh-GreeterWeather/istio/dest-rule.yaml
kubectl delete -f WeatherProxy/k8s/weather-proxy-svc.yaml
```

Deploy the services, but they will now be deployed with ServiceAccounts and into different namespaces.

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/authorization/greeter-weather-svc-sa.yaml
kubectl apply -f ServiceMesh-GreeterWeather/istio/authorization/greeter-weather-virtualservice.yaml
kubectl apply -f ServiceMesh-GreeterWeather/istio/authorization/dest-rule.yaml
kubectl apply -f WeatherProxy/istio/authorization/weather-proxy-svc-sa.yaml

kubectl get sa
kubectl get sa -n greet-ns

curl -v http://<gateway-ip>/weather/92022/units/imperial
```

Get a successful response

Enable authorization by deploying a new resource RbacConfig.

```
kubectl apply -f ServiceMesh-GreeterWeather/istio/authorization/rbac-config-enable.yaml
```

Execute the same previous curl command and this time the request will fail.

## Setup namespace level authorization

Take a look at the yaml file to make sure you understand what is being configured.

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/authorization/namespaceAuth/greeter-weather-namespace-policy.yaml
```

The defined ServiceRole specifies the following – each must be satisfied:

-   Applies to all services
-   The HTTP method must be a GET
-   The constraints are “the destination.labels[app] must be “weather-proxy”. If there were other services to be added to the values array then they would be comma-separated.

The defined ServiceRoleBinding specifies the following:

-   References the ServiceRole “greeter-weather-requester”
-   The user/subject making the request must be from one of the namespaces identified. In our specific case, the request to weather-proxy service is being made from the “greet-ns” namespace.

```bash
curl http://<gateway-ip>/weather/92022/units/imperial
```

Let’s remove the namespace authorization and then use authorization for a specific service and by a specific user and the all users.

```bash
kubectl delete -f ServiceMesh-GreeterWeather/istio/authorization/namespaceAuth/greeter-weather-namespace-policy.yaml
```

Apply the authorization by a specific user:

```bash
kubectl apply -f ServiceMesh-GreeterWeather/istio/authorization/serviceLevelAccess/specificuser.yaml
```

Review the yaml file to determine who the actual user is that is making the request. Where did this user come from?
