# Task 2: Transaction Routing
## Deliverables: Sample Transactions

---

## 2.1 POS to Source Station Routing (1 Hr)

### Overview

This section describes how a transaction originates from a POS terminal and reaches the Source Station (ss-wiseasy).

### POS Terminal Communication

#### **Connection Protocol**

- **Protocol**: Wiseasy (custom protocol or ISO 8583 variant)
- **Connection Type**: TCP/IP socket
- **Port**: Configured in `devel.properties` - `wiseasy_server_port=6022`
- **Host**: `localhost` (in development)

#### **Transaction Flow from POS**

```
POS Terminal
    ↓ (TCP Connection on port 6022)
    ↓ (Wiseasy Protocol)
Source Station (ss-wiseasy)
    ↓ (TCP Connection established)
    ↓ (Request received)
Incoming Listener
    ↓ (Message parsing)
    ↓ (Message format validation)
Q2 Transaction Manager
```

### Source Station Responsibilities

**Module**: `ss-wiseasy`

**Key Component**: `TransactionBuilder.java`

#### **Step 1: Receive Message from POS**

The `Incoming Listener` in ss-wiseasy:
- Accepts incoming TCP connections on port 6022
- Receives ISO 8583 message
- Validates message structure
- Creates transaction context

#### **Step 2: Parse Transaction Request**

**Message Example** (ISO 8583 format):

```
MTI:     0200                    (Authorization Request)
Field 2: 4111111111111111       (Card PAN)
Field 3: 000000                 (Processing Code - Auth)
Field 4: 000000000100           (Amount - $1.00)
Field 11: 000001                (STAN - System Trace Audit Number)
Field 14: 2512                  (Card Expiry - MM/YY)
Field 25: 05                    (Service Code)
Field 52: XXXXXXXXXXXXX         (PIN Block)
Field 103: 1234567890           (To Account)
Field 113.25: WISEASY           (Source Station Name)
```

#### **Step 3: Enrich Transaction with SS Name**

In `TransactionBuilder.java`:

```java
// Populate source station name
String ssName = ctx.getString(Constants.SS);
if (ssName != null) {
    req.set("113.25", ssName);  // Source Station in field 113.25
}

// Populate destination station name if supplied
String ds = ctx.getString(Constants.DS);
if (ds != null) {
    req.set("113.30", ds);      // Destination Station in field 113.30
}
```

#### **Step 4: Populate Invoice Number**

```java
if (isAuth(req) || isSale(req) || isCashWithdrawal(req)) {
    if (req.hasField(11) && !req.hasField("113.29")) {
        req.set("113.29", req.getString(11));  // Use STAN as invoice
    }
}
```

**Result**: Source Station enriches the transaction with routing information.

### Source Station Configuration

**Location**: `modules/ss-wiseasy/src/main/resources/`

**Typical Configuration** (q2 XML format):

```xml
<channel class="org.jpos.iso.channel.ASCIIChannel"
         host="0.0.0.0"
         port="6022"
         packager="org.jpos.iso.packager.GenericPackager"
         logger="Q2">
    <property name="timeout" value="300000" />
</channel>

<listener class="org.jpos.iso.ISOServer"
          channel="server-channel"
          port="6022"
          logger="Q2">
    <property name="sessions" value="100" />
</listener>
```

### Sample Transaction: Authorization

#### **1. POS Sends Request**

```
From POS Terminal to ss-wiseasy (port 6022):

MESSAGE (ISO 8583):
  MTI: 0200 (Auth Request)
  F2:  4111111111111111
  F3:  000000
  F4:  000000000100
  F11: 123456
  F14: 2512
  F25: 05
  F52: [PIN Block]
  F103: 1234567890
```

#### **2. Source Station Processing**

```java
// ss-wiseasy TransactionBuilder
- Validates message structure
- Adds source station identification (F113.25 = "WISEASY")
- Uses STAN (F11=123456) as invoice number (F113.29)
- Routing decision: Route to JPTS Core
- Creates transaction context with:
  * SS_REQUEST: The incoming Wiseasy message
  * SOURCE_STATION: "wiseasy"
  * INVOICE_NUMBER: "123456"
```

