# VCP v1.1 Generic Integration Patterns

**Document ID:** VSO-IMPL-001-GEN  
**Parent:** [Sidecar Overview](./overview.md)

---

## Overview

This guide covers generic integration patterns for systems beyond MT4/MT5, including REST APIs, FIX protocol, and message queues.

---

## Pattern 1: REST API Integration

### Architecture

```
[Your Application]
        │
        ▼
[VCP Client Library]
        │
        ▼ HTTP POST
[VCP Sidecar API]
```

### Python Client

```python
"""
VCP v1.1 Python Client - Generic Integration
"""
import hashlib
import json
import uuid
import requests
from datetime import datetime, timezone
from typing import Optional, Dict, Any

class VCPClient:
    """Generic VCP v1.1 client."""
    
    def __init__(
        self,
        policy_id: str,
        sidecar_url: str = "http://localhost:8080",
        tier: str = "SILVER"
    ):
        self.policy_id = policy_id
        self.sidecar_url = sidecar_url
        self.tier = tier
        self.session = requests.Session()
    
    def log(
        self,
        event_type: str,
        payload: Dict[str, Any],
        trace_id: Optional[str] = None
    ) -> Dict[str, Any]:
        """Log a VCP event."""
        now = datetime.now(timezone.utc)
        
        event = {
            "header": {
                "event_id": str(uuid.uuid4()),
                "event_type": event_type,
                "timestamp_iso": now.isoformat(),
                "timestamp_int": str(int(now.timestamp() * 1000)),
                "timestamp_precision": "MILLISECOND",
                "clock_sync_status": "BEST_EFFORT",
                "vcp_version": "1.1",
                "hash_algo": "SHA256"
            },
            "payload": payload,
            "policy_identification": {
                "version": "1.1",
                "policy_id": self.policy_id,
                "conformance_tier": self.tier,
                "verification_depth": {
                    "hash_chain_validation": False,
                    "merkle_proof_required": True,
                    "external_anchor_required": True
                }
            }
        }
        
        if trace_id:
            event["header"]["trace_id"] = trace_id
        
        # Send to sidecar
        response = self.session.post(
            f"{self.sidecar_url}/vcp/log",
            json=event,
            timeout=5
        )
        response.raise_for_status()
        
        return event
    
    def log_order(
        self,
        order_id: str,
        symbol: str,
        side: str,
        order_type: str,
        price: float,
        quantity: float,
        trace_id: Optional[str] = None
    ) -> Dict[str, Any]:
        """Log order event."""
        return self.log("ORD", {
            "trade_data": {
                "order_id": order_id,
                "symbol": symbol,
                "side": side,
                "order_type": order_type,
                "price": str(price),
                "quantity": str(quantity)
            }
        }, trace_id)
    
    def log_execution(
        self,
        order_id: str,
        symbol: str,
        side: str,
        exec_price: float,
        exec_qty: float,
        commission: float = 0,
        slippage: float = 0,
        trace_id: Optional[str] = None
    ) -> Dict[str, Any]:
        """Log execution event."""
        payload = {
            "trade_data": {
                "order_id": order_id,
                "symbol": symbol,
                "side": side,
                "execution_price": str(exec_price),
                "executed_qty": str(exec_qty)
            }
        }
        if commission:
            payload["trade_data"]["commission"] = str(commission)
        if slippage:
            payload["trade_data"]["slippage"] = str(slippage)
        
        return self.log("EXE", payload, trace_id)
    
    def log_error(
        self,
        error_type: str,
        error_code: str,
        error_message: str,
        severity: str,
        component: str,
        correlated_id: Optional[str] = None
    ) -> Dict[str, Any]:
        """Log error event."""
        payload = {
            "error_details": {
                "error_code": error_code,
                "error_message": error_message,
                "severity": severity,
                "affected_component": component
            }
        }
        if correlated_id:
            payload["error_details"]["correlated_event_id"] = correlated_id
        
        return self.log(error_type, payload)
```

### Usage

```python
# Initialize client
vcp = VCPClient(
    policy_id="com.mycompany:trading-system-v1",
    sidecar_url="http://localhost:8080"
)

# Log order
trace_id = str(uuid.uuid4())
vcp.log_order(
    order_id="ORD-001",
    symbol="EURUSD",
    side="BUY",
    order_type="LIMIT",
    price=1.0850,
    quantity=100000,
    trace_id=trace_id
)

# Log execution
vcp.log_execution(
    order_id="ORD-001",
    symbol="EURUSD",
    side="BUY",
    exec_price=1.0851,
    exec_qty=100000,
    slippage=0.0001,
    trace_id=trace_id
)
```

---

## Pattern 2: FIX Protocol Integration

### Architecture

```
[FIX Engine] ──────────────────────▶ [Broker]
      │
      │ (message listener)
      ▼
[VCP FIX Adapter]
      │
      ▼
[VCP Sidecar]
```

### FIX Message Adapter

