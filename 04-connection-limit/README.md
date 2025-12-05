# Connection Limit Example

This example demonstrates how to limit the number of concurrent connections from clients using ClientTrafficPolicy. Connection limits protect your Envoy Gateway and backend services from being overwhelmed by too many simultaneous connections.

## Overview

This example sets up:

- A simple HTTP echo service that responds with request information
- A Kubernetes Service to expose the application
- An HTTPRoute that routes traffic based on hostname
- A Backend resource that references the service endpoint
- A ClientTrafficPolicy that limits concurrent client connections to 5

## What You'll Learn

- How to configure connection limits using ClientTrafficPolicy
- How connection limits protect your infrastructure
- How to test and observe connection limit behavior
- How to apply connection limits at the Gateway level

## Prerequisites

- Envoy Gateway installed and running (see main [README](../README.md))
- `kubectl` configured to access your cluster
- Backend extension enabled (required for Backend resources)

## Enable Backend

If you haven't already enabled the Backend extension, configure it:

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

After updating the ConfigMap, restart the envoy-gateway deployment to apply the configuration:

```bash
kubectl rollout restart deployment envoy-gateway -n envoy-gateway-system
```

Wait for the deployment to be ready:

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## Installation

Deploy the application resources in order:

```bash
# Deploy the application
kubectl apply -f deployment.yaml

# Create the service
kubectl apply -f service.yaml

# Create the HTTPRoute and Backend resources
kubectl apply -f httproute.yaml

# Create the ClientTrafficPolicy with connection limit
kubectl apply -f client-traffic-policy.yaml
```

Wait for the deployment to be ready:

```bash
kubectl wait --for=condition=available --timeout=60s deployment/connection-limit-app -n default
```

## Configuration Details

### ClientTrafficPolicy

The ClientTrafficPolicy resource configures connection limits for client connections:

- **connectionLimit.value**: Maximum number of concurrent connections allowed (5 in this example)

When the connection limit is reached, new connection attempts will be rejected, protecting your gateway and backend services from being overwhelmed.

### How Connection Limits Work

Connection limits apply to the total number of concurrent connections from clients to the Envoy Gateway:

1. **Normal Operation**: Connections are accepted up to the limit
2. **Limit Reached**: When the limit is reached, new connections are rejected
3. **Connection Closed**: When a connection closes, a new one can be accepted
4. **Protection**: This prevents resource exhaustion and ensures fair access

### Policy Targeting

The policy targets the Gateway resource (`eg`), which means it applies to all routes through that gateway. You can also target specific HTTPRoutes if you need route-specific connection limits.

## Testing

### Step 1: Get the Gateway Address

```bash
export GATEWAY_HOST=$(kubectl get gateway/eg -n envoy-gateway-system -o jsonpath='{.status.addresses[0].value}')
echo "Gateway address: ${GATEWAY_HOST}"
```

### Step 2: Test Normal Operation

Send a single request to verify the route works:

```bash
curl -H "Host: connection-limit-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

**Expected Response:**

The echo service will return information about the request, including pod name, namespace, and request details.

### Step 3: Test Connection Limit

To observe the connection limit in action, open multiple concurrent connections:

```bash
# Open 10 concurrent connections (exceeding the limit of 5)
for i in {1..10}; do
  (
    echo "Connection $i:"
    curl -v --max-time 10 \
      -H "Host: connection-limit-app.alfabytes.xyz" \
      http://${GATEWAY_HOST} 2>&1 | head -5
    echo ""
  ) &
done
wait
```

Some connections should succeed, while others may be rejected when the connection limit is reached.

### Step 4: Monitor Connection Behavior

Watch Envoy Gateway logs to see connection limit behavior:

```bash
# In a separate terminal, watch the logs
kubectl logs -f -n envoy-gateway-system deployment/envoy-gateway | grep -i "connection\|limit"
```

### Step 5: Test with Persistent Connections

Test with persistent connections to better observe the limit:

```bash
# Keep connections open for a few seconds
for i in {1..8}; do
  (
    echo "Opening connection $i..."
    curl -v --keepalive-time 5 --max-time 5 \
      -H "Host: connection-limit-app.alfabytes.xyz" \
      http://${GATEWAY_HOST} 2>&1 | grep -E "(Connected|HTTP|connection)"
    sleep 1
  ) &
done
wait
```

## Verification

Check that all resources are created and ready:

```bash
# Check deployment
kubectl get deployment connection-limit-app -n default

# Check service
kubectl get service connection-limit-app -n default

# Check HTTPRoute
kubectl get httproute connection-limit-app-httproute -n default

# Check Backend resource
kubectl get backend connection-limit-app-backend -n default

# Check ClientTrafficPolicy
kubectl get clienttrafficpolicy connection-limit-app-client-traffic-policy -n envoy-gateway-system

# View HTTPRoute details
kubectl describe httproute connection-limit-app-httproute -n default

# View ClientTrafficPolicy details
kubectl describe clienttrafficpolicy connection-limit-app-client-traffic-policy -n envoy-gateway-system
```

### Verify Policy Configuration

Check the ClientTrafficPolicy configuration:

```bash
kubectl get clienttrafficpolicy connection-limit-app-client-traffic-policy -n envoy-gateway-system -o yaml
```

You should see the connection limit setting:
- `connection.connectionLimit.value: 5`

## Troubleshooting

If the connection limit isn't working as expected:

1. **Check ClientTrafficPolicy Status:**

   ```bash
   kubectl get clienttrafficpolicy connection-limit-app-client-traffic-policy -n envoy-gateway-system -o yaml
   ```

2. **Verify Policy Attachment:**

   ```bash
   kubectl describe clienttrafficpolicy connection-limit-app-client-traffic-policy -n envoy-gateway-system
   ```

3. **Check Envoy Gateway Logs:**

   ```bash
   kubectl logs -n envoy-gateway-system deployment/envoy-gateway | grep -i "connection\|limit"
   ```

4. **Verify Gateway Status:**

   ```bash
   kubectl get gateway eg -n envoy-gateway-system
   kubectl describe gateway eg -n envoy-gateway-system
   ```

5. **Check for Policy Conflicts:**

   ```bash
   # List all ClientTrafficPolicies
   kubectl get clienttrafficpolicy --all-namespaces
   ```

6. **Test Connection Count:**

   ```bash
   # Check active connections (may require additional tools)
   # Monitor gateway metrics if available
   ```

## Cleanup

Remove all resources created by this example:

```bash
# Delete ClientTrafficPolicy
kubectl delete -f client-traffic-policy.yaml

# Delete HTTPRoute and Backend
kubectl delete -f httproute.yaml

# Delete Service
kubectl delete -f service.yaml

# Delete Deployment
kubectl delete -f deployment.yaml
```

Or delete all resources at once:

```bash
kubectl delete -f deployment.yaml -f service.yaml -f httproute.yaml -f client-traffic-policy.yaml
```

Verify cleanup:

```bash
kubectl get deployment,service,httproute,backend -n default | grep connection-limit-app
kubectl get clienttrafficpolicy -n envoy-gateway-system | grep connection-limit-app
```

## Next Steps

Once you've mastered connection limits, explore other examples:

- [Circuit Breakers](../02-circuit-breakers/) - Add fault tolerance with circuit breakers
- [Failover](../06-failover/) - Implement high availability with automatic failover
- [Fault Injection](../07-fault-injection/) - Test resilience with controlled failures
- [Direct Response](../05-direct-response/) - Return static responses without backends
