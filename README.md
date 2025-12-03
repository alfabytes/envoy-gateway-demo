# Envoy Gateway Demo

A comprehensive collection of practical examples demonstrating various features and capabilities of [Envoy Gateway](https://gateway.envoyproxy.io/), a cloud-native API Gateway built on top of Envoy Proxy and the Kubernetes Gateway API.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Examples](#examples)
- [Cleanup](#cleanup)

## Overview

This repository contains hands-on examples showcasing different Envoy Gateway features:

- **Backend Routing** - Basic HTTP routing and service discovery

Each example includes:

- Kubernetes deployment manifests
- Service definitions
- HTTPRoute configurations
- Testing instructions
- Cleanup procedures

## Prerequisites

Before you begin, ensure you have:

- A Kubernetes cluster (v1.24+)
- `kubectl` configured to access your cluster
- `helm` v3.0+ installed
- Basic understanding of Kubernetes Gateway API

## Installation

### Step 1: Install Envoy Gateway

Install the Gateway API CRDs and Envoy Gateway using Helm:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.6.0 -n envoy-gateway-system --create-namespace
```

### Step 2: Wait for Envoy Gateway to be Ready

Wait for the Envoy Gateway deployment to become available:

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

### Step 3: Create Gateway Class and Gateway

Create the GatewayClass and default Gateway instance:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
EOF
```

### Step 4: Verify Installation

Verify that the Gateway is ready and has an assigned address:

```bash
kubectl get gateway eg -n envoy-gateway-system
```

You should see the Gateway in a `Ready` state with an assigned address.

## Examples

Each example demonstrates a specific Envoy Gateway feature. Navigate to the respective directory for detailed instructions.

### 1. Backend Routing

**Location:** [`01-backend-routing/`](01-backend-routing/)

Basic HTTP routing example showing how to route traffic to backend services using HTTPRoute resources. This is the foundation for all other examples.

**Key Concepts:**

- HTTPRoute configuration
- Service discovery
- Hostname-based routing
- Path-based matching

## Cleanup

To remove all resources created by the examples:

```bash
# Remove all example resources
kubectl delete httproute --all --all-namespaces
kubectl delete backend --all --all-namespaces
kubectl delete deployment --selector=app -n default
kubectl delete service --selector=app -n default

# Remove Envoy Gateway
kubectl delete gateway eg -n envoy-gateway-system
kubectl delete gatewayclass eg
helm uninstall eg -n envoy-gateway-system
kubectl delete namespace envoy-gateway-system
```

## Additional Resources

- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs)