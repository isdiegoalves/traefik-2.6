# Traefik 2.6 webinar

## Traefik 2.6 with Consul Catalog and Consul KV 

- Create Hashicorp Cloud platform account 
- Create Consul Cluster using the Hashicorp Cloud Platform UI - it usually takes around 10 minutes
- Create peering connection between Consul Cluster and the Kubernetes cluster. Currently, the only AWS is possible. 
- Once peering connection is established you should be able to ping the Consul endpoint from the cluster

```sh
 ping consul-cluster.private.consul.50108fb6-25db-4763-b37f-16866f03465a.aws.hashicorp.cloud
```
- get the token
- download the client configuration files 

```sh
kubectl create secret generic "consul-ca-cert" --from-file='tls.crt=./ca.pem'
```

```
kubectl create secret generic "consul-gossip-key" --from-literal="key=$(jq -r .encrypt client_config.json)"

```

export CONSUL_HTTP_TOKEN="4fe3a3a1-603b-cd4c-34de-9e942e2aa037"

```
kubectl create secret generic "consul-bootstrap-token" --from-literal="token=${CONSUL_HTTP_TOKEN}"
```

- proceed with Consul Agent installation thorugh HELM 
```
 helm install consul -f values.yaml hashicorp/consul --version "0.32.1" --set global.image=hashicorp/consul-enterprise:1.11.2-ent
```

```
 k exec -it consul-597zz -- consul members -token b0aac0df-8e6b-4356-79ea-298035227ee0
Defaulted container "consul" out of: consul, client-acl-init (init)
Node                                         Address               Status  Type    Build       Protocol  DC              Partition  Segment
ip-172-25-18-119                             172.25.18.119:8301    alive   server  1.11.2+ent  2         consul-cluster  default    <all>
ip-172-20-42-156.eu-west-3.compute.internal  100.103.182.87:8301   alive   client  1.11.2+ent  2         consul-cluster  default    <default>
ip-172-20-45-56.eu-west-3.compute.internal   100.100.175.142:8301  alive   client  1.11.2+ent  2         consul-cluster  default    <default>
ip-172-20-52-241.eu-west-3.compute.internal  100.123.246.162:8301  alive   client  1.11.2+ent  2         consul-cluster  default    <default>
```

### create Consul namespaces

The namespace `traefik` is used to keep additional Traefik's configuration e.g. middlewares. 
```
 k exec -it consul-597zz -n consul-agent -- consul namespace create -name traefik  -token <SECRET_TOKEN>
Defaulted container "consul" out of: consul, client-acl-init (init)
Name: traefik
Partition:   default
```

The namespace `apps` is used to keep discovered services from Kubernetes to Consul Catalog. 

```
❯ k exec -it consul-597zz -n consul-agent -- consul namespace create -name apps  -token <SECRET_TOKEN>
Defaulted container "consul" out of: consul, client-acl-init (init)
Name: apps
Partition:   default
```

Create the Stripprefix middleware in the Consul KV datastore: 

```sh
 k exec -it consul-597zz -n consul-agent -- consul kv put -token <SECRET_TOKEN> -namespace traefik traefik/http/middlewares/stripfoo/stripprefix/0 /foo
Defaulted container "consul" out of: consul, client-acl-init (init)
Success! Data written to: traefik/http/middlewares/stripfoo/stripprefix/0
```

## Gateway API 

The Traefik Gateway provider that is available in Traefik 2.6 supports v0.4.0 (v1alpha2)  and can be configured using the following resources: 

