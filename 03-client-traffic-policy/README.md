# Client Traffic Policy Example

This example demonstrates how to configure client-side traffic policies with Envoy Gateway using ClientTrafficPolicy resources. ClientTrafficPolicy allows you to configure how Envoy Gateway handles connections from clients, including TCP keepalive settings, timeouts, and other connection-level configurations.

## Overview

This example sets up:

- A simple HTTP echo service that responds with request information
- A Kubernetes Service to expose the application
- An HTTPRoute that routes traffic based on hostname
- A Backend resource that references the service endpoint
- A ClientTrafficPolicy that configures TCP keepalive settings for client connections

## What You'll Learn

- How to configure ClientTrafficPolicy resources
- How to set TCP keepalive parameters for client connections
- How client-side policies differ from backend policies
- How to apply policies at the Gateway level

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

# Create the ClientTrafficPolicy
kubectl apply -f client-traffic-policy.yaml
```

Wait for the deployment to be ready:

```bash
kubectl wait --for=condition=available --timeout=60s deployment/client-traffic-policy-app -n default
```

## Configuration Details

### ClientTrafficPolicy

The ClientTrafficPolicy resource configures how Envoy Gateway handles connections from clients. In this example, we configure TCP keepalive settings:

- **idleTime**: Duration (20 minutes) before sending the first keepalive probe when the connection is idle
- **interval**: Duration (60 seconds) between keepalive probes
- **probes**: Number of keepalive probes (3) to send before considering the connection dead

### How Client Traffic Policies Work

ClientTrafficPolicy applies to connections between clients and the Envoy Gateway, not between the gateway and backends. This allows you to:

- Manage connection lifecycle from clients
- Configure timeouts and keepalive settings
- Control how long idle connections are maintained
- Detect and close stale client connections

### Policy Targeting

The policy targets the Gateway resource (`eg`), which means it applies to all routes through that gateway. You can also target specific HTTPRoutes if you need route-specific policies.

## Testing

### Step 1: Get the Gateway Address

```bash
export GATEWAY_HOST=$(kubectl get gateway/eg -n envoy-gateway-system -o jsonpath='{.status.addresses[0].value}')
echo "Gateway address: ${GATEWAY_HOST}"
```

### Step 2: Test the Route

Send a request with the configured hostname:

```bash
curl -H "Host: client-traffic-policy-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

**Expected Response:**

The echo service will return information about the request, including pod name, namespace, and request details.

### Step 3: Test with Verbose Output

Test with verbose output to see connection details:

```bash
curl -v -H "Host: client-traffic-policy-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

### Step 4: Test Keepalive Behavior

To observe TCP keepalive behavior, you can:

1. **Monitor connection state:**

   ```bash
   # Watch Envoy Gateway logs for keepalive activity
   kubectl logs -f -n envoy-gateway-system deployment/envoy-gateway | grep -i keepalive
   ```

2. **Test with persistent connections:**

   ```bash
   # Use curl with keepalive enabled
   curl -v --keepalive-time 30 \
     -H "Host: client-traffic-policy-app.alfabytes.xyz" \
     http://${GATEWAY_HOST}
   ```

3. **Send multiple requests:**

   ```bash
   for i in {1..5}; do
     echo "Request $i:"
     curl -H "Host: client-traffic-policy-app.alfabytes.xyz" http://${GATEWAY_HOST}
     echo ""
     sleep 2
   done
   ```

## Verification

Check that all resources are created and ready:

```bash
# Check deployment
kubectl get deployment client-traffic-policy-app -n default

# Check service
kubectl get service client-traffic-policy-app -n default

# Check HTTPRoute
kubectl get httproute client-traffic-policy-app-httproute -n default

# Check Backend resource
kubectl get backend client-traffic-policy-app-backend -n default

# Check ClientTrafficPolicy
kubectl get clienttrafficpolicy enable-tcp-keepalive-policy -n envoy-gateway-system

# View HTTPRoute details
kubectl describe httproute client-traffic-policy-app-httproute -n default

# View ClientTrafficPolicy details
kubectl describe clienttrafficpolicy enable-tcp-keepalive-policy -n envoy-gateway-system
```

### Verify Policy Configuration

Check the ClientTrafficPolicy configuration:

```bash
kubectl get clienttrafficpolicy enable-tcp-keepalive-policy -n envoy-gateway-system -o yaml
```

You should see the TCP keepalive settings:
- `idleTime: 20m`
- `interval: 60s`
- `probes: 3`

## Troubleshooting

If the client traffic policy isn't working as expected:

1. **Check ClientTrafficPolicy Status:**

   ```bash
   kubectl get clienttrafficpolicy enable-tcp-keepalive-policy -n envoy-gateway-system -o yaml
   ```

2. **Verify Policy Attachment:**

   ```bash
   kubectl describe clienttrafficpolicy enable-tcp-keepalive-policy -n envoy-gateway-system
   ```

3. **Check Envoy Gateway Logs:**

   ```bash
   kubectl logs -n envoy-gateway-system deployment/envoy-gateway | grep -i "client\|keepalive\|tcp"
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
kubectl get deployment,service,httproute,backend -n default | grep client-traffic-policy-app
kubectl get clienttrafficpolicy -n envoy-gateway-system | grep enable-tcp-keepalive-policy
```

## Next Steps

Once you've mastered client traffic policies, explore other examples:

- [Connection Limit](../04-connection-limit/) - Protect your backends with connection limits
- [Circuit Breakers](../02-circuit-breakers/) - Add fault tolerance with circuit breakers
- [Failover](../06-failover/) - Implement high availability with automatic failover
- [Fault Injection](../07-fault-injection/) - Test resilience with controlled failures
