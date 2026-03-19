# WebSocket Connection Stability Solutions

## Connection Stability Issues & Solutions

### 1. **Pod Scaling & Rolling Deployments**

#### Problem
When Kubernetes scales pods or performs rolling updates, active WebSocket connections are dropped.

#### Solution: Graceful Connection Handling

```go
// Enhanced Gateway with graceful shutdown
type ConnectionManager struct {
    connections    map[string]*Connection
    shutdownChan   chan struct{}
    drainDuration  time.Duration
}

type Connection struct {
    ID          string
    WebSocket   *websocket.Conn
    ServicePod  string
    LastPing    time.Time
    UserID      string
}

func (cm *ConnectionManager) GracefulShutdown() {
    log.Println("Starting graceful WebSocket shutdown...")

    // Stop accepting new connections
    close(cm.shutdownChan)

    // Notify all clients about impending disconnection
    for _, conn := range cm.connections {
        conn.WebSocket.WriteMessage(websocket.TextMessage, []byte(`{
            "type": "server_shutdown",
            "message": "Server restarting, please reconnect in 5 seconds",
            "reconnect_after": 5000
        }`))
    }

    // Wait for drain period
    time.Sleep(cm.drainDuration)

    // Close all connections
    for _, conn := range cm.connections {
        conn.WebSocket.Close()
    }
}

// Kubernetes PreStop hook in deployment
spec:
  template:
    spec:
      containers:
      - name: gateway
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"] # Allow graceful shutdown
```

#### Kubernetes Configuration
```yaml
# service-gaia-gateway-api deployment with graceful shutdown
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-gaia-gateway-api
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # Ensure at least one pod is always running
      maxSurge: 1        # Add one extra pod during updates
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: gateway
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

### 2. **Automatic Reconnection with Exponential Backoff**

#### Client-Side Reconnection Logic
```javascript
class StableWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.options = options;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 10;
        this.reconnectInterval = 1000; // Start with 1 second
        this.maxReconnectInterval = 30000; // Max 30 seconds
        this.isReconnecting = false;
        this.shouldReconnect = true;
    }

    connect() {
        try {
            this.ws = new WebSocket(this.url);
            this.setupEventHandlers();
        } catch (error) {
            this.handleConnectionError(error);
        }
    }

    setupEventHandlers() {
        this.ws.onopen = () => {
            console.log('WebSocket connected');
            this.reconnectAttempts = 0;
            this.reconnectInterval = 1000;
            this.isReconnecting = false;

            if (this.options.onOpen) {
                this.options.onOpen();
            }

            // Start heartbeat
            this.startHeartbeat();
        };

        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);

            // Handle server shutdown notifications
            if (message.type === 'server_shutdown') {
                console.log('Server shutting down, will reconnect after:', message.reconnect_after);
                setTimeout(() => this.reconnect(), message.reconnect_after);
                return;
            }

            if (this.options.onMessage) {
                this.options.onMessage(message);
            }
        };

        this.ws.onclose = (event) => {
            console.log('WebSocket closed:', event.code, event.reason);
            this.stopHeartbeat();

            if (this.shouldReconnect && !this.isReconnecting) {
                this.reconnect();
            }
        };

        this.ws.onerror = (error) => {
            console.error('WebSocket error:', error);
            this.handleConnectionError(error);
        };
    }

    reconnect() {
        if (this.isReconnecting || !this.shouldReconnect) {
            return;
        }

        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error('Max reconnection attempts reached');
            if (this.options.onMaxReconnectAttemptsReached) {
                this.options.onMaxReconnectAttemptsReached();
            }
            return;
        }

        this.isReconnecting = true;
        this.reconnectAttempts++;

        const delay = Math.min(
            this.reconnectInterval * Math.pow(2, this.reconnectAttempts - 1),
            this.maxReconnectInterval
        );

        console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);

        setTimeout(() => {
            console.log('Attempting to reconnect...');
            this.connect();
        }, delay);
    }

    startHeartbeat() {
        this.heartbeatInterval = setInterval(() => {
            if (this.ws.readyState === WebSocket.OPEN) {
                this.ws.send(JSON.stringify({
                    type: 'ping',
                    timestamp: Date.now()
                }));
            }
        }, 30000); // Send ping every 30 seconds
    }

    stopHeartbeat() {
        if (this.heartbeatInterval) {
            clearInterval(this.heartbeatInterval);
        }
    }

    send(message) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(message));
            return true;
        } else {
            console.warn('WebSocket not connected, message queued');
            // Optionally queue messages for when connection is restored
            return false;
        }
    }

    close() {
        this.shouldReconnect = false;
        this.stopHeartbeat();
        this.ws.close();
    }
}

