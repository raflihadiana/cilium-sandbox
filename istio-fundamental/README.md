# Istio Fundamental Labs

One of the quickest ways to get started with Istio is to leverage the demo profile. The demo profile is designed to showcase Istio functionality with modest resource requirements. The demo profile contains an Istio control plane (also called Istiod), Istio ingress-gateway and egress-gateway, and a few add-on components.

## Install Istio

In this lab, you will install Istio with the demo profile. You will validate the installation is successful and examine the installation artifacts. While you can use either istioctl, Helm, or the Istio operator to install Istio, in this lab you will use istioctl.

Note: If you open the Prometheus/Grafana/Jaeger/Kiali UI tab, you will see the "Please wait, we are trying to connect to the service..." message. This is normal before these services are installed.

### Download Istio

1. Download the Istio release binary:

    ```
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
    ```

2. Add the istioctl client to the PATH:

    ```
    export PATH=$PWD/istio-${ISTIO_VERSION}/bin:$PATH
    ```

3. Check istioctl version:

    ```
    istioctl version
    ```

4. Check if your Kubernetes environment meets Istio's platform requirement:

    ```
    istioctl x precheck
    ```

The precheck response should indicate that no issues were found:

    ✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
        To get started, check out https://istio.io/latest/docs/setup/getting-started/

### Install Istio

1. List available installation profiles:

    ```
    istioctl profile list
    ```

2. Since this is a getting started workshop, you will use the demo profile to install Istio.

    ```
    istioctl install --set profile=demo -y
    ```

3. You should see output that indicates each Istio component is installed successfully. Check out the resources installed by Istio:

    ```
    kubectl get all,cm,secrets,envoyfilters -n istio-system
    ```

4. Check out Custom Resource Definitions (CRDs) installed by Istio:

    ```
    kubectl get crds -n istio-system
    ```

5. Verify the installation using the following command:

    ```
    istioctl verify-install
    ```

    You should see the following at the end of the output to indicate that your Istio is installed successfully:

    ✔ Istio is installed and verified successfully

### Install Istio Telemetry Add-ons

Istio telemetry add-ons are shipped as samples, but these add-ons are optimized for quick getting started and demo purposes and not for production usage. They provides a convenient way to install telemetry components that integrate with Istio.

```
kubectl apply -f istio-${ISTIO_VERSION}/samples/addons
```

1. Wait till all pods in the istio-system are running:

    ```
    kubectl get pods -n istio-system
    ```

2. Enable access to the Prometheus dashboard:

    ```
    istioctl dashboard prometheus --browser=false --address 0.0.0.0
    ```

Click on the Prometheus UI tab, you should be able to view the Prometheus UI from there. Press ctrl+C to end the prior command, and use the command below to enable access to the Grafana dashboard:

```
istioctl dashboard grafana --browser=false --address 0.0.0.0
```

Click on the Grafana UI tab, and you should be able to view the Grafana UI. Press ctrl+C to end the prior command, and use the command below to enable access to the Jaeger dashboard:

```
istioctl dashboard jaeger --browser=false --address 0.0.0.0
```

Click on the Jaeger UI tab, and you should be able to view the Jaeger UI. Press ctrl+C to end the prior command, and use the command below to enable access to the Kiali dashboard:

```
istioctl dashboard kiali --browser=false --address 0.0.0.0
```

Click on the Kiali UI tab, and you should be able to view the Kiali UI. Press ctrl+C to end the prior command. You will not see much telemetry data on any of these dashboards now, as you don't have any services defined in the Istio service mesh yet. You will revisit these dashboards soon.

## 🚩Istio Ingress Gateway

In this lab, you will deploy a sample application to your Kubernetes cluster, expose the web-api service to the Istio ingress gateway, and configure secure access to the service. The ingress gateway allows traffic into the mesh. If you need more sophisticated edge gateway capabilities (such as request transformation, OIDC, LDAP, OPA, etc.) then you should use a gateway specifically built for those use cases like Gloo Edge.

### Prerequisites

Verify you're in the correct folder for this lab: /root/istio-workshops/istio-basics. This lab builds on the first lab where you installed Istio and its add-on components using the demo profile.

```
cd /root/istio-workshops/istio-basics
```

