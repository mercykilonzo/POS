# Task 4: Error Codes
## Deliverables: Error code list

---

## 4.1 How to Find and Interpret Error Codes (1 Hr)

### Overview

Error codes in flocash-jPTS come from multiple sources:
1. **ISO 8583 Response Codes** (Field 39)
2. **CMF Result Codes** (Internal framework)
3. **Protocol-Specific Codes** (ACI, Abiy, Ethswitch, etc.)
4. **System Error Codes** (Database, network, configuration)

### ISO 8583 Standard Response Codes

These are the most common codes returned in Field 39 of ISO 8583 messages.

#### **Success Codes (0-09)**

| Code | Meaning | Action | Example |
|------|---------|--------|---------|
| 00 | Approved | Transaction successful - process normally | Card authorization approved |
| 01 | Refer to Issuer | Contact card issuer for approval | Call bank for clearance |
| 02 | Approved with ID | Approved, issuer ID required | Get issuer identification |
| 03 | Invalid Merchant | Merchant account invalid | Check merchant setup |
| 04 | Pick up Card | Confiscate card | Card blocked by issuer |

#### **Decline Codes (10-19)**

| Code | Meaning | Action | Example |
|------|---------|--------|---------|
| 05 | Do not honor | Card issuer declines transaction | Customer declined or limit reached |
| 06 | Error | General processing error | Retry transaction |
| 07 | Pick up Card, Special | Card flagged for confiscation | Fraud/lost card alert |
| 08 | Honor with ID | Issuer requests ID verification | Request additional verification |
| 09 | Request in progress | Try again later | Issuer system temporarily busy |
| 10 | Partial Approval | Only part of amount approved | Process partial transaction or decline |

#### **Try Again Codes (11-19)**

| Code | Meaning | Action | Example |
|------|---------|--------|---------|
| 11 | VIP Approval | Requires supervisor approval | Escalate for manual review |
| 12 | Invalid Transaction | Message format or data error | Check message structure |
| 13 | Invalid Amount | Amount outside limits | Validate transaction amount |
| 14 | Invalid Card Number | Card PAN fails validation | Verify card number |
| 15 | No Issuer | Issuer not found/unavailable | Retry or use backup processor |
| 16 | Approved, Update Track 3 | Update track 3 data | Update card data |
| 17 | Customer Authentication Failed | PIN/password incorrect | Request correct PIN |
| 18 | No Account | Card account not found | Invalid card/account |
| 19 | Transaction Not Permitted | Card not enabled for transaction | Check card capabilities |

#### **System Error Codes (20-29)**

| Code | Meaning | Cause | Action |
|------|---------|-------|--------|
| 20 | Issuer Unavailable | Connection to issuer failed | Retry or queue offline |
| 21 | No Action Taken | Request cannot be processed | Verify request completeness |
| 22 | Suspected Malfunction | System error detected | Investigate and retry |
| 23 | Unacceptable Transaction Fee | Fee exceeds limits | Reduce fee or use alternate route |
| 24 | File Update Not Performed | Database update failed | Retry operation |
| 25 | Unable to Locate Record | Record not found | Verify identifier |
| 26 | Duplicate Transmission | Message already processed | Check for duplicate |
| 27 | Field Error | Data field validation failed | Correct field data |
| 28 | Date Invalid | Transaction date invalid | Check system date |
| 29 | Original Transaction Not Found | Cannot reverse non-existent txn | Verify reversal details |

#### **Network & Timeout Codes (30-99)**

