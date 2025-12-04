# Task 1: System Setup
## Deliverables: JPTS Repo, JPTS Local Setup

---

## 1.1 System Structure (1 Hr)

### Architecture Overview

The flocash-jPTS system is a payment transaction processing platform built on the jPOS (Java Payment Processing System) framework. It follows a modular, layered architecture designed to handle payment card transactions across multiple channels (POS, ATM, MOTO, e-commerce).

### High-Level System Flow

```
POS/Terminal
    ↓
    ├─→ Source Station (ss-wiseasy, ssm)
        ↓
        ├─→ JPTS Core (jpts, gateway)
            ↓
            ├─→ Destination Station (ds-aci, ds-abiy, ds-ethswitch, ds-jcard)
                ↓
                ├─→ Acquirer/Payment Processor
```

### Core Architecture Components

#### **Layer 1: Entry Points**
- **Source Station (ss-wiseasy)**: Receives transactions from POS terminals
- **REST API (restapi)**: HTTP interface for external integrations
- **APP (app)**: Web-based management interface

#### **Layer 2: Gateway/Translation**
- **Gateway (gateway)**: Core message conversion and routing logic
- **Key Responsibilities**:
  - `Incoming.java`: Converts external protocol to internal CMF format
  - `Outgoing.java`: Converts CMF format back to external protocol
  - `TranslatePin.java`: PIN block translation between external and internal formats
  - `TxnName.java`: Derives transaction name for routing
  - `StationType.java`: Defines station types (SOURCE, DESTINATION)

#### **Layer 3: Core Processing**
- **JPTS Core (jpts)**: Transaction processing engine
- **Components**:
  - `CreateTranLog.java`: Creates transaction log records
  - Transaction routing logic
  - CMF (Card Management Framework) integration

#### **Layer 4: Destination Stations**
- **ds-aci**: ACI payment processor connector
- **ds-abiy**: Abiy bank connector
- **ds-ethswitch**: Ethswitch connector
- **ds-jcard**: JCard payment platform connector
- **ds-jcard**: Internal card payment processing

#### **Layer 5: Support Services**
- **CMF (cmf)**: Card holder management and message formatting
- **SSM (ssm)**: Security/HSM simulator module
- **Database (db-dev)**: PostgreSQL with migration scripts
- **Simulators (sim-wiseasy, sim-ds-abiy)**: Testing environments

### System Components Summary

| Component | Purpose | Type |
|-----------|---------|------|
| `gateway` | Message translation and routing | Core |
| `jpts` | Transaction processing engine | Core |
| `jpts-app` | Main application container | App |
| `ss-wiseasy` | Source station (Wiseasy protocol) | Connector |
| `ds-aci`, `ds-abiy`, `ds-ethswitch`, `ds-jcard` | Destination stations | Connectors |
| `restapi` | REST API interface | Interface |
| `cmf` | Card management framework | Utility |
| `ssm` | Security module simulator | Security |
| `db-dev` | Database migrations | Infrastructure |
| `app` | Web management interface | App |

### Key Data Structures

#### **ISOMsg (ISO 8583 Message)**
Standard financial transaction message format used throughout the system.

#### **CMF (Card Management Framework)**
Internal standardized format for transaction representation across modules.

#### **Context**
Transaction-scoped data container that flows through transaction participants.

#### **StationType Enum**
Defines whether a component is:
- **SOURCE**: Receives external transactions
- **DESTINATION**: Sends to external acquirers

---

## 1.2 Modules: Source Station, JPTS Core, Destination Stations (3 Hrs)

### Module Overview

#### **1. Source Station Module (ss-wiseasy)**

**Location**: `/modules/ss-wiseasy/`

**Purpose**: 
- Receives transactions from POS/ATM terminals via Wiseasy protocol
- Converts incoming protocol to internal CMF format
- Routes transactions to JPTS core

**Key Components**:
```java
// TransactionBuilder.java
- Populates invoice numbers (from STAN if not present)
- Sets source station name (SS_REQUEST field 113.25)
- Sets destination station name (DS_REQUEST field 113.30)
- Marks request for JPTS processing
```

**Configuration** (in `devel.properties`):
```properties
wiseasy_xml_server_port=9001
wiseasy_server_port=6022
```

**Flow**:
1. POS terminal connects to Source Station
2. Transaction received in Wiseasy protocol
3. TransactionBuilder populates required fields
4. Message converted to CMF format
5. Routed to JPTS Core

#### **2. JPTS Core Module (jpts)**

**Location**: `/modules/jpts/`

