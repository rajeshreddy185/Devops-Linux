# Helm to ArgoCD Migration

This repository contains the Helm chart and ArgoCD application configuration for deploying the hello-app with zero downtime.

## Structure

- `updated_yaml/` - Helm chart directory
  - `Chart.yaml` - Helm chart metadata
  - `values.yaml` - Default configuration values
  - `templates/` - Kubernetes resource templates
- `argocd-apps/` - ArgoCD application manifests
  - `hello-app.yaml` - ArgoCD application for hello-app

## Zero Downtime Configuration

The deployment is configured for zero downtime with the following settings:

- **Rolling Update Strategy**: Ensures gradual replacement of pods
- **maxSurge: 25%**: Allows 25% additional pods during updates
- **maxUnavailable: 0%**: Ensures no pods are unavailable during updates
- **Readiness and Liveness Probes**: Ensures pods are ready before traffic is sent
- **Multiple Replicas**: Default 2 replicas for high availability

## Deployment Instructions

1. **Update Repository URL**: Replace the `repoURL` in `argocd-apps/hello-app.yaml` with your actual Git repository URL.

2. **Apply ArgoCD Application**:
   ```bash
   kubectl apply -f argocd-apps/hello-app.yaml
   ```

3. **Sync in ArgoCD UI**: 
   - Navigate to ArgoCD UI
   - Find the `hello-app` application
   - Click "Sync" to deploy

## Migration from Existing Helm Deployment

Since you already have the hello-app deployed via Helm, this configuration is designed to:
- Take over existing resources without disruption
- Maintain the same service configuration (NodePort on port 8080)
- Use the same image and labels
- Preserve existing pods during transition

## Customization

Modify `updated_yaml/values.yaml` to customize:
- Replica count
- Resource limits/requests
- Image version
- Service type and ports
- Node affinity and tolerations

## Verification

Check deployment status:
```bash
kubectl get pods -l app=hello-app
kubectl get svc hello-app
```

Test the application:
```bash
NODE_PORT=$(kubectl get svc hello-app -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
curl http://$NODE_IP:$NODE_PORT
```