| Code | Meaning | Cause | Action |
|------|---------|-------|--------|
| 30 | Format Error | Message structure invalid | Validate message format |
| 39 | No Credit Account | Account not found | Verify account number |
| 40 | Requested Function Not Supported | Operation not available | Use different transaction type |
| 41 | Specified Card Issue Date Lost/Stolen | Card flagged | Confiscate card |
| 42 | No Chequeing Account | No checking account on card | Verify account type |
| 43 | No Savings Account | No savings account on card | Verify account type |
| 44 | Card Expired | Card validity period ended | Request new card |
| 45 | Ineligible Card | Card cannot be used | Contact issuer |
| 46 | Activity Violation | Transaction type not allowed | Check card restrictions |
| 47 | Contact Acquirer | Requires acquirer contact | Escalate to support |
| 48 | Conflicting Data | Contradictory data in message | Correct data conflicts |
| 49 | Reserved For Nationally Defined Use | Issuer-specific code | Check with issuer |
| 50 | Approved for Partial Amount | Only part approved | Process partial amount |
| 51 | Expired Card | Card expired | Request new card |
| 52 | Reserved For Issuer Use | Issuer-specific error | Contact issuer |
| 53 | Reserved For Issuer Use | Issuer-specific error | Contact issuer |
| 54 | Acceptable Surcharge Fee Exceeded | Surcharge too high | Reduce surcharge |
| 55 | Incorrect PIN | PIN validation failed | Request correct PIN |
| 56 | No Card Record | Card not in database | Verify card number |
| 57 | Transaction Type Not Permitted | Card cannot perform transaction | Verify transaction type |
| 58 | Merchant Not Permitted | Merchant restricted for this card | Verify merchant code |
| 59 | Suspected Fraud | Fraud detected | Block card/transaction |
| 60 | Contact Acquirer | Acquirer intervention required | Escalate to acquirer |
| 61 | Excessive Activity | Too many transactions | Wait before retry |
| 62 | Restricted Card | Card restrictions applied | Contact issuer |
| 63 | Security Violation | Security check failed | Verify credentials |
| 64 | Original Amount Incorrect | Reversal amount mismatch | Verify amount |
| 65 | Exceeds Withdrawal Limit | Daily limit exceeded | Try again later |
| 66 | Incorrect PIN (3 attempts) | PIN incorrect 3 times | Card locked |
| 67 | Hard Capture | Card must be retained | Confiscate card |
| 68 | Response Received Too Late | Timeout from issuer | Retry transaction |
| 69 | Duplicate Transaction | Transaction already processed | Verify duplicate |
| 70 | Contact Card Issuer | Issuer contact required | Call issuer |
| 71 | Declined - Referral | Requires referral approval | Manual review |
| 72 | Declined - Decline | Transaction declined | Transaction rejected |
| 73 | Customer Cancellation | Customer cancelled transaction | Normal cancellation |
| 74 | Duplicate Reversal | Reversal already processed | Verify reversal |
| 75 | Allowable Number Of PIN Tries Exceeded | Too many PIN attempts | Card locked |
| 76 | Invalid Response | Invalid response format | Retry or escalate |
| 77 | Intervening File Update | Database conflict | Retry transaction |
| 78 | Approved Scheduled Payment Transaction | Scheduled payment OK | Process normally |
| 79 | Approved Recurring Transaction | Recurring transaction OK | Process normally |
| 80 | Network Timeout | No response from network | Retry or offline |
| 81 | PIN Change Block | PIN change blocked | Contact support |
| 82 | No Suitable Account | No valid account | Verify account |
| 83 | Not From Terminal | Transaction invalid for terminal | Verify terminal type |
| 84 | No Account Of Type | Account type not available | Use different account |
| 85 | No Reason Provided | Generic decline | Contact issuer |
| 86 | Cannot Verify PIN | PIN validation failed | Request correct PIN |
| 87 | Integrity Check Failed | Data validation error | Retry transaction |
| 88 | Authentication Failure | Authentication failed | Verify credentials |
| 89 | Transaction Type Not Allowed | Transaction not permitted | Use different type |
| 90 | Cutoff In Progress | System maintenance | Retry later |
| 91 | Issuer Or Switch Inoperative | Service unavailable | Retry or use backup |
| 92 | Unable To Route | No route available | Check routing configuration |
| 93 | Transaction Cannot Be Completed | System issue | Retry or escalate |
| 94 | Duplicate Transmission | Already processed | Check for duplicates |
| 95 | Reconciliation Error | Balance mismatch | Investigate discrepancy |
| 96 | System Malfunction | System error | Report and retry |
| 97 | Reserved For National Use | Country-specific code | Check local codes |
| 98 | Reserved For International Interbank Use | International banking code | Check intl standards |
| 99 | Reserved For Private Use | Private implementation code | Check implementation |

### CMF Result Codes

Internal jPTS result codes from `org.jpos.rc.CMF`:

#### **Common CMF Codes**

```java
// From jPOS CMF result codes
public static final String APPROVED = "00";
public static final String DECLINED = "05";
public static final String INTERNAL_ERROR = "96";
public static final String ISSUER_TIMEOUT = "91";
```

