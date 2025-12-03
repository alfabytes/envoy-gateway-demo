# Backend Routing Example

This example demonstrates basic HTTP routing with Envoy Gateway, showing how to route incoming requests to backend services using HTTPRoute resources and Backend resources.

## Overview

This example sets up:

- A simple HTTP echo service that responds with a greeting message
- A Kubernetes Service to expose the application
- An HTTPRoute that routes traffic based on hostname
- A Backend resource that references the service endpoint

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

## Installation

Deploy the application resources in order:

```bash
# Deploy the application
kubectl apply -f deployment.yaml

# Create the service
kubectl apply -f service.yaml

# Create the HTTPRoute and Backend resources
kubectl apply -f httproute.yaml
```

Wait for the deployment to be ready:

```bash
kubectl wait --for=condition=available --timeout=60s deployment/backend-routing-app -n default
```

## Testing

### Step 1: Get the Gateway Address

```bash
export GATEWAY_HOST=$(kubectl get gateway/eg -n envoy-gateway-system -o jsonpath='{.status.addresses[0].value}')
echo "Gateway address: ${GATEWAY_HOST}"
```

### Step 2: Test the Route

Send a request with the configured hostname:

```bash
curl -H "Host: backend-routing-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

**Expected Response:**

```text
Hello from Backend Routing App! This is a simple backend-routing application using Envoy Gateway.
```

### Step 3: Verify Routing

You can also test with verbose output to see the routing in action:

```bash
curl -v -H "Host: backend-routing-app.alfabytes.xyz" http://${GATEWAY_HOST}
```

### Step 4: Test Multiple Requests

Send multiple requests to see load balancing across replicas:

```bash
for i in {1..5}; do
  echo "Request $i:"
  curl -H "Host: backend-routing-app.alfabytes.xyz" http://${GATEWAY_HOST}
  echo ""
done
```

## Verification

Check that all resources are created and ready:

```bash
# Check deployment
kubectl get deployment backend-routing-app -n default

# Check service
kubectl get service backend-routing-app -n default

# Check HTTPRoute
kubectl get httproute backend-routing-app-httproute -n default

# Check Backend resource
kubectl get backend backend-routing-app-backend -n default

# View HTTPRoute details
kubectl describe httproute backend-routing-app-httproute -n default
```

## Cleanup

Remove all resources created by this example:

```bash
# Delete HTTPRoute and Backend
kubectl delete -f httproute.yaml

# Delete Service
kubectl delete -f service.yaml

# Delete Deployment
kubectl delete -f deployment.yaml
```

Or delete all resources at once:

```bash
kubectl delete -f deployment.yaml -f service.yaml -f httproute.yaml
```

Verify cleanup:

```bash
kubectl get deployment,service,httproute,backend -n default | grep backend-routing-app
```

## Next Steps

Once you've mastered basic routing, explore other examples:

- [Circuit Breakers](../02-circuit-breakers/) - Add fault tolerance
- [Client Traffic Policy](../03-client-traffic-policy/) - Configure client-side policies
- [Connection Limit](../04-connection-limit/) - Protect your backends
