# Direct Client-to-Microservice Communication Architecture

## Addendum to WebSocket Architecture Document

This document explores alternative architectures that enable direct client-to-microservice communication, eliminating intermediate proxy layers for improved performance and reduced latency.

---

## Table of Contents

1. [Overview of Direct Communication Patterns](#overview)
2. [Option 1: Service Mesh with Client-Side Discovery](#option-1-service-mesh)
3. [Option 2: Edge Load Balancer with Multiple Endpoints](#option-2-edge-load-balancer)
4. [Option 3: Client-Side Service Discovery](#option-3-client-side-discovery)
5. [Option 4: Hybrid Approach](#option-4-hybrid-approach)
6. [Implementation Comparison](#implementation-comparison)
7. [Recommended Approach](#recommended-approach)

---

## 1. Overview of Direct Communication Patterns

### Benefits of Direct Communication

- **Reduced Latency**: Eliminate proxy overhead (typically 20-50ms improvement)
- **Better Scalability**: Distribute load across service instances directly
- **Improved Reliability**: Fewer single points of failure
- **Cost Efficiency**: Reduce proxy infrastructure costs
- **Service Autonomy**: Services handle their own traffic management

### Challenges to Address

- **Service Discovery**: Clients need to find available service instances
- **Load Balancing**: Distribute traffic without central load balancer
- **Security**: Authentication/authorization without gateway
- **Cross-Cutting Concerns**: Observability, rate limiting, etc.

---

## 2. Option 1: Service Mesh with Client-Side Discovery

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                      │
│   ┌─────────────────┐    ┌─────────────────┐                   │
│   │   Web Client    │    │  Mobile Client  │                   │
│   │ + Service Mesh  │    │ + Service Mesh  │                   │
│   │   Sidecar       │    │   Sidecar       │                   │
│   └─────────┬───────┘    └─────────┬───────┘                   │
└─────────────┼─────────────────────────┼───────────────────────────┘
              │ Service Discovery        │
              │ + Load Balancing         │
┌─────────────▼─────────────────────────▼───────────────────────────┐
│                    Service Mesh Control Plane                   │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │              Istio/Linkerd/Consul Connect              │     │
│   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │     │
│   │  │   Service   │ │   Traffic   │ │  Security   │       │     │
│   │  │ Discovery   │ │   Policies  │ │   Policies  │       │     │
│   │  └─────────────┘ └─────────────┘ └─────────────┘       │     │
│   └─────────────────────────────────────────────────────────┘     │
└─────────────────┬─────────────────┬───────────────────────────────┘
                  │ Direct secure   │ Direct secure
                  │ connections     │ connections
┌─────────────────▼─────────────────▼───────────────────────────────┐
│                   Kubernetes Cluster                             │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
│ │ WebSocket   │ │    Chat     │ │Notification │ │   Consult   │  │
│ │ Service     │ │   Service   │ │  Service    │ │  Service    │  │
│ │ + Sidecar   │ │ + Sidecar   │ │ + Sidecar   │ │ + Sidecar   │  │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation with Istio

```yaml
# Istio Gateway for direct service access
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: websocket-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  # Chat service direct access
  - port:
      number: 443
      name: chat-wss
      protocol: HTTPS
    hosts:
    - chat.dr-aesthetics.com
    tls:
      mode: SIMPLE
      credentialName: chat-tls-secret
  # Notification service direct access
  - port:
      number: 443
      name: notification-wss
      protocol: HTTPS
    hosts:
    - notifications.dr-aesthetics.com
    tls:
      mode: SIMPLE
      credentialName: notifications-tls-secret
  # Consult service direct access
  - port:
      number: 443
      name: consult-wss
      protocol: HTTPS
    hosts:
    - consult.dr-aesthetics.com
    tls:
      mode: SIMPLE
      credentialName: consult-tls-secret
---
# Virtual Service for WebSocket routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: chat-service-vs
spec:
  hosts:
  - chat.dr-aesthetics.com
  gateways:
  - websocket-gateway
  http:
  - match:
    - headers:
        upgrade:
          exact: websocket
    route:
    - destination:
        host: chat-service
        port:
          number: 8080
    timeout: 3600s
---
# Destination Rule for load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: chat-service-dr
spec:
  host: chat-service
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: "X-Client-ID"
  portLevelSettings:
  - port:
      number: 8080
    connectionPool:
      tcp:
        maxConnections: 1000
        connectTimeout: 30s
        keepalive:
          time: 7200s
          interval: 30s
```

### Client-Side Implementation

```javascript
// JavaScript client with service discovery
class ServiceMeshWebSocketClient {
  constructor() {
    this.serviceRegistry = new Map();
    this.connections = new Map();
    this.discoveryEndpoint = 'https://service-discovery.dr-aesthetics.com';
  }

  async connectToService(serviceName, clientId) {
    // Get service endpoints from mesh
    const serviceInfo = await this.discoverService(serviceName);

    // Select endpoint based on consistent hashing
    const endpoint = this.selectEndpoint(serviceInfo.endpoints, clientId);

    // Establish direct WebSocket connection
    const wsUrl = `wss://${endpoint.host}:${endpoint.port}/ws`;
    const ws = new WebSocket(wsUrl, [], {
      headers: {
        'X-Client-ID': clientId,
        'Authorization': `Bearer ${this.getAuthToken()}`
      }
    });

    this.setupConnectionHandlers(ws, serviceName, clientId);
    this.connections.set(serviceName, ws);

    return ws;
  }

  async discoverService(serviceName) {
    // Query service mesh for available endpoints
    const response = await fetch(
      `${this.discoveryEndpoint}/services/${serviceName}/endpoints`
    );
    return await response.json();
  }

  selectEndpoint(endpoints, clientId) {
    // Consistent hash-based selection
    const hash = this.hashString(clientId);
    const index = hash % endpoints.length;
    return endpoints[index];
  }

  setupConnectionHandlers(ws, serviceName, clientId) {
    ws.onopen = () => {
      console.log(`Connected to ${serviceName}`);
      this.heartbeat(ws);
    };

    ws.onclose = (event) => {
      console.log(`Disconnected from ${serviceName}: ${event.code}`);
      // Implement exponential backoff reconnection
      this.scheduleReconnect(serviceName, clientId);
    };

    ws.onerror = (error) => {
      console.error(`WebSocket error for ${serviceName}:`, error);
    };

    ws.onmessage = (event) => {
      this.handleMessage(JSON.parse(event.data), serviceName);
    };
  }

  async scheduleReconnect(serviceName, clientId, attempt = 1) {
    const delay = Math.min(1000 * Math.pow(2, attempt), 30000);

    setTimeout(async () => {
      try {
        await this.connectToService(serviceName, clientId);
      } catch (error) {
        console.error(`Reconnection attempt ${attempt} failed:`, error);
        this.scheduleReconnect(serviceName, clientId, attempt + 1);
      }
    }, delay);
  }

  heartbeat(ws) {
    const interval = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: 'ping', timestamp: Date.now() }));
      } else {
        clearInterval(interval);
      }
    }, 30000);
  }
}