**Purpose**:
- Central transaction processing engine
- Transaction logging
- Routing decisions
- Final processing coordination

**Key Components**:
```java
// CreateTranLog.java
- Creates transaction log records in database
- Extracts transaction details (amount, account, routing info)
- Stores routing information for audit/compliance
- Field 113.30 can override routing (ds name based routing)

// Transaction Participants
- Validates transaction data
- Updates transaction status
- Processes responses from destination stations
```

**Transaction Types Supported**:
- Auth (Authorization)
- Sale
- Cash Withdrawal
- Refund
- Balance Inquiry
- And others

**Field Mappings** (extracted from transaction):
```
Field 2: Card PAN
Field 3: Processing Code (txtype)
Field 4: Transaction Amount
Field 11: STAN (System Trace Audit Number) / Invoice Number
Field 14: Card Expiry
Field 103: To Account
Field 112.3: Source Station Processing Code
Field 113.29: Invoice Number
Field 113.30: Destination Station Override
```

**Database Tables** (created by migrations):
- `tranlog`: Main transaction log
- `cardproducts`: Card product definitions
- `binrange`: Card BIN (Bank Identification Number) ranges

#### **3. Destination Station Modules**

**General Structure** (all ds-* modules):
```
/modules/ds-{name}/
├── src/main/java/org/jpos/...
│   ├── Outgoing messages to acquirer
│   ├── Protocol-specific conversion
│   ├── Field mapping
│   └── Error handling
└── src/main/resources/
    └── packager configurations
```

##### **a) ACI Destination (ds-aci)**

**Purpose**: 
- Connect to ACI payment processor
- Convert CMF to ACI protocol
- Handle ACI-specific message formats

**Key Protocol**:
- Uses VERSION 87 message format
- Field transformations for ACI compatibility
- Standard ACI result codes

##### **b) Abiy Destination (ds-abiy)**

**Location**: `/modules/ds-abiy/`

**Purpose**: 
- Connect to Abiy bank's payment system
- Process through Abiy's acquisition platform

**Configuration**:
```properties
abiy_server_host=localhost
abiy_server_port=9997
abiy_remote_server_host=10.1.3.102
abiy_remote_server_port=6022
```

##### **c) Ethswitch Destination (ds-ethswitch)**

**Purpose**: 
- Connect to Ethswitch payment processor
- Handle Ethiopia-specific payment formats
- Result code mapping from ResultCode.properties

**Resources**:
```
rc/ResultCode.properties: Error code mappings
```

##### **d) JCard Destination (ds-jcard)**

**Purpose**: 
- Internal card payment processing
- Local acquiring for card-based transactions
- Balance checking and fund transfers

---

## 1.3 Database Setup (1 Hr)

### Database Configuration

**Default Database**: PostgreSQL (MySQL also supported)

**Configuration File**: `devel.properties`

```properties
# Current Configuration
dbname=flocash_jpts
dbuser=jpos
dbpass=password
dbhost=localhost
dbport=5432
```

### Database Setup Steps

#### **Step 1: Create PostgreSQL User and Database**

```sql
-- Log into PostgreSQL as superuser
CREATE USER jpos WITH PASSWORD 'password' SUPERUSER;
CREATE DATABASE flocash_jpts OWNER jpos;
```

#### **Step 2: Verify Connection**

```bash
psql -h localhost -U jpos -d flocash_jpts
```

#### **Step 3: Initialize Schema**

The database schema is created automatically via migrations:

**Migration Files Location**: `/modules/db-dev/src/main/resources/db/migration/`

**Migrations**:
1. `V1__base_schema.sql` - Initial schema creation
2. `V002__add_consumer_hash.sql` - Consumer hash table addition

### Database Schema Overview

#### **Table: tranlog**

Main transaction logging table
```sql
-- Fields include:
id              BIGINT PRIMARY KEY
created         TIMESTAMP
amount          DECIMAL
currency        VARCHAR
card_pan        VARCHAR (masked)
processing_code VARCHAR
source_station  VARCHAR -- ss name
destination     VARCHAR -- ds name
status          VARCHAR -- transaction status
response_code   VARCHAR
result_msg      VARCHAR
to_account      VARCHAR
invoice_number  VARCHAR
ds_transport_data JSON -- for custom data
```

#### **Table: cardproducts**

Card product definitions
```sql
id              BIGINT PRIMARY KEY
name            VARCHAR UNIQUE
active          CHAR(1) -- 'Y' or 'N'
pos             CHAR(1) -- Supports POS
atm             CHAR(1) -- Supports ATM
moto            CHAR(1) -- Supports MOTO
ecommerce       CHAR(1) -- Supports E-commerce
ds              VARCHAR -- Default destination station
```

