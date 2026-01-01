# VCP v1.1 MT4/MT5 Integration

**Document ID:** VSO-IMPL-001-MT5  
**Parent:** [Sidecar Overview](./overview.md)

---

## Overview

This guide covers integrating VCP into MetaTrader 4/5 Expert Advisors (EAs) using the sidecar pattern. The integration is **non-invasive** — your existing EA logic remains unchanged.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      MetaTrader 5 Terminal                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────┐     ┌──────────────────────┐            │
│  │    Your EA           │     │    VCP Logger        │            │
│  │                      │────▶│    (Include file)    │            │
│  │  [Trading Logic]     │     │                      │            │
│  │  [Risk Management]   │     │  [Event Formatting]  │            │
│  │  [Order Execution]   │     │  [HTTP POST]         │            │
│  └──────────────────────┘     └──────────┬───────────┘            │
│                                          │                         │
└──────────────────────────────────────────┼─────────────────────────┘
                                           │
                                           ▼
                               ┌──────────────────────┐
                               │    VCP Sidecar       │
                               │    (localhost:8080)  │
                               │                      │
                               │  [Hash] [Merkle]     │
                               │  [Sign] [Anchor]     │
                               └──────────────────────┘
```

---

## Setup

### 1. Enable WebRequest

In MT5 terminal:
1. **Tools → Options → Expert Advisors**
2. Check "Allow WebRequest for listed URL"
3. Add: `http://localhost:8080`

### 2. Include VCP Logger

Copy `VCPLogger.mqh` to your `MQL5/Include/` directory.

---

## VCP Logger Include File