// Usage
const client = new ServiceMeshWebSocketClient();

// Connect to multiple services directly
async function initializeConnections() {
  const clientId = generateClientId();

  await client.connectToService('chat', clientId);
  await client.connectToService('notifications', clientId);
  await client.connectToService('consult', clientId);
}
```

---

## 3. Option 2: Edge Load Balancer with Multiple Endpoints

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                            │
└─────────────────┬───────────────────────────────────────────────┘
                  │ Service-specific domains
┌─────────────────▼───────────────────────────────────────────────┐
│                   GCP Cloud Load Balancer                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Global Load Balancer                    │   │
│   │  ┌─────────────────┐    ┌─────────────────┐           │   │
│   │  │  chat.dr-...    │    │notifications.dr-│           │   │
│   │  │  (Backend A)    │    │  ... (Backend B)│           │   │
│   │  └─────────────────┘    └─────────────────┘           │   │
│   │  ┌─────────────────┐    ┌─────────────────┐           │   │
│   │  │ consult.dr-...  │    │   api.dr-...    │           │   │
│   │  │  (Backend C)    │    │  (Backend D)    │           │   │
│   │  └─────────────────┘    └─────────────────┘           │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────┬─────────────────┬───────────────────────────────┘
                  │ Direct to K8s   │ Via Nginx Ingress
                  │ Services        │ (existing)
┌─────────────────▼─────────────────▼───────────────────────────────┐
│                    Kubernetes Services                           │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
│ │    Chat     │ │Notification │ │   Consult   │ │   Gateway   │  │
│ │  Service    │ │  Service    │ │  Service    │ │   (HTTP)    │  │
│ │  (NodePort) │ │ (NodePort)  │ │ (NodePort)  │ │ (ClusterIP) │  │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### GCP Load Balancer Configuration

```yaml
# Terraform configuration for GCP Load Balancer
resource "google_compute_global_address" "websocket_ip" {
  name = "websocket-global-ip"
}