#### **3. Message Conversion**

From Wiseasy protocol → Internal CMF Format

**CMF Message**:
```
CMF_REQUEST {
  card: 4111111111111111
  amount: 100 (cents)
  txntype: "auth"
  pan: 4111111111111111
  expiry: 2512
  pin_block: [encrypted]
  to_account: 1234567890
  source_station: WISEASY
  invoice_number: 123456
  stand: 123456
}
```

---

## 2.2 Source Station to JPTS Core Routing (30 Mins)

### Overview

After the Source Station enriches the transaction, it routes to JPTS Core via CMF (Card Management Framework) protocol.

### Routing Mechanism

#### **1. Queue-Based Routing**

```
ss-wiseasy
    ↓ (Q2 Space)
    ↓ (Transaction Queue)
JPTS CMF Interface
    ↓ (CMF Protocol over TCP)
JPTS Core Engine
```

#### **2. CMF Protocol Connection**

**Configuration** (in `devel.properties`):

```properties
jpts_cmf_port=9999
jpts_cmf_host=localhost
jpts_cmf_timeout=30000
jpts_cmf_connect_timeout=30000
cmf_echo_interval=60000
```

**Connection Details**:
- **Protocol**: CMF (Card Management Framework)
- **Port**: 9999
- **Timeout**: 30 seconds
- **Echo/Keep-alive**: Every 60 seconds

### Transaction Participant Flow in JPTS Core

#### **Participant 1: Incoming**

**Class**: `org.jpos.gw.Incoming`

**Purpose**: Convert external protocol to CMF

```java
@Override
public int doPrepare(long id, Context ctx) throws BLException, ISOException {
    ISOMsg m = ctx.get(incomingRequest);      // SS_REQUEST
    ISOMsg converted = new ISOMsg();
    
    // Apply adapters (field mappings)
    for (CMFAdapter adapter : adapters) {
        adapter.toCMF(converted, m, originalRequest, ctx);
    }
    
    ctx.put(RESPONSE.name(), converted);
    return PREPARED | READONLY;
}
```

**Adapters Applied**:
- Track2Adapter: Convert track data
- MtiAndPCodeAdapter: MTI/Processing code conversion
- Field adapters: Individual field conversions

#### **Participant 2: TxnName**

**Class**: `org.jpos.gw.participant.TxnName`

**Purpose**: Derive transaction name for routing

```java
String txnname = m.getString(0).substring(1);  // e.g., "0200" → "200"

if (m.hasField(3)) {
    String pcode = m.getString(3);
    if (pcode.length() > 2) {
        txnname = txnname + "." + pcode.substring(0,2);  // e.g., "200.00"
    }
}
```

**Result**: `TXNNAME` = "0200.00" (determines routing)

#### **Participant 3: TranslatePin** (if PIN present)

**Class**: `org.jpos.gw.TranslatePin`

**Purpose**: Translate PIN from external to internal format

```java
if (request.hasField(52)) {  // PIN Block present
    CardHolder ch = new CardHolder(request);
    SMAdapter sm = getSecurityModule();
    SecureKeyStore keystore = getKeystore();
    
    SecureDESKey keyFrom = keystore.getKey(sourceKey);
    SecureDESKey keyTo = keystore.getKey(destKey);
    
    // Translate PIN from source format to destination format
    byte[] translatedPin = sm.translatePin(
        request.getBytes(52),
        keyFrom,
        keyTo,
        // other params
    );
    
    request.setBytes(52, translatedPin);
}
```

#### **Participant 4: CreateTranLog**

**Class**: `org.jpos.jpts.CreateTranLog`

**Purpose**: Create transaction log record

```java
protected int doPrepare(long id, Context ctx) throws Exception {
    ISOMsg m = ctx.get(REQUEST.name());
    
    TranLog tl = new TranLog();
    tl.setAmount(new BigDecimal(m.getString(4)));
    tl.setCurrency(m.getString(49));  // if present
    tl.setPan(m.getString(2));
    tl.setFromAccount(m.getString(102));
    tl.setToAccount(m.getString(103));
    tl.setSsPcode(m.getString(3));
    
    // Routing: Check for DS override
    if(m.hasField("113.30")) {
        tl.setDs(m.getString("113.30"));  // Destination Station
    }
    
    db.save(tl);  // Save to database
}
```