```mql5
//+------------------------------------------------------------------+
//| VCPLogger.mqh - VCP v1.1 Logger for MT5                          |
//+------------------------------------------------------------------+
#property copyright "VeritasChain Standards Organization"
#property version   "1.10"

#include <JAson.mqh>

//--- Input parameters (set in your EA)
input string VCP_PolicyID = "com.example:silver-mt5";
input string VCP_SidecarURL = "http://localhost:8080/vcp/log";
input int    VCP_Timeout = 5000;

//+------------------------------------------------------------------+
//| VCP Logger Class                                                  |
//+------------------------------------------------------------------+
class CVCPLogger
{
private:
    string m_policyId;
    string m_sidecarUrl;
    int    m_timeout;
    
public:
    //--- Constructor
    CVCPLogger()
    {
        m_policyId = VCP_PolicyID;
        m_sidecarUrl = VCP_SidecarURL;
        m_timeout = VCP_Timeout;
    }
    
    //--- Initialize with custom settings
    void Init(string policyId, string sidecarUrl, int timeout = 5000)
    {
        m_policyId = policyId;
        m_sidecarUrl = sidecarUrl;
        m_timeout = timeout;
    }
    
    //--- Log Order Event
    bool LogOrder(string orderId, string symbol, string side, 
                  string orderType, double price, double qty, 
                  string traceId = "")
    {
        CJAVal event;
        BuildHeader(event, "ORD", traceId);
        
        event["payload"]["trade_data"]["order_id"] = orderId;
        event["payload"]["trade_data"]["symbol"] = symbol;
        event["payload"]["trade_data"]["side"] = side;
        event["payload"]["trade_data"]["order_type"] = orderType;
        event["payload"]["trade_data"]["price"] = DoubleToString(price, 5);
        event["payload"]["trade_data"]["quantity"] = DoubleToString(qty, 2);
        
        return SendEvent(event);
    }
    
    //--- Log Execution Event
    bool LogExecution(string orderId, string symbol, string side,
                      double execPrice, double execQty, 
                      double commission = 0, double slippage = 0,
                      string traceId = "")
    {
        CJAVal event;
        BuildHeader(event, "EXE", traceId);
        
        event["payload"]["trade_data"]["order_id"] = orderId;
        event["payload"]["trade_data"]["symbol"] = symbol;
        event["payload"]["trade_data"]["side"] = side;
        event["payload"]["trade_data"]["execution_price"] = DoubleToString(execPrice, 5);
        event["payload"]["trade_data"]["executed_qty"] = DoubleToString(execQty, 2);
        
        if(commission != 0)
            event["payload"]["trade_data"]["commission"] = DoubleToString(commission, 2);
        if(slippage != 0)
            event["payload"]["trade_data"]["slippage"] = DoubleToString(slippage, 5);
        
        return SendEvent(event);
    }
    
    //--- Log Signal Event
    bool LogSignal(string algoId, string symbol, string signalType,
                   double confidence, string reason, string traceId = "")
    {
        CJAVal event;
        BuildHeader(event, "SIG", traceId);
        
        event["payload"]["vcp_gov"]["algo_id"] = algoId;
        event["payload"]["vcp_gov"]["signal_type"] = signalType;
        event["payload"]["vcp_gov"]["confidence"] = DoubleToString(confidence, 4);
        event["payload"]["vcp_gov"]["reason"] = reason;
        event["payload"]["trade_data"]["symbol"] = symbol;
        
        return SendEvent(event);
    }
    
    //--- Log Risk Event
    bool LogRisk(string metricName, double currentValue, double limitValue,
                 string action, string traceId = "")
    {
        CJAVal event;
        BuildHeader(event, "RSK", traceId);
        
        event["payload"]["vcp_risk"]["metric_name"] = metricName;
        event["payload"]["vcp_risk"]["current_value"] = DoubleToString(currentValue, 4);
        event["payload"]["vcp_risk"]["limit_value"] = DoubleToString(limitValue, 4);
        event["payload"]["vcp_risk"]["action"] = action;
        
        return SendEvent(event);
    }
    
    //--- Log Error Event
    bool LogError(string errorType, string errorCode, string errorMsg,
                  string severity, string component, string correlatedId = "")
    {
        CJAVal event;
        BuildHeader(event, errorType, "");
        
        event["payload"]["error_details"]["error_code"] = errorCode;
        event["payload"]["error_details"]["error_message"] = errorMsg;
        event["payload"]["error_details"]["severity"] = severity;
        event["payload"]["error_details"]["affected_component"] = component;
        
        if(correlatedId != "")
            event["payload"]["error_details"]["correlated_event_id"] = correlatedId;
        
        return SendEvent(event);
    }
    
    //--- Log Custom Event
    bool LogCustom(string eventType, CJAVal &payload, string traceId = "")
    {
        CJAVal event;
        BuildHeader(event, eventType, traceId);
        event["payload"] = payload;
        return SendEvent(event);
    }

private:
    //--- Build event header
    void BuildHeader(CJAVal &event, string eventType, string traceId)
    {
        // Generate event ID (simplified UUID)
        string eventId = GenerateEventId();
        
        // Timestamps
        datetime now = TimeCurrent();
        long nowMs = (long)now * 1000;
        
        event["header"]["event_id"] = eventId;
        event["header"]["event_type"] = eventType;
        event["header"]["timestamp_iso"] = TimeToString(now, TIME_DATE|TIME_SECONDS);
        event["header"]["timestamp_int"] = IntegerToString(nowMs);
        event["header"]["timestamp_precision"] = "MILLISECOND";
        event["header"]["clock_sync_status"] = "BEST_EFFORT";
        event["header"]["vcp_version"] = "1.1";
        event["header"]["hash_algo"] = "SHA256";
        
        if(traceId != "")
            event["header"]["trace_id"] = traceId;
        
        // Policy Identification (REQUIRED in v1.1)
        event["policy_identification"]["version"] = "1.1";
        event["policy_identification"]["policy_id"] = m_policyId;
        event["policy_identification"]["conformance_tier"] = "SILVER";
        event["policy_identification"]["verification_depth"]["hash_chain_validation"] = false;
        event["policy_identification"]["verification_depth"]["merkle_proof_required"] = true;
        event["policy_identification"]["verification_depth"]["external_anchor_required"] = true;
    }
    
    //--- Generate simple event ID
    string GenerateEventId()
    {
        return StringFormat("%08x-%04x-%04x-%04x-%012x",
            (uint)TimeCurrent(),
            MathRand() % 0xFFFF,
            0x7000 | (MathRand() % 0x0FFF),
            0x8000 | (MathRand() % 0x3FFF),
            MathRand());
    }
    
    //--- Send event to sidecar
    bool SendEvent(CJAVal &event)
    {
        string jsonData = event.Serialize();
        char data[], result[];
        string headers = "Content-Type: application/json\r\n";
        string resultHeaders;
        
        StringToCharArray(jsonData, data, 0, StringLen(jsonData));
        
        ResetLastError();
        int res = WebRequest(
            "POST",
            m_sidecarUrl,
            headers,
            m_timeout,
            data,
            result,
            resultHeaders
        );
        
        if(res == -1)
        {
            int error = GetLastError();
            Print("VCP Error: WebRequest failed, error=", error);
            return false;
        }
        
        if(res != 200 && res != 201)
        {
            Print("VCP Error: Sidecar returned ", res);
            return false;
        }
        
        return true;
    }
};

//--- Global instance
CVCPLogger g_VCP;
```

