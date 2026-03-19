# WebSocket Architecture Proposal
## Based on Current Dr. Aesthetics Infrastructure

---

## Current Infrastructure Analysis

### Your Existing Architecture
```
Client → Nginx Ingress → External Gateway (Go) → Internal Gateway (Traefik) → Microservices
       api.dr-aesthetics.com    service-gaia-gateway-api                    service-{scope}-{topic}-api
```

### Current Microservices (Based on infrastructure.md)
- **`service-gaia-user-api`** - User management, authentication
- **`service-dermacode-skin-api`** - AI skin analysis
- **`service-finance-payment-api`** - Payment processing
- **`service-patient-app-bff`** - Mobile BFF
- **`service-dermacode-consult-api`** - Consultation services

### Current Routing Pattern
- **Domain**: `api.dr-aesthetics.com` (single entry point)
- **URL Format**: `https://api.dr-aesthetics.com/v1/{topic}/{resource}`
- **Examples**:
  - `POST /v1/user/login` → `service-gaia-user-api`
  - `GET /v1/skin/analysis/123` → `service-dermacode-skin-api`
  - `POST /v1/mobile/home` → `service-patient-app-bff`

---

## WebSocket Integration Strategy

### Option 1: Minimal Change - Add WebSocket to External Gateway ⭐ **RECOMMENDED**

This leverages your existing infrastructure with minimal changes.

#### Architecture
```
Client → Nginx Ingress → External Gateway (Enhanced) → Microservices
       api.dr-aesthetics.com    service-gaia-gateway-api     service-{scope}-{topic}-api
                                    (HTTP + WebSocket)          (HTTP + WebSocket)
```

#### Benefits

##### 🚀 **Infrastructure & Operational Benefits**
- ✅ **Zero infrastructure changes** - Uses existing Nginx Ingress, Load Balancer, and Kubernetes setup
- ✅ **Cost effective** - No additional infrastructure costs or new components to manage
- ✅ **Operational simplicity** - Single entry point for both HTTP and WebSocket traffic
- ✅ **Consistent monitoring** - WebSocket metrics integrate with existing Prometheus/Grafana setup
- ✅ **Same DNS management** - No additional subdomains or SSL certificates needed

##### 🔧 **Development & Team Benefits**
- ✅ **Familiar technology stack** - Extends existing Go + Gin-gonic expertise
- ✅ **Consistent patterns** - Follows current naming conventions and routing logic (`/v1/{topic}` → `/ws/v1/{topic}`)
- ✅ **Shared middleware** - Reuses existing JWT authentication, rate limiting, logging, and request ID generation
- ✅ **Unified codebase** - WebSocket logic integrated into existing `service-gaia-gateway-api`
- ✅ **Minimal learning curve** - Team already familiar with gateway codebase

##### 🔒 **Security & Reliability Benefits**
- ✅ **Centralized security** - Single point for authentication, authorization, and security policies
- ✅ **Consistent rate limiting** - Same Redis-based rate limiting for WebSocket connections
- ✅ **Unified logging** - WebSocket traffic appears in same logs as HTTP traffic for easier debugging
- ✅ **Session management** - Can leverage existing session handling and user context
- ✅ **Compliance alignment** - Maintains current security audit trail and access patterns

##### 📈 **Scalability & Performance Benefits**
- ✅ **Horizontal scaling** - Gateway instances can be scaled based on WebSocket + HTTP load
- ✅ **Resource efficiency** - WebSocket connections share resources with HTTP handling
- ✅ **Connection pooling** - Can implement sophisticated connection management at the gateway level
- ✅ **Load distribution** - WebSocket traffic distributed across multiple gateway instances

##### 🔄 **Migration & Evolution Benefits**
- ✅ **Gradual rollout** - Can implement WebSocket endpoints incrementally per service
- ✅ **Backward compatibility** - HTTP APIs remain unchanged during WebSocket implementation
- ✅ **Feature flags** - Easy to enable/disable WebSocket features without deployment changes
- ✅ **Future flexibility** - Can evolve to direct service access later if performance demands it

#### Drawbacks

##### ⚠️ **Performance Limitations**
- ❌ **Additional proxy layer** - WebSocket traffic still goes through gateway (adds ~20-50ms latency)
- ❌ **Gateway bottleneck** - All WebSocket connections must pass through gateway instances
- ❌ **Connection limits** - Gateway memory/CPU limits total concurrent WebSocket connections
- ❌ **Not optimal for high-frequency trading** - Extra hop makes it unsuitable for ultra-low latency requirements