```python
"""
VCP FIX Protocol Adapter
Maps FIX messages to VCP events
"""
from quickfix import Message

class VCPFIXAdapter:
    """Adapter for FIX messages to VCP events."""
    
    # FIX field tags
    TAG_CLORDID = 11
    TAG_ORDERID = 37
    TAG_SYMBOL = 55
    TAG_SIDE = 54
    TAG_ORDTYPE = 40
    TAG_PRICE = 44
    TAG_ORDERQTY = 38
    TAG_AVGPX = 6
    TAG_CUMQTY = 14
    TAG_EXECTYPE = 150
    TAG_ORDSTATUS = 39
    
    # Side mapping
    SIDE_MAP = {"1": "BUY", "2": "SELL"}
    
    # Order type mapping
    ORDTYPE_MAP = {"1": "MARKET", "2": "LIMIT", "3": "STOP", "4": "STOP_LIMIT"}
    
    def __init__(self, vcp_client):
        self.vcp = vcp_client
    
    def process_message(self, message: Message):
        """Process FIX message and log to VCP."""
        msg_type = message.getHeader().getField(35)
        
        if msg_type == "D":  # NewOrderSingle
            self._process_new_order(message)
        elif msg_type == "8":  # ExecutionReport
            self._process_execution_report(message)
        elif msg_type == "9":  # OrderCancelReject
            self._process_cancel_reject(message)
    
    def _process_new_order(self, message: Message):
        """Process NewOrderSingle (35=D)."""
        self.vcp.log_order(
            order_id=message.getField(self.TAG_CLORDID),
            symbol=message.getField(self.TAG_SYMBOL),
            side=self.SIDE_MAP.get(message.getField(self.TAG_SIDE), "UNKNOWN"),
            order_type=self.ORDTYPE_MAP.get(message.getField(self.TAG_ORDTYPE), "UNKNOWN"),
            price=float(message.getField(self.TAG_PRICE)) if message.isSetField(self.TAG_PRICE) else 0,
            quantity=float(message.getField(self.TAG_ORDERQTY))
        )
    
    def _process_execution_report(self, message: Message):
        """Process ExecutionReport (35=8)."""
        exec_type = message.getField(self.TAG_EXECTYPE)
        
        if exec_type in ["1", "2", "F"]:  # Partial/Full fill
            self.vcp.log_execution(
                order_id=message.getField(self.TAG_CLORDID),
                symbol=message.getField(self.TAG_SYMBOL),
                side=self.SIDE_MAP.get(message.getField(self.TAG_SIDE), "UNKNOWN"),
                exec_price=float(message.getField(self.TAG_AVGPX)),
                exec_qty=float(message.getField(self.TAG_CUMQTY))
            )
        elif exec_type == "8":  # Rejected
            self.vcp.log_error(
                "ERR_REJECT",
                "ORDER_REJECTED",
                message.getField(58) if message.isSetField(58) else "Unknown reason",
                "WARNING",
                "fix-gateway"
            )
```

---

## Pattern 3: Message Queue Integration

### Kafka Consumer

```python
"""
VCP Kafka Consumer - Tap into existing event streams
"""
from kafka import KafkaConsumer
import json

class VCPKafkaConsumer:
    """Consume trading events from Kafka and log to VCP."""
    
    def __init__(self, vcp_client, bootstrap_servers: list, topics: list):
        self.vcp = vcp_client
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            group_id='vcp-sidecar',
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
    
    def start(self):
        """Start consuming messages."""
        for message in self.consumer:
            try:
                self._process_message(message.topic, message.value)
            except Exception as e:
                print(f"Error processing message: {e}")
    
    def _process_message(self, topic: str, data: dict):
        """Process Kafka message based on topic."""
        if topic == "orders":
            self.vcp.log_order(
                order_id=data["order_id"],
                symbol=data["symbol"],
                side=data["side"],
                order_type=data["type"],
                price=data["price"],
                quantity=data["quantity"]
            )
        elif topic == "executions":
            self.vcp.log_execution(
                order_id=data["order_id"],
                symbol=data["symbol"],
                side=data["side"],
                exec_price=data["price"],
                exec_qty=data["quantity"]
            )
        elif topic == "errors":
            self.vcp.log_error(
                data.get("type", "ERR_SYSTEM"),
                data["code"],
                data["message"],
                data.get("severity", "WARNING"),
                data.get("component", "unknown")
            )

# Usage
consumer = VCPKafkaConsumer(
    vcp_client=vcp,
    bootstrap_servers=['kafka:9092'],
    topics=['orders', 'executions', 'errors']
)
consumer.start()
```

### Redis Stream Consumer

```python
"""
VCP Redis Stream Consumer
"""
import redis

class VCPRedisConsumer:
    """Consume trading events from Redis Streams."""
    
    def __init__(self, vcp_client, redis_url: str, streams: list):
        self.vcp = vcp_client
        self.redis = redis.from_url(redis_url)
        self.streams = {s: '0' for s in streams}
    
    def start(self):
        """Start consuming from streams."""
        while True:
            # Read from all streams
            results = self.redis.xread(self.streams, block=5000)
            
            for stream, messages in results:
                for msg_id, data in messages:
                    self._process_message(stream.decode(), data)
                    self.streams[stream.decode()] = msg_id.decode()
    
    def _process_message(self, stream: str, data: dict):
        """Process Redis stream message."""
        # Decode bytes to strings
        data = {k.decode(): v.decode() for k, v in data.items()}
        
        event_type = data.get("event_type", "UNKNOWN")
        
        self.vcp.log(event_type, {
            "trade_data": {
                k: v for k, v in data.items() if k != "event_type"
            }
        })
```

