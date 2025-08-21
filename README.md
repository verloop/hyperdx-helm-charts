# HyperDX V2 Helm Charts

Welcome to the official HyperDX Helm charts repository. This guide provides instructions on how to install, configure, and manage your HyperDX V2 deployment using Helm.

## Table of Contents

- [Quick Start](#quick-start)
- [Deployment Options](#deployment-options)
  - [Full Stack (Default)](#full-stack-default)
  - [External ClickHouse](#external-clickhouse)
  - [External OTEL Collector](#external-otel-collector)
  - [Minimal Deployment](#minimal-deployment)
- [Cloud Deployment](#cloud-deployment)
  - [Google Kubernetes Engine (GKE)](#google-kubernetes-engine-gke)
  - [Amazon EKS](#amazon-eks)
  - [Azure AKS](#azure-aks)
- [Configuration](#configuration)
  - [API Key Setup](#api-key-setup)
  - [Task Configuration](#task-configuration)
  - [Using Secrets](#using-secrets)
  - [Ingress Setup](#ingress-setup)
- [Operations](#operations)
  - [Upgrading](#upgrading-the-chart)
  - [Uninstalling](#uninstalling-hyperdx)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## Quick Start

### Prerequisites

- [Helm](https://helm.sh/) v3+
- Kubernetes cluster (v1.20+ recommended)
- `kubectl` configured to interact with your cluster

### Install HyperDX (Full Stack)

```sh
# Add the HyperDX Helm repository
helm repo add hyperdx https://hyperdxio.github.io/helm-charts
helm repo update

# Install with default values (includes ClickHouse, OTEL collector, MongoDB)
helm install my-hyperdx hyperdx/hdx-oss-v2

# Get the external IP (for cloud deployments)
kubectl get services

# Access the UI at http://<EXTERNAL-IP>:3000
```

**That's it!** HyperDX is now running with all components included.

## Deployment Options

### Full Stack (Default)

By default, this Helm chart deploys the complete HyperDX stack including:
- **HyperDX Application** (API, UI, and OpAMP server)
- **ClickHouse** (for storing logs, traces, and metrics)
- **OTEL Collector** (for receiving and processing telemetry data)
- **MongoDB** (for application metadata)

To install the full stack with default values:

```sh
helm install my-hyperdx hyperdx/hdx-oss-v2
```

### External ClickHouse

If you have an existing ClickHouse cluster, you have two options for configuring connections:

#### Option 1: Inline Configuration (Simple)

```yaml
# values-external-clickhouse.yaml
clickhouse:
  enabled: false  # Disable the built-in ClickHouse

otel:
  clickhouseEndpoint: "tcp://your-clickhouse-server:9000"
  clickhousePrometheusEndpoint: "http://your-clickhouse-server:9363"  # Optional

hyperdx:
  defaultConnections: |
    [
      {
        "name": "External ClickHouse",
        "host": "http://your-clickhouse-server:8123",
        "port": 8123,
        "username": "your-username",
        "password": "your-password"
      }
    ]
```

#### Option 2: External Secret (Recommended for Production)

For production deployments where you want to keep credentials separate from your Helm configuration:

```yaml
# values-external-clickhouse-secret.yaml
clickhouse:
  enabled: false  # Disable the built-in ClickHouse

otel:
  clickhouseEndpoint: "tcp://your-clickhouse-server:9000"
  clickhousePrometheusEndpoint: "http://your-clickhouse-server:9363"  # Optional

hyperdx:
  # Use an existing secret for complete configuration (connections + sources)
  useExistingConfigSecret: true
  existingConfigSecret: "hyperdx-external-config"
  existingConfigConnectionsKey: "connections.json"
  existingConfigSourcesKey: "sources.json"
```

Create your configuration secret:

```bash
# Create the connections JSON
cat <<EOF > connections.json
[
  {
    "name": "Production ClickHouse",
    "host": "https://your-production-clickhouse.com:8123",
    "port": 8123,
    "username": "hyperdx_user",
    "password": "your-secure-password"
  }
]
EOF

# Create the sources JSON
cat <<EOF > sources.json
[
  {
    "from": {
      "databaseName": "default",
      "tableName": "otel_logs"
    },
    "kind": "log",
    "name": "Logs",
    "connection": "Production ClickHouse",
    "timestampValueExpression": "TimestampTime",
    "displayedTimestampValueExpression": "Timestamp",
    "implicitColumnExpression": "Body",
    "serviceNameExpression": "ServiceName",
    "bodyExpression": "Body",
    "eventAttributesExpression": "LogAttributes",
    "resourceAttributesExpression": "ResourceAttributes",
    "severityTextExpression": "SeverityText",
    "traceIdExpression": "TraceId",
    "spanIdExpression": "SpanId"
  },
  {
    "from": {
      "databaseName": "default",
      "tableName": "otel_traces"
    },
    "kind": "trace",
    "name": "Traces",
    "connection": "Production ClickHouse",
    "timestampValueExpression": "Timestamp",
    "displayedTimestampValueExpression": "Timestamp",
    "implicitColumnExpression": "SpanName",
    "serviceNameExpression": "ServiceName",
    "traceIdExpression": "TraceId",
    "spanIdExpression": "SpanId",
    "durationExpression": "Duration"
  }
]
EOF

# Create the Kubernetes secret
kubectl create secret generic hyperdx-external-config \
  --from-file=connections.json=connections.json \
  --from-file=sources.json=sources.json

# Clean up the local files
rm connections.json sources.json
```

### External OTEL Collector

If you have an existing OTEL collector setup:

```yaml
# values-external-otel.yaml
otel:
  enabled: false  # Disable the built-in OTEL collector

hyperdx:
  # Point to your external OTEL collector endpoint
  otelExporterEndpoint: "http://your-otel-collector:4318"
```

#### Configuring Ingress for OTEL Collector

For instructions on exposing your OTEL collector endpoints via ingress (including example configuration and best practices), see the [OTEL Collector Ingress](#otel-collector-ingress) section in the [Ingress Setup](#ingress-setup) chapter above.

### Minimal Deployment

For organizations with existing infrastructure:

```yaml
# values-minimal.yaml
clickhouse:
  enabled: false

otel:
  enabled: false

hyperdx:
  otelExporterEndpoint: "http://your-otel-collector:4318"
  # Option 1: Inline configuration (for testing/development)
  defaultConnections: |
    [
      {
        "name": "External ClickHouse",
        "host": "http://your-clickhouse-server:8123",
        "port": 8123,
        "username": "your-username",
        "password": "your-password"
      }
    ]
  
  # Option 2: External secret (recommended for production)
  # useExistingConfigSecret: true
  # existingConfigSecret: "my-external-config"
  # existingConfigConnectionsKey: "connections.json"
  # existingConfigSourcesKey: "sources.json"
```

## Configuration

### API Key Setup

After successfully deploying HyperDX, you'll need to configure the API key to enable the app's telemetry data collection:

1. **Access your HyperDX instance** via the configured ingress or service endpoint
2. **Log into the HyperDX dashboard** and navigate to Team settings to generate or retrieve your API key
3. **Update your deployment** with the API key using one of the following methods:

#### Method 1: Update via Helm upgrade with values file

Add the API key to your `values.yaml`:

```yaml
hyperdx:
  apiKey: "your-api-key-here"
```

Then upgrade your deployment:

```sh
helm upgrade my-hyperdx hyperdx/hdx-oss-v2 -f values.yaml
```

#### Method 2: Update via Helm upgrade with --set flag

```sh
helm upgrade my-hyperdx hyperdx/hdx-oss-v2 --set hyperdx.apiKey="your-api-key-here"
```

**Important:** After updating the API key, you need to restart the pods to pick up the new configuration:

```sh
kubectl rollout restart deployment my-hyperdx-hdx-oss-v2-app my-hyperdx-hdx-oss-v2-otel-collector
```

**Note:** The chart automatically creates a Kubernetes secret (`<release-name>-app-secrets`) with your API key. No additional secret configuration is needed unless you want to use an external secret.

## Using Secrets

For handling sensitive data such as API keys or database credentials, use Kubernetes secrets. The HyperDX Helm charts provide default secret files that you can modify and apply to your cluster.

### Using Pre-Configured Secrets

The Helm chart includes a default secret template located at [`charts/hdx-oss-v2/templates/secrets.yaml`](https://github.com/hyperdxio/helm-charts/blob/main/charts/hdx-oss-v2/templates/secrets.yaml). This file provides a base structure for managing secrets.


If you need to manually apply a secret, modify and apply the provided `secrets.yaml` template:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hyperdx-secret
  annotations:
    "helm.sh/resource-policy": keep
type: Opaque
data:
  API_KEY: <base64-encoded-api-key>
```

Apply the secret to your cluster:

```sh
kubectl apply -f secrets.yaml
```

### Creating a Custom Secret

If you prefer, you can create a custom Kubernetes secret manually:

```sh
kubectl create secret generic hyperdx-secret \
  --from-literal=API_KEY=my-secret-api-key
```

### Referencing a Secret in `values.yaml`

```yaml
hyperdx:
  apiKey:
    valueFrom:
      secretKeyRef:
        name: hyperdx-secret
        key: API_KEY
```

## Task Configuration

By default, there is one task in the chart setup as a cronjob, responsible for checking whether alerts should fire. Here are its configuration options:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `tasks.enabled` | Enable/Disable cron tasks in the cluster. By default, the HyperDX image will run cron tasks intra process. Change to true if you'd rather use a separate cron task in the cluster. | `false` |
| `tasks.checkAlerts.schedule` | Cron schedule for the check-alerts task | `*/1 * * * *` |
| `tasks.checkAlerts.resources` | Resource requests and limits for the check-alerts task | See `values.yaml` |

## Ingress Setup

- [General Ingress Setup](#general-ingress-setup)
- [OTEL Collector Ingress](#otel-collector-ingress)
- [Troubleshooting Ingress](#troubleshooting-ingress)

### General Ingress Setup

To expose the HyperDX UI and API via a domain name, enable ingress in your `values.yaml`:

```yaml
hyperdx:
  ingress:
    enabled: true
    host: "hyperdx.yourdomain.com"  # Set this to your desired domain
```

#### Configuring `ingress.host` and `hyperdx.frontendUrl`

- **`hyperdx.ingress.host`**: Set to the domain you want to use for accessing HyperDX (e.g., `hyperdx.yourdomain.com`).
- **`hyperdx.frontendUrl`**: Should match the ingress host and include the protocol (e.g., `https://hyperdx.yourdomain.com`).

**Example:**
```yaml
hyperdx:
  frontendUrl: "https://hyperdx.yourdomain.com"
  ingress:
    enabled: true
    host: "hyperdx.yourdomain.com"
```

This ensures that all generated links, cookies, and redirects work correctly.

#### Enabling TLS (HTTPS)

To secure your deployment with HTTPS, enable TLS in your ingress configuration:

```yaml
hyperdx:
  ingress:
    enabled: true
    host: "hyperdx.yourdomain.com"
    tls:
      enabled: true
      tlsSecretName: "hyperdx-tls"  # Name of the Kubernetes TLS secret
```

- Create a Kubernetes TLS secret with your certificate and key:
  ```sh
  kubectl create secret tls hyperdx-tls \
    --cert=path/to/tls.crt \
    --key=path/to/tls.key
  ```
- The ingress will reference this secret to terminate HTTPS connections.

#### Example Minimal Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hyperdx-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: hyperdx.yourdomain.com
      http:
        paths:
          - path: /(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: <service-name>
                port:
                  number: 3000
  tls:
    - hosts:
        - hyperdx.yourdomain.com
      secretName: hyperdx-tls
```

#### Common Pitfalls

- **Path and Rewrite Configuration:**
  - For Next.js and other SPAs, always use a regex path and rewrite annotation as shown above. Do not use just `path: /` without a rewrite, as this will break static asset serving.
- **Mismatched `frontendUrl` and `ingress.host`:**
  - If these do not match, you may experience issues with cookies, redirects, and asset loading.
- **TLS Misconfiguration:**
  - Ensure your TLS secret is valid and referenced correctly in the ingress.
  - Browsers may block insecure content if you access the app over HTTP when TLS is enabled.
- **Ingress Controller Version:**
  - Some features (like regex paths and rewrites) require recent versions of nginx ingress controller. Check your version with:
    ```sh
    kubectl -n ingress-nginx get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath="{.items[0].spec.containers[0].image}"
    ```

---

### OTEL Collector Ingress

If you need to expose your OTEL collector endpoints (for traces, metrics, logs) through ingress, you can use the `additionalIngresses` configuration. This is useful for organizations that want to send telemetry data from outside the cluster or use a custom domain for the collector.

**Example configuration:**

```yaml
hyperdx:
  ingress:
    enabled: true
    additionalIngresses:
      - name: otel-collector
        annotations:
          nginx.ingress.kubernetes.io/ssl-redirect: "false"
          nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
          nginx.ingress.kubernetes.io/use-regex: "true"
        ingressClassName: nginx
        hosts:
          - host: collector.yourdomain.com
            paths:
              - path: /v1/(traces|metrics|logs)
                pathType: Prefix
                port: 4318
                name: otel-collector
        tls:
          - hosts:
              - collector.yourdomain.com
            secretName: collector-tls
```

- This creates a separate ingress resource for the OTEL collector endpoints.
- You can use a different domain, configure specific TLS settings, and apply custom annotations for the collector ingress.
- The regex path rule allows you to route all OTLP signals (traces, metrics, logs) through a single rule.

**Note:**
- If you do not need to expose the OTEL collector externally, you can skip this section.
- For most users, the general ingress setup is sufficient.

---

### Troubleshooting Ingress

- **Check Ingress Resource:**
  ```sh
  kubectl get ingress -A
  kubectl describe ingress <ingress-name>
  ```
- **Check Pod Logs:**
  ```sh
  kubectl logs -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx
  ```
- **Test Asset URLs:**
  Use `curl` to verify static assets are served as JS, not HTML:
  ```sh
  curl -I https://hyperdx.yourdomain.com/_next/static/chunks/main-xxxx.js
  # Should return Content-Type: application/javascript
  ```
- **Browser DevTools:**
  - Check the Network tab for 404s or assets returning HTML instead of JS.
  - Look for errors like "Unexpected token <" in the console (indicates HTML returned for JS).
- **Check for Path Rewrites:**
  - Ensure the ingress is not stripping or incorrectly rewriting asset paths.
- **Clear Browser and CDN Cache:**
  - After changes, clear your browser cache and any CDN/proxy cache to avoid stale assets.

---

## Operations

### Upgrading the Chart

To upgrade to a newer version:

```sh
helm upgrade my-hyperdx hyperdx/hdx-oss-v2 -f values.yaml
```

To check available chart versions:

```sh
helm search repo hyperdx
```

### Uninstalling HyperDX

To remove the deployment:

```sh
helm uninstall my-hyperdx
```

This will remove all resources associated with the release, but persistent data (if any) may remain.

## Cloud Deployment

### Google Kubernetes Engine (GKE)

When deploying to GKE, you may need to override certain values due to cloud-specific networking behavior:

#### LoadBalancer DNS Resolution Issue

GKE's LoadBalancer service can cause internal DNS resolution issues where pod-to-pod communication resolves to external IPs instead of staying within the cluster network. This specifically affects the OTEL collector's connection to the OpAMP server.

**Symptoms:**
- OTEL collector logs showing "connection refused" errors with cluster IP addresses
- OpAMP connection failures like: `dial tcp 34.118.227.30:4320: connect: connection refused`

**Solution:**
Use the fully qualified domain name (FQDN) for the OpAMP server URL:

```bash
helm install my-hyperdx hyperdx/hdx-oss-v2 \
  --set hyperdx.frontendUrl="http://your-external-ip-or-domain.com" \
  --set otel.opampServerUrl="http://my-hyperdx-hdx-oss-v2-app.default.svc.cluster.local:4320"
```

#### Other GKE Considerations

```yaml
# values-gke.yaml
hyperdx:
  frontendUrl: "http://34.123.61.99"  # Use your LoadBalancer external IP

otel:
  opampServerUrl: "http://my-hyperdx-hdx-oss-v2-app.default.svc.cluster.local:4320"

# Adjust for GKE pod networking if needed
clickhouse:
  config:
    clusterCidrs:
      - "10.8.0.0/16"  # GKE commonly uses this range
      - "10.0.0.0/8"   # Fallback for other configurations
```

### Amazon EKS

For EKS deployments, consider these common configurations:

```yaml
# values-eks.yaml
hyperdx:
  frontendUrl: "http://your-alb-domain.com"

# EKS typically uses these pod CIDRs
clickhouse:
  config:
    clusterCidrs:
      - "192.168.0.0/16"
      - "10.0.0.0/8"

# Enable ingress for production
hyperdx:
  ingress:
    enabled: true
    host: "hyperdx.yourdomain.com"
    tls:
      enabled: true
```

### Azure AKS

For AKS deployments:

```yaml
# values-aks.yaml
hyperdx:
  frontendUrl: "http://your-azure-lb.com"

# AKS pod networking
clickhouse:
  config:
    clusterCidrs:
      - "10.244.0.0/16"  # Common AKS pod CIDR
      - "10.0.0.0/8"
```

### Production Cloud Deployment Checklist

- [ ] Configure proper `frontendUrl` with your external domain/IP
- [ ] Set up ingress with TLS for HTTPS access
- [ ] Override `otel.opampServerUrl` with FQDN if experiencing connection issues
- [ ] Adjust `clickhouse.config.clusterCidrs` for your pod network CIDR
- [ ] Configure persistent storage for production workloads
- [ ] Set appropriate resource requests and limits
- [ ] Enable monitoring and alerting

### Browser Compatibility Notes

For HTTP-only deployments (development/testing), some browsers may show crypto API errors due to secure context requirements. For production deployments, use HTTPS with proper TLS certificates through ingress configuration.

## Troubleshooting

### Checking Logs

```sh
kubectl logs -l app.kubernetes.io/name=hdx-oss-v2
```