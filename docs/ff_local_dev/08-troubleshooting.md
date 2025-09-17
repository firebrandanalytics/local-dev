# Troubleshooting Guide

Common issues and solutions when working with FireFoundry.

## Image Pull Errors

### Symptoms

Pods stuck in `ImagePullBackOff` status

### Causes

- Incorrect ACR credentials
- Wrong Azure subscription context
- Missing registry secrets

### Solutions

```bash
# Verify ACR access
az acr login --name firebranddevet
docker pull firebranddevet.azurecr.io/ff-broker:latest

# Recreate registry secrets
kubectl delete secret myregistrycreds -n ff-dev
kubectl delete secret myregistrycreds -n ff-control-plane

# Follow registry setup steps again from [Deployment Guide](04-deployment.md)
```

## Pod Startup Issues

### Symptoms

Pods stuck in `Pending`, `CrashLoopBackOff`, or `Error` states

### Common Causes

- Insufficient resources
- Missing secrets or configmaps
- Image pull issues
- Port conflicts

### Solutions

```bash
# Check pod status and events
kubectl describe pod <pod-name> -n ff-dev

# Check resource availability
kubectl top nodes
kubectl describe node <node-name>

# Check logs
kubectl logs <pod-name> -n ff-dev --previous

# Restart problematic pods
kubectl delete pod <pod-name> -n ff-dev
```

## Service Connectivity Issues

### Symptoms

Services can't communicate with each other

### Common Causes

- Network policies blocking traffic
- Incorrect service names or ports
- DNS resolution issues

### Solutions

```bash
# Test service connectivity
kubectl run test-pod --image=busybox -it --rm -- nslookup firefoundry-ff-broker.ff-dev.svc.cluster.local

# Check service endpoints
kubectl get endpoints -n ff-dev

# Test port forwarding
kubectl port-forward svc/firefoundry-ff-broker 50061:50061 -n ff-dev
```

## Helm Installation Issues

### Symptoms

Helm install/upgrade fails

### Common Causes

- Missing dependencies
- Invalid values files
- Resource conflicts

### Solutions

```bash
# Update dependencies
helm dependency update

# Check for conflicts
helm list -n ff-dev
helm list -n ff-control-plane

# Dry run to validate
helm install firefoundry-core . --dry-run --debug -n ff-dev

# Clean up failed installations
helm uninstall firefoundry-core -n ff-dev
```

## Minikube Issues

### Symptoms

Minikube won't start or has performance issues

### Common Causes

- Insufficient system resources
- Docker not running
- Conflicting virtualization

### Solutions

```bash
# Check minikube status
minikube status

# Restart minikube
minikube stop
minikube start --memory=8192 --cpus=4

# Check system resources
docker system df
minikube ssh "df -h"

# Reset minikube (nuclear option)
minikube delete
minikube start --memory=8192 --cpus=4 --disk-size=20g
```

## GitHub Token Issues

### Symptoms

ff-cli commands fail with authentication errors

### Common Causes

- Missing or invalid GITHUB_TOKEN
- Token doesn't have required scopes
- Environment variable not set

### Solutions

```bash
# Check if token is set
echo $GITHUB_TOKEN

# Test token validity
curl -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user

# Recreate token with correct scopes
# See [Prerequisites Guide](01-prerequisites.md) for detailed steps
```

## Performance Issues

### Symptoms

Slow response times, high resource usage

### Common Causes

- Insufficient resources allocated
- Memory leaks
- Inefficient queries or operations

### Solutions

```bash
# Monitor resource usage
kubectl top pods -n ff-dev
kubectl top nodes

# Check resource limits
kubectl describe pod <pod-name> -n ff-dev | grep -A 5 "Requests\|Limits"

# Scale up resources
kubectl scale deployment <deployment-name> --replicas=2 -n ff-dev
```

## Getting Additional Help

### Debug Information Collection

```bash
# Collect system information
kubectl get all -n ff-dev
kubectl get all -n ff-control-plane
kubectl describe nodes
minikube logs

# Export logs for analysis
kubectl logs deployment/firefoundry-ff-broker -n ff-dev > ff-broker.log
kubectl logs deployment/firefoundry-context-service -n ff-dev > context-service.log
```

### Support Channels

- **Documentation**: Check the Agent SDK docs included in generated projects
- **Logs**: Always check pod logs first when troubleshooting
- **Community**: Internal Teams channels and team knowledge sharing
- **Support**: Reach out to the FireFoundry platform team for infrastructure issues:
  - **Doug**: doug@firebrand.ai
  - **Augustus**: augustus@firebrand.ai
  - **Jose**: jose@firebrand.ai

## Prevention Tips

1. **Regular Updates**: Keep your tools and dependencies updated
2. **Resource Monitoring**: Monitor resource usage regularly
3. **Backup Configurations**: Save your working configurations
4. **Test Changes**: Use staging environments for testing changes
5. **Document Issues**: Keep track of solutions that work for your environment
