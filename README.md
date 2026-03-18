# Paymob Payment Initialization — IBM ACE 12

A production-ready message flow built with IBM App Connect Enterprise 12 that handles the full payment initialization process with Paymob payment gateway.

---

## What It Does

The client sends one request → ACE handles everything → returns a ready-to-use payment iFrame URL.

```
Client
  │
  │  POST /payment/initialize
  ▼
ACE (PaymentInit.msgflow)
  │
  ├── 1. Validate all input fields
  ├── 2. Load user + address from SQL Server
  ├── 3. Call Paymob: Get auth token
  ├── 4. Call Paymob: Create order
  ├── 5. Call Paymob: Generate payment key
  ├── 6. Build iFrame URL
  ├── 7. Save order + transaction to DB
  │
  ▼
Return iFrame URL to client
```

---

## Flow Nodes

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | HTTPInput | HTTPInput | Receive POST /payment/initialize |
| 2 | validateInput | Compute | Validate all request fields |
| 3 | LoadUserData | Compute | Load user + address from SQL Server |
| 4 | buildAuthRequest | Compute | Build Paymob auth request body |
| 5 | CallPaymobAuth | HTTPRequest | POST /auth/tokens |
| 6 | ParseAuthResponse | Compute | Extract auth token |
| 7 | BuildOrderRequest | Compute | Build Paymob order request body |
| 8 | CallPaymobOrder | HTTPRequest | POST /ecommerce/orders |
| 9 | ParseOrderResponse | Compute | Extract Paymob order ID |
| 10 | BuildPaymentKeyRequest | Compute | Build payment key request body |
| 11 | CallPaymobPayKey | HTTPRequest | POST /acceptance/payment_keys |
| 12 | ParsePaymentKeyResponse | Compute | Extract payment key |
| 13 | BuildIframeUrl | Compute | Construct iFrame URL |
| 14 | SaveToDB | Compute | Insert order + transaction to DB |
| 15 | BuildSuccessResponse | Compute | Build success JSON response |
| 16 | ErrorHandler | Compute | Handle and log all errors |
| 17 | HTTPReply | HTTPReply | Return response to client |

---

## Request

```
POST http://localhost:1234/payment/initialize
Content-Type: application/json
```

```json
{
  "userId": 1,
  "addressId": 1,
  "amountCents": 10000,
  "currency": "EGP",
  "items": [
    {
      "name": "Product Name",
      "amountCents": 10000,
      "description": "Product description",
      "quantity": 1
    }
  ]
}
```

### Field Rules

| Field | Rule |
|---|---|
| `userId` | Required — must exist in database |
| `addressId` | Required — must belong to this user |
| `amountCents` | Required — amount × 100 (e.g. 100 EGP = 10000) |
| `currency` | Required — 3 characters (e.g. EGP) |
| `items` | Required — at least 1 item |

---

## Response

### Success

```json
{
  "success": true,
  "orderId": 3,
  "paymobOrderId": "488850328",
  "merchantOrderRef": "ORD-1-20260318090751",
  "iframeUrl": "https://accept.paymob.com/api/acceptance/iframes/IFRAME_ID?payment_token=...",
  "amountCents": 10000,
  "currency": "EGP"
}
```

### Error

```json
{
  "success": false,
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user found with the provided userId",
    "stage": "LoadUserData"
  }
}
```

---

## Error Codes

| Code | Meaning |
|---|---|
| `INVALID_USER_ID` | userId is missing or not positive |
| `INVALID_ADDRESS_ID` | addressId is missing or not positive |
| `INVALID_AMOUNT` | amountCents is missing or zero |
| `INVALID_CURRENCY` | currency is missing or not 3 characters |
| `INVALID_ITEMS` | items array is missing or empty |
| `USER_NOT_FOUND` | userId does not exist in database |
| `ADDRESS_NOT_FOUND` | addressId does not belong to this user |
| `CONFIG_ERROR` | Paymob credentials are not configured |
| `AUTH_TOKEN_FAILED` | Could not get auth token from Paymob |
| `CREATE_ORDER_FAILED` | Could not create order on Paymob |
| `PAYMENT_KEY_FAILED` | Could not generate payment key |
| `MAX_RETRIES_EXCEEDED` | All retry attempts failed |