##### 🎯 **Single Point of Failure Concerns**
- ❌ **Gateway dependency** - If gateway goes down, both HTTP and WebSocket traffic affected
- ❌ **Centralized risk** - Gateway bugs or issues impact all real-time communications
- ❌ **Resource contention** - WebSocket connections consume gateway resources, potentially affecting HTTP performance
- ❌ **Scaling coupling** - WebSocket scaling tied to overall gateway scaling decisions

##### 🛠️ **Development & Maintenance Challenges**
- ❌ **Gateway complexity** - Increases complexity of the critical `service-gaia-gateway-api` component
- ❌ **Mixed responsibilities** - Gateway now handles both HTTP proxy and WebSocket connection management
- ❌ **Debugging complexity** - WebSocket issues require debugging through gateway layer
- ❌ **Development overhead** - Changes to WebSocket logic require gateway deployment

##### 📊 **Operational & Monitoring Challenges**
- ❌ **Gateway resource monitoring** - Need to monitor WebSocket memory/connection usage at gateway level
- ❌ **Connection state management** - Must track WebSocket connections across gateway instances
- ❌ **Failover complexity** - WebSocket connections lost when gateway instances restart
- ❌ **Mixed metrics** - WebSocket and HTTP metrics intermingled, potentially complicating analysis

##### ⚡ **Scaling & Performance Limitations**
- ❌ **Memory usage** - Gateway must maintain state for all active WebSocket connections
- ❌ **Connection affinity** - Requires session affinity or complex connection migration
- ❌ **Horizontal scaling complexity** - Adding gateway instances requires WebSocket connection rebalancing
- ❌ **Resource overhead** - Each WebSocket connection consumes gateway memory and file descriptors

##### 🔧 **Technical Debt & Future Concerns**
- ❌ **Gateway bloat** - Gateway service accumulates more responsibilities over time
- ❌ **Migration difficulty** - Moving to direct service access later requires significant refactoring
- ❌ **Testing complexity** - Must test WebSocket functionality within gateway integration tests
- ❌ **Deployment coupling** - WebSocket feature changes tied to gateway deployment cycle

#### Mitigation Strategies for Drawbacks

##### 🛡️ **For Performance & Bottleneck Issues**
- **Connection pooling** - Implement efficient WebSocket connection management
- **Resource limits** - Set per-user and total connection limits to prevent resource exhaustion
- **Monitoring** - Comprehensive metrics on gateway WebSocket performance
- **Caching** - Cache authentication/authorization decisions for WebSocket connections

##### 🔄 **For Single Point of Failure**
- **Multiple gateway instances** - Run 3+ gateway replicas with proper load balancing
- **Graceful degradation** - Implement circuit breakers and fallback to HTTP polling
- **Health checks** - Robust health checking and automatic instance replacement
- **Connection migration** - Design for eventual connection state migration between instances

##### 🧪 **For Development & Operational Complexity**
- **Clear separation** - Separate WebSocket logic into dedicated modules within gateway
- **Feature flags** - Use feature flags to enable/disable WebSocket features independently
- **Comprehensive testing** - Automated tests for WebSocket functionality
- **Documentation** - Clear operational runbooks for WebSocket debugging and troubleshooting

#### Implementation Details

**1. Enhanced External Gateway (service-gaia-gateway-api)**
```go
// Enhanced gateway with WebSocket support
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

type EnhancedGateway struct {
    // Existing HTTP routing
    httpRoutes map[string]string

    // NEW: WebSocket routing
    wsRoutes map[string]string
    upgrader websocket.Upgrader

    // Connection management
    connections map[string]*websocket.Conn
}

// Existing HTTP routing (unchanged)
func (g *EnhancedGateway) HandleHTTP(c *gin.Context) {
    // Your existing HTTP routing logic
    // POST /v1/user/login → service-gaia-user-api
}

// NEW: WebSocket routing
func (g *EnhancedGateway) HandleWebSocket(c *gin.Context) {
    // Extract routing info from path
    // /ws/v1/skin/analysis → service-dermacode-skin-api
    // /ws/v1/consult/session → service-dermacode-consult-api

    path := c.Request.URL.Path
    targetService := g.determineWebSocketTarget(path)

    // Upgrade to WebSocket
    conn, err := g.upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        return
    }

    // Apply existing middleware logic
    userID := g.authenticateWebSocket(c)
    g.rateLimitWebSocket(userID)

    // Forward to target service
    g.forwardWebSocket(conn, targetService, path)
}

func (g *EnhancedGateway) determineWebSocketTarget(path string) string {
    // /ws/v1/skin/analysis → service-dermacode-skin-api
    // /ws/v1/consult/session → service-dermacode-consult-api
    // /ws/v1/user/notifications → service-gaia-user-api

    parts := strings.Split(path, "/")
    if len(parts) >= 4 {
        topic := parts[3] // "skin", "consult", "user"
        return g.wsRoutes[topic]
    }
    return ""
}
```

