# VCP v1.1 Sidecar Architecture

**Document ID:** VSO-IMPL-001-SIDE  
**Parent:** [VCP v1.1 Implementation Guide](../README.md)

---

## Overview

VCP operates as a **sidecar** — a non-invasive component that runs alongside existing trading systems without modifying their core logic.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXISTING TRADING INFRASTRUCTURE                   │
│                         (NO MODIFICATIONS)                           │
├─────────────────────────────────────────────────────────────────────┤
│  [Trading Algo] [Risk Mgmt] [Order Mgmt] [Market Data]              │
│                         │                                            │
│                   [Event Stream]                                     │
│                         │                                            │
├─────────────────────────┼────────────────────────────────────────────┤
│                         ▼                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                       VCP SIDECAR                            │    │
│  │                                                              │    │
│  │  [Capture] → [Hash] → [Sign] → [Merkle] → [Anchor]         │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                          VCP LAYER                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Design Principles

| Principle | Description | Rationale |
|-----------|-------------|-----------|
| **Non-invasive** | No changes to trading logic | Zero risk to production |
| **Fail-safe** | VCP failure ≠ trading failure | Trading continuity preserved |
| **Async-first** | Event capture is asynchronous | No latency impact |
| **Idempotent** | Duplicate handling is safe | Network reliability |
| **Recoverable** | Support replay after outages | Data completeness |

---

## Integration Patterns

### 1. API Interception

Copy REST/FIX traffic without modifying endpoints.

```
[Trading System] ──────────────────────▶ [Broker]
        │
        │ (copy)
        ▼
[VCP Sidecar]
```

**Best For:** Gold/Platinum tiers, REST APIs, FIX gateways

**Implementation:**

```python
# Nginx-based interception
location /api/orders {
    mirror /vcp-mirror;
    proxy_pass http://broker-backend;
}

location /vcp-mirror {
    internal;
    proxy_pass http://vcp-sidecar:8080/capture;
}
```

### 2. In-Process Hook

Library/DLL integration within the trading application.

```
┌─────────────────────────────┐
│     Trading Application     │
│                             │
│  [Order Logic] ──▶ [Hook]   │
│                      │      │
│                      ▼      │
│               [VCP Client]  │
└─────────────────────────────┘
```

**Best For:** Silver tier, MT4/MT5, custom applications

**Implementation:** See [mt5.md](./mt5.md)

### 3. Message Queue Tap

Subscribe to existing Kafka/Redis streams.

```
[Trading System] ──▶ [Kafka] ──▶ [Broker]
                        │
                        │ (subscribe)
                        ▼
                  [VCP Sidecar]
```

**Best For:** Institutional, event-driven architectures

**Implementation:**

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'orders',
    'executions',
    bootstrap_servers=['kafka:9092'],
    group_id='vcp-sidecar'
)

for message in consumer:
    event = parse_trading_event(message.value)
    vcp_client.log(event.type, event.payload)
```

---

## Component Responsibilities

### Event Capture

```python
class EventCapture:
    """Capture trading events from various sources."""
    
    def __init__(self, sources: list):
        self.sources = sources
        self.queue = asyncio.Queue()
    
    async def start(self):
        for source in self.sources:
            asyncio.create_task(source.stream(self.queue))
    
    async def get_event(self):
        return await self.queue.get()
```

### Event Processing

```python
class EventProcessor:
    """Process captured events into VCP format."""
    
    def __init__(self, vcp_client):
        self.client = vcp_client
    
    async def process(self, raw_event):
        # Transform to VCP format
        vcp_event = self.transform(raw_event)
        
        # Log to VCP
        logged = self.client.log(
            vcp_event["type"],
            vcp_event["payload"]
        )
        
        return logged
    
    def transform(self, raw_event) -> dict:
        """Transform platform-specific event to VCP format."""
        # Implementation varies by source
        pass
```

### Batch Management

```python
class BatchManager:
    """Manage event batches and anchoring."""
    
    def __init__(self, anchor_interval_seconds: int):
        self.interval = anchor_interval_seconds
        self.tree = MerkleTree()
        self.events = []
        self.last_anchor = None
    
    def add_event(self, event: dict):
        self.tree.add(event["security"]["event_hash"])
        self.events.append(event)
        
        # Check if anchor due
        if self._should_anchor():
            self.anchor()
    
    def _should_anchor(self) -> bool:
        if not self.last_anchor:
            return False
        elapsed = (datetime.now() - self.last_anchor).total_seconds()
        return elapsed >= self.interval
    
    def anchor(self):
        root = self.tree.root()
        # Anchor to external store
        # Reset for next batch
        self.tree = MerkleTree()
        self.events = []
        self.last_anchor = datetime.now()
```

---

## Failure Handling

### Sidecar Failure

```python
class FailSafeWrapper:
    """Ensure trading continues if VCP fails."""
    
    def __init__(self, vcp_client):
        self.client = vcp_client
        self.fallback_queue = []
    
    def log(self, event_type, payload):
        try:
            return self.client.log(event_type, payload)
        except Exception as e:
            # Queue for later
            self.fallback_queue.append({
                "type": event_type,
                "payload": payload,
                "queued_at": datetime.now()
            })
            logger.error(f"VCP log failed: {e}")
            return None  # Don't block trading
    
    def replay_queue(self):
        """Replay queued events when VCP recovers."""
        while self.fallback_queue:
            event = self.fallback_queue.pop(0)
            try:
                self.client.log(event["type"], event["payload"])
            except:
                self.fallback_queue.insert(0, event)
                break
```

### Network Failure

```python
class RetryingClient:
    """Client with automatic retry."""
    
    def __init__(self, base_client, max_retries=3):
        self.client = base_client
        self.max_retries = max_retries
    
    def log(self, event_type, payload):
        for attempt in range(self.max_retries):
            try:
                return self.client.log(event_type, payload)
            except NetworkError:
                if attempt < self.max_retries - 1:
                    time.sleep(2 ** attempt)  # Exponential backoff
                else:
                    raise
```

---

## Monitoring

### Health Checks

```python
@app.get("/health")
def health_check():
    return {
        "status": "healthy",
        "vcp_version": "1.1",
        "events_logged": metrics.events_logged,
        "last_anchor": metrics.last_anchor,
        "queue_size": len(fallback_queue),
        "uptime_seconds": metrics.uptime
    }
```

### Metrics

| Metric | Description |
|--------|-------------|
| `vcp_events_total` | Total events logged |
| `vcp_events_failed` | Failed log attempts |
| `vcp_anchor_total` | Total anchors performed |
| `vcp_anchor_latency` | Anchor operation latency |
| `vcp_queue_size` | Pending events in fallback queue |

---

## Deployment Checklist

```
[ ] Event capture point identified
[ ] Network/API access confirmed
[ ] Storage provisioned (local + anchor)
[ ] Clock synchronization configured
[ ] Failover procedures documented
[ ] Performance impact measured (<1% latency)
[ ] Key management established
[ ] Monitoring and alerting configured
[ ] Recovery procedures tested
```

---

## See Also

- [mt5.md](./mt5.md) — MT4/MT5 specific integration
- [generic.md](./generic.md) — Generic integration patterns

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