---

## Pattern 4: Decorator-Based Integration

### Python Decorator

```python
"""
VCP Decorator - Automatic logging for trading functions
"""
import functools
from typing import Callable

def vcp_logged(event_type: str, extract_payload: Callable = None):
    """
    Decorator to automatically log VCP events.
    
    Usage:
        @vcp_logged("ORD", lambda args, result: {"order_id": args[0]})
        def place_order(order_id, symbol, side, price, qty):
            # Your order logic
            return result
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # Execute function
            result = func(*args, **kwargs)
            
            # Extract payload
            if extract_payload:
                payload = extract_payload(args, kwargs, result)
            else:
                payload = {"function": func.__name__, "result": str(result)}
            
            # Log to VCP
            global_vcp_client.log(event_type, payload)
            
            return result
        return wrapper
    return decorator

# Usage
@vcp_logged("ORD", lambda args, kwargs, result: {
    "trade_data": {
        "order_id": kwargs.get("order_id") or args[0],
        "symbol": kwargs.get("symbol") or args[1],
        "side": kwargs.get("side") or args[2]
    }
})
def place_order(order_id: str, symbol: str, side: str, price: float, qty: float):
    """Your actual order placement logic."""
    # ... implementation ...
    return {"status": "success", "order_id": order_id}
```

---

## Pattern 5: Async Integration

### asyncio Client

```python
"""
VCP Async Client for high-throughput applications
"""
import aiohttp
import asyncio
from typing import Optional, Dict, Any

class AsyncVCPClient:
    """Asynchronous VCP client."""
    
    def __init__(self, policy_id: str, sidecar_url: str):
        self.policy_id = policy_id
        self.sidecar_url = sidecar_url
        self.session: Optional[aiohttp.ClientSession] = None
    
    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        return self
    
    async def __aexit__(self, *args):
        if self.session:
            await self.session.close()
    
    async def log(self, event_type: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Log event asynchronously."""
        event = self._build_event(event_type, payload)
        
        async with self.session.post(
            f"{self.sidecar_url}/vcp/log",
            json=event
        ) as response:
            response.raise_for_status()
            return event
    
    def _build_event(self, event_type: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        # Same as sync client
        pass

# Usage
async def main():
    async with AsyncVCPClient("com.example:async-v1", "http://localhost:8080") as vcp:
        # Log multiple events concurrently
        tasks = [
            vcp.log("ORD", {"trade_data": {"order_id": f"ORD-{i}"}})
            for i in range(100)
        ]
        await asyncio.gather(*tasks)

asyncio.run(main())
```

---

## Sidecar Implementation

### Generic Sidecar (Python/Flask)

```python
"""
Generic VCP Sidecar Server
"""
from flask import Flask, request, jsonify
from datetime import datetime, timezone
import hashlib
import json
import threading
import time

app = Flask(__name__)

# State
events = []
merkle_tree = MerkleTree()
lock = threading.Lock()

@app.route('/vcp/log', methods=['POST'])
def log_event():
    """Receive and process VCP event."""
    event = request.json
    
    # Compute hash
    hashable = {"header": event["header"], "payload": event["payload"]}
    canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
    event_hash = hashlib.sha256(canonical.encode()).hexdigest()
    event["security"] = {"event_hash": event_hash}
    
    with lock:
        merkle_tree.add(event_hash)
        events.append(event)
    
    return jsonify({
        "status": "ok",
        "event_id": event["header"]["event_id"],
        "event_hash": event_hash
    }), 201

@app.route('/vcp/anchor', methods=['POST'])
def anchor():
    """Perform anchoring."""
    with lock:
        root = merkle_tree.root()
        count = len(events)
        
        # Anchor logic here...
        
        merkle_tree.reset()
        events.clear()
    
    return jsonify({
        "merkle_root": root,
        "event_count": count,
        "anchored_at": datetime.now(timezone.utc).isoformat()
    })

@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint."""
    return jsonify({
        "status": "healthy",
        "pending_events": len(events),
        "current_root": merkle_tree.root()
    })

# Background anchor scheduler
def anchor_scheduler(interval_seconds: int):
    while True:
        time.sleep(interval_seconds)
        with app.test_client() as client:
            client.post('/vcp/anchor')

if __name__ == '__main__':
    # Start daily anchoring (24 hours)
    scheduler = threading.Thread(
        target=anchor_scheduler,
        args=(86400,),
        daemon=True
    )
    scheduler.start()
    
    app.run(host='0.0.0.0', port=8080)
```

---

## See Also

- [overview.md](./overview.md) — Sidecar architecture overview
- [mt5.md](./mt5.md) — MT4/MT5 specific integration

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
