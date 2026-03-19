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
- ✅ **Zero infrastructure changes** - uses existing components
- ✅ **Consistent with current pattern** - same domain, same routing logic
- ✅ **Leverages existing middleware** - authentication, rate limiting, logging
- ✅ **Familiar to team** - extends current `service-gaia-gateway-api`

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

## Recommended Choice: Option 1

Based on your infrastructure philosophy of **"골든 패스"** (Golden Path) and operational simplicity, **Option 1** is recommended because:

1. **Minimal Change**: Extends your existing `service-gaia-gateway-api`
2. **Consistent Pattern**: Follows your current URL/routing strategy
3. **Operational Simplicity**: Single entry point, centralized middleware
4. **Team Familiarity**: Uses existing Gin-gonic/Go expertise
5. **Cost Effective**: No additional infrastructure components

You can always evolve to Option 2 later if performance requirements demand it.

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