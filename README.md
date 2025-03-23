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
  // Price goes 5% beyond min/max - we must stop
  "stopPlankPercentage": 0.05,  
  // Check for signal every 1 hour
  "checkFrequency": 60,         
  // Minimum time (in minutes) between consecutive BUY or SELL signals (4 hours)
  "minBuySellFrequency": 240    
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
  "status": "INITIALIZED",                              
  // Service creation timestamp
  "createdAt": "2025-03-23T14:00:00Z",                  
  // Price zone configuration
  "zoneConfig": {                                       
    // Emergency upper boundary
    "stopHighPrice": 92565.51,                          
    // Upper sell zone boundary
    "sellTopPrice": 90750.50,                           
    // Lower sell zone boundary
    "sellBottomPrice": 87431.25,                        
    // Upper buy zone boundary
    "buyTopPrice": 87068.75,                            
    // Lower buy zone boundary
    "buyBottomPrice": 83750.50,                         
    // Emergency lower boundary
    "stopLowPrice": 82075.49,                           
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
  // Signal type (BUY/SELL/PAUSE/STOP)
  "signal": "BUY"                                       
}
```

### Zone Recalculation Event
Triggered when price zones are recalculated, either automatically or by manual user request.

```js
{
  // Type of event
  "eventType": "ZONE_RECALCULATION",                    
  // When recalculation occurred
  "timestamp": "2025-03-24T00:00:00Z",                  
  // User identifier
  "userId": "user123",                                  
  // Service instance ID
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", 
  // Trading pair
  "symbol": "BTC/USDT",                                 
  // New price zones
  "zoneConfig": {                                       
    // New emergency upper boundary
    "stopHighPrice": 93125.22,                          
    // New upper sell zone boundary
    "sellTopPrice": 91300.25,                           
    // New lower sell zone boundary
    "sellBottomPrice": 87937.50,                        
    // New upper buy zone boundary
    "buyTopPrice": 87562.50,                            
    // New lower buy zone boundary
    "buyBottomPrice": 84200.75,                         
    // New emergency lower boundary
    "stopLowPrice": 82516.74,                           
    // Start of new analysis period
    "zoneStartsAt": "2025-03-14T00:00:00Z",             
    // End of new analysis period
    "zoneEndsAt": "2025-03-24T00:00:00Z"                
  },
  // Reason for recalculation
  "reason": "ZONE_CHANGE_PROPOSAL"                      
}
```

### Zone Crossing Event
Generated when price moves from one zone to another (e.g., from BUY_ZONE to BUFFER_ZONE), providing opportunity to track price movement patterns.

```js
{
  // Type of event
  "eventType": "ZONE_CROSSING",                         
  // When zone crossing occurred
  "timestamp": "2025-03-23T18:15:00Z",                  
  // User identifier
  "userId": "user123",                                  
  // Service instance ID
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", 
  // Trading pair
  "symbol": "BTC/USDT",                                 
  // Current price
  "price": 87500.25,                                    
  // Zone price was in
  "previousZone": "BUY_ZONE",                           
  // Zone price moved to
  "currentZone": "BUFFER_ZONE",                         
  // Price movement direction
  "direction": "UPWARD"                                 
}
```

### Status Change Event
Occurs when the bot's operational status changes (e.g., from ACTIVE to STOPPED), typically due to emergency conditions or user actions.

```js
{
  // Type of event
  "eventType": "STATUS_CHANGE",                         
  // When status changed
  "timestamp": "2025-03-23T22:45:00Z",                  
  // User identifier
  "userId": "user123",                                  
  // Service instance ID
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", 
  // Trading pair
  "symbol": "BTC/USDT",                                 
  // Previous service status
  "previousStatus": "ACTIVE",                           
  // New service status
  "newStatus": "STOPPED",                               
  // Reason for status change
  "reason": "STOP_LEVEL_REACHED",                       
  // Price at status change
  "price": 82000.15                                     
}
```

### Error Event
Fired when an operational error occurs during bot execution that requires attention but doesn't necessarily stop the bot.

```js
{
  // Type of event
  "eventType": "ERROR",                                 
  // When error occurred
  "timestamp": "2025-03-23T16:20:00Z",                  
  // User identifier
  "userId": "user123",                                  
  // Service instance ID
  "instanceId": "550e8400-e29b-41d4-a716-446655440000", 
  // Trading pair
  "symbol": "BTC/USDT",                                 
  // Error code
  "errorCode": "DATA_FETCH_FAILURE",                    
  // Error description
  "errorMessage": "Failed to retrieve price data from exchange API" 
}
```

## Workflow

1. **Initialization**: Load config, fetch data, calculate zones
2. **Monitoring**: Check price at intervals, compare to zones, emit signals
   - Price is checked at the frequency defined by `checkFrequency`
   - BUY/SELL signals are only emitted if at least `minBuySellFrequency` minutes have passed since the last signal of the same type
3. **Manual Recalculation**: User explicitly requests zone recalculation when appropriate
4. **Signals**:
   - **BUY**: Price in BUY_ZONE
   - **SELL**: Price in SELL_ZONE
   - **PAUSE**: Price in BUFFER_ZONE or out of buy/sell boundaries
   - **STOP**: Price beyond stop boundaries (requires manual restart)

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

#### Zone Management

##### Recalculate Zones
- **Method**: POST
- **Path**: `/api/v1/instances/{instanceId}/zones/recalculate`
- **Response**: Zone recalculation event data

##### Get Current Zones
- **Method**: GET
- **Path**: `/api/v1/instances/{instanceId}/zones`
- **Response**: Current zone configuration

#### Event Retrieval

##### Get Events
- **Method**: GET
- **Path**: `/api/v1/instances/{instanceId}/events?type={eventType}`
- **Response**: Last 100 events of specified type (or all types if not specified) ordered by timestamp descending
- **Notes**: 
  - To get signals, use `type=SIGNAL`
  - Other valid types: `ZONE_RECALCULATION`, `ZONE_CROSSING`, `STATUS_CHANGE`, `ERROR`
  - If no type is specified, returns all event types

#### Service Control

##### Change Service Status
- **Method**: PUT
- **Path**: `/api/v1/instances/{instanceId}/status`
- **Request Body**:
```js
{
  "status": "ACTIVE", // ACTIVE, PAUSED, STOPPED
}
```
- **Response**: Status change event data

##### Update Configuration
- **Method**: PUT
- **Path**: `/api/v1/instances/{instanceId}/config`
- **Request Body**: Updated configuration parameters
- **Response**: Updated instance details

## Implementation -> Persistence

### Database Architecture

The service uses a document database (such as MongoDB, Firestore, or similar) to store instance configurations, state, and generated events. This approach provides flexibility for schema evolution and naturally represents the JSON structures used throughout the service.

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
  "stopPlankPercentage": Number,     // e.g., 0.05 (5%)
  "checkFrequency": Number,          // e.g., 60 (in minutes)
  "minBuySellFrequency": Number,     // e.g., 240 (in minutes)
  "status": String,                  // e.g., "ACTIVE", "INITIALIZED", "PAUSED", "STOPPED"
  "createdAt": Timestamp,            // e.g., "2025-03-23T14:00:00Z"
  "lastUpdatedAt": Timestamp,        // e.g., "2025-03-23T14:30:00Z"
  "lastSignalTimestamps": {
    "BUY": Timestamp,                // e.g., "2025-03-23T14:30:00Z"
    "SELL": Timestamp                // e.g., "2025-03-22T10:15:00Z"
  },
  "currentZone": String,             // e.g., "BUFFER_ZONE", "BUY_ZONE", "SELL_ZONE"
  "zoneConfig": {
    "stopHighPrice": Number,         // e.g., 92565.51
    "sellTopPrice": Number,          // e.g., 90750.50
    "sellBottomPrice": Number,       // e.g., 87431.25
    "buyTopPrice": Number,           // e.g., 87068.75
    "buyBottomPrice": Number,        // e.g., 83750.50
    "stopLowPrice": Number,          // e.g., 82075.49
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
- Updating instance status (with timestamps and reason)
- Modifying configuration parameters
- Recalculating and updating price zones
- Tracking current price zone and last signal timestamps

#### Events Collection

**Document Structure**

```
{
  "eventId": UUID,                   // e.g., "7b83d310-5c99-4e69-a9a2-0cb81368f000"
  "eventType": String,               // e.g., "SIGNAL", "ZONE_RECALCULATION", "ZONE_CROSSING", "STATUS_CHANGE", "ERROR"
  "timestamp": Timestamp,            // e.g., "2025-03-23T14:30:00Z"
  "userId": String,                  // e.g., "user123"
  "instanceId": UUID,                // e.g., "550e8400-e29b-41d4-a716-446655440000"
  "symbol": String,                  // e.g., "BTC/USDT"
  
  // Fields for SIGNAL events
  "price": Number,                   // e.g., 87250.45
  "signal": String,                  // e.g., "BUY", "SELL", "PAUSE", "STOP"
  
  // Fields for ZONE_RECALCULATION events
  "zoneConfig": {
    "stopHighPrice": Number,         // e.g., 93125.22
    "sellTopPrice": Number,          // e.g., 91300.25
    "sellBottomPrice": Number,       // e.g., 87937.50
    "buyTopPrice": Number,           // e.g., 87562.50
    "buyBottomPrice": Number,        // e.g., 84200.75
    "stopLowPrice": Number,          // e.g., 82516.74
    "zoneStartsAt": Timestamp,       // e.g., "2025-03-14T00:00:00Z"
    "zoneEndsAt": Timestamp          // e.g., "2025-03-24T00:00:00Z"
  },
  "reason": String,                  // e.g., "ZONE_CHANGE_PROPOSAL", "SCHEDULED_RECALCULATION"
  
  // Fields for ZONE_CROSSING events
  "previousZone": String,            // e.g., "BUY_ZONE"
  "currentZone": String,             // e.g., "BUFFER_ZONE"
  "direction": String,               // e.g., "UPWARD", "DOWNWARD"
  
  // Fields for STATUS_CHANGE events
  "previousStatus": String,          // e.g., "ACTIVE"
  "newStatus": String,               // e.g., "STOPPED"
  "reason": String,                  // e.g., "STOP_LEVEL_REACHED", "USER_REQUESTED"
  
  // Fields for ERROR events
  "errorCode": String,               // e.g., "DATA_FETCH_FAILURE"
  "errorMessage": String             // e.g., "Failed to retrieve price data"
}
```

The Events collection supports these operations:
- Recording events of all types (SIGNAL, ZONE_RECALCULATION, ZONE_CROSSING, STATUS_CHANGE, ERROR)
- Retrieving events by instance ID with optional filtering by event type
- Filtering events by time range with pagination support
- Finding the most recent signal of a specific type to enforce minimum signal frequency