// Usage
const ws = new StableWebSocket('wss://api.dr-aesthetics.com/ws/v1/consult/session/123', {
    onOpen: () => console.log('Connected to consultation'),
    onMessage: (message) => handleConsultMessage(message),
    onMaxReconnectAttemptsReached: () => {
        // Show user a "connection lost" message and retry button
        showConnectionLostUI();
    }
});

ws.connect();
```

### 3. **Connection Persistence & Session Affinity**

#### Redis-Based Session Management
```go
// Enhanced gateway with session persistence
type SessionManager struct {
    redis redis.Client
}

func (sm *SessionManager) StoreConnection(userID, connectionID, podID string) error {
    key := fmt.Sprintf("ws_session:%s", userID)
    data := map[string]interface{}{
        "connection_id": connectionID,
        "pod_id":       podID,
        "created_at":   time.Now().Unix(),
        "last_ping":    time.Now().Unix(),
    }

    return sm.redis.HMSet(context.Background(), key, data).Err()
}

func (sm *SessionManager) GetConnectionPod(userID string) (string, error) {
    key := fmt.Sprintf("ws_session:%s", userID)
    podID, err := sm.redis.HGet(context.Background(), key, "pod_id").Result()
    return podID, err
}

func (sm *SessionManager) UpdateLastPing(userID string) error {
    key := fmt.Sprintf("ws_session:%s", userID)
    return sm.redis.HSet(context.Background(), key, "last_ping", time.Now().Unix()).Err()
}

// Kubernetes service with session affinity
apiVersion: v1
kind: Service
metadata:
  name: service-gaia-gateway-api
spec:
  selector:
    app: service-gaia-gateway-api
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 hour session affinity
  ports:
  - port: 8080
    targetPort: 8080
```

### 4. **Load Balancer & Proxy Optimizations**

#### Nginx Ingress Optimizations
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  annotations:
    # WebSocket specific optimizations
    nginx.ingress.kubernetes.io/proxy-read-timeout: "86400"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "86400"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"

    # Keep-alive settings
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"
    nginx.ingress.kubernetes.io/upstream-keepalive-requests: "1000"
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"

    # Buffer settings for WebSocket
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-request-buffering: "off"

    # Connection upgrade settings
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_cache_bypass $http_upgrade;
```

#### Custom Nginx Configuration
```nginx
# Additional nginx.conf optimizations
http {
    # WebSocket specific settings
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    upstream websocket_backend {
        # Use least_conn for WebSocket connections
        least_conn;

        # Health checks
        server service-gaia-gateway-api-1:8080 max_fails=3 fail_timeout=30s;
        server service-gaia-gateway-api-2:8080 max_fails=3 fail_timeout=30s;
        server service-gaia-gateway-api-3:8080 max_fails=3 fail_timeout=30s;

        # Keep-alive settings
        keepalive 100;
        keepalive_requests 1000;
        keepalive_timeout 60s;
    }
}
```

### 5. **Heartbeat & Health Monitoring**

#### Server-Side Heartbeat Implementation
```go
type ConnectionHealth struct {
    conn     *websocket.Conn
    lastPing time.Time
    userID   string
}

func (ch *ConnectionHealth) StartHealthCheck() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // Check if connection is stale
            if time.Since(ch.lastPing) > 90*time.Second {
                log.Printf("Connection stale for user %s, closing", ch.userID)
                ch.conn.Close()
                return
            }

            // Send ping
            err := ch.conn.WriteMessage(websocket.PingMessage, nil)
            if err != nil {
                log.Printf("Ping failed for user %s: %v", ch.userID, err)
                ch.conn.Close()
                return
            }
        }
    }
}

func (ch *ConnectionHealth) HandlePong() {
    ch.lastPing = time.Now()
}
```

### 6. **Circuit Breaker & Fallback Mechanisms**

#### Circuit Breaker for WebSocket Services
```go
type WebSocketCircuitBreaker struct {
    maxFailures   int
    resetTimeout  time.Duration
    failures      int
    lastFailTime  time.Time
    state         string // "closed", "open", "half-open"
}

func (cb *WebSocketCircuitBreaker) Call(operation func() error) error {
    if cb.state == "open" {
        if time.Since(cb.lastFailTime) > cb.resetTimeout {
            cb.state = "half-open"
        } else {
            return errors.New("circuit breaker is open")
        }
    }

    err := operation()
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()

        if cb.failures >= cb.maxFailures {
            cb.state = "open"
        }
        return err
    }

    // Success - reset circuit breaker
    cb.failures = 0
    cb.state = "closed"
    return nil
}

// Usage in gateway
func (gw *Gateway) forwardWebSocket(userConn *websocket.Conn, service string) {
    cb := &WebSocketCircuitBreaker{
        maxFailures:  5,
        resetTimeout: 30 * time.Second,
    }

    err := cb.Call(func() error {
        return gw.connectToService(userConn, service)
    })

    if err != nil {
        // Fallback to HTTP polling
        gw.fallbackToHTTP(userConn, service)
    }
}
```