**Database Entry Created**:
```sql
INSERT INTO tranlog (
    pan, amount, from_account, to_account, 
    source_station, destination, processing_code, 
    invoice_number, status
) VALUES (
    '4111111111111111', 100, '9876543210', '1234567890',
    'WISEASY', 'ACI', '000000',
    '123456', 'PENDING'
);
```

### Sample Flow: Authorization Through JPTS Core

#### **Transaction State at Each Participant**

```
Initial Request (ss-wiseasy):
  SS_REQUEST = {MTI: 0200, F2: 4111..., F4: 100, ...}
  
After Incoming (convert to CMF):
  REQUEST = {CMF Format, all fields mapped}
  
After TxnName:
  TXNNAME = "0200.00"
  
After TranslatePin:
  REQUEST.F52 = [translated PIN block]
  
After CreateTranLog:
  TRAN_LOG_ID = 12345
  Database: tranlog record created
  
Transaction Context for Next Participant:
  REQUEST: CMF formatted message
  TXNNAME: "0200.00" (for routing)
  TRAN_LOG_ID: 12345
```

### Routing Decision Table

**Based on TXNNAME and destination configuration:**

| TXNNAME | Processing Code | Destination | Route |
|---------|-----------------|-------------|-------|
| 0200.00 | 000000 | Default | ds-aci |
| 0200.00 | 000000 | Specified | Override |
| 0200.31 | 310000 | N/A | ds-jcard |
| 0400.00 | 000000 | Default | ds-aci |
| 0420.00 | 000000 | Default | ds-abiy |

**Routing Logic**:
1. Check for DS override in field 113.30
2. If set, route to specified destination
3. Otherwise, use default destination for card product
4. Transaction routed to appropriate DS module

---

## 2.3 JPTS Core to Destination Station Routing (1 Hr)

### Overview

After processing in JPTS Core, the transaction is routed to a Destination Station (ds-aci, ds-abiy, etc.) for final processing.

### Destination Station Types

#### **Available Destinations**

1. **ds-aci** - ACI Payment Processor
2. **ds-abiy** - Abiy Bank
3. **ds-ethswitch** - Ethswitch Processor
4. **ds-jcard** - Internal Card Processor

### Destination Routing Process

#### **Step 1: SelectGroup Participant**

Routes based on TXNNAME to appropriate destination:

```java
// Pseudo-code routing logic
String txnname = ctx.get(TXNNAME.name());  // e.g., "0200.00"
String ds = getDSFromConfig(txnname);      // e.g., "ds-aci"

ctx.put("current.destination", ds);        // Set for next participants
```

#### **Step 2: QueryHost Participant** (if applicable)

Queries destination before sending:

```java
// Send query to destination station
// Receive: balance, limits, restrictions
ctx.put("DS_RESPONSE", queryResponse);
```

#### **Step 3: Outgoing Participant**

**Class**: `org.jpos.gw.Outgoing`

**Purpose**: Convert CMF to external protocol

```java
@Override
public int doPrepare(long id, Context ctx) throws ISOException, BLException {
    ISOMsg message = ctx.get(REQUEST.name());       // CMF message
    ISOMsg converted = new ISOMsg();
    
    // For DESTINATION type, reverse adapters
    List<CMFAdapter> incomingConverters = ctx.get(Incoming.ADAPTERS);
    switch (stationType) {
        case DESTINATION:
            // Apply inverse transformation (CMF → External)
            CMFAdapter.toCMF(incomingConverters, converted, message, originalRequest, ctx);
            CMFAdapter.toCMF(converters, converted, message, originalRequest, ctx);
            break;
    }
    
    // Store as outgoing response
    ctx.put(outgoingResponse, converted);
    return PREPARED | READONLY;
}
```

#### **Step 4: SendResponse Participant**

Sends converted message to destination station via CMF protocol:

```java
// Send DS_REQUEST to destination station via TCP
// Destination listens on its configured port
// Receive: DS_RESPONSE
ctx.put("DS_RESPONSE", response);
```

