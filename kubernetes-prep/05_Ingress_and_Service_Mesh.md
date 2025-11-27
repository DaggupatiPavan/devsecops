# 05. Ingress & Service Mesh

## 1. Ingress Controllers (Advanced)

An Ingress Controller acts as a reverse proxy and load balancer for your cluster.

### Nginx Ingress Controller
The most popular controller.

**Advanced Features via Annotations:**
- **Rewrite Target**: Change the path before sending to backend.
  ```yaml
  nginx.ingress.kubernetes.io/rewrite-target: /$2
  ```
- **Rate Limiting**: Protect services from abuse.
  ```yaml
  nginx.ingress.kubernetes.io/limit-rps: "5"
  ```
- **SSL Passthrough**: Send encrypted traffic directly to the pod without termination.
  ```yaml
  nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  ```
- **Canary Deployments**: Split traffic by percentage or header.
  ```yaml
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "10" # 10% traffic to canary
  ```

## 2. Service Mesh (Istio, Linkerd)

A Service Mesh manages traffic *between* services (East-West traffic), adding reliability, security, and observability without changing application code.

### Architecture
- **Control Plane**: Manages configuration (Istiod, Linkerd Control Plane).
- **Data Plane**: Intelligent proxies (Envoy, Linkerd-proxy) deployed as **Sidecars** next to every app container.

### Key Capabilities

#### 1. mTLS (Mutual TLS)
Automatically encrypts all traffic between services.
- **Security**: Zero-trust network.
- **Identity**: Verifies the identity of the caller (SPIFFE IDs).

#### 2. Traffic Management
Advanced routing beyond simple load balancing.
- **Circuit Breaking**: Stop sending traffic to a failing service to prevent cascading failures.
- **Retries & Timeouts**: Automatically retry failed requests.
- **Fault Injection**: Intentionally introduce delays/errors to test resilience (Chaos Engineering).

#### 3. Observability
- **Metrics**: Golden signals (Latency, Traffic, Errors, Saturation) for every hop.
- **Tracing**: Distributed tracing (Jaeger/Zipkin) to visualize request flow.
- **Visualization**: Tools like **Kiali** (for Istio) provide a topology graph of the mesh.

### Istio vs Linkerd

| Feature | Istio | Linkerd |
| :--- | :--- | :--- |
| **Complexity** | High (Steep learning curve) | Low (Focus on simplicity) |
| **Proxy** | Envoy (C++) | Linkerd-proxy (Rust - ultra light) |
| **Features** | Everything imaginable | "Just works" essentials |
| **Performance** | Good (but heavier sidecar) | Best in class (micro-proxy) |

### Example: Istio VirtualService (Traffic Splitting)
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```
