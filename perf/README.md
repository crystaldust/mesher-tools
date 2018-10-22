## Mesher performance tests

### The test project

The project contains a simple http server and client. User first makes a call to the client, then client requests for server, after the server responses, the client responses the user with http code 200.

### Prerequisites

- Docker
- Kubernetes 1.9 or higher, with [MutatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) enabled

### Build the project

Run the `build.sh` script under `server/` and `client/` folder, the script will build the binaries with a container go environment and build the server and client images. After that, make the images available to the Kubernetes cluster. Then install the project with one of the following deployment types.

### The deployments

- **Original under Kubernetes**
The original project with Kubernetes. This is used as a benchmark standard. Deploy the project with the commands:
```bash
$ kubectl apply -f server/server-perf.yaml
$ kubectl apply -f client/client-perf.yaml
```

- **Deployed with Mesher**
The original project deployed with mesher as sidecar. [Install mesher control plane](#install-mesher-control-plane-components) before deploying it with the commands:
```bash
$ kubectl apply -f server/server-with-mesher.yaml
$ kubectl apply -f client/client-with-mesher.yaml
```

- **Deployed with Istio**
The original project deployed with envoy as sidecar. Make sure [Istio is installed](#install-istio) before deploying the projects:
```bash
$ kubectl apply -f server/server-with-istio.yaml
$ kubectl apply -f client/client-with-istio.yaml
```

### Install mesher control plane components
Mesher's core abilities(service discovery, handler chain...) are derived from [go-chassis](https://github.com/go-chassis/go-chassis), which takes [ServiceComb](http://servicecomb.io) as the control plane. Beyond that, mesher also provide a [sidecar-injector](https://github.com/go-mesh/sidecar-injector) to inject mesher as sidecar under Kubernetes. To install the complete control plane, run the script:

```
$ cd perf/servicecomb
$ bash -x install.sh
```

It will create a namespace `servicecomb` which contains service-center and sidecar-injector. Make sure the namespace for testing projects is labeled with `sidecar-injector=enabled`:

```bash
kubectl label namespace default istio-injection=enabled
```

### Install Istio

After extracting the istio archive, use [helm](https://helm.sh/) to generate the Istio deployment resources with custom options, the following options will make helm generate a minimum Istio with only essential components and sidecar-injection enabled.
```bash
helm template install/kubernetes/helm/istio --name istio --namespace istio-system \
                     --set ingress.enabled=false \
                     --set gateways.istio-ingressgateway.enabled=false \
                     --set gateways.istio-egressgateway.enabled=false \
                     --set galley.enabled=false \
                     --set sidecarInjectorWebhook.enabled=true \
                     --set mixer.enabled=false \
                     --set prometheus.enabled=false \
                     --set global.proxy.envoyStatsd.enabled=false > istio-min-withinjection.yaml
```

Then install Istio with the generated yaml:
```bash
$ kubectl apply -f istio-min-withinjection.yaml
```

Make sure the namespace for testing projects is labeled with `istio-injection`:
```bash
kubectl label namespace default istio-injection=enabled
```

### Run the tests
After deploying one of the 3 projects, make the client service available to the load test client by defining [externalIPs](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips) or changing the client service from NodePort to [load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) according to your Kubernetes cluster. Then make a load test on client([wrk](https://github.com/wg/wrk) as an example):
```bash
$ wrk -t12 -c40 -d60s http://{CLIENT-SERVICE-ADDR}:9000
```



### Sample test results

The sample result are taken from a 3 node virtual machine(4U4G) Kubernetes cluster, with a [wrk](https://github.com/wg/wrk) test command `wrk -t12 -c40 -d60s http://{CLIENT-SERVICE-ADDR}:9000`. The output might vary by different environments and test tools and even test arguments. So the side effect is more important than the absolute performance value. More test results(the consumption on CPU, memory; tests with governance abilities enabled, etc.) will be added in the future.

|                  | Original | With Mesher | With Istio(Min) |
| ---------------- | -------- | ----------- | --------------- |
| QPS              | 2303.69  | 1876.708    | 1756.6          |
| Reponse time(ms) | 17.702   | 19.48       | 20.874          |

