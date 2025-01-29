![image alt](https://github.com/Ashsatsan/microservices-googlecloud/blob/main/istioproj5.png?raw=true)

# Istio Project Documentation

## Overview
This project demonstrates a secure Istio configuration to manage microservices traffic. The setup ensures:

1. External traffic is restricted to only the `frontend` namespace.
2. Internal communication between services in the `backend` and `frontend` namespaces is secure and allowed.
3. Fine-grained traffic control and load balancing are implemented using Istio resources.

## Prerequisites
- Kubernetes cluster
- Istio installed (version 1.24.2)
- `kubectl` and `istioctl` CLI tools installed
- Domain configured (e.g., `frontend.satsan.site`)

---

## Project Structure

### Namespaces
- **frontend**: Hosts the `frontend` service and its associated resources.
- **backend**: Hosts the microservices that provide business logic and data.
- **istio-system**: Hosts Istio control plane components.

### Key Configuration Files

#### Authorization Policies
1. **deny-authorization.yml**: Denies traffic to `backend` namespace from outside the cluster.
2. **backend-internal.yml**: Allows communication between services within the `backend` namespace and `frontend` namespace.
3. **frontend-internal.yml**: Allows communication within the `frontend` namespace.
4. **frontend-authorization.yml**: Permits external traffic only to the `frontend` namespace while restricting access to `backend`.

#### Gateway
- **gateway.yml**: Configures the Istio ingress gateway to accept traffic on ports 443 and 80 only.

#### Peer Authentication
1. **peer.yml**: Configures mutual TLS (mTLS) for the `default` namespace.
2. **peer2.yml**: Configures mTLS for the `frontend` namespace.
   ```yaml
   apiVersion: security.istio.io/v1beta1
   kind: PeerAuthentication
   metadata:
     name: default
     namespace: frontend
   spec:
     mtls:
       mode: STRICT
   ```
3. **peer3.yml**: Configures mTLS for the `backend` namespace.

#### Virtual Services
1. **virtualservice.yml**: Routes traffic accessing the host to the corresponding service securely.
2. **virtualservice-canary.yml**: Configures canary deployment for the `cartservice` to route 20% of traffic to version 2 (`cartservice-v2`).

#### Destination Rules
- **destinationrule.yml**: Configures load balancing and traffic routing policies for the services.

---

## Deployment Steps

### Step 1: Istio Installation and Namespace Setup

Install Istio:
```bash
istioctl install 
```

Example output:
```bash
✔ Istio core installed
✔ Istiod installed
✔ Installation complete
```

If using a custom configuration: if want to see an example link: [configuration profiles]()
```bash
istioctl install -f istio-operator.yaml  
```
We can use the same file for better control and security for the smesh service

Create the frontend and backend namespaces:
```bash
kubectl create namespace frontend
kubectl create namespace backend
```

Enable automatic sidecar injection:
```bash
kubectl label namespace frontend istio-injection=enabled
kubectl label namespace backend istio-injection=enabled
```
verify label with :
```bash
kubectl get namespaces --show-labels
```

### Step 2: Namespace and Certificate Setup
1. Create the `frontend` and `backend` namespaces.
2. Generate TLS certificates:
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout frontend.key -x509 -out frontend.crt -subj "/CN=frontend.satsan.site"
   ```
3. Create a Kubernetes secret for the ingress gateway:
   ```bash
   kubectl create -n istio-system secret tls istio-ingressgateway-certs \
     --cert=frontend.crt \
     --key=frontend.key
   ```
4. Verify the secret:
   ```bash
   kubectl get secret -n istio-system
   ```
   Example output:
   ```
   NAME                         TYPE                DATA   AGE
   istio-ca-secret              istio.io/ca-root    5      25h
   istio-ingressgateway-certs   kubernetes.io/tls   2      23h
   ```

### Step 2: Apply Configurations
1. Deploy the authorization policies:
   ```bash
   kubectl apply -f deny-authorization.yml
   kubectl apply -f backend-internal.yml
   kubectl apply -f frontend-internal.yml
   kubectl apply -f frontend-authorization.yml
   ```
2. Configure the ingress gateway:
   ```yaml
   tls:
     mode: SIMPLE  # Ensures Istio terminates the TLS connection
     credentialName: istio-ingressgateway-certs  # Refers to the secret with the TLS certs created earlier
     minProtocolVersion: TLSV1_2
   hosts:
     - "frontend.satsan.site"  # The domain you're preferring
   ```
   ```bash
   kubectl apply -f gateway.yml
   ```
3. Enable mTLS for namespaces:
   ```bash
   kubectl apply -f peer.yml    # for default namespace
   kubectl apply -f peer2.yml   # for frontend namespace
   kubectl apply -f peer3.yml   # for backend namespace
   ```
4. Set up virtual services and destination rules:
   ```yaml
   kind: VirtualService
   metadata:
     name: frontend-route
     namespace: frontend
   spec:
     hosts:
     - "frontend.satsan.site"  # Match the host in Gateway
     gateways:
     - istio-system/microservices-gateway  # Reference the gateway defined above along with namespace
     http:
     - match:
       - uri:
           prefix: /  # Match any URL path starting with "/"
       route:
       - destination:
           host: frontend.frontend.svc.cluster.local  # Internal DNS for the frontend service
           port:
             number: 80
   ```
   ```bash
   kubectl apply -f virtualservice.yml
   kubectl apply -f virtualservice-canary.yml
   kubectl apply -f destinationrule.yml
   ```

### Step 3: Verify Deployment
1. Check pod statuses:
   ```bash
   kubectl get pods -n frontend
   kubectl get pods -n backend
   ```
2. Verify services are synced with Istio:
   ```bash
   istioctl proxy-status
   ```
   ```bash
    NAME                                                   CLUSTER        CDS                LDS                EDS                RDS                ECDS        ISTIOD                     VERSION
   adservice-74bdc9bcf6-xl8sr.backend                     Kubernetes     SYNCED (22m)       SYNCED (22m)       SYNCED (22m)       SYNCED (22m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   cartservice-7f7b9fc469-67gph.backend                   Kubernetes     SYNCED (26m)       SYNCED (26m)       SYNCED (26m)       SYNCED (26m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   cartservice-v2-d7fb846db-px9dt.backend                 Kubernetes     SYNCED (22m)       SYNCED (22m)       SYNCED (22m)       SYNCED (22m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   checkoutservice-6bbccb4788-8nfl6.backend               Kubernetes     SYNCED (16m)       SYNCED (16m)       SYNCED (16m)       SYNCED (16m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   currencyservice-795445fcb8-b4728.backend               Kubernetes     SYNCED (28m)       SYNCED (28m)       SYNCED (28m)       SYNCED (28m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   emailservice-c498b5f8b-62qh5.backend                   Kubernetes     SYNCED (5m57s)     SYNCED (5m57s)     SYNCED (5m57s)     SYNCED (5m57s)     IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   frontend-7c96fdb969-jl2s5.frontend                     Kubernetes     SYNCED (22m)       SYNCED (22m)       SYNCED (22m)       SYNCED (22m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   istio-ingressgateway-7f57549c9f-cffpz.istio-system     Kubernetes     SYNCED (19m)       SYNCED (19m)       SYNCED (19m)       SYNCED (19m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   loadgenerator-55b7ff944b-gr9pk.backend                 Kubernetes     SYNCED (14m)       SYNCED (14m)       SYNCED (14m)       SYNCED (14m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   paymentservice-6578f9dcfd-s2jfn.backend                Kubernetes     SYNCED (6m30s)     SYNCED (6m30s)     SYNCED (6m30s)     SYNCED (6m30s)     IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   productcatalogservice-5865bf7d98-vd9cn.backend         Kubernetes     SYNCED (14m)       SYNCED (14m)       SYNCED (14m)       SYNCED (14m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   recommendationservice-758d9b68c4-hvk9c.backend         Kubernetes     SYNCED (5m31s)     SYNCED (5m31s)     SYNCED (5m31s)     SYNCED (5m31s)     IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   redis-cart-7ff8f4d6ff-dvfwx.backend                    Kubernetes     SYNCED (15m)       SYNCED (15m)       SYNCED (15m)       SYNCED (15m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   shippingservice-65cc774694-pfbh9.backend               Kubernetes     SYNCED (28m)       SYNCED (28m)       SYNCED (28m)       SYNCED (28m)       IGNORED     istiod-ddcf4fdd9-wfbfp     1.24.2
   ```
3. Analyze namespace configurations:
   ```bash
   istioctl analyze -n frontend
   istioctl analyze -n backend
   ```
   ```bash
    Abhijeet Singh@Abhijeet MINGW64 /c/project/istio/istio-1.24.2 (main)
   $ istioctl analyze -n backend

   ✔ No validation issues found when analyzing namespace: backend.

   Abhijeet Singh@Abhijeet MINGW64 /c/project/istio/istio-1.24.2 (main)
   $ istioctl analyze -n frontend

   ✔ No validation issues found when analyzing namespace: frontend.

   ```
   
4. Test external access:
   Access the `frontend` service using the configured domain (e.g., `https://frontend.satsan.site`).

---
### Step 4: Troubleshoot
---
1. ✘ Istiod encountered an error: failed to wait for resource: resources not ready after 5m0s: context deadline exceeded
   Deployment/istio-system/istiod (containers with unready status: [discovery])
  ✘ CNI encountered an error: failed to update resource with server-side apply for obj ConfigMap/kube-system/istio-cni- 
   config: configmaps "istio-cni-config" is forbidden: User "xyz@gmail.com" cannot patch resource "configmaps" in API group 
   "" in the namespace 
  "kube-system": GKE Warden authz [denied by managed-namespaces-limitation]: the namespace "kube-system" is managed and the 
  request's verb "patch" is denied
---
  Solution:
  Install Istio without CNI: Since you don't need the CNI plugin right now, you can install Istio without it. You can use the istioctl command to install Istio with the --set flag to explicitly disable the CNI.

  Run the following command:
  ```bash
   istioctl install --set profile=default --set values.cni.enabled=false
  ```
2. Manual Cleanup: Manually clean up CNI configurations or leftover Istio resources that might be causing the issue.

  For example, deleting CNI-related resources in the kube-system namespace:
  ```bash
    kubectl delete configmap istio-cni-config -n kube-system
    kubectl delete daemonset istio-cni -n kube-system
  ```


  ## Load Balancing and Canary Deployment
  - **Load Balancing**: Configured via `destinationrule.yml` for consistent and efficient traffic distribution across pods.
  - **Canary Deployment**: 20% of traffic to `cartservice` is routed to `cartservice-v2` using `virtualservice-canary.yml`.

---
### Step 5: Monitoring
After applying the istioctl install a dictoring named sample will get installed inside it there is one more directory named addons inside it all the monitoring tools yaml file will be found.
  ```bash
    kubectl apply -f samples/addons/kiali.yaml
    kubectl apply -f samples/addons/prometheus.yaml
    kubectl apply -f samples/addons/grafana.yaml
  ```
Verify:
  ```bash
   kubectl get pod -n istio-system
 ```

1. Configuration applied in this project via kaili dashboard:
   a. command to access the dashboard:
      ```bash
       kubectl port-forward service/kiali 30001 -n istio-system  # copy the url and paste into browser
      ```
   ![image alt](https://github.com/Ashsatsan/microservices-googlecloud/blob/main/istioproj12.png?raw=true)

2. View secure communication betweeen microservices and also view the communication between v2 version of cartservice
    
    ![image alt](https://github.com/Ashsatsan/microservices-googlecloud/blob/main/istioproj2.png?raw=true)

3. Now Monitoring metrix via Grafana:
  a. To access grafana follow similar steps as kiali.
  b. Grafana Dashboard ask for the username and password.
   ```bash
    Username: admin
    To access password execute the following command:
    kubectl get secret <grafana-secret-name> -n <namespace> -o jsonpath="{.data.admin-password}" | base64 --decode

   ```
   
 ![image alt](https://github.com/Ashsatsan/microservices-googlecloud/blob/main/istioproj8.png?raw=true) 
 
 ![image alt](hhttps://github.com/Ashsatsan/microservices-googlecloud/blob/main/istioproj9.png?raw=true)

4. The overall project setup

   ![image alt](https://github.com/Ashsatsan/microservices-googlecloud/blob/main/istioproj4.png?raw=true)







