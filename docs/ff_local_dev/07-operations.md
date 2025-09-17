# Operations and Maintenance

Learn how to monitor, debug, and maintain your FireFoundry deployment.

## Updating Deployments

### Update Core Services

```bash
# Update core services with new configuration or code
helm upgrade firefoundry-core . \
  -f values/dev.yaml \
  -f secrets.yaml \
  --namespace ff-dev

# Update control plane services
helm upgrade firefoundry-control . \
  -f values/control-plane.yaml \
  -f secrets.yaml \
  --namespace ff-control-plane

# Update individual agent services
./deploy-app-runner.sh my-service v1.2.0 ff-dev ../my-ff-project/apps/my-service
```

## Monitoring and Debugging

**Recommended approach**: Use k9s for visual monitoring and debugging instead of complex kubectl commands.

```bash
# Open k9s to monitor both namespaces simultaneously
k9s -n ff-dev -n ff-control-plane

# Access ff-console for web-based monitoring
kubectl -n ff-control-plane port-forward svc/firefoundry-control-ff-console 3001:3001
# Open http://localhost:3001 in your browser
```

**k9s provides**:

- Visual pod status and logs
- Resource usage monitoring
- Easy navigation between services
- Built-in port-forwarding capabilities
- No need to memorize kubectl syntax

### Manual Monitoring Commands

If you prefer command-line monitoring:

```bash
# Check pod status
kubectl get pods -n ff-dev
kubectl get pods -n ff-control-plane

# View logs
kubectl logs -n ff-dev deployment/firefoundry-ff-broker
kubectl logs -n ff-control-plane deployment/firefoundry-control-concourse-web

# Check resource usage
kubectl top pods -n ff-dev
kubectl top pods -n ff-control-plane

# Describe problematic resources
kubectl describe pod <pod-name> -n ff-dev
```

## Cleanup and Environment Reset

### Remove Specific Deployments

```bash
# Remove specific deployments
helm uninstall firefoundry-core --namespace ff-dev
helm uninstall firefoundry-control --namespace ff-control-plane
```

### Complete Environment Reset

```bash
# Complete environment reset
kubectl delete namespace ff-dev
kubectl delete namespace ff-control-plane
minikube delete  # Nuclear option - removes entire cluster
```

## Performance Optimization

### Resource Monitoring

```bash
# Monitor resource usage
kubectl top nodes
kubectl top pods -n ff-dev
kubectl top pods -n ff-control-plane

# Check resource limits and requests
kubectl describe pod <pod-name> -n ff-dev | grep -A 5 "Requests\|Limits"
```

### Scaling Services

```bash
# Scale deployments
kubectl scale deployment firefoundry-ff-broker --replicas=3 -n ff-dev

# Check scaling status
kubectl get deployment firefoundry-ff-broker -n ff-dev
```

## Backup and Recovery

### Backup Configuration

```bash
# Backup Helm releases
helm get values firefoundry-core -n ff-dev > ff-core-backup.yaml
helm get values firefoundry-control -n ff-control-plane > ff-control-backup.yaml

# Backup secrets
kubectl get secret myregistrycreds -n ff-dev -o yaml > registry-secret-backup.yaml
```

### Recovery

```bash
# Restore from backup
helm upgrade firefoundry-core . -f ff-core-backup.yaml -n ff-dev
helm upgrade firefoundry-control . -f ff-control-backup.yaml -n ff-control-plane
```

## Next Steps

For troubleshooting specific issues, see:

1. **[Troubleshooting Guide](08-troubleshooting.md)** - Common issues and solutions