**2. Nginx Ingress Configuration (minimal change)**
```yaml
# Add WebSocket support to existing Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    # NEW: WebSocket support
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  tls:
  - hosts:
    - api.dr-aesthetics.com
    secretName: api-tls-secret
  rules:
  - host: api.dr-aesthetics.com
    http:
      paths:
      # Existing HTTP routes (unchanged)
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: service-gaia-gateway-api
            port:
              number: 8080
      # NEW: WebSocket routes
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: service-gaia-gateway-api
            port:
              number: 8080  # Same service, same port
```

**3. Microservice Updates (Add WebSocket endpoints)**
```python
# Example: service-dermacode-consult-api
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse

app = FastAPI()

# Existing HTTP endpoints (unchanged)
@app.get("/sessions")
async def get_sessions():
    return {"sessions": [...]}

@app.post("/start-session")
async def start_session():
    return {"session_id": "123"}

# NEW: WebSocket endpoint
@app.websocket("/ws/session/{session_id}")
async def websocket_session(websocket: WebSocket, session_id: str):
    await websocket.accept()

    # WebSocket logic for real-time consultation
    while True:
        data = await websocket.receive_text()
        # Process consultation messages
        await websocket.send_text(f"Response: {data}")

# Health check (add if not exists)
@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

#### URL Structure
```
# HTTP (unchanged)
https://api.dr-aesthetics.com/v1/user/login
https://api.dr-aesthetics.com/v1/skin/analysis
https://api.dr-aesthetics.com/v1/consult/sessions

# WebSocket (new)
wss://api.dr-aesthetics.com/ws/v1/consult/session/123
wss://api.dr-aesthetics.com/ws/v1/skin/analysis/live
wss://api.dr-aesthetics.com/ws/v1/user/notifications
```

---

---

## Alternative: Option 2 - Direct Service Access (Higher Performance)

If you want maximum performance and are willing to make more infrastructure changes:

#### Architecture
```
Client → GCP Load Balancer → Individual Services (Direct)
```

#### Service-Specific Domains
- **WebSocket Chat**: `wss://chat.dr-aesthetics.com/ws` → `service-patient-app-bff`
- **WebSocket Consult**: `wss://consult.dr-aesthetics.com/ws` → `service-dermacode-consult-api`
- **WebSocket Skin Analysis**: `wss://skin.dr-aesthetics.com/ws` → `service-dermacode-skin-api`
- **HTTP APIs (unchanged)**: `https://api.dr-aesthetics.com/v1/*` → `service-gaia-gateway-api`

#### Benefits
- ✅ **Maximum performance** - direct connections, no proxy overhead
- ✅ **Independent scaling** - each WebSocket service scales separately
- ✅ **Service isolation** - WebSocket failures don't affect HTTP APIs

#### Drawbacks
- ❌ **More DNS management** - multiple subdomains
- ❌ **Authentication duplication** - each service needs auth logic
- ❌ **Monitoring complexity** - multiple entry points to monitor

---

## Recommended Choice: Option 1 (with Caveats)

### Why Option 1 is Recommended for Your Current Situation

Based on your infrastructure philosophy of **"골든 패스"** (Golden Path) and operational simplicity, **Option 1** is recommended because:

#### ✅ **Strong Alignment with Current Infrastructure**
1. **Minimal Change**: Extends your existing `service-gaia-gateway-api` without new components
2. **Consistent Pattern**: Follows your current URL/routing strategy (`/v1/{topic}` → `/ws/v1/{topic}`)
3. **Operational Simplicity**: Single entry point, centralized middleware, unified monitoring
4. **Team Familiarity**: Uses existing Gin-gonic/Go expertise your team already has
5. **Cost Effective**: No additional infrastructure costs or new SSL certificates needed

#### ✅ **Perfect for Getting Started**
- **Low Risk**: Extends proven architecture rather than introducing new patterns
- **Fast Implementation**: Can be implemented in 4-6 weeks
- **Incremental Rollout**: Can enable WebSocket per service gradually
- **Fallback Safe**: Easy to disable if issues arise

### ⚠️ **Important Considerations & When to Reconsider**

#### **Option 1 is Great For:**
- **Real-time notifications** (low frequency, user-facing)
- **Chat functionality** (moderate frequency, small groups)
- **Live consultation updates** (session-based, temporary connections)
- **Dashboard live updates** (periodic data refresh)