### Deploy the sample application

You will use the web-api, recommendation, and purchase-history services built using the fake service as your sample application. The web-api service calls the recommendation service via HTTP, and the recommendation service calls the purchase-history service, also via HTTP.

1. Set up the istioinaction namespace for our services:

    ```
    kubectl create ns istioinaction
    ```

2. Deploy the web-api, recommendation and purchase-history services along with the sleep service into the istioinaction namespace:

    ```
    kubectl apply -n istioinaction -f sample-apps/web-api.yaml
    kubectl apply -n istioinaction -f sample-apps/recommendation.yaml
    kubectl apply -n istioinaction -f sample-apps/purchase-history-v1.yaml
    kubectl apply -n istioinaction -f sample-apps/sleep.yaml
    ```

3. After running these commands, you should check that all pods are running in the istioinaction namespace:

    ```
    kubectl get po -n istioinaction
    ```

Wait a few seconds until all of them show a Running status.

### Configure the inbound traffic

The Istio ingress gateway will create a Kubernetes Service of type LoadBalancer. Use this GATEWAY_IP address to reach the gateway:

```
kubectl get svc -n istio-system
```

Store the ingress gateway IP address in an environment variable.

```
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
export INGRESS_PORT=80
export SECURE_INGRESS_PORT=443
```

### Expose our apps

![Alt text](diagram-istio-ingress.png)

Even though you don't have apps defined in the istioinaction namespace in the mesh yet, you can still use the Istio ingress gateway to route traffic to them. Using Istio's Gateway resource, you can configure what ports should be exposed, what protocol to use, etc. Using Istio's VirtualService resource, you can configure how to route traffic from the Istio ingress gateway to your web-api service.

1. Review the Gateway resource:

    ```
    cat sample-apps/ingress/web-api-gw.yaml
    ```

2. Review the VirtualService resource:

    ```
    cat sample-apps/ingress/web-api-gw-vs.yaml
    ```
Why is port number 8080 shown in the destination route configuration for the web-api-gw-vs VirtualService resource? Check the service port for the web-api service in the istioinaction namespace:

```
kubectl get service web-api -n istioinaction
```

You can see the service listens on port 8080.

3. Apply the Gateway and VirtualService resources to expose your web-api service outside of the Kubernetes cluster:

    ```
    kubectl -n istioinaction apply -f sample-apps/ingress/
    ```

The Istio ingress gateway will create new routes on the proxy that you should be able to call from outside of the Kubernetes cluster:

```
curl -H "Host: istioinaction.io" http://$GATEWAY_IP:$INGRESS_PORT
```

4. Query the gateway configuration using the istioctl proxy-config command:

    ```
    istioctl proxy-config routes deploy/istio-ingressgateway.istio-system
    ```

5. If you want to see an individual route, you can ask for its output as json like this:

    ```
    istioctl proxy-config routes deploy/istio-ingressgateway.istio-system --name http.8080 -o json
    ```

### Secure the inbound traffic

![Secure Istio Ingress Gateway](diagram-secure-istio-ingress.png)

To secure inbound traffic with HTTPS, you need a certificate with the appropriate SAN and you will need to configure the Istio ingress-gateway to use it.

1. Create a TLS secret for istioinaction.io in the istio-system namespace:

    ```
    kubectl create -n istio-system secret tls istioinaction-cert --key labs/02/certs/istioinaction.io.key --cert labs/02/certs/istioinaction.io.crt
    ```

2. Update the Istio ingress-gateway to use this cert:

    ```
    cat labs/02/web-api-gw-https.yaml
    ```

        Note, we are pointing to the istioinaction-cert and that the cert must be in the same namespace as the ingress gateway deployment. Even though the Gateway resource is in the istioinaction namespace, the cert must be where the gateway is actually deployed.

3. Apply the web-api-gw-https.yaml in the istioinaction namespace. Since this gateway resource is also called web-api-gateway, it will replace our prior web-api-gateway configuration for port 80.

    ```
    kubectl -n istioinaction apply -f labs/02/web-api-gw-https.yaml
    ```

4. Call the web-api service through the Istio ingress-gateway on the secure 443 port:

    ```
    curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
    ```

