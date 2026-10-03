# Day 82 -- EKS Networking with Gateway API and Persistent Storage

## Task

Your EKS cluster was running and the AI-BankApp was deployed with raw manifests. But production needed proper ingress, HTTPS, session persistence, and reliable storage. The AI-BankApp project used the Kubernetes Gateway API with Envoy Gateway instead of traditional Ingress -- the next generation of Kubernetes traffic management.

Today you set up the Gateway API, configured TLS with cert-manager, understood EBS storage in action, and explored the AI-BankApp's production networking setup.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch `feat/gitops`) -- `k8s/gateway.yml`, `k8s/cert-manager.yml`, `k8s/pv.yml`, `k8s/pvc.yml`

---

## project

[AI-BankApp-DevOps](https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps)

---

## All screenshot's

[screenshots of AI-BankApp-DevOps](screenshots/)

---

## Challenge Tasks

### Task 1: Understand Gateway API vs Ingress

The AI-BankApp used the Gateway API instead of the traditional Ingress resource. The differences were:

| **Feature**       | **Ingress**          | **Gateway API**                                          |
| ----------------- | -------------------- | -------------------------------------------------------- |
| API maturity      | Stable but limited   | GA since Kubernetes 1.26                                 |
| Traffic splitting | Not supported        | Built-in (weighted backends)                             |
| Header matching   | Annotation-dependent | Native HTTPRoute rules                                   |
| Role separation   | Single resource      | GatewayClass (infra) -> Gateway (ops) -> HTTPRoute (dev) |
| TLS management    | Annotation-based     | Native TLS config in Gateway listeners                   |
| Session affinity  | Not standardized     | BackendTrafficPolicy (with Envoy)                        |

**The AI-BankApp's Gateway architecture:**

```text
[Internet]
    |
[AWS NLB] (created by Envoy Gateway)
    |
[Gateway: bankapp-gateway]
  |-- Listener: HTTP (port 80)
  |-- Listener: HTTPS (port 443, TLS terminated)
    |
[HTTPRoute: bankapp-route]
    |
[Service: bankapp-service:8080]
    |
[Pods: bankapp x2-4] (with session affinity via cookie)

```

---

### Task 2: Install Envoy Gateway

Envoy Gateway was the Gateway API implementation the AI-BankApp used.

Installed via Helm:

```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait
```


Verified:

```bash
kubectl get pods -n envoy-gateway-system

kubectl get gatewayclass
```


---

### Task 3: Deploy the AI-BankApp with Gateway API

The AI-BankApp was deployed with the core manifests:

```bash
cd AI-BankApp-DevOps
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml
```

**Now the Gateway configuration was studied and applied.**

**1. GatewayClass** -- defined which controller handled Gateways:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**2. Gateway** -- created the actual load balancer with listeners:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      hostname: <your-ip>.nip.io
      tls:
        mode: Terminate
        certificateRefs:
          - name: bankapp-tls
```

When this was applied, Envoy Gateway created an AWS NLB automatically.

**3. HTTPRoute** -- routed traffic to the BankApp service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-route
  namespace: bankapp
spec:
  parentRefs:
    - name: bankapp-gateway
      sectionName: https
    - name: bankapp-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: bankapp-service
          port: 8080
```

**4. BackendTrafficPolicy** -- provided session persistence via cookies:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: bankapp-session
  namespace: bankapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: bankapp-route
  loadBalancer:
    type: ConsistentHash
    consistentHash:
      type: Cookie
      cookie:
        name: BANKAPP_AFFINITY
        ttl: 3600s
