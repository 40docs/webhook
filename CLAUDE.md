# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes webhook handler application that integrates with FortiWeb WAF systems. The webhook listens for Kubernetes Ingress events and automatically updates FortiWeb server policy configurations with intermediate certificate groups based on Ingress annotations.

## Architecture

### Core Components

**Webhook Handler (server.js)**
- HTTP server listening on port 8080 for POST requests
- Processes Kubernetes Ingress `Updated` events from admission controllers
- Extracts FortiWeb configuration from Ingress annotations
- Makes authenticated API calls to FortiWeb to update server policies

**Key Annotations**
The webhook expects these annotations on Ingress resources:
- `server-policy-intermediate-certificate-group`: Certificate group to assign
- `fortiweb-ip`: FortiWeb management IP address  
- `fortiweb-port`: FortiWeb management port
- `fortiweb-login`: Kubernetes secret containing FortiWeb credentials

### Workflow
1. Receives webhook POST with Ingress event metadata
2. Extracts namespace and ingress name from event
3. Fetches Ingress annotations via `kubectl`
4. Retrieves FortiWeb credentials from Kubernetes secret
5. Checks if server policy exists via GET API call (with retry logic)
6. Updates server policy with certificate group via PUT API call

## Development Commands

### Local Development
```bash
# Install dependencies (if package.json existed)
npm install

# Run the server locally
node server.js

# Test webhook endpoint
curl -X POST http://localhost:8080 -H "Content-Type: application/json" -d '{"eventmeta":{"kind":"Ingress","reason":"Updated","namespace":"test","name":"test-ingress"}}'
```

### Docker Development
```bash
# Build container image
docker build -t webhook:latest .

# Run container locally
docker run -p 8080:8080 webhook:latest

# Test containerized webhook
curl -X POST http://localhost:8080 -H "Content-Type: application/json" -d '{"test": "data"}'
```

### Helm Chart Management
```bash
# Package chart for distribution
helm package ./charts/webhook -d docs/

# Update Helm repository index
helm repo index docs/ --url https://amerintlxperts.github.io/webhook/

# Deploy to Kubernetes cluster
helm install webhook ./charts/webhook

# Upgrade existing deployment
helm upgrade webhook ./charts/webhook

# Publish to GitHub Pages (automated via doit.sh)
./doit.sh
```

## Configuration

### Helm Values (charts/webhook/values.yaml)
- **image.repository**: Container image location (`ghcr.io/amerintlxperts/webhook`)
- **replicaCount**: Number of webhook instances (default: 2)
- **tolerations/nodeSelector**: Runs on system node pool with critical addon tolerance

### Required Kubernetes Resources
- **ServiceAccount**: Needs permissions to read Ingress resources and secrets
- **ClusterRole/ClusterRoleBinding**: RBAC for kubectl operations
- **Secret**: FortiWeb credentials (referenced by `fortiweb-login` annotation)

## Security Considerations

### Credential Handling
- FortiWeb credentials stored as base64 in Kubernetes secrets
- Credentials extracted and used for API authentication
- No credentials logged in plaintext

### Network Security
- Uses `--insecure` for FortiWeb API calls (accepts self-signed certificates)
- HTTP webhook endpoint (consider TLS termination at ingress level)
- Runs on port 8080 with ClusterIP service

### Container Security
- Based on `node:current-alpine` (minimal attack surface)
- Installs kubectl binary for Kubernetes API access
- Runs as root user (consider non-root execution)

## API Integration

### FortiWeb REST API
- **Authentication**: Base64 encoded JSON with username/password/vdom
- **Check Policy**: `GET /api/v2.0/cmdb/server-policy/policy?mkey={name}_{namespace}`
- **Update Policy**: `PUT /api/v2.0/cmdb/server-policy/policy?mkey={name}_{namespace}`
- **Retry Logic**: 6 attempts with 30-second delays for policy existence check

### Kubernetes API
- Uses `kubectl` commands for resource access
- Extracts Ingress annotations and secret data
- Requires appropriate RBAC permissions

## Error Handling

The webhook implements comprehensive error handling:
- JSON parsing errors for webhook payload and API responses
- kubectl command execution errors
- FortiWeb API call failures with retry logic
- Missing annotation validation
- Secret data validation

All errors are logged but don't prevent webhook response (returns 200 OK).

## Monitoring and Debugging

### Logging
- Webhook payload logging on receipt
- Command execution logging with full kubectl/curl commands
- API response logging for FortiWeb interactions
- Error logging with context for troubleshooting

### Health Checks
- HTTP 200 response for all POST requests (even on errors)
- HTTP 405 for non-POST requests
- Server startup logging on port 8080

## Deployment Architecture

The webhook is designed to run in Kubernetes clusters with:
- FortiWeb WAF providing ingress traffic filtering
- Automatic certificate management requiring policy updates
- Multiple ingress resources needing synchronized configuration
- High availability via 2 replica deployment on system nodes