If you call it on the 80 port with http, it will not work as you no longer have the gateway resource configured to be exposed on port 80.

```
curl -H "Host: istioinaction.io" http://$GATEWAY_IP:$INGRESS_PORT
```

### Mesh Metrics

Now that we have an Istio gateway running and receiving traffic, we can now visualize information about the traffic we just generated.

Click on the Grafana UI tab and we should start to see some mesh metrics appearing.

## Adding Services to the Mesh

In this lab, you will incrementally add services to the mesh. The mesh is actually integrated with the services themselves which makes it mostly transparent to the service implementation.

### Sidecar injection

Adding services to the mesh requires that the client-side proxies be associated with the service components and registered with the control plane. With Istio, you have two methods to inject the Envoy Proxy sidecar into the microservice Kubernetes pods:

- Automatic sidecar injection
- Manual sidecar injection.

1. To enable the automatic sidecar injection, use the command below to add the istio-injection label to the istioinaction namespace:
    ```
    kubectl label namespace istioinaction istio-injection=enabled
    ```

2. Validate the istioinaction namespace is annotated with the istio-injection label:
    ```
    kubectl get namespace -L istio-injection
    ```

Now that you have an istioinaction namespace with automatic sidecar injection enabled, you are ready to start adding services in your istioinaction namespace to the mesh. Since you added the istio-injection label to the istioinaction namespace, the Istio mutating admission controller automatically injects the Envoy Proxy sidecar during the initial deployment or restart of the pod.

### Review Service requirements

Before you add Kubernetes services to the mesh, you need to be aware of the [application requirements](https://istio.io/latest/docs/ops/deployment/requirements/) to ensure that your Kubernetes services meet the minimum requirements.

**Service descriptors:**

- each service port name must start with the protocol name, for example name: http

**Deployment descriptors:**

- The pods must be associated with a Kubernetes service.
- The pods must not run as a user with UID 1337
- App and version labels are added to provide contextual information for metrics and tracing

Check the above requirements for each of the Kubernetes services and make adjustments as necessary. If you don't have NET_ADMIN security rights, you would need to explore the [Istio CNI plugin](https://istio.io/latest/docs/setup/additional-setup/cni/) to remove the NET_ADMIN requirement for deploying services.

Using the web-api service as an example, you can review its service and deployment descriptor:

```
cat sample-apps/web-api.yaml
```

From the service descriptor, the name: http declares the http protocol for the service port 8080:

```yaml
  - name: http
    protocol: TCP
    port: 8080
    targetPort: 8081
```

From the deployment descriptor, the app: web-api label matches the web-api service's selector of app: web-api so this deployment and its pod are associated with the web-api service. Further, the app: web-api label and version: v1 labels provide contextual information for metrics and tracing. The containerPort: 8081 declares the listening port for the container, which matches the targetPort: 8081 in the web-api service descriptor earlier.

```yaml
  template:
    metadata:
      labels:
        app: web-api
        version: v1
      annotations:
    spec:
      serviceAccountName: web-api
      containers:
      - name: web-api
        image: nicholasjackson/fake-service:v0.7.8
        ports:
        - containerPort: 8081
```

Check the purchase-history-v1, recommendation, and sleep services to validate they all meet the above requirements.

### Adding services to the mesh

1. You can add a sidecar to each of the services in the istioinaction namespace, starting with the web-api service:

    ```
    kubectl rollout restart deployment web-api -n istioinaction
    ```

2. Validate the web-api pod is running with Istio's default sidecar proxy injected:

    ```
    kubectl get pod -l app=web-api -n istioinaction
    ```

You should see 2/2 in the output. This indicates the sidecar proxy is running alongside the web-api application container in the web-api pod:

```
NAME                       READY   STATUS    RESTARTS   AGE
web-api-7d5ccfd7b4-m7lkj   2/2     Running   0          9m4s
```

3. Validate the web-api pod log looks good:

    ```
    kubectl logs deploy/web-api -c web-api -n istioinaction
    ```

4. Validate you can continue to call the web-api service securely:

    ```
    curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
    ```

### Understand what happens

Use the command below to get the details of the web-api pod:

```
kubectl get pod -l app=web-api -n istioinaction -o yaml
```

From the output, the web-api pod contains 1 init container and 2 normal containers. The Istio mutating admission controller was responsible for injecting the istio-init container and the istio-proxy container.

#### The istio-init container:

The istio-init container uses the proxyv2 image. The entry point of the container is pilot-agent, which contains the istio-iptables command to set up port forwarding for Istio's sidecar proxy.

```
    initContainers:
    - args:
      - istio-iptables
      - -p
      - "15001"
      - -z
      - "15006"
      - -u
      - "1337"
      - -m
      - REDIRECT
      - -i
      - '*'
      - -x
      - ""
      - -b
      - '*'
      - -d
      - 15090,15021,15020
      image: docker.io/istio/proxyv2:latest
      imagePullPolicy: Always
      name: istio-init
```

Interested in knowing more about the flags for istio-iptables? Run the following command:

```
kubectl exec deploy/web-api -c istio-proxy -n istioinaction -- /usr/local/bin/pilot-agent istio-iptables --help
```

The output explains the flags such as -u, -m, and -i which are used in the istio-init container's args. You will notice that all inbound ports are redirected to the Envoy Proxy container within the pod. You can also see a few ports such as 15021 which are excluded from redirection (you'll soon learn why this is the case). You may also notice the following securityContext for the istio-init container. This means that a service deployer must have the NET_ADMIN and NET_RAW security capabilities to run the istio-init container for the web-api service or other services in the Istio service mesh. If the service deployer can't have these security capabilities, you can use the Istio CNI plugin which removes the NET_ADMIN and NET_RAW requirement for users deploying pods into Istio service mesh.

