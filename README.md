# FortiWeb Kubernetes Webhook Handler

A Kubernetes webhook service that automatically updates FortiWeb WAF server policies when Ingress resources are modified. This webhook integrates with the FortiWeb ingress controller to maintain synchronized certificate configurations.

## Overview

This webhook listens for Kubernetes Ingress `Updated` events and automatically configures FortiWeb server policies with the appropriate intermediate certificate groups based on Ingress annotations. It provides seamless integration between Kubernetes certificate management and FortiWeb WAF security policies.

### Key Features

- **Automatic Policy Updates**: Updates FortiWeb server policies when Ingress resources change
- **Certificate Integration**: Synchronizes intermediate certificate groups from Ingress annotations
- **Retry Logic**: Robust error handling with exponential backoff for API calls
- **Kubernetes Native**: Uses kubectl and Kubernetes APIs for resource access
- **High Availability**: Deployed with 2 replicas for reliability

## Quick Start

### Prerequisites

- Kubernetes cluster with RBAC enabled
- FortiWeb WAF with REST API access
- Kubernetes secrets containing FortiWeb credentials
- Ingress resources with required annotations

### Installation

#### Using Helm

```bash
# Add the Helm repository
helm repo add webhook https://amerintlxperts.github.io/webhook/
helm repo update

# Install the webhook
helm install webhook webhook/webhook

# Or install from local charts
helm install webhook ./charts/webhook
```

#### Manual Deployment

```bash
# Apply Kubernetes manifests
kubectl apply -f charts/webhook/templates/

# Verify deployment
kubectl get pods -l app.kubernetes.io/name=webhook-instance
```

## Configuration

### Required Ingress Annotations

For the webhook to process an Ingress resource, it must include these annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # Certificate group to assign to server policy
    server-policy-intermediate-certificate-group: "intermediate-cert-group-1"
    
    # FortiWeb management interface details
    fortiweb-ip: "10.0.1.100"
    fortiweb-port: "8443"
    
    # Kubernetes secret containing FortiWeb credentials
    fortiweb-login: "fortiweb-admin-secret"
spec:
  # ... ingress configuration
```

### FortiWeb Credentials Secret

Create a Kubernetes secret with FortiWeb authentication credentials:

```bash
kubectl create secret generic fortiweb-admin-secret \
  --from-literal=username='admin' \
  --from-literal=password='your-password'
```

### Helm Configuration

Customize the deployment using Helm values:

```yaml
# values.yaml
image:
  repository: ghcr.io/amerintlxperts/webhook
  tag: "latest"
  pullPolicy: Always

replicaCount: 2

service:
  type: ClusterIP
  port: 8080

# Deploy to system node pool
nodeSelector:
  system-pool: "true"

tolerations:
  - key: "CriticalAddonsOnly"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

## How It Works

### Webhook Flow

1. **Event Reception**: Webhook receives HTTP POST with Ingress event metadata
2. **Annotation Extraction**: Fetches Ingress annotations using kubectl
3. **Credential Retrieval**: Reads FortiWeb credentials from Kubernetes secret
4. **Policy Verification**: Checks if server policy exists in FortiWeb
5. **Policy Update**: Updates server policy with certificate group configuration

### Integration Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Kubernetes    │───▶│     Webhook     │───▶│    FortiWeb     │
│     Ingress     │    │     Handler     │    │   Server Policy │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                       │                       │
        │                       │                       │
   Annotations              kubectl API             REST API
   - cert-group             - Get ingress           - Check policy
   - fortiweb-ip           - Get secret             - Update policy
   - fortiweb-port         - Extract creds          - Retry logic
   - fortiweb-login
```

## Development

### Local Development

```bash
# Run the webhook server
node server.js

# Test webhook endpoint
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{
    "eventmeta": {
      "kind": "Ingress",
      "reason": "Updated", 
      "namespace": "default",
      "name": "test-ingress"
    }
  }'
```

### Docker Development

```bash
# Build container image
docker build -t webhook:latest .

# Run containerized webhook
docker run -p 8080:8080 webhook:latest
```

### Helm Chart Development

```bash
# Validate Helm templates
helm template webhook ./charts/webhook

# Package chart for distribution
helm package ./charts/webhook -d docs/

# Update repository index
helm repo index docs/ --url https://amerintlxperts.github.io/webhook/

# Publish to GitHub Pages
./doit.sh
```

## API Reference

### Webhook Endpoint

**POST** `/` - Process Ingress webhook events

**Request Body:**
```json
{
  "eventmeta": {
    "kind": "Ingress",
    "reason": "Updated",
    "namespace": "example-namespace", 
    "name": "example-ingress"
  }
}
```

**Response:**
```json
{
  "status": "ok",
  "received": "..."
}
```

### FortiWeb API Integration

The webhook uses FortiWeb's REST API v2.0:

- **Authentication**: Base64 encoded JSON with username/password/vdom
- **Policy Check**: `GET /api/v2.0/cmdb/server-policy/policy?mkey={name}_{namespace}`
- **Policy Update**: `PUT /api/v2.0/cmdb/server-policy/policy?mkey={name}_{namespace}`

## Security Considerations

### Authentication & Authorization

- **RBAC Required**: Webhook needs permissions to read Ingress resources and secrets
- **Secure Credentials**: FortiWeb credentials stored in Kubernetes secrets
- **API Authentication**: Uses FortiWeb's token-based authentication

### Network Security

- **TLS Considerations**: FortiWeb API calls use `--insecure` for self-signed certificates
- **Internal Traffic**: Webhook runs as ClusterIP service for internal access only
- **Container Security**: Based on Alpine Linux for minimal attack surface

### Recommended RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webhook-reader
rules:
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
```

## Monitoring & Troubleshooting

### Health Monitoring

```bash
# Check webhook pods
kubectl get pods -l app.kubernetes.io/name=webhook-instance

# View webhook logs
kubectl logs -l app.kubernetes.io/name=webhook-instance

# Test webhook endpoint
kubectl port-forward svc/webhook 8080:8080
curl -X POST http://localhost:8080 -d '{"test": "data"}'
```

### Common Issues

**Missing Annotations**
- Ensure all required annotations are present on Ingress resources
- Verify annotation names match expected values

**FortiWeb API Errors**
- Check FortiWeb credentials in Kubernetes secret
- Verify FortiWeb IP and port accessibility
- Confirm server policy exists in FortiWeb

**Permission Errors**
- Validate webhook ServiceAccount has required RBAC permissions
- Check kubectl can access Ingress resources and secrets

## Contributing

### Chart Updates

1. Modify chart version in `charts/webhook/Chart.yaml`
2. Update values or templates as needed
3. Test changes with `helm template`
4. Package and publish with `./doit.sh`

### Container Updates

1. Modify `server.js` for functionality changes
2. Update `Dockerfile` if needed
3. Build and test container image
4. Update image tag in Helm values

## License

This project is part of the 40docs platform ecosystem for documentation automation and Kubernetes security integration.