### Sample: Authorization to ACI Destination

#### **Configuration for DS-ACI**

**Location**: Configuration for ds-aci module

```
Destination Station: ACI
Protocol: Version 87 (Postilion compatible)
Port: 6022 (default)
Host: localhost (development)
```

#### **Message Transformation**

**CMF Message** (from JPTS Core):
```
REQUEST {
  MTI: 0200
  F2: 4111111111111111
  F3: 000000
  F4: 000000000100
  F14: 2512
  F52: [translated PIN]
  F113.25: WISEASY (source)
  F113.29: 123456 (invoice)
}
```

**Adapters Applied by DS-ACI**:

1. **MtiAndPCodeAdapter**: Convert MTI/processing code
   - CMF MTI 0200 → ACI format
   - Processing code mapping

2. **Track2Adapter**: Handle track data
   - Field 35, 45 conversion

3. **V87FeeAdapter**: Apply ACI fee handling
   - Extract fees from field 30, 31
   - Add to field 46

4. **V87STANAdapter**: STAN formatting
   - Ensure correct STAN format for ACI

**Output Message** (ACI compatible):
```
MESSAGE {
  MTI: 0200 (ACI format)
  F2: 4111111111111111
  F3: 000000 (ACI processing code)
  F4: 000000000100
  F14: 2512
  F30: 0000000000 (fee)
  F46: [fee details]
  F52: [ACI PIN format]
  F113.25: WISEASY
  F113.29: 123456
}
```

#### **Message Flow**

```
JPTS Core (CMF Message)
    ↓ (SendHost Participant)
    ↓ (TCP connection to ACI on port 6022)
DS-ACI Module
    ↓ (Outgoing conversion)
    ↓ (Adapters applied)
ACI Destination Station (CMF protocol)
    ↓ (Message received)
    ↓ (Processing)
ACI Processor
    ↓ (Returns response)
DS-ACI Module
    ↓ (Response converted back to CMF)
JPTS Core
    ↓ (Updates transaction log)
```

### Processing Code Mapping

**DS-ACI Processing Codes**:

| CMF Code | ACI Code | Description |
|----------|----------|-------------|
| 000000 | 000000 | Purchase |
| 310000 | 200000 | Balance inquiry |
| 400000 | 400000 | Reversal |
| 420000 | 400000 | Completion |

---

## 2.4 Destination Station to Acquirer Routing (2 Hrs)

### Overview

The Destination Station communicates with the actual payment acquirer or processor to settle the transaction.

### DS-ACI to ACI Processor Flow

#### **Network Connection**

```
DS-ACI Module (JPTS Local)
    ↓ (ISO 8583 over TCP)
    ↓ (Port configured for ACI)
ACI Processor (External/Remote)
    ↓ (Processes transaction)
    ↓ (Settlement network)
```

#### **Connection Configuration**

**For DS-ACI**: (in `devel.properties` or module config)
```properties
aci_processor_host=aci.example.com
aci_processor_port=6023
```

#### **Transaction Processing at ACI Processor**

```
DS-ACI sends:
  0200 Authorization Request
    ↓ (ACI validates card)
    ↓ (Check with issuer)
    ↓ (Authorization decision)
  
ACI responds:
  0210 Authorization Response
    Field 39: Response Code (00 = Approved, 05 = Declined, etc.)
    Field 61: Invoice Number
    Field 62: Authorization Code
    Field 63: Response Text
```

#### **Response Codes**

| Code | Meaning | Action |
|------|---------|--------|
| 00 | Approved | Transaction accepted |
| 05 | Declined | Transaction rejected |
| 19 | Re-enter transaction | Retry |
| 76 | Invalid response | Error |
| 91 | Issuer unavailable | Retry |

### DS-ABIY to Abiy Bank Flow

#### **Configuration**

```properties
abiy_server_host=localhost
abiy_server_port=9997
abiy_remote_server_host=10.1.3.102
abiy_remote_server_port=6022
```

#### **Two-Tier Architecture**

```
JPTS DS-Abiy Module
    ↓ (Local port 9997)
Abiy Adapter (acts as gateway)
    ↓ (Remote connection to 10.1.3.102:6022)
Abiy Bank Processor
    ↓ (Settlement)
```