# SSL Certificate for multiple domains
resource "google_compute_managed_ssl_certificate" "websocket_ssl" {
  name = "websocket-ssl-cert"

  managed {
    domains = [
      "chat.dr-aesthetics.com",
      "notifications.dr-aesthetics.com",
      "consult.dr-aesthetics.com"
    ]
  }
}

# Backend service for Chat
resource "google_compute_backend_service" "chat_backend" {
  name        = "chat-websocket-backend"
  port_name   = "websocket"
  protocol    = "HTTP"
  timeout_sec = 3600

  enable_cdn = false

  backend {
    group = google_container_node_pool.primary.instance_group_urls[0]
  }

  health_checks = [google_compute_health_check.websocket_health.id]

  custom_request_headers = [
    "X-Service-Name: chat"
  ]
}

# Backend service for Notifications
resource "google_compute_backend_service" "notifications_backend" {
  name        = "notifications-websocket-backend"
  port_name   = "websocket"
  protocol    = "HTTP"
  timeout_sec = 3600

  backend {
    group = google_container_node_pool.primary.instance_group_urls[0]
  }

  health_checks = [google_compute_health_check.websocket_health.id]
}

# URL Map for routing
resource "google_compute_url_map" "websocket_url_map" {
  name            = "websocket-url-map"
  default_service = google_compute_backend_service.chat_backend.id

  host_rule {
    hosts        = ["chat.dr-aesthetics.com"]
    path_matcher = "chat-matcher"
  }

  host_rule {
    hosts        = ["notifications.dr-aesthetics.com"]
    path_matcher = "notifications-matcher"
  }

  host_rule {
    hosts        = ["consult.dr-aesthetics.com"]
    path_matcher = "consult-matcher"
  }

  path_matcher {
    name            = "chat-matcher"
    default_service = google_compute_backend_service.chat_backend.id
  }

  path_matcher {
    name            = "notifications-matcher"
    default_service = google_compute_backend_service.notifications_backend.id
  }

  path_matcher {
    name            = "consult-matcher"
    default_service = google_compute_backend_service.consult_backend.id
  }
}

# HTTPS Proxy
resource "google_compute_target_https_proxy" "websocket_proxy" {
  name             = "websocket-https-proxy"
  url_map          = google_compute_url_map.websocket_url_map.id
  ssl_certificates = [google_compute_managed_ssl_certificate.websocket_ssl.id]
}

# Forwarding Rule
resource "google_compute_global_forwarding_rule" "websocket_forwarding_rule" {
  name       = "websocket-forwarding-rule"
  target     = google_compute_target_https_proxy.websocket_proxy.id
  port_range = "443"
  ip_address = google_compute_global_address.websocket_ip.address
}
```

### Kubernetes Service Configuration

```yaml
# Chat Service with NodePort
apiVersion: v1
kind: Service
metadata:
  name: chat-service-nodeport
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"ports": {"websocket":"chat-backend-config"}}'
spec:
  type: NodePort
  selector:
    app: chat-service
  ports:
  - name: websocket
    port: 8080
    targetPort: 8080
    nodePort: 30080
---
# Backend Config for Chat Service
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: chat-backend-config
spec:
  timeoutSec: 3600
  connectionDraining:
    drainingTimeoutSec: 60
  sessionAffinity:
    affinityType: "CLIENT_IP"
    affinityCookieTtlSec: 3600
  healthCheck:
    checkIntervalSec: 30
    timeoutSec: 10
    healthyThreshold: 1
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /health
    port: 8080