```

**Why cookie-based session affinity?** The AI-BankApp used Spring Security with form-based login. Without session affinity, a user's requests could hit different pods, and they could be logged out. The `BANKAPP_AFFINITY` cookie ensured all requests from a user went to the same pod.

Applied the Gateway configuration:

```bash
kubectl apply -f k8s/gateway.yml
```

Waited for the NLB to be provisioned:

```bash
kubectl get gateway -n bankapp -w
```

The Gateway became programmed and received an AWS NLB address:

Got the external address:

Tested access:

```bash
curl -i http://$GATEWAY_IP
```

The AWS NLB and Envoy Gateway were reachable, but the request returned `404 Not Found` because the configured HTTPRoute hostname did not match the AWS NLB hostname used by the request.

---

### Task 4: Set Up TLS with cert-manager

The AI-BankApp used cert-manager with Let's Encrypt for automatic HTTPS certificates.

Installed cert-manager:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true \
  --wait
```

The installation completed successfully with cert-manager `v1.21.2`.

Verified:

```bash
kubectl get pods -n cert-manager
```

The cert-manager installation was then upgraded to enable Gateway API support:

```bash
helm upgrade cert-manager jetstack/cert-manager \
  -n cert-manager \
  --set crds.enabled=true \
  --set config.enableGatewayAPI=true \
  --wait
```

The `ClusterIssuer` from `k8s/cert-manager.yml` was applied:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - group: gateway.networking.k8s.io
                kind: Gateway
                name: bankapp-gateway
                namespace: bankapp
```

**How it works:**

1. cert-manager requests a certificate from Let's Encrypt
2. Let's Encrypt sends an HTTP-01 challenge
3. cert-manager creates a temporary HTTPRoute to respond to the challenge
4. Let's Encrypt verifies and issues the certificate
5. cert-manager stores the certificate in the `bankapp-tls` Secret
6. The Gateway uses this Secret for HTTPS termination

The Gateway address was restored and checked:

```bash
export GATEWAY_IP=$(kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}')
echo "$GATEWAY_IP"
```

The `nip.io` hostname command was attempted:

```bash
export HOSTNAME="${GATEWAY_IP}.nip.io"
echo "HTTPS URL: https://$HOSTNAME"
```
---

### Task 5: Understand EBS Persistent Storage in Action

The AI-BankApp used EBS volumes for MySQL (5Gi) and Ollama (10Gi). The storage setup was checked on EKS.

Check the storage setup:

```bash
# StorageClass
kubectl get storageclass gp3

# PVCs
kubectl get pvc -n bankapp

# PVs (dynamically provisioned)
kubectl get pv
```

**Find the actual EBS volumes in AWS:**

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-by,Values=ebs.csi.aws.com" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,AZ:AvailabilityZone,State:State}" \
  --output table \
  --region us-west-2
```

## Test persistence

```bash
# Check current MySQL data
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"

```

```bash
# Delete the pod
kubectl delete pod -n bankapp -l app=mysql
```


```bash
# Watch it recreate
kubectl get pods -n bankapp -l app=mysql -w
```

```bash
# Verify data survived
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
```

The database was intact because the EBS-backed persistent volume remained independent of the pod.

---

### Task 6: Explore HPA and Node Capacity

The AI-BankApp's HPA scaled pods between 2 and 4 based on CPU.

```bash
kubectl get hpa -n bankapp
```

```bash
kubectl top nodes
```

```bash
kubectl top pods -n bankapp
```

**Resource budget for the AI-BankApp on 3x t3.medium nodes:**

| **Component**       | **CPU Request**     | **Memory Request** | **Instances** |
| ------------------- | ------------------- | ------------------ | ------------- |
| BankApp             | 250m                | 256Mi              | 2-4 pods      |
| MySQL               | 250m                | 256Mi              | 1 pod         |
| Ollama              | 900m                | 2Gi                | 1 pod         |
| Init containers     | 50m                 | 32Mi               | temporary     |
| System pods         | ~500m               | ~500Mi              | per node      |
| **Total available** | **6000m (3 nodes)** | **12Gi (3 nodes)** |               |

Ollama was the heaviest consumer. If BankApp was scaled to 4 pods, total CPU requests reached ~2.9 cores + system overhead.

