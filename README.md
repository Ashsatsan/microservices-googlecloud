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
2. **backend-internal.yml**: Allows communication between services within the `backend` namespace and 'frontend' namespace.
3. **frontend-internal.yml**: Allows communication within the `frontend` namespace.
4. **frontend-authorization.yml**: Permits external traffic only to the `frontend` namespace while restricting access to `backend`.

#### Gateway
- **gateway.yml**: Configures the Istio ingress gateway to accept traffic on ports 443 and 80 only.

#### Peer Authentication
1. **peer.yml**: Configures mutual TLS (mTLS) for the `frontend` namespace.
2. **peer2.yml**: Configures mTLS for the `backend` namespace.
3. **peer3.yml**: Configures mTLS for the `default` namespace.

#### Virtual Services
1. **virtualservice.yml**: Routes traffic accessing the host to the corresponding service securely.
2. **virtualservice-canary.yml**: Configures canary deployment for the `cartservice` to route 20% of traffic to version 2 (`cartservice-v2`).

#### Destination Rules
- **destinationrule.yml**: Configures load balancing and traffic routing policies for the services.

---

## Deployment Steps

### Step 1: Namespace and Certificate Setup
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
   To ensure the certificate got created 
   Abhijeet Singh@Abhijeet MINGW64 /c/project/istio/istio-1.24.2 (main)
   $ kubectl get secret -n istio-system
NAME                         TYPE                DATA   AGE
istio-ca-secret              istio.io/ca-root    5      25h
istio-ingressgateway-certs   kubernetes.io/tls   2      23h

### Step 2: Apply Configurations
1. Deploy the authorization policies:
   ```bash
   kubectl apply -f deny-authorization.yml
   kubectl apply -f backend-internal.yml
   kubectl apply -f frontend-internal.yml
   kubectl apply -f frontend-authorization.yml
   ```
2. Configure the ingress gateway:
   ```bash
   kubectl apply -f gateway.yml
   ```
3. Enable mTLS for namespaces:
   ```bash
   kubectl apply -f peer.yml  #for default namespace
     apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: backend
spec:
  mtls:
    mode: STRICT

   
   kubectl apply -f peer2.yml  # for frontend namespace
   kubectl apply -f peer3.yml  # for backend namespace
   ```
4. Set up virtual services and destination rules:
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
    $ kubectl get pod -n backend
NAME                                     READY   STATUS    RESTARTS       AGE
adservice-74bdc9bcf6-xl8sr               2/2     Running   0              24h
cartservice-7f7b9fc469-67gph             2/2     Running   0              24h
cartservice-v2-d7fb846db-px9dt           2/2     Running   0              19m
checkoutservice-6bbccb4788-8nfl6         2/2     Running   0              24h
currencyservice-795445fcb8-b4728         2/2     Running   4 (157m ago)   24h
emailservice-c498b5f8b-62qh5             2/2     Running   0              24h
loadgenerator-55b7ff944b-gr9pk           2/2     Running   0              24h
paymentservice-6578f9dcfd-s2jfn          2/2     Running   4 (49m ago)    24h
productcatalogservice-5865bf7d98-vd9cn   2/2     Running   0              24h
recommendationservice-758d9b68c4-hvk9c   2/2     Running   0              24h
redis-cart-7ff8f4d6ff-dvfwx              2/2     Running   0              24h
shippingservice-65cc774694-pfbh9         2/2     Running   0              24h

$ kubectl get pod -n frontend
NAME                        READY   STATUS    RESTARTS   AGE
frontend-7c96fdb969-jl2s5   2/2     Running   0          25h
   
3. Verify services are synced with Istio:
   ```bash
   istioctl proxy-status
   ```
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

4. Analyze namespace configurations:
   ```bash
   istioctl analyze -n frontend
   istioctl analyze -n backend
   ```
  Abhijeet Singh@Abhijeet MINGW64 /c/project/istio/istio-1.24.2 (main)
$ istioctl analyze -n backend

✔ No validation issues found when analyzing namespace: backend.

Abhijeet Singh@Abhijeet MINGW64 /c/project/istio/istio-1.24.2 (main)
$ istioctl analyze -n frontend

✔ No validation issues found when analyzing namespace: frontend.

   
4. Test external access:
   Access the `frontend` service using the configured domain (e.g., `https://frontend.satsan.site`). Ensure no direct access to `backend` services.

---

## Load Balancing and Canary Deployment
- **Load Balancing**: Configured via `destinationrule.yml` for consistent and efficient traffic distribution across pods.
- **Canary Deployment**: 20% of traffic to `cartservice` is routed to `cartservice-v2` using `virtualservice-canary.yml`.

---

## Notes for Demonstration
1. **Explain Security**:
   - Highlight how mTLS ensures secure communication between services.
   - Explain the role of AuthorizationPolicies in restricting access.

2. **Emphasize Traffic Control**:
   - Demonstrate canary deployment and load balancing.
   - Show how `gateway.yml` restricts traffic to ports 443 and 80.

3. **Troubleshooting Tips**:
   - Restart pods if issues persist:
     ```bash
     kubectl rollout restart deployment -n backend
     ```
   - Check secrets and certificates:
     ```bash
     kubectl get secret -n istio-system
     ```

---


