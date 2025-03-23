# Spot Trading Signaling Service

## Concept

The Spot Trading Signaling Service monitors cryptocurrency prices and generates BUY and SELL signals based on strategic price zones. It analyzes recent price history to establish buy and sell zones around a middle buffer zone, then emits actionable trading signals when prices enter these zones. The service focuses exclusively on signal generation, not trade execution, allowing for easy integration with various trading systems.

## Zone Structure

Divides price range into three strategic zones:
- `SELL_ZONE`: Upper profit-taking region (`SELL_TOP_PRICE` to `SELL_BOTTOM_PRICE`) where SELL signals are generated when price enters
- `BUFFER_ZONE`: Middle neutral zone (0.4% by default) accounting for transaction fees where no signals are generated
- `BUY_ZONE`: Lower accumulation region (`BUY_TOP_PRICE` to `BUY_BOTTOM_PRICE`) where BUY signals are generated when price enters

![zone-trading-illustration](https://github.com/user-attachments/assets/da5c89d7-a7c5-4dfc-ac8c-2ee34b493513)

**Calculation:** Find min/max prices over specified period (e.g 10 days). Place `BUFFER_ZONE` around midpoint. The `SELL_TOP_PRICE` equals the maximum price and the `BUY_BOTTOM_PRICE` equals the minimum price observed during the period.

## Configuration Input

```js
{
  // User identifier
  "userId": "user123",          
  // Trading pair
  "symbol": "BTC/USDT",         
  // In minutes (60 = 1 hour candles)
  "timeframe": 60,              
  // In minutes (14400 = 10 days)
  "historyMinutes": 14400,      
  // 0.4% for middle buffer (to cover fees)
  "bufferPercentage": 0.004,    

  // Check for signal every 1 hour
  "checkFrequency": 60         
}
```

## Instance Creation Output

```js
{
  // User identifier
  "userId": "user123",                                  
  // UUID for this instance
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", 
  // Current service status
  "status": "INACTIVE",                              
  // Service creation timestamp
  "createdAt": "2025-03-23T14:00:00Z",                  
  // Price zone configuration
  "zoneConfig": {                                       
    // Upper sell zone boundary
    "sellTopPrice": 90750.50,                           
    // Lower sell zone boundary
    "sellBottomPrice": 87431.25,                        
    // Upper buy zone boundary
    "buyTopPrice": 87068.75,                            
    // Lower buy zone boundary
    "buyBottomPrice": 83750.50,                         
    // When zones were calculated
    "calculatedAt": "2025-03-23T14:00:00Z",             
    // Start of analysis period
    "zoneStartsAt": "2025-03-13T14:00:00Z",             
    // End of analysis period
    "zoneEndsAt": "2025-03-23T14:00:00Z"                
  }
}
```

## Event Types

### Signal Event
Emitted when price enters a buy or sell zone and the minimum time since the last signal has elapsed.

```js
{
  // Type of event
  "eventType": "SIGNAL",                                
  // When signal was generated
  "timestamp": "2025-03-23T14:30:00Z",                  
  // User identifier
  "userId": "user123",                                  
  // Service instance ID
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", 
  // Trading pair
  "symbol": "BTC/USDT",                                 
  // Current price
  "price": 87250.45,                                    
  // Signal type (BUY/SELL)
  "signal": "BUY"                                       
}
```

## Workflow

1. **Initialization**: Load config, fetch data, calculate zones
2. **Monitoring**: Check price at intervals, compare to zones, emit signals
   - Price is checked at the frequency defined by `checkFrequency`
3. **Manual Recalculation**: User explicitly requests zone recalculation when appropriate
4. **Signals**:
   - **BUY**: Price in BUY_ZONE
   - **SELL**: Price in SELL_ZONE

## Implementation

This section outlines the technical implementation details of the Spot Trading Signaling Service.

### REST API

The service exposes the following REST endpoints for management and data retrieval:

#### Instance Management

##### Create Signal Service Instance
- **Method**: POST
- **Path**: `/api/v1/instances`
- **Request Body**: Configuration input (as shown in Configuration Input section)
- **Response**: Instance creation output (as shown in Instance Creation Output section)

##### Get Instance Details
- **Method**: GET
- **Path**: `/api/v1/instances/{instanceId}`
- **Response**: Current instance details with zones and status

##### List User Instances
- **Method**: GET
- **Path**: `/api/v1/instances?userId={userId}`
- **Response**: List of active instances for the user

##### Update Instance
- **Method**: PUT
- **Path**: `/api/v1/instances/{instanceId}`
- **Request Body**: Updated configuration parameters and/or status
```js
{
  // Optional - update configuration
  "timeframe": 30,
  "historyMinutes": 7200,
  "bufferPercentage": 0.005,
  "checkFrequency": 30,
  
  // Optional - update status
  "status": "ACTIVE" // ACTIVE, INACTIVE
}
```
- **Response**: Updated instance details

##### Recalculate Zones
- **Method**: POST
- **Path**: `/api/v1/instances/{instanceId}/recalculate`
- **Response**: Updated instance details with new zone configuration

#### Signal Retrieval

##### Get Signals
- **Method**: GET
- **Path**: `/api/v1/instances/{instanceId}/signals?limit=100`
- **Response**: Signal events ordered by timestamp descending, up to the specified limit

## Implementation -> Persistence

### Database Architecture

The service uses a document database (such as MongoDB, Firestore, or similar) to store instance configurations, state, and generated signals. This approach provides flexibility for schema evolution and naturally represents the JSON structures used throughout the service.

### Collections

#### Instances Collection

**Document Structure**

```
{
  "instanceId": UUID,                // e.g., "550e8400-e29b-41d4-a716-446655440000"
  "userId": String,                  // e.g., "user123"
  "symbol": String,                  // e.g., "BTC/USDT"
  "timeframe": Number,               // e.g., 60 (in minutes)
  "historyMinutes": Number,          // e.g., 14400 (10 days)
  "bufferPercentage": Number,        // e.g., 0.004 (0.4%)

  "checkFrequency": Number,          // e.g., 60 (in minutes)
  "status": String,                  // e.g., "ACTIVE" or "INACTIVE"
  "createdAt": Timestamp,            // e.g., "2025-03-23T14:00:00Z"
  "lastUpdatedAt": Timestamp,        // e.g., "2025-03-23T14:30:00Z",
  "zoneConfig": {
    "sellTopPrice": Number,          // e.g., 90750.50
    "sellBottomPrice": Number,       // e.g., 87431.25
    "buyTopPrice": Number,           // e.g., 87068.75
    "buyBottomPrice": Number,        // e.g., 83750.50
    "calculatedAt": Timestamp,       // e.g., "2025-03-23T14:00:00Z"
    "zoneStartsAt": Timestamp,       // e.g., "2025-03-13T14:00:00Z"
    "zoneEndsAt": Timestamp          // e.g., "2025-03-23T14:00:00Z"
  }
}
```

The Instances collection supports these operations:
- Creating new service instances from user configuration
- Retrieving instance details by ID
- Listing all instances for a specific user
- Updating instance status
- Modifying configuration parameters
- Recalculating and updating price zones
- Tracking current price zone and last signal timestamps

#### Signals Collection

**Document Structure**

```
{
  "signalId": UUID,                  // e.g., "7b83d310-5c99-4e69-a9a2-0cb81368f000"
  "timestamp": Timestamp,            // e.g., "2025-03-23T14:30:00Z"
  "userId": String,                  // e.g., "user123"
  "instanceId": UUID,                // e.g., "550e8400-e29b-41d4-a716-446655440000"
  "symbol": String,                  // e.g., "BTC/USDT"
  "price": Number,                   // e.g., 87250.45
  "signal": String                   // e.g., "BUY", "SELL"
}
```

The Signals collection supports these operations:
- Recording signal events
- Retrieving signals by instance ID
- Filtering signals by time range with pagination support
- Finding the most recent signal of a specific type to enforce minimum signal frequency