- [Simple Gateway](https://gateway-api.sigs.k8s.io/v1alpha2/guides/simple-gateway/)
- [HTTP Routing](https://gateway-api.sigs.k8s.io/v1alpha2/guides/http-routing/)
- [TLS](https://gateway-api.sigs.k8s.io/v1alpha2/guides/tls/)

Let's follow the steps to play with Traefik Gateway Provider

1. Deploy Gateway API CRD's:

```sh
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.4.0" \
| kubectl apply -f -
```
Alternatively, all CRD's are placed in a dedicated directory. (01-gateway-api-crds) 

2.  Deploying Traefik Proxy 2.6

The following command will deploy all necassary files including i
 - namespace, 
 - Service Account, 
 - RBAC for Traefik Proxy as well as Gateway API resources
 - Kubernetes Service Load Balancer
 - Dashboard

```sh
kubectl apply -f 02-traefik
```

Please note that experminal features have been added to the static configuration. 

```sh
- --experimental.kubernetesgateway=true
- --providers.kubernetesgateway=true
```

The dashabord can be accessed by creating `port-forward`, execute the following command 

```sh
kubectl -n traefik port-forward  $(kubectl get pods -l "app=traefik" -n traefik --output=name) 9000:9000
```

and than navigate to `http://localhost:9000/dashboard/`

3. Create `GatewaClass` resource that creates a new controller called `traefik.io/gateway-controller`

```sh
kubectl apply -f 03-gateway-config
```

```yaml
cat << EOF | kubectl apply -f - 
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: GatewayClass
metadata:
  name: traefik-gateway-class
spec:
  controllerName: traefik.io/gateway-controller
EOF
```
4. Deploying sample applications. 

We will deploy two test commonly known WHOAMI applications. Those two deployments are nearly the same, the difference is in the name of the application. We will be using those two deployments to present traffic splitting. 

```sh
kubectl apply -f 04-app/01-whoami.yaml
```

5. Create Gateway resource that referrs to the previously created Gateway Class. The gateway create also listeners, such as HTTP, HTTPS, and TCP. 

Please note that HTTPS refers to the Kubernetes secret with TLS cerificate. In our case we use just a dummy certificate. It can be created using openssl command 

```sh
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem -subj "/CN=traefik26.demo.traefiklabs.tech"
```
```sh
kubectl create secret tls self-signed --cert=certificate.pem --key=key.pem
```
The TLS certificate can be easily created by executing:

```sh
kubectl apply -f 04-app/02-tls-self-signed.yaml
```

Let's create Gateway resource: 


```yaml
cat << EOF | kubectl apply -f - 
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
metadata:
  name: traefik-gateway
  namespace: default
spec:
  gatewayClassName: traefik-gateway-class  
  listeners: # Use GatewayClass defaults for listener definition.
    - name: http
      protocol: HTTP
      port: 80

    - name: https  
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate # Passthrough
        certificateRefs:
          - kind: Secret
            name: self-signed
EOF	    
```	

6. Create the basic HTTP router to expose the whoami application: 

```yaml
cat << EOF | kubectl apply -f - 
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: whoami-app
  namespace: default
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: default
  hostnames:
    - whoami.f1.demo.traefiklabs.tech # make sure to update the hostname 
  rules:
    - matches:
        - path:
            type: Exact
            value: /

      backendRefs:
        - name: whoamiv1
          port: 80
          weight: 1
EOF	  
```

Once the HTTP router has been created you should be able to reach the application using the curl command: `curl -k https://whoami.f1.demo.traefiklabs.tech`. 

7. Create the Traffic Splitting HTTP router. 

In that example we will use built-in feature to split traffic between to created services. Based on the defined weights, the incoming HTTP requests will be reaching those two application that we deployed in the previous steps.


```yaml
cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: whoami-splitting
  namespace: default
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: default
  hostnames:
    - canary.f1.demo.traefiklabs.tech
  rules:
    - backendRefs:
      - name: whoamiv1
        port: 80
        weight: 1
      - name: whoamiv2
        port: 80
        weight: 5
EOF
```
Once the HTTP router is created, you can verify it by using `curl -k https://canary.f1.demo.traefiklabs.tech`. The reponse should be returned from services `whoaamiv1` and `whoamiv2` based on the defined weights. 

8. Create similar service to previous one, but with custom header. By attaching the customer header the request will be redirected to the only selected application. That is a great feature that should help with performing QA testing. 


```yaml
cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: whoami-splitting-header
  namespace: default

spec:
  parentRefs:
    - name: traefik-gateway
      namespace: default

  hostnames:
    - canary-header.f1.demo.traefiklabs.tech

  rules:
    - backendRefs:
      - name: whoamiv1
        port: 80
    - matches:
      - headers:
        - name: X-canary-header
          value: traefik 
      backendRefs:
       - name: whoamiv2
         port: 80
EOF
```

If the custom header will be added to the request it will reach only `whoamiv2` applications. 

```sh
curl https://canary-header.f1.demo.traefiklabs.tech -k -H "X-Canary-header: traefik"
```

Without specifing the header, the `whoamiv1` application will be always reached. 

## HTTP/3

Follow the tutorial at this repository:  https://github.com/traefik-tech-blog/traefik-http3

## Inflight Connections Middleware

- Add the following configuration to the static configuration to manage incoming TCP connections:

```sh
--entryPoints.footcp.address=:3179
```
- Create Load Balancer type service to the newly created TCP entrypoint.
- Deploy test application `whoamitcp`
- Deploy Inflight Connections Middleware and IngressRouteTCP to expose `whoamitcp` service.
- Validate connectivity by using `netcat` and the endpoint created by load balancer type service (`kubectl get svc/trefik-tcp`)