---

## Prerequisites

- IBM ACE 12.x
- SQL Server with ODBC DSN configured
- Paymob sandbox or production account

---

## Database Tables Used

| Table | Usage |
|---|---|
| `users` | Load user first/last name, email, phone |
| `addresses` | Load billing address for payment key |
| `orders` | Insert new order record |
| `transactions` | Insert transaction with payment key |
| `errorLogs` | Log any errors that occur |

---

## Configuration

### 1. ODBC

Create a System DSN named `userData` pointing to your SQL Server, then run:

```bash
mqsisetdbparms -n userData -u YOUR_USERNAME -p YOUR_PASSWORD
```

Set `userData` as the data source on each Compute node that queries the database.

### 2. Policy

Fill in `PaymobIntegrationPolicy/Policy/PaymobConfig.policyxml`:

```xml
<paymobBaseUrl>https://accept.paymob.com/api</paymobBaseUrl>
<paymobApiKey>YOUR_API_KEY</paymobApiKey>
<paymobIntegrationId>YOUR_INTEGRATION_ID</paymobIntegrationId>
<paymobIframeId>YOUR_IFRAME_ID</paymobIframeId>
<paymobHmacSecret>YOUR_HMAC_SECRET</paymobHmacSecret>
```

Get these from your Paymob Dashboard:

| Value | Location |
|---|---|
| `paymobApiKey` | Settings → Account Info → API Key |
| `paymobIntegrationId` | Developers → Payment Integrations → ID |
| `paymobIframeId` | Developers → iFrames → ID |
| `paymobHmacSecret` | Settings → Account Info → HMAC |

### 3. SSL Certificate

Paymob uses HTTPS. Import the certificate chain into ACE truststore:

```powershell
# Run in PowerShell as Administrator
$tcpClient = New-Object System.Net.Sockets.TcpClient("accept.paymob.com", 443)
$sslStream = New-Object System.Net.Security.SslStream($tcpClient.GetStream(), $false, {$true})
$sslStream.AuthenticateAsClient("accept.paymob.com")
$chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
$chain.Build($sslStream.RemoteCertificate)
$keytool = "C:\Program Files\IBM\ACE\12.0.12.19\common\jdk\bin\keytool.exe"
$i = 0
foreach ($element in $chain.ChainElements) {
    $bytes = $element.Certificate.Export([Security.Cryptography.X509Certificates.X509ContentType]::Cert)
    [IO.File]::WriteAllBytes("C:\paymob_cert_$i.cer", $bytes)
    & $keytool -import -alias "paymob_$i" -file "C:\paymob_cert_$i.cer" `
               -keystore "C:\paymob_trust.jks" -storepass paymob123 -noprompt
    $i++
}
$sslStream.Close(); $tcpClient.Close()
```

Then add to your integration server's `server.conf.yaml`:

```yaml
ResourceManagers:
  JVM:
    truststoreType: 'JKS'
    truststoreFile: 'C:/paymob_trust.jks'
    truststorePass: 'paymob123'
```

---

## Retry Mechanism

Each of the 3 Paymob HTTP calls has automatic retry with exponential backoff:

```
Attempt 1 fails → wait 1 second  → retry
Attempt 2 fails → wait 2 seconds → retry
Attempt 3 fails → wait 3 seconds → error
```

---

## How to Deploy

```
1. Open ACE Toolkit
2. Right-click PaymobIntegrationApp → Deploy
3. Select your Integration Server
4. Click Finish
5. Verify: mqsilist YOUR_NODE -r
```

---

## How to Test

Use Postman or curl:

```bash
curl -X POST http://localhost:1234/payment/initialize \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "addressId": 1,
    "amountCents": 10000,
    "currency": "EGP",
    "items": [{"name":"Test","amountCents":10000,"description":"Test","quantity":1}]
  }'
```

Open the `iframeUrl` from the response in a browser to see the Paymob payment form.



## Tech Stack

```
IBM ACE 12    — Integration platform
ESQL          — Business logic in Compute nodes
Java          — PolicyReader JavaCompute node
SQL Server    — Database via ODBC
Paymob API    — Payment gateway (REST)
SSL/TLS       — Secure HTTPS communication
```