---
# Similar configurations for other services...
```

---

## 4. Option 3: Client-Side Service Discovery

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                      │
│   ┌─────────────────┐    ┌─────────────────┐                   │
│   │   Web Client    │    │  Mobile Client  │                   │
│   │ + Discovery SDK │    │ + Discovery SDK │                   │
│   └─────────┬───────┘    └─────────┬───────┘                   │
└─────────────┼─────────────────────────┼───────────────────────────┘
              │ Query endpoints         │
┌─────────────▼─────────────────────────▼───────────────────────────┐
│                   Service Discovery API                         │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │              Consul/etcd/Custom API                 │     │
│   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │     │
│   │  │  Service    │ │   Health    │ │   Load      │     │     │
│   │  │ Registry    │ │   Status    │ │ Balancing   │     │     │
│   │  └─────────────┘ └─────────────┘ └─────────────┘     │     │
│   └─────────────────────────────────────────────────────────┘     │
└─────────────────┬─────────────────┬───────────────────────────────┘
                  │ Service info    │ Service info
┌─────────────────▼─────────────────▼───────────────────────────────┐
│                     Kubernetes Services                          │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
│ │    Chat     │ │Notification │ │   Consult   │ │    Auth     │  │
│ │  Service    │ │  Service    │ │  Service    │ │  Service    │  │
│ │  (ClusterIP)│ │ (ClusterIP) │ │ (ClusterIP) │ │ (ClusterIP) │  │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Service Discovery Implementation

```go
// Service Discovery Server
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type ServiceDiscoveryServer struct {
    k8sClient kubernetes.Interface
    services  map[string]*ServiceInfo
}

type ServiceInfo struct {
    Name      string              `json:"name"`
    Endpoints []ServiceEndpoint   `json:"endpoints"`
    Health    string              `json:"health"`
    Updated   time.Time           `json:"updated"`
}

type ServiceEndpoint struct {
    Host   string            `json:"host"`
    Port   int               `json:"port"`
    Scheme string            `json:"scheme"`
    Weight int               `json:"weight"`
    Tags   map[string]string `json:"tags"`
}

func NewServiceDiscoveryServer(k8sClient kubernetes.Interface) *ServiceDiscoveryServer {
    return &ServiceDiscoveryServer{
        k8sClient: k8sClient,
        services:  make(map[string]*ServiceInfo),
    }
}

func (sds *ServiceDiscoveryServer) StartDiscovery() {
    // Watch Kubernetes services and endpoints
    go sds.watchServices()

    // Periodic health checks
    go sds.healthCheckLoop()
}

func (sds *ServiceDiscoveryServer) GetService(c *gin.Context) {
    serviceName := c.Param("service")

    service, exists := sds.services[serviceName]
    if !exists {
        c.JSON(http.StatusNotFound, gin.H{
            "error": "Service not found",
        })
        return
    }

    // Filter healthy endpoints
    healthyEndpoints := make([]ServiceEndpoint, 0)
    for _, endpoint := range service.Endpoints {
        if sds.isEndpointHealthy(endpoint) {
            healthyEndpoints = append(healthyEndpoints, endpoint)
        }
    }

    response := &ServiceInfo{
        Name:      service.Name,
        Endpoints: healthyEndpoints,
        Health:    service.Health,
        Updated:   service.Updated,
    }

    c.JSON(http.StatusOK, response)
}

func (sds *ServiceDiscoveryServer) watchServices() {
    // Watch for Kubernetes service and endpoint changes
    for {
        services, err := sds.k8sClient.CoreV1().Services("").List(
            context.TODO(), metav1.ListOptions{
                LabelSelector: "websocket=enabled",
            })

        if err != nil {
            time.Sleep(5 * time.Second)
            continue
        }

        for _, service := range services.Items {
            sds.updateServiceEndpoints(service.Name, service)
        }

        time.Sleep(30 * time.Second)
    }
}

func (sds *ServiceDiscoveryServer) updateServiceEndpoints(serviceName string, service interface{}) {
    // Implementation to update service endpoints
    // This would query Kubernetes endpoints and update sds.services
}

func (sds *ServiceDiscoveryServer) isEndpointHealthy(endpoint ServiceEndpoint) bool {
    // Implement health check logic
    client := &http.Client{Timeout: 5 * time.Second}

    healthURL := fmt.Sprintf("%s://%s:%d/health",
        endpoint.Scheme, endpoint.Host, endpoint.Port)

    resp, err := client.Get(healthURL)
    if err != nil {
        return false
    }
    defer resp.Body.Close()

    return resp.StatusCode == http.StatusOK
}
```

### Client-Side Discovery SDK

```javascript
// JavaScript Service Discovery SDK
class ServiceDiscovery {
  constructor(discoveryEndpoint) {
    this.discoveryEndpoint = discoveryEndpoint;
    this.serviceCache = new Map();
    this.cacheExpiry = 30000; // 30 seconds
  }