---

## Usage in Your EA

```mql5
//+------------------------------------------------------------------+
//| MyTradingEA.mq5                                                   |
//+------------------------------------------------------------------+
#property copyright "Your Company"
#property version   "1.00"

#include <VCPLogger.mqh>

// Your EA inputs
input double LotSize = 0.1;
input int    StopLoss = 50;

//+------------------------------------------------------------------+
//| Expert initialization function                                    |
//+------------------------------------------------------------------+
int OnInit()
{
    // Initialize VCP logger
    g_VCP.Init(
        "com.yourcompany:my-ea-v1",  // Your PolicyID
        "http://localhost:8080/vcp/log"
    );
    
    // Log EA startup
    g_VCP.LogError("INIT", "EA_START", "Expert Advisor initialized", "INFO", "my-ea");
    
    return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                  |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
    g_VCP.LogError("INIT", "EA_STOP", "Expert Advisor stopped, reason=" + IntegerToString(reason), "INFO", "my-ea");
}

//+------------------------------------------------------------------+
//| Expert tick function                                              |
//+------------------------------------------------------------------+
void OnTick()
{
    // Your trading logic here...
    
    // Example: Log signal when your strategy triggers
    if(ShouldBuy())
    {
        string traceId = GenerateTraceId();
        
        // Log signal
        g_VCP.LogSignal(
            "my-strategy-v1",
            _Symbol,
            "BUY",
            0.85,  // confidence
            "MA crossover detected",
            traceId
        );
        
        // Execute trade
        MqlTradeRequest request = {};
        MqlTradeResult result = {};
        
        request.action = TRADE_ACTION_DEAL;
        request.symbol = _Symbol;
        request.volume = LotSize;
        request.type = ORDER_TYPE_BUY;
        request.price = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
        
        // Log order
        g_VCP.LogOrder(
            "pending",
            _Symbol,
            "BUY",
            "MARKET",
            request.price,
            request.volume,
            traceId
        );
        
        if(OrderSend(request, result))
        {
            // Log execution
            g_VCP.LogExecution(
                IntegerToString(result.order),
                _Symbol,
                "BUY",
                result.price,
                result.volume,
                0,  // commission (fill in if known)
                result.price - request.price,  // slippage
                traceId
            );
        }
        else
        {
            // Log rejection
            g_VCP.LogError(
                "ERR_REJECT",
                "ORDER_FAILED",
                "OrderSend failed: " + IntegerToString(result.retcode),
                "WARNING",
                "order-manager",
                traceId
            );
        }
    }
}

//+------------------------------------------------------------------+
//| Trade transaction handler                                         |
//+------------------------------------------------------------------+
void OnTradeTransaction(const MqlTradeTransaction& trans,
                        const MqlTradeRequest& request,
                        const MqlTradeResult& result)
{
    // Automatically log trade events
    if(trans.type == TRADE_TRANSACTION_ORDER_ADD)
    {
        g_VCP.LogOrder(
            IntegerToString(trans.order),
            trans.symbol,
            (trans.order_type == ORDER_TYPE_BUY) ? "BUY" : "SELL",
            "MARKET",
            trans.price,
            trans.volume
        );
    }
    else if(trans.type == TRADE_TRANSACTION_DEAL_ADD)
    {
        g_VCP.LogExecution(
            IntegerToString(trans.order),
            trans.symbol,
            (trans.deal_type == DEAL_TYPE_BUY) ? "BUY" : "SELL",
            trans.price,
            trans.volume
        );
    }
}

//+------------------------------------------------------------------+
//| Helper functions                                                  |
//+------------------------------------------------------------------+
bool ShouldBuy()
{
    // Your buy signal logic
    return false;
}

string GenerateTraceId()
{
    return StringFormat("%08x-%04x", (uint)TimeCurrent(), MathRand() % 0xFFFF);
}
```

