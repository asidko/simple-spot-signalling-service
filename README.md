# Spot Trading Signaling Service

## Concept

A service that monitors cryptocurrency price zones and emits trading signals. For better modularity, it only focuses on signal generation and not responsible for trade execution.

## Zone Structure

Divides price range into strategic zones:
- `STOP_HIGH_PRICE`: Emergency upper boundary that triggers full stop when breached, protecting from extreme upward volatility
- `SELL_TOP_PRICE` to `SELL_BOTTOM_PRICE`: Upper profit-taking region where SELL signals are generated when price enters
- `BUFFER_ZONE`: Middle neutral zone (0.4% by default) accounting for transaction fees where no signals are generated
- `BUY_TOP_PRICE` to `BUY_BOTTOM_PRICE`: Lower accumulation region where BUY signals are generated when price enters  
- `STOP_LOW_PRICE`: Emergency lower boundary that triggers full stop when breached, limiting downside risk

![zone-structure](https://github.com/user-attachments/assets/fe64a599-c645-4deb-b92f-527f8e1a0000)

**Calculation:** Find min/max prices over specified period (e.g 10 days). Place `BUFFER_ZONE` around midpoint. Place stop boundaries beyond min/max by configurable percentage.

## Configuration Input

```json
{
  "userId": "user123",          // User identifier
  "symbol": "BTC/USDT",
  "timeframe": 60,              // In minutes (60 = 1 hour candles)
  "historyMinutes": 14400,      // In minutes (14400 = 10 days)
  "bufferPercentage": 0.004,    // 0.4% for middle buffer (to cover fees)
  "stopPlankPercentage": 0.05,  // Price goes 5% beyond min/max - we must stop
  "checkFrequency": 60          // Check for signal every 1 hour
}
```

## Bot Creation Output

```json
{
  "userId": "user123",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", // UUID for this instance
  "status": "INITIALIZED",
  "createdAt": "2025-03-23T14:00:00Z",
  "zoneConfig": {
    "stopHighPrice": 92565.51,
    "sellTopPrice": 90750.50,
    "sellBottomPrice": 87431.25,
    "buyTopPrice": 87068.75,
    "buyBottomPrice": 83750.50,
    "stopLowPrice": 82075.49,
    "calculatedAt": "2025-03-23T14:00:00Z",
    "zoneStartsAt": "2025-03-13T14:00:00Z", // Start of analysis period
    "zoneEndsAt": "2025-03-23T14:00:00Z"    // End of analysis period
  }
}
```

## Event Types

### Signal Event
```json
{
  "eventType": "SIGNAL",
  "timestamp": "2025-03-23T14:30:00Z",
  "userId": "user123",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "symbol": "BTC/USDT",
  "price": 87250.45,
  "signal": "BUY"
}
```

### Zone Recalculation Event
```json
{
  "eventType": "ZONE_RECALCULATION",
  "timestamp": "2025-03-24T00:00:00Z",
  "userId": "user123",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "symbol": "BTC/USDT",
  "zoneConfig": {
    "stopHighPrice": 93125.22,
    "sellTopPrice": 91300.25,
    "sellBottomPrice": 87937.50,
    "buyTopPrice": 87562.50,
    "buyBottomPrice": 84200.75,
    "stopLowPrice": 82516.74,
    "zoneStartsAt": "2025-03-14T00:00:00Z",
    "zoneEndsAt": "2025-03-24T00:00:00Z"
  },
  "reason": "ZONE_CHANGE_PROPOSAL"
}
```

### Zone Crossing Event
```json
{
  "eventType": "ZONE_CROSSING",
  "timestamp": "2025-03-23T18:15:00Z",
  "userId": "user123",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "symbol": "BTC/USDT",
  "price": 87500.25,
  "previousZone": "BUY_ZONE",
  "currentZone": "BUFFER_ZONE",
  "direction": "UPWARD"
}
```

### Status Change Event
```json
{
  "eventType": "STATUS_CHANGE",
  "timestamp": "2025-03-23T22:45:00Z",
  "userId": "user123",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "symbol": "BTC/USDT",
  "previousStatus": "ACTIVE",
  "newStatus": "STOPPED",
  "reason": "STOP_LEVEL_REACHED",
  "price": 82000.15
}
```

### Error Event
```json
{
  "eventType": "ERROR",
  "timestamp": "2025-03-23T16:20:00Z",
  "userId": "user123",
  "instanceId": "550e8400-e29b-41d4-a716-446655440000",
  "symbol": "BTC/USDT",
  "errorCode": "DATA_FETCH_FAILURE",
  "errorMessage": "Failed to retrieve price data from exchange API"
}
```

## Workflow

1. **Initialization**: Load config, fetch data, calculate zones
2. **Monitoring**: Check price at intervals, compare to zones, emit signals
3. **Manual Recalculation**: User explicitly requests zone recalculation when appropriate
4. **Signals**:
   - **BUY**: Price in BUY_ZONE
   - **SELL**: Price in SELL_ZONE
   - **PAUSE**: Price in BUFFER_ZONE or out of buy/sell boundaries
   - **STOP**: Price beyond stop boundaries (requires manual restart)

## Future improvements

- Smart zone boundaries recalculation to ignore manipulations

## Additional nodes

When writing this doc we want to be consice and specific but not at the expense of illustration