**Clean up the workload (keep the cluster for Day 83):**

```bash
kubectl delete -f k8s/gateway.yml 2>/dev/null
kubectl delete -f k8s/hpa.yml
kubectl delete -f k8s/bankapp-deployment.yml
kubectl delete -f k8s/ollama-deployment.yml
kubectl delete -f k8s/mysql-deployment.yml
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/secrets.yml
kubectl delete -f k8s/configmap.yml
kubectl delete -f k8s/pvc.yml
kubectl delete -f k8s/pv.yml
kubectl delete -f k8s/namespace.yml
```

---

## Gateway API Architecture Diagram

```text
Internet
   |
   v
AWS NLB
   |
   v
Gateway
   |
   v
HTTPRoute
   |
   v
Service
   |
   v
AI-BankApp Pods
```

## Gateway API vs Ingress

| Feature          | Gateway API                             | Ingress                  |
| ---------------- | --------------------------------------- | ------------------------ |
| API Structure    | GatewayClass → Gateway → HTTPRoute      | Ingress                  |
| Role Separation  | Yes                                     | Limited                  |
| Routing          | HTTPRoute and other route types         | Mainly HTTP/HTTPS        |
| Traffic Policies | Supported through Gateway API ecosystem | Controller-dependent     |
| Extensibility    | High                                    | More limited             |
| Configuration    | More structured                         | Simpler                  |
| Use Case         | Modern and complex traffic management   | Basic HTTP/HTTPS routing |

## Gateway API Resources

### GatewayClass

`GatewayClass` defines which controller manages the Gateway resources.

For AI-BankApp, Envoy Gateway was used as the controller.

### Gateway

`Gateway` defines how the application was exposed to external traffic.

It configured the HTTP and HTTPS listeners and used the `envoy-gateway` GatewayClass.

### HTTPRoute

`HTTPRoute` defined how incoming HTTP/HTTPS requests were routed to the AI-BankApp Service.

The route forwarded requests to `bankapp-service` on port `8080`.

### BackendTrafficPolicy

`BackendTrafficPolicy` configured traffic behavior for the backend.

AI-BankApp used consistent hashing with a cookie named `BANKAPP_AFFINITY` for session persistence.

## Cookie-Based Session Affinity

Cookie-based session affinity was needed because AI-BankApp used application sessions that were stored in the backend pod.

When multiple BankApp pods were running, requests from the same user needed to reach the same backend pod.

The `BANKAPP_AFFINITY` cookie allowed Envoy Gateway to consistently route the user's requests to the same backend pod.

## cert-manager and TLS

cert-manager automated TLS certificate management for AI-BankApp.

It used a Let's Encrypt `ClusterIssuer` and the HTTP-01 challenge to validate domain ownership.

After successful validation, cert-manager created and managed the TLS certificate stored in a Kubernetes Secret.

The Gateway used this Secret to terminate HTTPS traffic.

## EBS Persistent Storage Flow

```text
StorageClass
     |
     v
    PVC
     |
     v
    PV
     |
     v
EBS Volume
     |
     v
    Pod
```

The `gp3` StorageClass used the AWS EBS CSI driver to dynamically provision EBS volumes.

AI-BankApp used persistent volumes for MySQL and Ollama so that application data remained available even when their pods were recreated.

## Resource Budget for AI-BankApp on EKS

| Component       | CPU Request | Memory Request | Instances |
| --------------- | ----------- | -------------- | --------- |
| BankApp         | 250m        | 256Mi          | 2-4 pods  |
| MySQL           | 250m        | 256Mi          | 1 pod     |
| Ollama          | 900m        | 2Gi            | 1 pod     |
| Init containers | 50m         | 32Mi           | Temporary |
| System pods     | ~500m       | ~500Mi         | Per node  |
| Total available | 6000m       | 12Gi           | 3 nodes   |

```
```