  async discoverService(serviceName) {
    // Check cache first
    const cached = this.serviceCache.get(serviceName);
    if (cached && Date.now() - cached.timestamp < this.cacheExpiry) {
      return cached.data;
    }

    try {
      const response = await fetch(
        `${this.discoveryEndpoint}/services/${serviceName}`
      );

      if (!response.ok) {
        throw new Error(`Service discovery failed: ${response.statusText}`);
      }

      const serviceInfo = await response.json();

      // Cache the result
      this.serviceCache.set(serviceName, {
        data: serviceInfo,
        timestamp: Date.now()
      });

      return serviceInfo;
    } catch (error) {
      console.error(`Failed to discover service ${serviceName}:`, error);

      // Return cached data if available, even if expired
      const cached = this.serviceCache.get(serviceName);
      if (cached) {
        console.warn(`Using stale cache for service ${serviceName}`);
        return cached.data;
      }

      throw error;
    }
  }

  selectEndpoint(endpoints, clientId, algorithm = 'consistent-hash') {
    if (!endpoints || endpoints.length === 0) {
      throw new Error('No healthy endpoints available');
    }

    switch (algorithm) {
      case 'consistent-hash':
        return this.consistentHashSelection(endpoints, clientId);
      case 'weighted-random':
        return this.weightedRandomSelection(endpoints);
      case 'round-robin':
        return this.roundRobinSelection(endpoints);
      default:
        return endpoints[0];
    }
  }

  consistentHashSelection(endpoints, clientId) {
    const hash = this.hashString(clientId);
    const index = hash % endpoints.length;
    return endpoints[index];
  }

  weightedRandomSelection(endpoints) {
    const totalWeight = endpoints.reduce((sum, ep) => sum + (ep.weight || 1), 0);
    let random = Math.random() * totalWeight;

    for (const endpoint of endpoints) {
      random -= (endpoint.weight || 1);
      if (random <= 0) {
        return endpoint;
      }
    }

    return endpoints[0];
  }

  hashString(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }

  // Invalidate cache for a service
  invalidateCache(serviceName) {
    this.serviceCache.delete(serviceName);
  }

  // Clear all cached data
  clearCache() {
    this.serviceCache.clear();
  }
}

// WebSocket Client with Service Discovery
class DirectWebSocketClient {
  constructor(discoveryEndpoint) {
    this.discovery = new ServiceDiscovery(discoveryEndpoint);
    this.connections = new Map();
    this.reconnectAttempts = new Map();
  }

  async connect(serviceName, clientId, options = {}) {
    try {
      // Discover service endpoints
      const serviceInfo = await this.discovery.discoverService(serviceName);

      // Select best endpoint
      const endpoint = this.discovery.selectEndpoint(
        serviceInfo.endpoints,
        clientId,
        options.loadBalancingAlgorithm
      );

      // Create WebSocket connection
      const wsUrl = `${endpoint.scheme}://${endpoint.host}:${endpoint.port}/ws`;
      const ws = new WebSocket(wsUrl, [], {
        headers: {
          'X-Client-ID': clientId,
          'X-Service-Name': serviceName,
          'Authorization': options.authToken ? `Bearer ${options.authToken}` : undefined
        }
      });

      // Setup connection handlers
      this.setupConnectionHandlers(ws, serviceName, clientId, options);

      // Store connection
      this.connections.set(serviceName, ws);

      return ws;
    } catch (error) {
      console.error(`Failed to connect to ${serviceName}:`, error);
      throw error;
    }
  }

  setupConnectionHandlers(ws, serviceName, clientId, options) {
    ws.onopen = () => {
      console.log(`Connected to ${serviceName}`);
      this.reconnectAttempts.delete(serviceName);

      // Start heartbeat
      this.startHeartbeat(ws, serviceName);

      // Notify application
      if (options.onConnect) {
        options.onConnect(serviceName);
      }
    };

    ws.onclose = (event) => {
      console.log(`Disconnected from ${serviceName}: ${event.code}`);

      // Clean up
      this.connections.delete(serviceName);

      // Attempt reconnection if not intentional close
      if (event.code !== 1000 && options.autoReconnect !== false) {
        this.scheduleReconnect(serviceName, clientId, options);
      }

      if (options.onDisconnect) {
        options.onDisconnect(serviceName, event);
      }
    };

    ws.onerror = (error) => {
      console.error(`WebSocket error for ${serviceName}:`, error);

      if (options.onError) {
        options.onError(serviceName, error);
      }
    };

    ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);