#### The istio-proxy container:

When you continue looking through the list of containers in the pod, you will see the istio-proxy container. The istio-proxy container also uses the proxyv2 image. You'll notice the istio-proxy container has requested 0.01 CPU and 40 MB memory to start with as well as 2 CPU and 1 GB memory for limits. You will need to budget for these settings when managing the capacity for the cluster. These resources can be customized during the Istio installation thus may vary per your installation profile.

When you reviewed the istio-init container configuration earlier, you may have noticed that ports 15021, 15090, and 15020 are on the list of inbound ports to be excluded from redirection to Envoy. The reason is that port 15021 is for health check, and port 15090 is for the Envoy Proxy to emit its metrics to Prometheus, and port 15020 is for the merged Prometheus metrics colllected from the Istio agent, the Envoy Proxy, and the application container. Thus it is not necessary to redirect inbound traffic for these ports since they are being used by the istio-proxy container.

Also notice that the istiod-ca-cert and istio-token volumes are mounted on the pod for the purpose of implementing mutual TLS (mTLS), which will be covered in the lab 04.

### Add more services to the Istio service mesh

1. Next, you can add the istio-proxy sidecar to the other services in the istioinaction namespace

    ```
    kubectl rollout restart deployment purchase-history-v1 -n istioinaction
    kubectl rollout restart deployment recommendation -n istioinaction
    kubectl rollout restart deployment sleep -n istioinaction
    ```

2. Validate that all the pods in the istioinaction namespace are running with Istio's default sidecar proxy injected:

    ```
    kubectl get pods -n istioinaction
    ```

3. Validate that you can continue to call the web-api service securely:

    ```
    curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
    ```

### What have you gained?

Congratulations on adding services in the istioinaction namespace to the Istio service mesh. One of the values of using a service mesh is that you will gain immediate insight into the behavior and interactions between your services. Istio delivers a set of dashboards as add-on components that give you access to important telemetry data, just by adding services to the mesh.
Distributed tracing

You can view distributed tracing information using the Jaeger UI tab.

- Navigate to the Jaeger UI tab. On the "Service" dropdown, select "istio-ingressgateway". Click on the "Find Traces" button at the bottom. You should see some traces, which show every request to the web-api service through the Istio's ingress gateway.

- Click on one of the traces to view the details of the distributed traces for that request. You can click on each trace span to learn more about it. You may notice all trace spans have the same value for the x-request-id header. Why? This is how Jaeger knows these trace spans are part of the same request. In order for your services' distributed tracing to work properly in your Istio service mesh, the B-3 trace headers including x-request-id have to be propagated between your services.
