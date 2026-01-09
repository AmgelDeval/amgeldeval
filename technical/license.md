# License Generation & Usage Guide

This document explains **how to generate a license file from scratch** and use it with `config/license.json`.

---

## Overview

The licensing mechanism uses **asymmetric cryptography (RSA)**.

* The **private key** is used to **generate (sign)** licenses
* The **public key** is embedded in the application to **verify** licenses
* The application **never** contains the private key

---

## License File Format

The license file is a JSON file stored at:

```
config/license.json
```

### Example

```json
{
  "organization": "KSVTBALKUR",
  "hostname": "Sudarshans-MacBook-Air.local",
  "validTill": "2027-03-31",
  "signature": "BASE64_SIGNATURE_HERE"
}
```

---

## Step 1: Generate RSA Keys (One Time)

### Generate Private Key

```bash
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048
```

This creates:

```
private.pem
```

> ⚠️ This is the **PRIVATE KEY**
> Keep it secure. Do not share or commit it.

---

### Generate Public Key

```bash
openssl rsa -pubout -in private.pem -out public.pem
```

This creates:

```
public.pem
```

* `private.pem` → used only for license generation
* `public.pem` → embedded in the application

---

## Step 2: Decide License Values

You must decide the following values:

| Field        | Description                        |
| ------------ | ---------------------------------- |
| organization | Customer / temple identifier       |
| hostname     | Exact system hostname              |
| validTill    | License expiry date (`YYYY-MM-DD`) |

### Example values

```
organization = KSVTBALKUR
hostname     = Sudarshans-MacBook-Air.local
validTill    = 2027-03-31
```

---

## Step 3: Create the Data to Be Signed

The **exact string format** to sign is:

```
organization|hostname|validTill
```

### Example

```
KSVTBALKUR|Sudarshans-MacBook-Air.local|2027-03-31
```

⚠️ **Important**

* Order matters
* No spaces
* No extra newlines
* Case-sensitive

---

## Step 4: Generate the Signature

### Create data file

```bash
echo -n "KSVTBALKUR|Sudarshans-MacBook-Air.local|2027-03-31" > data.txt
```

### Sign using private key

```bash
openssl dgst -sha256 -sign private.pem data.txt | base64
```

### Output

```
MEQCIG9H5z8DqJ2Z9...
```

This output is the **signature**.

---

## Step 5: Create `config/license.json`

Insert the generated signature into the license file.

```json
{
  "organization": "KSVTBALKUR",
  "hostname": "Sudarshans-MacBook-Air.local",
  "validTill": "2027-03-31",
  "signature": "MEQCIG9H5z8DqJ2Z9..."
}
```

Place the file at:

```
config/license.json
```

(relative to application startup directory)

---

## How the Application Uses This File

At startup, the application:

1. Loads `config/license.json`
2. Reads the current system hostname
3. Verifies the license signature using the embedded public key
4. Validates expiry date
5. Starts only if all checks pass

---

## Security Rules (Must Follow)

### DO NOT

* Put `private.pem` inside the application
* Commit private key to Git
* Share private key with customers
* Store private key on servers

### DO

* Keep private key offline
* Embed only the public key in code
* Regenerate license when hostname changes

---

## Mental Model

| Item         | Owner       | Purpose           |
| ------------ | ----------- | ----------------- |
| Private key  | Vendor      | Create licenses   |
| Public key   | Application | Verify licenses   |
| License file | Customer    | Proof of validity |

---

## Summary

1. Generate RSA key pair (`private.pem`, `public.pem`)
2. Decide license values (organization, hostname, expiry)
3. Sign `organization|hostname|validTill` using `private.pem`
4. Create `config/license.json`
5. Application verifies license at startup

---