        if (options.onMessage) {
          options.onMessage(serviceName, message);
        }
      } catch (error) {
        console.error(`Failed to parse message from ${serviceName}:`, error);
      }
    };
  }

  async scheduleReconnect(serviceName, clientId, options) {
    const attempts = this.reconnectAttempts.get(serviceName) || 0;
    const maxAttempts = options.maxReconnectAttempts || 10;

    if (attempts >= maxAttempts) {
      console.error(`Max reconnection attempts reached for ${serviceName}`);
      return;
    }

    const delay = Math.min(1000 * Math.pow(2, attempts), 30000);
    this.reconnectAttempts.set(serviceName, attempts + 1);

    console.log(`Scheduling reconnect to ${serviceName} in ${delay}ms (attempt ${attempts + 1})`);

    setTimeout(async () => {
      try {
        // Invalidate cache to get fresh endpoints
        this.discovery.invalidateCache(serviceName);

        // Attempt reconnection
        await this.connect(serviceName, clientId, options);
      } catch (error) {
        console.error(`Reconnection attempt ${attempts + 1} failed:`, error);
      }
    }, delay);
  }

  startHeartbeat(ws, serviceName) {
    const interval = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({
          type: 'ping',
          timestamp: Date.now()
        }));
      } else {
        clearInterval(interval);
      }
    }, 30000);
  }

  send(serviceName, message) {
    const ws = this.connections.get(serviceName);
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(message));
      return true;
    }

    console.warn(`Cannot send message to ${serviceName}: connection not available`);
    return false;
  }

  disconnect(serviceName) {
    const ws = this.connections.get(serviceName);
    if (ws) {
      ws.close(1000, 'Client disconnect');
      this.connections.delete(serviceName);
    }
  }

  disconnectAll() {
    for (const [serviceName, ws] of this.connections) {
      ws.close(1000, 'Client shutdown');
    }
    this.connections.clear();
  }
}

// Usage Example
async function initializeDirectConnections() {
  const client = new DirectWebSocketClient('https://discovery.dr-aesthetics.com');
  const clientId = generateClientId();

  const connectionOptions = {
    authToken: getAuthToken(),
    autoReconnect: true,
    maxReconnectAttempts: 5,
    loadBalancingAlgorithm: 'consistent-hash',

    onConnect: (serviceName) => {
      console.log(`Successfully connected to ${serviceName}`);
    },

    onMessage: (serviceName, message) => {
      handleServiceMessage(serviceName, message);
    },

    onDisconnect: (serviceName, event) => {
      console.log(`Disconnected from ${serviceName}:`, event.code);
    },

    onError: (serviceName, error) => {
      console.error(`Error with ${serviceName}:`, error);
    }
  };

  // Connect to multiple services directly
  try {
    await Promise.all([
      client.connect('chat', clientId, connectionOptions),
      client.connect('notifications', clientId, connectionOptions),
      client.connect('consult', clientId, connectionOptions)
    ]);

    console.log('All WebSocket connections established');

    // Example usage
    client.send('chat', {
      type: 'join_room',
      room: 'general',
      user: getCurrentUser()
    });

  } catch (error) {
    console.error('Failed to establish connections:', error);
  }
}
```

---

## 5. Option 4: Hybrid Approach

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"content": "Analyze direct client-to-microservice communication patterns", "status": "completed", "activeForm": "Analyzing direct client-to-microservice communication patterns"}, {"content": "Design service mesh architecture with client-side load balancing", "status": "completed", "activeForm": "Designing service mesh architecture with client-side load balancing"}, {"content": "Evaluate edge computing and CDN-based solutions", "status": "in_progress", "activeForm": "Evaluating edge computing and CDN-based solutions"}, {"content": "Update architecture document with direct communication options", "status": "pending", "activeForm": "Updating architecture document with direct communication options"}]