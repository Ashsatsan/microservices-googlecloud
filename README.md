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
2. **backend-internal.yml**: Allows communication between services within the `backend` namespace.
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
   kubectl apply -f peer.yml
   kubectl apply -f peer2.yml
   kubectl apply -f peer3.yml
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
2. Verify services are synced with Istio:
   ```bash
   istioctl proxy-status
   ```
3. Analyze namespace configurations:
   ```bash
   istioctl analyze -n frontend
   istioctl analyze -n backend
   ```
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