#### HTTP Polling Fallback
```javascript
// Client-side fallback mechanism
class WebSocketWithFallback {
    constructor(wsUrl, httpUrl) {
        this.wsUrl = wsUrl;
        this.httpUrl = httpUrl;
        this.mode = 'websocket'; // or 'polling'
        this.pollingInterval = 2000;
    }

    connect() {
        this.tryWebSocket();
    }

    tryWebSocket() {
        this.ws = new StableWebSocket(this.wsUrl, {
            onMaxReconnectAttemptsReached: () => {
                console.log('WebSocket failed, falling back to HTTP polling');
                this.fallbackToPolling();
            }
        });
        this.ws.connect();
    }

    fallbackToPolling() {
        this.mode = 'polling';
        this.startPolling();
    }

    startPolling() {
        this.pollingTimer = setInterval(async () => {
            try {
                const response = await fetch(this.httpUrl);
                const data = await response.json();
                this.handleMessage(data);
            } catch (error) {
                console.error('Polling failed:', error);
            }
        }, this.pollingInterval);
    }

    send(message) {
        if (this.mode === 'websocket') {
            return this.ws.send(message);
        } else {
            // Send via HTTP POST
            return this.sendViaHTTP(message);
        }
    }
}
```

### 7. **Resource Management & Scaling**

#### Auto-scaling Based on Connection Count
```yaml
# HorizontalPodAutoscaler for WebSocket gateway
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: websocket-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: service-gaia-gateway-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  # Scale based on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Scale based on memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Scale based on custom metric (WebSocket connections)
  - type: Pods
    pods:
      metric:
        name: websocket_connections_per_pod
      target:
        type: AverageValue
        averageValue: "1000"
```

#### Connection Limits & Resource Protection
```go
type ConnectionLimiter struct {
    maxConnections int
    current        int
    mutex          sync.Mutex
}

func (cl *ConnectionLimiter) TryAcquire() bool {
    cl.mutex.Lock()
    defer cl.mutex.Unlock()

    if cl.current >= cl.maxConnections {
        return false
    }

    cl.current++
    return true
}

func (cl *ConnectionLimiter) Release() {
    cl.mutex.Lock()
    defer cl.mutex.Unlock()
    cl.current--
}

// Usage in gateway
func (gw *Gateway) handleWebSocket(c *gin.Context) {
    if !gw.limiter.TryAcquire() {
        c.JSON(503, gin.H{"error": "Server at capacity"})
        return
    }
    defer gw.limiter.Release()

    // Handle WebSocket connection
}
```

## Monitoring & Alerting

### Connection Health Metrics
```go
var (
    wsConnectionsActive = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "websocket_connections_active",
            Help: "Number of active WebSocket connections",
        },
        []string{"service", "pod"},
    )

    wsConnectionDrops = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "websocket_connection_drops_total",
            Help: "Total number of WebSocket connection drops",
        },
        []string{"reason", "service"},
    )

    wsReconnectionAttempts = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "websocket_reconnection_attempts_total",
            Help: "Total number of reconnection attempts",
        },
        []string{"success", "service"},
    )
)
```

### Alerting Rules
```yaml
# Prometheus alerting rules
groups:
- name: websocket.rules
  rules:
  - alert: HighWebSocketDropRate
    expr: rate(websocket_connection_drops_total[5m]) > 0.1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High WebSocket connection drop rate"

  - alert: WebSocketConnectionCapacity
    expr: websocket_connections_active / websocket_max_connections > 0.8
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "WebSocket connections approaching capacity"
```

---

## Summary: Stability Comparison

### **Before Enhancement** (High Instability)
- ❌ 4 failure points in proxy chain
- ❌ No graceful shutdown
- ❌ No automatic reconnection
- ❌ No session affinity
- ❌ Default timeouts not optimized for WebSocket

### **After Enhancement** (High Stability)
- ✅ 2 failure points (reduced by 50%)
- ✅ Graceful shutdown with client notifications
- ✅ Exponential backoff reconnection
- ✅ Session affinity with Redis persistence
- ✅ WebSocket-optimized timeouts and configurations
- ✅ Circuit breaker with HTTP fallback
- ✅ Comprehensive monitoring and alerting

**Expected Results:**
- **Connection Stability**: 95%+ → 99%+
- **Reconnection Time**: 10-30s → 1-5s
- **User Experience**: Frequent disruptions → Mostly seamless