| Code | Meaning | Handled By | Recovery |
|------|---------|-----------|----------|
| APPROVED (00) | Transaction approved | All participants | Update DB status |
| DECLINED (05) | Transaction declined | Participant chain | Return to POS |
| INTERNAL_ERROR (96) | System error | Error handler | Log and alert |
| ISSUER_TIMEOUT (91) | Issuer unavailable | Retry logic | Queue for retry |

### Protocol-Specific Error Codes

#### **ACI Result Codes** (ds-aci module)

ACI processors return specific codes in response:

```
Version 87 Response Codes:
00 - Transaction Approved
01 - Refer to Issuer
05 - Declined
```

**Location**: Response code in Field 39 of 0210 message

#### **Abiy Error Codes** (ds-abiy module)

Abiy bank specific codes:

| Code | Meaning | Abiy Action |
|------|---------|-------------|
| 00 | Approved | Continue |
| 05 | Declined | Decline transaction |
| 30 | Format Error | Invalid message format |
| 40 | Function Not Supported | Unsupported txn type |
| 91 | Timeout | Retry with backoff |

#### **Ethswitch Result Codes**

**File**: `/modules/ds-ethswitch/src/main/resources/rc/ResultCode.properties`

```properties
00=Approved
01=Refer to issuer
02=Approved with ID
03=Invalid merchant
04=Pick up card
05=Do not honor
...
```

**Usage in Code**:
```java
// Load result codes
Properties resultCodes = new Properties();
resultCodes.load(new FileInputStream("ResultCode.properties"));

// Map response code to message
String approvalCode = resultCodes.getProperty(responseCode);
```

### Finding Error Codes in Logs

#### **Search for Error Code in Response**

```bash
# Find all responses with code 05 (declined)
grep -E "Field.*39|response.*code" logs/jpts.log | grep "05"

# Expected log entry:
# 2025-12-04 10:25:45,123 [Thread-1] DEBUG org.jpos.gw.Outgoing - Response code: 05
```

#### **Search for Transaction with Specific Error**

```bash
# Find transaction with error
grep "id=12345" logs/jpts.log | grep -E "ERROR|Exception|code.*05"

# Get full context
grep -B10 -A10 "id=12345.*ERROR" logs/jpts.log
```

#### **Search for Timeout Errors**

```bash
# Find timeout conditions (code 91)
grep -E "timeout|Timeout|91" logs/jpts.log

# Expected pattern:
# [Thread-1] ERROR ... Timeout waiting for response: 91 (Issuer Unavailable)
```

### Database Error Code Storage

#### **Query Error Codes from Database**

```sql
-- Find all transactions with errors
SELECT id, pan, response_code, created, ds
FROM tranlog
WHERE response_code NOT IN ('00', '02', '10', '50')
ORDER BY created DESC
LIMIT 20;

-- Result codes and counts
SELECT response_code, COUNT(*) as count
FROM tranlog
WHERE created > NOW() - INTERVAL '24 hours'
GROUP BY response_code
ORDER BY count DESC;

-- Response rate by destination station
SELECT ds, response_code, COUNT(*) as count
FROM tranlog
WHERE created > NOW() - INTERVAL '24 hours'
GROUP BY ds, response_code
ORDER BY ds, count DESC;
```

### Interpreting Error Codes in Context

#### **Example 1: Response Code 05 (Declined)**

**Log Entry**:
```
2025-12-04 10:25:45,456 [Thread-1] INFO org.jpos.gw.Outgoing - Response: 05 (Do not honor)
```

**Interpretation**:
- Card holder declined by issuer
- Possible causes:
  - Insufficient funds
  - Card limit exceeded
  - Account restrictions
  - Fraud detection

**Action**:
- Inform customer
- Suggest trying different card
- Check with card issuer

#### **Example 2: Response Code 91 (Issuer Timeout)**

**Log Entry**:
```
2025-12-04 10:26:00,789 [Thread-1] ERROR org.jpos.transaction.participant.SendHost - Timeout waiting for issuer: 91
```

**Interpretation**:
- No response from card issuer within timeout period
- Possible causes:
  - Network connectivity issue
  - Issuer system overload
  - Configuration timeout too short

**Action**:
- Retry transaction
- Check network connectivity
- Verify issuer status