#### **Table: binrange**

Card BIN range definitions
```sql
id              BIGINT PRIMARY KEY
start           VARCHAR -- 6-12 digit BIN start
"end"           VARCHAR -- 6-12 digit BIN end
weight          INT
cardproduct     BIGINT FOREIGN KEY
```

### Initial Data Setup

**Create Default Card Product**:

```sql
INSERT INTO cardproducts (id, name, active, pos, atm, moto, ecommerce, ds) 
VALUES (1, 'Default', 'Y', 'Y', 'Y', 'Y', 'Y', 'xml');

INSERT INTO binrange (id, start, "end", weight, cardproduct) 
VALUES (1, '000000000000', '999999999999', 0, 1);
```

This allows all test cards to be processed through the default destination station.

---

## 1.4 System Setup (2 Hrs)

### Prerequisites

**Required Software**:
- Java Development Kit (JDK) 1.8+
- Gradle (build tool)
- PostgreSQL (database)
- Git (version control)

**Verify Prerequisites**:
```bash
java -version          # Should show 1.8+
gradle --version       # Should show current version
psql --version         # Should show PostgreSQL version
```

### Build Process

#### **Step 1: Configure Development Properties**

Edit `/devel.properties`:

```properties
# Database Configuration
dbname=flocash_jpts
dbport=5432
dbhost=localhost
dbuser=jpos
dbpass=password

# Service Ports
wiseasy_xml_server_port=9001
wiseasy_server_port=6022
jpts_cmf_port=9999
jpts_cmf_host=localhost

# Security
security_module=hsm-sim
key_store=ks
```

#### **Step 2: Build All Modules**

```bash
cd /home/student/flocash-jPTS
gradle -Ptarget=devel clean installApp
```

**What This Does**:
- Compiles all modules (gateway, jpts, ss-wiseasy, ds-*, etc.)
- Runs unit tests
- Packages applications
- Creates executable scripts

**Build Artifacts**:
```
modules/jpts-app/build/install/jpts-app/  -- Main application
bin/q2                                     -- Q2 command utility
```

#### **Step 3: Create Database Schema**

**DANGER**: This command wipes the database!

```bash
./modules/jpts-app/build/install/jpts-app/bin/q2 \
  -c "createschema /tmp/schema.sql yes;shutdown --force"
```

#### **Step 4: Load Initial Data**

```sql
-- Insert default card product
INSERT INTO cardproducts (id, name, active, pos, atm, moto, ecommerce, ds) 
VALUES (1, 'Default', 'Y', 'Y', 'Y', 'Y', 'Y', 'xml');

-- Insert BIN range (all test cards)
INSERT INTO binrange (id, start, "end", weight, cardproduct) 
VALUES (1, '000000000000', '999999999999', 0, 
        (SELECT id FROM cardproducts WHERE name = 'Default'));
```

### Configuration Files

#### **devel.properties**
Main configuration file for development environment. Contains:
- Database connection parameters
- Service ports
- Security module settings
- Station-specific configuration

#### **jpts-app Configuration** 
Location: `modules/jpts-app/src/main/resources/`

Contains:
- Transaction routing rules
- Message adapters
- Protocol converters
- Participant configurations

### Running the System

#### **Start Main Application**

```bash
cd modules/jpts-app/build/install/jpts-app
./bin/q2
```

#### **Check System Status**

```bash
# From jPTS shell
status
```

#### **Shutdown**

```bash
# From jPTS shell
shutdown
```

### Verification Checklist

- [ ] Database created and accessible
- [ ] All modules compiled successfully
- [ ] Schema created with tables
- [ ] Default card product inserted
- [ ] jpts-app starts without errors
- [ ] Source station listening on port 6022
- [ ] CMF service on port 9999
- [ ] No connection errors in logs

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Database connection refused | PostgreSQL not running | Start PostgreSQL service |
| Build fails | Gradle not installed | Install Gradle or use gradlew |
| Schema creation fails | User permissions | Ensure jpos user is SUPERUSER |
| Application won't start | Port already in use | Change ports in devel.properties |

---

## Summary

Task 1 establishes:
1. ✓ System architecture understanding
2. ✓ Module organization and purpose
3. ✓ Database schema and initialization
4. ✓ Build and deployment process
5. ✓ Configuration management
6. ✓ Running the system locally

**Next Step**: Move to Task 2 (Transaction Routing) to understand how transactions flow through the system.
