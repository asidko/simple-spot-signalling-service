# Spot Trading Signaling Service

## Concept

A service that monitors cryptocurrency price zones and emits trading signals. For better modularity, it only focuses on signal generation and not responsible for trade execution.

## Zone Structure

Divides price range into strategic zones:
- `stop_high_plank`: Emergency upper boundary that triggers full stop when breached, protecting from extreme upward volatility
- `sell_zone_top` to `sell_zone_bottom`: Upper profit-taking region where SELL signals are generated when price enters
- Middle buffer zone (0.4% by default): Neutral zone accounting for transaction fees where no signals are generated
- `buy_zone_top` to `buy_zone_bottom`: Lower accumulation region where BUY signals are generated when price enters  
- `stop_low_plank`: Emergency lower boundary that triggers full stop when breached, limiting downside risk

![zone-structure](https://github.com/user-attachments/assets/fe64a599-c645-4deb-b92f-527f8e1a0000)

**Calculation:** Find min/max prices over specified period (e.g 10 days). Place buffer zone around midpoint. Plase stop boundaries beyound min/max by configurable percentage.

## Configuration Input

```json
{
  "symbol": "BTC/USDT",
  "timeframe": 60,             // In minutes (60 = 1 hour candles)
  "historyMinutes": 14400,     // In minutes (14400 = 10 days)
  "autoRecalculate": true,     // Toggle for automatic recalculation
  "recalculateTimeframe": 120  // Recalculate boundaries each 2 hours
  "bufferPercentage": 0.004,   // 0.4% for middle buffer (to cover fees)
  "stopPlankPercentage": 0.05, // Price goes 5% beyond min/max - we must stop
  "checkFrequency": 60         // Check for signal every 1 hour
}
```

## Bot Creation Output

```json
{
  "botId": "trading-bot-btc-1",
  "status": "initialized",
  "createdAt": "2025-03-23T14:00:00Z",
  "zones": {
    "stopHighPlank": 92565.51,
    "sellZoneTop": 90750.50,
    "sellZoneBottom": 87431.25,
    "buyZoneTop": 87068.75,
    "buyZoneBottom": 83750.50,
    "stopLowPlank": 82075.49,
    "calculatedAt": "2025-03-23T14:00:00Z"
  }
}
```

## Event Types

### Signal Event
```json
{
  "eventType": "signal",
  "timestamp": "2025-03-23T14:30:00Z",
  "botId": "trading-bot-btc-1",
  "symbol": "BTC/USDT",
  "price": 87250.45,
  "signal": "BUY"
}
```

### Zone Recalculation Event
```json
{
  "eventType": "zoneRecalculation",
  "timestamp": "2025-03-24T00:00:00Z",
  "botId": "trading-bot-btc-1",
  "symbol": "BTC/USDT",
  "zone": {
    "stopHighPlank": 93125.22,
    "sellZoneTop": 91300.25,
    "sellZoneBottom": 87937.50,
    "buyZoneTop": 87562.50,
    "buyZoneBottom": 84200.75,
    "stopLowPlank": 82516.74
  },
  "reason": "auto_recalculation"
}
```

### Zone Crossing Event
```json
{
  "eventType": "zoneCrossing",
  "timestamp": "2025-03-23T18:15:00Z",
  "botId": "trading-bot-btc-1",
  "symbol": "BTC/USDT",
  "price": 87500.25,
  "previousZone": "buy_zone",
  "currentZone": "buffer_zone",
  "direction": "upward"
}
```

### Status Change Event
```json
{
  "eventType": "statusChange",
  "timestamp": "2025-03-23T22:45:00Z",
  "botId": "trading-bot-btc-1",
  "symbol": "BTC/USDT",
  "previousStatus": "active",
  "newStatus": "stopped",
  "reason": "stop_level_reached",
  "price": 82000.15
}
```

### Error Event
```json
{
  "eventType": "error",
  "timestamp": "2025-03-23T16:20:00Z",
  "botId": "trading-bot-btc-1",
  "symbol": "BTC/USDT",
  "errorCode": "DATA_FETCH_FAILURE",
  "errorMessage": "Failed to retrieve price data from exchange API"
}
```

## Workflow

1. **Initialization**: Load config, fetch data, calculate zones
2. **Monitoring**: Check price at intervals, compare to zones, emit signals
3. **Recalculation** (if enabled): Daily update using rolling time window
4. **Signals**:
   - **BUY**: Price in buy zone
   - **SELL**: Price in sell zone
   - **PAUSE**: Price in buffer zone
   - **STOP**: Price beyond stop boundaries (requires manual restart)

## Future improvements

- Smart zone boundaries recalculation to ignore manipulations