#### **Example 3: Response Code 30 (Format Error)**

**Log Entry**:
```
2025-12-04 10:27:15,234 [Thread-1] ERROR org.jpos.gw.Incoming - Format validation failed: 30
```

**Interpretation**:
- Message structure/format invalid
- Possible causes:
  - Malformed ISO 8583 message
  - Missing mandatory fields
  - Invalid field values

**Action**:
- Review message structure
- Check field validation rules
- Verify message building code

### Error Code Troubleshooting Guide

#### **Common Error Scenarios**

**Scenario 1: Multiple Declined Transactions (05)**

```bash
# Check for pattern
grep "response.*code.*05" logs/jpts.log | wc -l

# If > 10% of transactions in last hour:
SELECT COUNT(*) as total_txns, 
       SUM(CASE WHEN response_code = '05' THEN 1 ELSE 0 END) as declined
FROM tranlog
WHERE created > NOW() - INTERVAL '1 hour';

# Action: Check with card issuer or processor
```

**Scenario 2: Timeout Errors (91)**

```bash
# Check timeout frequency
grep "91" logs/jpts.log | wc -l

# If increasing:
# 1. Check network connectivity
# 2. Verify issuer availability
# 3. Increase timeout in devel.properties:
#    jpts_cmf_timeout=60000  (from 30000)
```

**Scenario 3: Format Errors (30)**

```bash
# Check for message structure issues
grep "30" logs/jpts.log | grep -E "Field|format"

# Validate against message standards:
# - Check field 0 (MTI) format
# - Verify mandatory fields present
# - Validate field types and lengths
```

### Error Code Mapping Reference

#### **Quick Lookup Table**

| Symptom | Codes | Cause | Solution |
|---------|-------|-------|----------|
| Transaction declined | 05, 18, 44, 51 | Card or account issue | Contact issuer |
| Cannot connect | 15, 20, 91, 92 | Network/routing issue | Check connectivity |
| Invalid data | 12, 14, 27, 30 | Message format error | Validate fields |
| System error | 22, 96, 76 | Processing error | Retry or escalate |
| PIN issues | 17, 55, 66, 75 | PIN related | Request correct PIN |
| Account issues | 25, 39, 42, 43 | Account problem | Verify account |

### Setting Up Error Alerts

#### **Log-Based Alerts**

```bash
# Monitor for critical errors
while true; do
  error_count=$(grep -c "ERROR\|Exception" logs/jpts.log)
  if [ $error_count -gt 100 ]; then
    echo "Alert: High error count detected" | mail -s "jPTS Error Alert" admin@example.com
  fi
  sleep 300  # Check every 5 minutes
done
```

#### **Database-Based Alerts**

```sql
-- Create alert trigger for multiple failures
CREATE OR REPLACE FUNCTION check_error_threshold()
RETURNS VOID AS $$
DECLARE
  error_count INT;
BEGIN
  SELECT COUNT(*) INTO error_count
  FROM tranlog
  WHERE response_code NOT IN ('00', '02', '10', '50')
    AND created > NOW() - INTERVAL '1 hour';
  
  IF error_count > 50 THEN
    INSERT INTO system_alerts (type, message, severity)
    VALUES ('HIGH_ERROR_RATE', 'Over 50 errors in last hour', 'CRITICAL');
  END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## Summary: Error Codes

### Key Takeaways

1. **ISO 8583 Field 39** contains response codes (00-99)
2. **Codes 00-09**: Approval-related
3. **Codes 10-29**: Decline/error conditions
4. **Codes 30-99**: System/network errors
5. **CMF codes** are internal framework codes
6. **Protocol-specific codes** vary by destination (ACI, Abiy, etc.)

### Error Code Hierarchy

```
ISO 8583 Response (F39)
    ↓
CMF Internal Code
    ↓
Protocol-Specific Code (ACI/Abiy)
    ↓
Log Entry
    ↓
Database tranlog record
```

### Quick Reference Commands

```bash
# Find all errors
grep "ERROR\|05\|91" logs/jpts.log

# Count by code
grep -o "code [0-9][0-9]" logs/jpts.log | sort | uniq -c

# Search database
SELECT COUNT(*), response_code FROM tranlog GROUP BY response_code;
```

**Next Step**: Move to Task 5 (PIN Translation) for security understanding.
