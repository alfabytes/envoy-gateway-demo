# Circuit Breakers Example

This example demonstrates how to implement circuit breaker patterns with Envoy Gateway using BackendTrafficPolicy. Circuit breakers protect your system from cascading failures by automatically stopping requests to unhealthy backends and allowing them to recover.

## Overview

This example sets up:

- A simple HTTP echo service that responds with a greeting message
- A Kubernetes Service to expose the application
- An HTTPRoute that routes traffic based on hostname
- A Backend resource that references the service endpoint
- A BackendTrafficPolicy with circuit breaker configuration to protect the backend

## Prerequisites

- Envoy Gateway installed and running (see main [README](../README.md))
- `kubectl` configured to access your cluster

## Enable Backend

- By default Backend is disabled. Lets enable it in the EnvoyGateway startup configuration
- The default installation of Envoy Gateway installs a default EnvoyGateway configuration and attaches it using a ConfigMap. In the next step, we will update this resource to enable Backend.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-gateway-config
  namespace: envoy-gateway-system
data:
  envoy-gateway.yaml: |
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: EnvoyGateway
    provider:
      type: Kubernetes
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    extensionApis:
      enableBackend: true
EOF
```

After updating the ConfigMap, you will need to wait the configuration kicks in.
You can force the configuration to be reloaded by restarting the envoy-gateway deployment.

```bash
kubectl rollout restart deployment envoy-gateway -n envoy-gateway-system
```

## What You'll Learn

- How to configure circuit breakers using BackendTrafficPolicy
- How circuit breakers prevent cascading failures
- How to set limits on pending and parallel requests
- How circuit breakers automatically recover when backends become healthy

## Installation

Deploy the application resources in order:

```bash
# Deploy the application
kubectl apply -f deployment.yaml

# Create the service
kubectl apply -f service.yaml

# Create the HTTPRoute and Backend resources
kubectl apply -f httproute.yaml

# Create the BackendTrafficPolicy with circuit breaker configuration
kubectl apply -f traffic-policy.yaml
```

Wait for the deployment to be ready:

```bash
kubectl wait --for=condition=available --timeout=60s deployment/circuit-breakers-app -n default
```

## Configuration Details

### BackendTrafficPolicy

The BackendTrafficPolicy resource configures circuit breaker behavior:

- **maxPendingRequests**: Maximum number of pending requests allowed (0 = unlimited)
- **maxParallelRequests**: Maximum number of parallel requests allowed (10 in this example)

When these limits are exceeded, the circuit breaker opens and requests are rejected immediately, preventing the backend from being overwhelmed.

### How Circuit Breakers Work

1. **Closed State**: Normal operation, requests flow through
2. **Open State**: Circuit opens when limits are exceeded, requests are rejected immediately
3. **Half-Open State**: Circuit periodically tests if backend has recovered
4. **Recovery**: When backend is healthy, circuit closes and normal operation resumes

## Testing

### Step 1: Get the Gateway Address

```bash
export GATEWAY_HOST=$(kubectl get gateway/eg -n envoy-gateway-system -o jsonpath='{.status.addresses[0].value}')
echo "Gateway address: ${GATEWAY_HOST}"
```

### Step 2: Test the Route

Send a request with the configured hostname:

```bash
curl -H "Host: circuit-breakers-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

**Expected Response:**

```text
Hello from Circuit Breakers App! This is a simple circuit-breakers application using Envoy Gateway.
```

### Step 3: Verify Routing

You can also test with verbose output to see the routing in action:

```bash
curl -v -H "Host: circuit-breakers-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

### Step 4: Test Multiple Requests

Send multiple requests to see load balancing across replicas:

```bash
for i in {1..5}; do
  echo "Request $i:"
  curl -H "Host: circuit-breakers-app.alfabytes.xyz" http://${GATEWAY_HOST}
  echo ""
done
```

## Verification

Check that all resources are created and ready:

```bash
# Check deployment
kubectl get deployment circuit-breakers-app -n default

# Check service
kubectl get service circuit-breakers-app -n default

# Check HTTPRoute
kubectl get httproute circuit-breakers-app-httproute -n default

# Check Backend resource
kubectl get backend circuit-breakers-app-backend -n default

# Check BackendTrafficPolicy
kubectl get backendtrafficpolicy circuit-breakers-app-traffic-policy -n default

# View HTTPRoute details
kubectl describe httproute circuit-breakers-app-httproute -n default

# View BackendTrafficPolicy details
kubectl describe backendtrafficpolicy circuit-breakers-app-traffic-policy -n default
```

## Troubleshooting

If the circuit breaker isn't working as expected:

1. **Check BackendTrafficPolicy Status:**

   ```bash
   kubectl get backendtrafficpolicy circuit-breakers-app-traffic-policy -n default -o yaml
   ```

2. **Verify Policy Attachment:**

   ```bash
   kubectl describe backendtrafficpolicy circuit-breakers-app-traffic-policy -n default
   ```

3. **Check Envoy Gateway Logs:**

   ```bash
   kubectl logs -n envoy-gateway-system deployment/envoy-gateway | grep -i circuit
   ```

4. **Monitor Request Metrics:**

   ```bash
   # Watch for circuit breaker state changes
   kubectl logs -f -n envoy-gateway-system deployment/envoy-gateway
   ```

## Cleanup

Remove all resources created by this example:

```bash
# Delete BackendTrafficPolicy
kubectl delete -f traffic-policy.yaml

# Delete HTTPRoute and Backend
kubectl delete -f httproute.yaml

# Delete Service
kubectl delete -f service.yaml

# Delete Deployment
kubectl delete -f deployment.yaml
```

Or delete all resources at once:

```bash
kubectl delete -f deployment.yaml -f service.yaml -f httproute.yaml -f traffic-policy.yaml
```

Verify cleanup:

```bash
kubectl get deployment,service,httproute,backend,backendtrafficpolicy -n default | grep circuit-breakers-app
```

## Next Steps

Once you've mastered circuit breakers, explore other examples:

- [Client Traffic Policy](../03-client-traffic-policy/) - Configure client-side policies
- [Connection Limit](../04-connection-limit/) - Protect your backends with connection limits
- [Failover](../06-failover/) - Implement high availability with automatic failover
- [Fault Injection](../07-fault-injection/) - Test resilience with controlled failures