---

## VCP Sidecar (Python)

Run this sidecar alongside MT5:

```python
"""
VCP Sidecar for MT5 - Receives events and handles VCP processing
"""
from flask import Flask, request, jsonify
from datetime import datetime, timezone
import hashlib
import json

app = Flask(__name__)

# In-memory storage (use proper database in production)
events = []
merkle_tree = MerkleTree()

@app.route('/vcp/log', methods=['POST'])
def log_event():
    """Receive VCP event from MT5."""
    event = request.json
    
    # Compute event hash
    hashable = {"header": event["header"], "payload": event["payload"]}
    canonical = json.dumps(hashable, sort_keys=True, separators=(',', ':'))
    event_hash = hashlib.sha256(canonical.encode()).hexdigest()
    
    # Add security section
    event["security"] = {"event_hash": event_hash}
    
    # Add to Merkle tree
    merkle_tree.add(event_hash)
    events.append(event)
    
    return jsonify({"status": "ok", "event_id": event["header"]["event_id"]}), 201

@app.route('/vcp/anchor', methods=['POST'])
def anchor():
    """Trigger daily anchoring."""
    root = merkle_tree.root()
    
    # Anchor to OpenTimestamps
    import opentimestamps
    timestamp = opentimestamps.stamp(bytes.fromhex(root))
    
    result = {
        "merkle_root": root,
        "event_count": len(events),
        "anchored_at": datetime.now(timezone.utc).isoformat()
    }
    
    # Reset for next batch
    merkle_tree.reset()
    events.clear()
    
    return jsonify(result)

@app.route('/health', methods=['GET'])
def health():
    return jsonify({
        "status": "healthy",
        "events_pending": len(events),
        "merkle_root": merkle_tree.root()
    })

if __name__ == '__main__':
    app.run(host='localhost', port=8080)
```

---

## Troubleshooting

### WebRequest Fails (Error 4014)

**Cause:** URL not in allowed list.

**Solution:** Add `http://localhost:8080` to Expert Advisors settings.

### Sidecar Not Responding

**Cause:** Sidecar not running or wrong port.

**Solution:**
1. Check sidecar is running: `curl http://localhost:8080/health`
2. Verify port matches EA configuration

### Events Not Logging

**Cause:** JSON serialization issue.

**Solution:**
1. Check MT5 Experts log for errors
2. Verify JAson.mqh is properly included

---

## See Also

- [overview.md](./overview.md) — Sidecar architecture overview
- [generic.md](./generic.md) — Generic integration patterns

---

*Copyright © 2025 VeritasChain Standards Organization. CC BY 4.0 License.*