#### **Option 1 May Not Be Sufficient For:**
- **High-frequency trading** or ultra-low latency requirements (<10ms)
- **Large-scale multiplayer gaming** (100+ concurrent users per session)
- **High-throughput streaming** (video/audio streaming)
- **IoT device management** (1000+ concurrent device connections)

#### **Migration Path: Start with Option 1, Evolve Later**

**Phase 1 (Months 1-6): Option 1 Implementation**
- Get WebSocket working with current architecture
- Gather real-world performance data and usage patterns
- Build team expertise with WebSocket technology
- Establish monitoring and operational procedures

**Phase 2 (Months 6+): Evaluate & Potentially Migrate**
```bash
# If you observe:
Connection count > 10,000 concurrent     # Consider Option 2
Latency requirements < 50ms              # Consider Option 2
Gateway CPU usage > 80% due to WebSocket # Consider Option 2
Frequent gateway-related WebSocket issues # Consider Option 2
```

### **Decision Matrix**

| Factor | Option 1 Score | Reasoning |
|--------|---------------|-----------|
| **Implementation Speed** | ⭐⭐⭐⭐⭐ | 4-6 weeks vs 8-12 weeks |
| **Team Learning Curve** | ⭐⭐⭐⭐⭐ | Extends existing knowledge |
| **Operational Complexity** | ⭐⭐⭐⭐⭐ | Single system to manage |
| **Initial Cost** | ⭐⭐⭐⭐⭐ | Zero additional infrastructure |
| **Performance (< 1000 connections)** | ⭐⭐⭐⭐ | Adequate for most use cases |
| **Performance (> 5000 connections)** | ⭐⭐ | May require Option 2 migration |
| **Scalability Ceiling** | ⭐⭐⭐ | Limited by gateway resources |
| **Future Flexibility** | ⭐⭐⭐⭐ | Can evolve to Option 2 later |

### **Recommendation Summary**

**Start with Option 1** because:
1. **Immediate Value**: Get WebSocket features working quickly with minimal risk
2. **Learn and Measure**: Gather real-world data on your WebSocket usage patterns
3. **Build Expertise**: Team learns WebSocket technology in familiar environment
4. **Preserve Options**: Can migrate to Option 2 when/if performance demands it

**Plan for Option 2** if your business grows to need:
- 5,000+ concurrent WebSocket connections
- <50ms latency requirements
- Independent scaling of different WebSocket services

This approach follows your **"골든 패스"** philosophy: start simple, prove value, then optimize based on real needs rather than theoretical requirements.

---

## Implementation Plan

### Phase 1: Foundation (Week 1-2)
**Goal**: Add WebSocket support to existing External Gateway

#### Week 1: Gateway Enhancement
**Day 1-2: Setup WebSocket Infrastructure**
```bash
# Add WebSocket dependencies to service-gaia-gateway-api
go mod add github.com/gorilla/websocket
```

**Day 3-4: Implement WebSocket Routing**
- Add WebSocket upgrader to existing gateway
- Implement path-based routing logic (`/ws/v1/{topic}`)
- Add WebSocket connection management

**Day 5: Testing & Integration**
- Update Nginx Ingress with WebSocket annotations
- Test WebSocket upgrade and routing
- Integration tests with existing middleware

#### Week 2: Service Integration
**Day 1-2: Update Target Services**
- Add WebSocket endpoints to `service-dermacode-consult-api`
- Add health checks for WebSocket services

**Day 3-4: Authentication Integration**
- Extend existing JWT middleware for WebSocket
- Test authentication flow for WebSocket connections

**Day 5: Monitoring Setup**
- Add WebSocket metrics to existing Prometheus setup
- Update Grafana dashboards

### Phase 2: Service Implementation (Week 3-4)

#### Week 3: Core WebSocket Services
**Priority Services for WebSocket Integration:**

1. **`service-dermacode-consult-api`** (High Priority)
   - Real-time consultation features
   - Doctor-patient communication
   - Live session management

2. **`service-patient-app-bff`** (High Priority)
   - Real-time notifications
   - Chat functionality
   - Live updates

3. **`service-dermacode-skin-api`** (Medium Priority)
   - Live analysis feedback
   - Progress updates

#### Week 4: Testing & Optimization
- Load testing WebSocket connections
- Performance optimization
- Error handling improvements

### Phase 3: Production Deployment (Week 5-6)

#### Week 5: Staging Deployment
- Deploy to staging environment
- End-to-end testing with real client applications
- Performance benchmarking

#### Week 6: Production Rollout
- Gradual rollout using feature flags
- Production monitoring and alerting
- Documentation and team training