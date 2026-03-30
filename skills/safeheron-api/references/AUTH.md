# Safeheron API Authentication

## Request Encryption & Signing (Step-by-Step)

### Step 1: Prepare bizContent
Serialize your business parameters to a JSON string.

### Step 2: Generate AES key
Randomly generate:
- **32 bytes** AES key
- **16 bytes** IV (initialization vector)

### Step 3: Encrypt bizContent
```
Algorithm: AES/GCM/NoPadding
Output: base64(AES_GCM_Encrypt(jsonString, key, iv))  → bizContent field
```

### Step 4: Encrypt the AES key
```
Algorithm: RSA/ECB/OAEPWithSHA-256AndMGF1Padding
Input: 48 bytes (AES key + IV), encrypted with Safeheron's RSA Public Key
Output: base64(RSA_Encrypt(aesKey + iv))  → key field
```

### Step 5: Build signature string
Sort all parameters by key ascending (exclude `rsaType` and `aesType`):
```
apiKey=...&bizContent=...&key=...&timestamp=...
```

### Step 6: Sign
```
Algorithm: SHA256WithRSA
Input: Signature string, signed with YOUR RSA Private Key
Output: base64(RSA_Sign(sigString))  → sig field
```

### Final Request Body
```json
{
  "apiKey": "YOUR_API_KEY",
  "timestamp": "1623038312088",
  "bizContent": "<aes-encrypted-base64>",
  "key": "<rsa-encrypted-aes-key-base64>",
  "sig": "<rsa-signature-base64>",
  "rsaType": "ECB_OAEP",
  "aesType": "GCM_NOPADDING"
}
```

---

## Response Verification & Decryption

### Step 1: Build response signature string
Sort by key ascending (exclude `rsaType`, `aesType`):
```
bizContent=...&code=...&key=...&message=...&timestamp=...
```

### Step 2: Verify signature
```
Algorithm: SHA256WithRSA — verify with Safeheron's RSA Public Key
```
If verification fails, reject the response.

### Step 3: Decrypt the AES key
```
Algorithm: RSA/ECB/OAEPWithSHA-256AndMGF1Padding
Decrypt key field using YOUR RSA Private Key → 48-byte AES key+IV
```

### Step 4: Decrypt bizContent
```
Algorithm: AES/GCM/NoPadding
→ Yields plaintext JSON response
```

> All four SDKs (Java, JS/TS, Go, Python) automatically handle request signing, payload encryption, and response signature verification — these steps are built-in and non-optional. Manual implementation is only needed if you are calling the API directly without an SDK.

---

## OpenSSL Key Generation
```bash
# Generate RSA 4096-bit private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Export public key (upload to Safeheron Console)
openssl rsa -in api_private.pem -out api_public.pem -pubout

# Convert to PKCS8 (use base64 content as rsaPrivateKey in SafeheronConfig builder)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
```

