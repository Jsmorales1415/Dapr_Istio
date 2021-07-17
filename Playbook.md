# Kubernetes .NetCore services with Tye, Dapr and Istio
This solution use Project Tye for the services deployment on a Kubernetes environment, Dapr as sidecar and Istio as service discover.

## Init Project Tye 
From the rooth path, run the follow command to create the tye.yaml file with all the services integrated

```sh
tye init
```

After that, you will see a tye.yaml file created.

```yaml
# tye application configuration file
# read all about it at https://github.com/dotnet/tye
#
# when you've given us a try, we'd love to know what you think:
#    https://aka.ms/AA7q20u
#
name: darp_istio
services:
- name: servicea
  project: ServiceA/ServiceA.csproj
- name: serviceb
  project: ServiceB/ServiceB.csproj
```

## Init Dapr in your cluster
We are going to launch Dapr in our cluster, but also we need to disable the mTLS because we are going to let istio do that.

If Dapr is running yet, uninstall it before execute the init command:
```sh
dapr uninstall -k
```

Output:
```sh
ℹ️  Removing Dapr from your cluster...
✅  Dapr has been removed successfully
```

Init Dapr:
```sh
dapr init --enable-mtls=false -k
```

Output:
```sh
⌛  Making the jump to hyperspace...
ℹ️  Note: To install Dapr using Helm, see here: https://docs.dapr.io/getting-started/install-dapr-kubernetes/#install-with-helm-advanced

✅  Deploying the Dapr control plane to your cluster...
✅  Success! Dapr has been installed to namespace dapr-system. To verify, run `dapr status -k' in your terminal. To get started, go here: https://aka.ms/dapr-getting-started
```

Check the mutual tls on your cluster by executing:
```sh
dapr mtls -k
```

Expected output:
```sh
Mutual TLS is disabled in your Kubernetes cluster 
```

## Install and init Istio
Before this steps, make sure that you have installed istioctl on your pc(see references for help).

Install Istio
```sh
istioctl install --set profile=demo -y
```

Output
```sh
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
```

## Label the namespace
Look for the namespace used, it should have a label to indicate that it will inject automatically the istio inside the deployments

Show the default namespace labels:
```sh
kubectl get ns default --show-labels
```

Set an instio label on the namespace:
```sh
kubectl label namespace default istios-injection=enabled
```

Output:
```sh
namespace/default labeled
```

## Deploy the services
Complete the needed information in the tye.yaml file to set up all the services, then, Tye will help us to deploy 
```yaml
name: darp_istio

extensions:
- name: dapr

services:
- name: servicea
  project: ServiceA/ServiceA.csproj
  bindings:
  - protocol: http
    port: 5000  

- name: serviceb
  project: ServiceB/ServiceB.csproj
  bindings:
  - protocol: http
    port: 5001   
```

```sh
tye deploy -i tye.yaml
```

After that, a docker registry must be provide to push and build up the images.

## Deploy the gateway
The gateway is needed to allow the external requests, it also help to reject the requests that try to hit to different services inside the cluster, it become the ingress controller in the only accessible entry point.

```sh
kubectl apply -f istio_components/gateway.yaml
```

Inside the gateway.yaml

## Set the ports and the host for ingress 
```sh
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
```

If you are running on minikube, take it port as host.
```sh
export INGRESS_HOST=$(minikube ip)
```

For a different environment, try:
```sh
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
```

Finally, set the gateway url by executing:
```sh
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

## Calling the ServiceA from outside the cluster
Inside the Service A there is a call, where Service B is call and it give a basic information that is retrieved by the Service A.

Test:
```sh
curl http://192.168.64.14:31520/home
``` 

You can also call the service from browser.

## Define ingress rules
The rules to allow and manage the traffic must be definied as virtual services, every service that will be able to access from outside the cluster, should have its own VirtualService defining it routes, it versions, it subsets, and so on. Don't forget to specified the gateway to which they will obey.

Service A configuring:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-services-a
spec:
  hosts:
  - "192.168.64.14" 
  gateways:
  - services-gateway
  http:
  - match:
    - uri:
        prefix: /home
    route: 
    - destination:
        host: servicea.default.svc.cluster.local
```

Service B configuring:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-services-b
spec:
  hosts:
  - "192.168.64.14" 
  gateways:
  - services-gateway
  http:
  - match:
    - uri:
        prefix: /weatherforecast
    route: 
    - destination:
        host: serviceb.default.svc.cluster.local
```

## References
- [Dapr](https://docs.dapr.io/getting-started/install-dapr-cli/)
- [Istio](https://istio.io/latest/docs/setup/getting-started/)
- [Enable Istio](https://istio.io/latest/docs/examples/bookinfo/#deploying-the-application)
