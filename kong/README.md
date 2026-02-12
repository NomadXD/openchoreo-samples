# Kong Gateway Module for OpenChoreo Data Plane

This document provides comprehensive documentation for integrating Kong Gateway as the API gateway in the OpenChoreo data plane, replacing the default kgateway (Envoy-based) implementation.

## Table of Contents

- [Overview](#overview)
- [High-Level Architecture](#high-level-architecture)
- [Installation](#installation)
- [Configuration](#configuration)
- [Maintenance](#maintenance)
- [Customization](#customization)

---

## Overview

OpenChoreo uses the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) as the standard API for exposing component endpoints to public or internal networks. Because the Gateway API is a vendor-neutral Kubernetes standard, the gateway layer is easily pluggable and extensible across vendors — any Gateway API-compliant controller can serve as the ingress layer without changes to the control plane or the OpenChoreo ComponentTypes.

The Kong Gateway module replaces the default kgateway (Envoy) with [Kong Gateway](https://konghq.com/) and the [Kong Kubernetes Ingress Controller (KIC)](https://docs.konghq.com/kubernetes-ingress-controller/latest/), providing advanced API management capabilities such as rate limiting, authentication, request/response transformation, and observability - all through Kubernetes-native CRDs.

### Key Design Decisions

- **Standard Gateway API as the contract**: OpenChoreo components create `HTTPRoute` resources that reference a `Gateway` by name. The gateway implementation is transparent to the control plane.
- **Helm-driven configuration**: The `gatewayClassName` in the data plane Helm chart determines which gateway controller processes the `Gateway` CR and its routes.
- **No control plane changes required**: Switching gateways only requires data plane reconfiguration. The rendering pipeline, endpoint resolution, and release controllers work unchanged.

---

## High-Level Architecture

### Gateway Integration in OpenChoreo

```
┌─────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                           │
│                                                             │
│   Renders component templates and applies resources         │
│   (Deployment, Service, HTTPRoute) to the data plane        │
│                                                             │
└─────────────────────────────┬───────────────────────────────┘
                              │
                     applies resources
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     DATA PLANE                              │
│                                                             │
│              ┌──────────────────────────────┐               │
│              │   Component Resources        │               │
│              │                              │               │
│              │                              │               │
│              │   ┌────────────┐             │               │
│              │   │ Deployment │             │               │
│              │   └────────────┘             │               │
│              │   ┌────────────┐             │               │
│              │   │  Service   │             │               │
│              │   └─────┬──────┘             │               │
│              │         │ backendRef         │               │
│              │   ┌─────┴──────┐             │               │
│              │   │ HTTPRoute  │             │               │
│              │   │            │             │               │
│              │   │ parentRef ─┼────┐        │               │
│              │   └────────────┘    │        │               │
│              └─────────────────────┼────────┘               │
│                                    │                        │
│              ┌─────────────────────┴──────────┐             │
│              │   Gateway CR                   │             │
│              │   name: gateway-default        │             │
│              │   gatewayClassName: kong  ◄──── Configurable │
│              │                                │             │
│              │   listeners:                   │             │
│              │     - http  (port 19080)       │             │
│              │     - https (port 19443, TLS)  │             │
│              └───────────────┬────────────────┘             │
│                              │ watches                      │
│              ┌───────────────┴────────────────┐             │
│              │   Kong Ingress Controller      │             │
│              │   (KIC)                        │             │
│              │                                │             │
│              │   - Watches Gateway, HTTPRoute │             │
│              │   - Translates to Kong config  │             │
│              │   - Pushes to Kong Gateway     │             │
│              └───────────────┬────────────────┘             │
│                              │ configures                   │
│              ┌───────────────┴────────────────┐             │
│              │   Kong Gateway (Proxy)         │             │
│              │                                │             │
│              │   - Processes traffic          │             │
│              │   - TLS termination            │             │
│              │   - Plugin execution           │             │
│              │   - Routes to backends         │             │
│              └───────────────┬────────────────┘             │
│                              │                              │
│                         LoadBalancer                        │
│                         :19080 (HTTP)                       │
│                         :19443 (HTTPS)                      │
└─────────────────────────────────────────────────────────────┘
                              │
                          Client Traffic
```

### Component Breakdown

| Component                                    | Role                                                                                                                           |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Kong Kubernetes Ingress Controller (KIC)** | Watches Gateway API resources (Gateway, HTTPRoute) and translates them into Kong proxy configuration                           |
| **Kong Gateway (Proxy)**                     | Processes ingress traffic, terminates TLS, executes plugins (rate limiting, auth, etc.), and routes to backend services        |
| **Gateway CR**                               | Kubernetes Gateway API resource that defines listeners (ports, protocols, TLS). Created by Helm during data plane installation |
| **GatewayClass**                             | Declares that the `konghq.com/kic-gateway-controller` handles Gateway CRs with class `kong`                                    |
| **HTTPRoute**                                | Gateway API route resource. Created by OpenChoreo release pipeline per component. References the Gateway CR via `parentRefs`   |
| **KongPlugin / KongClusterPlugin**           | Kong-specific CRDs for applying API management policies (rate limiting, auth, transforms) to routes via annotations            |

### How Endpoint URLs Are Resolved

The ReleaseBinding controller resolves endpoint URLs by inspecting rendered HTTPRoutes:

1. Extracts `backendRef` port from the HTTPRoute (matches to workload endpoint)
2. Extracts `hostname` from the HTTPRoute spec
3. Looks up the Gateway referenced in `parentRefs`
4. Resolves the HTTPS port from DataPlane/Environment gateway configuration
5. Constructs the invoke URL: `https://<hostname>[:<port>]/<path>`

This resolution is gateway-implementation-agnostic — it only reads standard Gateway API fields.

### Traffic Flow

```
Client
  │
  ▼
LoadBalancer (:19443)
  │
  ▼
Kong Gateway (TLS termination)
  │
  ├─ Match HTTPRoute rules (hostname + path)
  ├─ Execute plugins (rate limit, auth, transform)
  │
  ▼
Service (ClusterIP)
  │
  ▼
Pod (application container)
```

---

## Installation

### Prerequisites

- An existing OpenChoreo deployment, with or without the default kgateway installed
- Helm 3.x
- kubectl configured with cluster access
- cert-manager installed (for TLS certificate management)

### Step 1: Remove kgateway (if currently installed)

If the data plane was previously deployed with kgateway, remove the existing Gateway CR so it can be recreated with the Kong GatewayClass:

```bash
# Delete the existing data plane Gateway CR
kubectl delete gateway gateway-default -n openchoreo-data-plane
```

> **Single cluster mode:** Do not remove the kgateway controller, GatewayClass, or its deployments. The control plane and observability plane gateways depend on kgateway. Only the data plane gateway is pluggable.

In multi-cluster deployments where the data plane runs on a separate cluster, kgateway can be fully removed:

```bash
# Multi-cluster only: remove kgateway entirely from the data plane cluster
kubectl delete gatewayclass kgateway
kubectl delete deployment -l app.kubernetes.io/name=kgateway -n openchoreo-data-plane
kubectl delete svc -l app.kubernetes.io/name=kgateway -n openchoreo-data-plane
```

### Step 2: Install Kong Gateway

```bash
# Add the Kong Helm repository
helm repo add kong https://charts.konghq.com
helm repo update

# Install Kong with Gateway API support
helm install kong kong/ingress \
  --namespace openchoreo-data-plane \
  --set gateway.enabled=true \
  --set ingressController.enabled=true \
  --set ingressController.installCRDs=true \
  --set ingressController.gatewayAPI.enabled=true \
  --set gateway.env.proxy_listen="0.0.0.0:19080\, 0.0.0.0:19443 http2 ssl" \
  --set gateway.proxy.type=LoadBalancer \
  --set gateway.proxy.http.enabled=true \
  --set gateway.proxy.http.servicePort=19080 \
  --set gateway.proxy.http.containerPort=19080 \
  --set gateway.proxy.tls.enabled=true \
  --set gateway.proxy.tls.servicePort=19443 \
  --set gateway.proxy.tls.containerPort=19443

# Wait for Kong to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=kong \
  -n openchoreo-data-plane \
  --timeout=300s
```

### Step 3: Create the Kong GatewayClass

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: "true"
spec:
  controllerName: konghq.com/kic-gateway-controller
EOF
```

Verify:

```bash
kubectl get gatewayclass kong
# ACCEPTED should be True
```

### Step 4: Deploy the Data Plane with Kong

Install or upgrade the OpenChoreo data plane Helm chart with the Kong `gatewayClassName`:

```bash
helm install openchoreo-data-plane ./install/helm/openchoreo-data-plane \
  --namespace openchoreo-data-plane \
  --set gatewayController.enabled=false \
  --set gateway.gatewayClassName=kong \
  --set gateway.httpPort=19080 \
  --set gateway.httpsPort=19443
```

This creates the `gateway-default` Gateway CR referencing the `kong` GatewayClass instead of `kgateway`.

### Step 5: Create TLS Certificate

If using cert-manager with a self-signed issuer:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: openchoreo-gateway-tls
  namespace: openchoreo-data-plane
spec:
  secretName: openchoreo-gateway-tls
  issuerRef:
    name: openchoreo-selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
    - "*.openchoreoapis.localhost"
EOF
```

### Step 6: Verify the Installation

```bash
# Check Kong pods
kubectl get pods -n openchoreo-data-plane -l app.kubernetes.io/instance=kong

# Check the Gateway CR status (PROGRAMMED should be True)
kubectl get gateway gateway-default -n openchoreo-data-plane

# Check Gateway listeners
kubectl describe gateway gateway-default -n openchoreo-data-plane
```

Expected pods:

| Pod                 | Role                                                               |
| ------------------- | ------------------------------------------------------------------ |
| `kong-controller-*` | Kong Kubernetes Ingress Controller — watches Gateway API resources |
| `kong-gateway-*`    | Kong Gateway proxy — processes traffic                             |

---

## Configuration

### Helm Values Reference

The following values control gateway behavior in the data plane Helm chart:

| Value                              | Type   | Default                        | Description                                                              |
| ---------------------------------- | ------ | ------------------------------ | ------------------------------------------------------------------------ |
| `gatewayController.enabled`        | bool   | `true`                         | Enable the kgateway controller sub-chart. Set to `false` when using Kong |
| `gateway.enabled`                  | bool   | `true`                         | Create the `gateway-default` Gateway CR                                  |
| `gateway.gatewayClassName`         | string | `"kgateway"`                   | GatewayClass name referenced by the Gateway CR. Set to `"kong"` for Kong |
| `gateway.httpPort`                 | int    | `9080`                         | HTTP listener port                                                       |
| `gateway.httpsPort`                | int    | `9443`                         | HTTPS listener port                                                      |
| `gateway.tls.hostname`             | string | `"*.openchoreoapis.localhost"` | Wildcard hostname for TLS certificate                                    |
| `gateway.tls.certName`             | string | `"openchoreo-gateway-tls"`     | Secret name for the TLS certificate                                      |
| `gateway.tls.clusterIssuer`        | string | `""`                           | cert-manager ClusterIssuer name                                          |
| `gateway.selfSignedIssuer.enabled` | bool   | `true`                         | Create a self-signed ClusterIssuer                                       |
| `gateway.infrastructure`           | object | `{}`                           | Cloud provider load balancer annotations                                 |

### DataPlane CR Gateway Configuration

The DataPlane CR defines gateway metadata used by the control plane for endpoint URL resolution:

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: DataPlane
metadata:
  name: default
spec:
  gateway:
    publicVirtualHost: "example.com" # Domain for public endpoints
    publicHTTPSPort: 19443 # Port included in endpoint URLs
    publicHTTPPort: 19080
    publicGatewayName: "gateway-default" # Must match the Gateway CR name
    publicGatewayNamespace: "openchoreo-data-plane"
    organizationVirtualHost: "org.example.com" # Optional: org-scoped domain
    organizationHTTPSPort: 19444
```

### Environment-Level Overrides

Environments can override gateway configuration from the DataPlane:

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Environment
metadata:
  name: production
spec:
  gateway:
    publicVirtualHost: "prod.example.com" # Overrides DataPlane value
```

If `publicVirtualHost` is set on the Environment, its gateway config takes full precedence over the DataPlane config.

### Port Configuration

Kong must listen on the same ports that the Gateway CR declares. The port mapping must be consistent across:

| Layer                    | HTTP  | HTTPS | Configured Via                                                       |
| ------------------------ | ----- | ----- | -------------------------------------------------------------------- |
| Kong `KONG_PROXY_LISTEN` | 19080 | 19443 | Kong Helm `--set gateway.env.proxy_listen`                           |
| Kong Service ports       | 19080 | 19443 | Kong Helm `--set gateway.proxy.http.servicePort` / `tls.servicePort` |
| Gateway CR listeners     | 19080 | 19443 | Data plane Helm `gateway.httpPort` / `gateway.httpsPort`             |
| DataPlane CR             | 19080 | 19443 | `spec.gateway.publicHTTPPort` / `publicHTTPSPort`                    |

A mismatch at any layer will cause listener errors or broken endpoint URLs.

### Kong-Specific Plugin Configuration

Kong plugins are applied to HTTPRoutes via annotations. Define plugins as `KongPlugin` CRDs and reference them on the route:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-5rpm
  namespace: <component-namespace>
config:
  minute: 5
  policy: local
plugin: rate-limiting
```

Apply to an HTTPRoute:

```yaml
annotations:
  konghq.com/plugins: rate-limit-5rpm
```

For cluster-wide plugins, use `KongClusterPlugin` with `labels: { global: "true" }`.

---

## Maintenance

### Monitoring Kong Health

```bash
# Check Kong pod status
kubectl get pods -n openchoreo-data-plane -l app.kubernetes.io/instance=kong

# Check Gateway CR programmed status
kubectl get gateway gateway-default -n openchoreo-data-plane

# View KIC controller logs
kubectl logs -n openchoreo-data-plane -l app.kubernetes.io/component=controller-manager,app.kubernetes.io/instance=kong -f

# View Kong proxy logs
kubectl logs -n openchoreo-data-plane -l app.kubernetes.io/component=app,app.kubernetes.io/instance=kong -f
```

### Accessing the Kong Admin API

The Admin API provides runtime visibility into Kong's configuration:

```bash
# Port-forward to Admin API
kubectl port-forward -n openchoreo-data-plane svc/kong-gateway-admin 8444:8444 &

# List configured routes
curl -k https://localhost:8444/routes

# List configured services
curl -k https://localhost:8444/services

# List active plugins
curl -k https://localhost:8444/plugins

# Check Kong status
curl -k https://localhost:8444/status
```

### Upgrading Kong

```bash
# Update Helm repo
helm repo update kong

# Upgrade Kong release
helm upgrade kong kong/ingress \
  --namespace openchoreo-data-plane \
  --reuse-values

# Verify pods are restarted
kubectl rollout status deployment/kong-gateway -n openchoreo-data-plane
kubectl rollout status deployment/kong-controller -n openchoreo-data-plane
```

### TLS Certificate Renewal

If using cert-manager, certificates are renewed automatically. To check certificate status:

```bash
# Check certificate status
kubectl get certificate -n openchoreo-data-plane

# Check secret expiry
kubectl get secret openchoreo-gateway-tls -n openchoreo-data-plane -o jsonpath='{.metadata.annotations}'
```

For manual certificate rotation:

```bash
# Delete the secret to trigger re-issuance (cert-manager)
kubectl delete secret openchoreo-gateway-tls -n openchoreo-data-plane

# Or update the secret directly
kubectl create secret tls openchoreo-gateway-tls \
  --cert=tls.crt --key=tls.key \
  -n openchoreo-data-plane --dry-run=client -o yaml | kubectl apply -f -
```

### Troubleshooting

**Gateway not PROGRAMMED**

```bash
kubectl describe gateway gateway-default -n openchoreo-data-plane
```

Common causes:

- Kong not listening on the ports declared in Gateway listeners. Verify `KONG_PROXY_LISTEN` matches `gateway.httpPort`/`gateway.httpsPort`.
- GatewayClass not found or not accepted. Verify `kubectl get gatewayclass kong`.
- KIC controller not running. Check `kubectl get pods -l app.kubernetes.io/component=controller-manager`.

**HTTPRoutes not taking effect**

```bash
# Check HTTPRoute status
kubectl get httproute -A

# Describe a specific route
kubectl describe httproute <name> -n <namespace>
```

Common causes:

- HTTPRoute `parentRef` name/namespace does not match the Gateway CR.
- Cross-namespace routing not allowed (Gateway must have `allowedRoutes.namespaces.from: All`).
- Backend service not found or port mismatch.

**Endpoint URLs not resolving**

Verify the DataPlane CR gateway config matches the actual Gateway CR:

```bash
kubectl get dataplane default -o yaml | grep -A 10 gateway
```

Ensure `publicGatewayName` and `publicGatewayNamespace` match the Gateway CR's name and namespace.

---

## Customization

### Switching Between Gateway Implementations

The gateway implementation is controlled entirely by Helm values. To switch:

**To Kong:**

```bash
helm upgrade openchoreo-data-plane ./install/helm/openchoreo-data-plane \
  --namespace openchoreo-data-plane \
  --set gatewayController.enabled=false \
  --set gateway.gatewayClassName=kong
```

**Back to kgateway:**

```bash
helm upgrade openchoreo-data-plane ./install/helm/openchoreo-data-plane \
  --namespace openchoreo-data-plane \
  --set gatewayController.enabled=true \
  --set gateway.gatewayClassName=kgateway
```

### Adding Kong Plugins to ComponentType Templates

To apply Kong plugins to all instances of a ComponentType, add annotations in the HTTPRoute template:

```yaml
# In ComponentType spec.resourceTemplates
- id: httproute
  template:
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: ${metadata.name}
      namespace: ${metadata.namespace}
      annotations:
        konghq.com/plugins: rate-limit,request-transform
    spec:
      parentRefs:
        - name: gateway-default
          namespace: openchoreo-data-plane
      hostnames:
        - ${environment.publicVirtualHost}
      rules:
        - backendRefs:
            - name: ${metadata.name}
              port: 80
```

> **Note:** Kong plugin annotations on HTTPRoutes create a dependency on Kong as the gateway implementation. Standard Gateway API fields remain portable.

### Custom Listener Ports

To use non-default ports (e.g., standard 80/443):

1. Configure Kong to listen on the desired ports:

```bash
--set gateway.env.proxy_listen="0.0.0.0:80\, 0.0.0.0:443 http2 ssl"
--set gateway.proxy.http.servicePort=80
--set gateway.proxy.http.containerPort=80
--set gateway.proxy.tls.servicePort=443
--set gateway.proxy.tls.containerPort=443
```

2. Update the data plane Helm values:

```bash
--set gateway.httpPort=80
--set gateway.httpsPort=443
```

3. Update the DataPlane CR:

```yaml
spec:
  gateway:
    publicHTTPPort: 80
    publicHTTPSPort: 443
```

### Cloud Provider Load Balancer Configuration

Use the `gateway.infrastructure` value to add cloud-specific annotations to the Gateway's generated Service:

```yaml
gateway:
  infrastructure:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "external"
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

### Scaling Kong for Production

```bash
# Scale the Kong proxy
kubectl scale deployment kong-gateway -n openchoreo-data-plane --replicas=3

# Scale the KIC controller (active-passive, single replica recommended)
kubectl scale deployment kong-controller -n openchoreo-data-plane --replicas=1
```

For production Helm values:

```yaml
# kong-production-values.yaml
gateway:
  replicaCount: 3
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "2Gi"
```

### Enabling Kong Enterprise Features

For production deployments requiring advanced API management:

- **Kong Manager UI**: Access via `kong-gateway-manager` service for visual route and plugin management
- **Developer Portal**: Requires Kong Enterprise license
- **RBAC / Workspaces**: Requires Kong Enterprise license
- **Vitals / Analytics**: Requires Kong Enterprise license

See the [Kong Enterprise documentation](https://docs.konghq.com/gateway/latest/) for license setup.
