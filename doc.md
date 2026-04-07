# Sistegra Payment Partner Reference

Dokumen integrasi partner untuk modul `sistegra-payment` :

- create payment link
- inquiry payment
- security request partner
- callback dari Sistegra ke partner

## Base URL

Gunakan base URL sesuai environment:

- Stage: `https://payment-stage.tachyon.net.id/api`

## Security Request Partner

Endpoint partner-facing `sistegra-payment` menggunakan:

- body field `request_time`
- header `x-signature`

Shared secret:

- gunakan shared secret yang sama antara partner dan mid

### Format `request_time`

- wajib ISO 8601 UTC
- wajib berakhiran `Z`
- contoh:

```text
2026-04-07T05:50:00.000Z
```

- toleransi waktu default: 5 menit

### Header Wajib

```http
Content-Type: application/json
x-signature: <hmac-sha256-hex>
```

## 1. Create Payment Link

### Endpoint

- Method: `POST`
- Path: `/sistegra-payment/generate-link`

### String To Sign

```text
external_id#order_id#request_time
```

### Request Body

```json
{
  "request_time": "2026-04-07T05:50:00.000Z",
  "external_id": "abcbd3973498c4-e91bcd910904mp",
  "order_id": "ORDER904p",
  "transaction_description": "this is order description VA",
  "invoice_details": {
    "idp": "DOX-000000003",
    "discount": 0,
    "package_id": 2490,
    "main_product": true,
    "partner_reference_code": "test 123"
  },
  "expired_time": 1440,
  "callback_url": "https://partner.example.com/api/payment/callback",
  "failed_redirect_url": "https://partner.example.com/payment/failed",
  "success_redirect_url": "https://partner.example.com/payment/success"
}
```

### Request Header

```http
Content-Type: application/json
x-signature: <hmac-sha256-hex>
```

### Success Response

```json
{
  "payment_id": "CPL_tw8ah2vhMSagUoXFbVzu",
  "order_id": "ORDER904p",
  "payment_url": "https://example.com/payment/CPL_tw8ah2vhMSagUoXFbVzu",
  "amount": 28000,
  "status": "PENDING",
  "expired_at": "2026-04-08T05:51:26.655Z",
  "success_redirect_url": "https://partner.example.com/payment/success",
  "failed_redirect_url": "https://partner.example.com/payment/failed",
  "channels": []
}
```

## 2. Inquiry Payment

### Endpoint

- Method: `POST`
- Path: `/sistegra-payment/inquiry`

### String To Sign

```text
payment_id#request_time
```

### Request Body

```json
{
  "payment_id": "CPL_2i56yGjMZxgvsHBu5Wuk",
  "request_time": "2026-04-07T05:55:00.000Z"
}
```

### Request Header

```http
Content-Type: application/json
x-signature: <hmac-sha256-hex>
```

### Success Response Gateway

```json
{
  "order_id": "ORDER904n",
  "transaction_id": "019d628e-034a-7a2c-aa5b-9d42ed7ae82b",
  "reference_id": "269137_ORDER904n",
  "response_code": "00",
  "currency": "IDR",
  "payment_details": {
    "amount": 28283,
    "total_amount": 28283,
    "expired_time": "2026-04-07T18:44:28.231+07:00",
    "transaction_time": "2026-04-07T18:29:27.648+07:00",
    "transaction_description": "this is order description VA"
  },
  "payment_method": "WALLET",
  "payment_channel": "NOBU",
  "external_id": "253573ee-d31f-4c58-8e48-7aebf900fde8",
  "transaction_status": "ACTIVE",
  "customer_details": {
    "phone": "081234567890",
    "email": "dodoxx@gmail.com",
    "full_name": "Test customer Guard Lagi"
  },
  "callback_url": "https://example.com/api/sistegra-payment/callback",
  "wallet_details": {
    "id": "081234567890",
    "id_type": "HP"
  },
  "qr_response": {
    "qr_string": "..."
  },
  "payment_id": "CPL_nEUqqNPk4PFcRmMNCJVu",
  "invoice": null,
  "channel_fee": 283
}
```

## 3. Callback dari Middleware ke Partner

Callback partner menggunakan:

### Callback Header

```http
Content-Type: application/json
x-signature: <hmac-sha256-hex>
```

### Callback String To Sign

```text
payment_id#order_id#request_time
```

### Callback Payload Example

```json
{
  "transaction_id": "019d626c-a12f-7ce3-9695-b2a251cc0cef",
  "external_id": "72b0af03-a069-4b43-ae36-717fbc64f08a",
  "reference_id": "269136_ORDER904m",
  "order_id": "ORDER904m",
  "merchant_id": "IFP2024063173",
  "response_code": "00",
  "response_message": "SUCCESS",
  "currency": "IDR",
  "payment_method": "WALLET",
  "payment_channel": "NOBU",
  "payment_system": "QR",
  "transaction_status": "PAID",
  "payment_details": {
    "amount": 28283,
    "expired_time": "2026-04-07T18:08:00.366+07:00",
    "transaction_description": "this is order description VA",
    "transaction_time": "2026-04-07T17:52:59.783+07:00",
    "total_amount": 28283,
    "paid_time": "2026-04-07T17:53:45.850+07:00"
  },
  "customer_details": {
    "phone": "081234567890",
    "email": "dodoxx@gmail.com",
    "full_name": "Test customer Guard Lagi"
  },
  "issuer": "NOBU",
  "wallet_details": {
    "id": "081234567890",
    "id_type": "HP"
  },
  "acquirer_issuer_relation": "on_us",
  "payment_id": "CPL_NmvNkYCYvFQpUZKGjYW9",
  "invoice": "INVRA01/2026/00200",
  "channel_fee": 283,
  "request_time": "2026-04-07T05:50:00.000Z"
}
```

### Verifikasi Callback di Sisi Partner

Partner harus:

1. ambil `payment_id`, `order_id`, `request_time` dari body
2. susun string:

```text
payment_id#order_id#request_time
```

3. generate HMAC SHA256 dengan shared secret yang sama
4. bandingkan dengan header `x-signature`

Contoh pseudocode:

```javascript
const crypto = require('crypto');

const payload = `${body.payment_id}#${body.order_id}#${body.request_time}`;
const expectedSignature = crypto
  .createHmac('sha256', sharedSecret)
  .update(payload, 'utf8')
  .digest('hex');

const isValid = expectedSignature === headers['x-signature'];
```

## 4. Error Response Security

Jika signature partner request tidak valid, endpoint partner-facing akan merespons:

```json
{
  "statusCode": 401,
  "message": "Invalid signature"
}
```

## 5. Ringkasan Kontrak Signature

### Generate Link

```text
external_id#order_id#request_time
```

### Inquiry

```text
payment_id#request_time
```

### Callback ke Partner

```text
payment_id#order_id#request_time
```