#### **Abiy-Specific Processing**

**Invoice Number Requirement**:
```
Field 113.29: MUST contain invoice number
If not provided, STAN (F11) is used as fallback
```

**Abiy Response**:
```
Response Code 00: Approved - Transaction successful
Response Code 05: Declined - Card declined
Response Code 30: Format error - Message format issue
Response Code 40: Function not supported
```

### DS-Ethswitch Processing

#### **Purpose**

- Ethiopia-specific payment processing
- Local card network handling
- Domestic transaction routing

#### **Result Code Mapping**

**File**: `/modules/ds-ethswitch/src/main/resources/rc/ResultCode.properties`

```properties
00=Approved
01=Refer to issuer
02=Approved with ID
03=Invalid merchant
04=Pick up card
05=Do not honor
...
99=System error
```

### DS-JCard Internal Processing

#### **Purpose**

- Internal card payment processing
- No external network required
- Direct database operations

#### **Processing Steps**

```java
// DS-JCard Processing
1. Validate card details against database
2. Check card status (active, blocked, expired)
3. Validate PIN (if required)
4. Check account balance
5. If sufficient balance:
   - Debit account
   - Record transaction
   - Return approval (00)
6. If insufficient balance:
   - Do NOT debit
   - Return declined (05)
7. Generate authorization code
8. Return response
```

#### **Response Generation**

```
0210 Response Message {
  F39: Response Code (00 for approved)
  F62: Authorization Code (6 digit)
  F61: Invoice Number (from request)
  F63: "APPROVED" or error message
}
```

### Complete Transaction Example: Authorization

#### **Initial Request (POS)**

```
POS → Source Station (ss-wiseasy)
  0200 Auth Request
  F2: 4111111111111111
  F4: 100 (USD)
  F11: 123456 (STAN)
```

#### **Step 1: SS-Wiseasy Processing**

```
Add routing information:
  F113.25: WISEASY (source)
  F113.29: 123456 (invoice)
Forward to JPTS
```

#### **Step 2: JPTS Core Processing**

```
Incoming: Convert to CMF
TxnName: Derive "0200.00"
TranslatePin: Translate PIN
CreateTranLog: Create DB record
  → tranlog inserted with status = PENDING
Route to: ds-aci (default)
```

#### **Step 3: DS-ACI Processing**

```
Outgoing: Convert CMF to ACI format
Apply adapters:
  - MtiAndPCodeAdapter
  - Field mapping
Send to ACI: 0200 Request
```

#### **Step 4: ACI Processor**

```
Receive: 0200 from DS-ACI
Process: Validate with issuer
Issuer response: Approved (00)
Send response: 0210 (Approved, Code 00)
```

#### **Step 5: Response Back to JPTS**

```
DS-ACI receives: 0210 Response
Convert back: CMF format
Send to JPTS: Response
JPTS updates: tranlog status = APPROVED
```

#### **Step 6: Response Back to POS**

```
JPTS sends to SS-Wiseasy: 0210 Response
SS-Wiseasy sends to POS: 0210 (Approved)
POS displays: "APPROVED - Thank You"
```

#### **Final Database State**

```sql
SELECT * FROM tranlog WHERE invoice_number = '123456';

Result:
| id    | pan              | amount | status     | response_code | ds      |
|-------|------------------|--------|------------|---------------|---------|
| 12345 | 4111111111111111 | 100    | APPROVED   | 00            | ACI     |
```

---

## 2.5 Using the Simulator (1 Hr)

### Simulator Modules

#### **1. HSM Simulator (ssm)**

**Purpose**: Simulate Hardware Security Module

**Features**:
- PIN encryption/decryption without real HSM
- Key generation and management
- PIN translation between formats

**Configuration**:
```properties
security_module=hsm-sim
key_store=ks
```

**Usage**:
```java
// In TranslatePin participant
SMAdapter sm = getSecurityModule();  // Returns HSM simulator
SecureKeyStore ks = getKeystore();
```

#### **2. Wiseasy Simulator (sim-wiseasy)**

**Purpose**: Simulate POS/ATM sending transactions

