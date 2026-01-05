# AWS Payment Cryptography: Visa dCVV2 Generation & Verification Guide

This guide demonstrates how to successfully generate and verify Visa Dynamic Card Verification Value (dCVV2) using AWS Payment Cryptography. 

It addresses a critical configuration detail regarding the **TR-31 Key Usage** required for EMV dynamic validation, which is often a stumbling block for developers.

## 📋 Prerequisites

* **AWS CLI** installed and configured.
* Access to **AWS Payment Cryptography** service.
* Understanding of basic EMV concepts (PAN, ATC, PSN).

---

## 🚀 The Critical Discovery

The most challenging part of generating dCVV2 on AWS is selecting the correct Key Usage. After extensive testing, the confirmed working configuration is:

* **Key Usage:** `TR31_E6_EMV_MKEY_OTHER`
* **Key Algorithm:** `TDES_2KEY`
* **Key Mode:** `DeriveKey`

> **Note:** Even though dCVV2 is a "verification" value, the Master Key requires `DeriveKey` permission because the actual verification involves deriving a session key based on the ATC (Application Transaction Counter).

---

## 🛠️ Step-by-Step Implementation

### 1. Define Test Variables (Mock Data)

Set up the environment variables with standard Visa test vector data.

```bash
export MOCK_PAN="4012345678901234"
export MOCK_EXP="2512"       # YYMM
export MOCK_ATC="0001"       # Application Transaction Counter
export MOCK_PSN="01"         # PAN Sequence Number
export MOCK_SERVICE_CODE="101"
```

### 2. Create the Master Key (The "Secret" Step)

Create a Symmetric Key with the specific `TR31_E6` usage. This key acts as the Issuer Master Key (IMK) used to derive chip-specific session keys.

```bash
export KEY_ARN=$(aws payment-cryptography create-key \
  --exportable \
  --key-attributes '{
    "KeyUsage": "TR31_E6_EMV_MKEY_OTHER",
    "KeyAlgorithm": "TDES_2KEY",
    "KeyModesOfUse": { "DeriveKey": true },
    "KeyClass": "SYMMETRIC_KEY"
  }' \
  --tags 'Key=Name,Value=VisaTest-E6-dCVV2' \
  --query 'Key.KeyArn' --output text)

echo "Created Key ARN: $KEY_ARN"
```

### 3. Generate dCVV2

Use the `generate-card-validation-data` API. Note that we pass the EMV parameters (ATC, PSN) into the `DynamicCardVerificationValue` object.

```bash
aws payment-cryptography-data generate-card-validation-data \
    --key-identifier $KEY_ARN \
    --primary-account-number $MOCK_PAN \
    --generation-attributes "DynamicCardVerificationValue={CardExpiryDate=$MOCK_EXP,PanSequenceNumber=$MOCK_PSN,ApplicationTransactionCounter=$MOCK_ATC,ServiceCode=$MOCK_SERVICE_CODE}"
```

**Output:**
```json
{
    "KeyArn": "arn:aws:payment-cryptography:us-west-2:128761886083:key/3ngjp7jwutndasts",
    "KeyCheckValue": "85777B",
    "ValidationData": "221"
}
```
*The generated dCVV2 value is **221**.*

### 4. Verify dCVV2 (Success Scenario)

Now, simulate a verification request (e.g., during a transaction authorization). We use the dCVV2 value generated in the previous step (`221`).

```bash
# Capture the result from previous step
export DCVV2_VALUE="221"

aws payment-cryptography-data verify-card-validation-data \
    --key-identifier $KEY_ARN \
    --primary-account-number $MOCK_PAN \
    --validation-data $DCVV2_VALUE \
    --verification-attributes "DynamicCardVerificationValue={CardExpiryDate=$MOCK_EXP,PanSequenceNumber=$MOCK_PSN,ApplicationTransactionCounter=$MOCK_ATC,ServiceCode=$MOCK_SERVICE_CODE}"
```

**Output:**
```json
{
    "KeyArn": "arn:aws:payment-cryptography:us-west-2:128761886083:key/3ngjp7jwutndasts",
    "KeyCheckValue": "85777B"
}
```
*A successful response (HTTP 200 with JSON body) indicates the verification passed.*

### 5. Verify dCVV2 (Failure Scenario)

To ensure the validation logic is actually checking the dynamic data, we purposely provide an incorrect ATC (`0009` instead of `0001`).

```bash
# Check current ATC variable
echo $MOCK_ATC
# Output: 0001

# Run verification with WRONG ATC (0009)
aws payment-cryptography-data verify-card-validation-data \
    --key-identifier $KEY_ARN \
    --primary-account-number $MOCK_PAN \
    --validation-data $DCVV2_VALUE \
    --verification-attributes "DynamicCardVerificationValue={CardExpiryDate=$MOCK_EXP,PanSequenceNumber=$MOCK_PSN,ApplicationTransactionCounter=0009,ServiceCode=$MOCK_SERVICE_CODE}"
```

**Output:**
```text
An error occurred (VerificationFailedException) when calling the VerifyCardValidationData operation: Card validation data verification failed.
```
*The system correctly rejected the validation due to the ATC mismatch, confirming the dynamic check is active.*

---

## 🎯 Summary

| Parameter | Value |
| :--- | :--- |
| **Algorithm** | TDES_2KEY |
| **Key Usage** | **TR31_E6_EMV_MKEY_OTHER** |
| **Mode** | DeriveKey |
| **Validation Type** | DynamicCardVerificationValue (dCVV2) |