**Features**:
- Send test ISO 8583 messages
- Configurable test scenarios
- Response verification

**Usage**:
```bash
# From jPTS console
# Send test transaction
```

#### **3. Abiy Simulator (sim-ds-abiy)**

**Purpose**: Simulate Abiy bank responses

**Features**:
- Return predefined responses
- Error condition simulation
- Load testing

**Configuration**:
```
Listen on: localhost:9997
Return responses: Approval, Decline, Error codes
```

### Running Simulators

#### **1. Start Main System**

```bash
cd modules/jpts-app/build/install/jpts-app
./bin/q2
```

#### **2. Send Test Transaction**

**Using jPTS Console**:
```
q2> send-test-transaction \
    --card=4111111111111111 \
    --amount=100 \
    --type=auth
```

Or use external client:

```bash
# Connect to port 6022 and send ISO message
nc localhost 6022 < test_message.iso
```

#### **3. Monitor Responses**

```
q2> tail -f logs/jpts.log
# Watch for transaction processing
```

### Test Scenarios

#### **Scenario 1: Successful Authorization**

```
POS Request:
  Card: 4111111111111111
  Amount: 100 USD
  Auth code: 123456

Expected Result:
  Response Code: 00 (Approved)
  tranlog status: APPROVED
```

#### **Scenario 2: Declined Transaction**

```
POS Request:
  Card: 5555555555554444
  Amount: 1000 USD

Expected Result:
  Response Code: 05 (Declined)
  tranlog status: DECLINED
  Reason: Issuer declined
```

#### **Scenario 3: Timeout Handling**

```
Configuration:
  jpts_cmf_timeout=30000

Test:
  Send request to slow destination
  Wait > 30 seconds

Expected Result:
  Response Code: 91 (Timeout)
  tranlog status: TIMEOUT
```

#### **Scenario 4: PIN Translation Test**

```
Source PIN Key: test_key_1
Destination PIN Key: test_key_2

Message with F52 (PIN block):
  Before: Encrypted with test_key_1
  After TranslatePin: Encrypted with test_key_2
  Verification: Compare with expected value
```

### Verification Tools

#### **Database Query**

```sql
-- Check transaction status
SELECT id, pan, amount, status, response_code, ds, created
FROM tranlog
ORDER BY created DESC
LIMIT 10;
```

#### **Log Analysis**

```bash
# Check for errors
grep -i error logs/jpts.log

# Monitor participant execution
grep "doPrepare" logs/jpts.log

# Track routing decisions
grep "TXNNAME\|ds\|destination" logs/jpts.log
```

#### **Latency Monitoring**

```sql
-- Calculate avg response time
SELECT 
  ds,
  AVG(EXTRACT(EPOCH FROM (updated - created))) as avg_time_sec
FROM tranlog
WHERE status = 'APPROVED'
GROUP BY ds;
```

---

## Summary: Complete Transaction Flow

### Simplified Flow Diagram

```
POS (Port 6022)
  ↓ (Wiseasy)
SS-Wiseasy (Source Station)
  ↓ (CMF on port 9999)
JPTS Core (Incoming/TxnName/TranslatePin/CreateTranLog)
  ↓ (Select based on TXNNAME)
Destination Station (ds-aci/ds-abiy/ds-ethswitch/ds-jcard)
  ↓ (TCP to acquirer)
External Acquirer (ACI/Abiy/Ethswitch)
  ↓ (Response)
JPTS Core (Update TranLog)
  ↓ (Response)
SS-Wiseasy
  ↓ (Response on port 6022)
POS Terminal
```

### Key Takeaways

1. **Entry**: Transactions enter via Source Station (ss-wiseasy)
2. **Enrichment**: Source station adds routing metadata
3. **CMF Conversion**: JPTS core converts to internal format
4. **Routing**: Transaction name determines destination station
5. **PIN Translation**: Secure PIN translation between formats
6. **Logging**: Database record created for audit
7. **Destination Processing**: Destination station converts and sends to acquirer
8. **Response Flow**: Response flows back through same path
9. **Status Update**: Database updated with final result

**Next Step**: Move to Task 3 (Monitoring and Logs) to understand system observation.
