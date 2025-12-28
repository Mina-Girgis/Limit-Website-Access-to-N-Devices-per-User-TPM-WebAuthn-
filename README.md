# Limit Website Access to N Devices per User (TPM / WebAuthn)

## Overview

This project demonstrates how to limit user access to a fixed number of devices (N) using **TPM (Trusted Platform Module)** through **WebAuthn**.

### Key Features

- A user can log in from only **N devices**
- The same device can be used by **multiple users**
- Keys are stored securely in **hardware**
- No unreliable identifiers like MAC or IP addresses

---

## Problem Statement

**Business Requirement:** Each user should be allowed to access the website from only N devices.

### Why MAC Address Does Not Work

Using MAC addresses may sound reasonable, but in reality it does not work:

- Operating systems **randomize MAC addresses**
- MAC addresses **change across network hops**
- Browsers **do not expose MAC addresses**
- Servers **never see the real client MAC**

**Conclusion:** MAC addresses cannot be used to identify devices reliably.

---

## The TPM / WebAuthn Approach

### What is TPM?

A **Trusted Platform Module (TPM)** is a hardware chip that:

- Generates cryptographic key pairs
- Keeps private keys inside hardware
- Prevents key extraction
- Performs signing securely

### Important Rule

**You never read or export the private key.** You only ask the TPM to sign data.

### Core Idea

- Each user gets a **separate key per device**
- The **private key stays inside the TPM**
- The backend stores:
  - Public key
  - Credential ID
- The backend **enforces the device limit (N)**

---

## How It Works (High Level)

```
Browser → TPM → Backend
```

1. Browser requests authentication
2. TPM signs a challenge
3. Backend verifies the signature using the stored public key

### Multiple Users on One Device

This is **fully supported**:

- One device can store multiple TPM keys
- Each user has their own credential
- No conflict between users

---

## Database Structure

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL
);

CREATE TABLE user_device_keys (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT REFERENCES users(id),
    credential_id TEXT NOT NULL,
    public_key TEXT NOT NULL,
    device_info TEXT,
    registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Authentication Flow

### 1. Device Registration

- Browser creates a credential using WebAuthn
- TPM generates a key pair
- Public key and credential ID are sent to the backend
- Backend checks device count before saving

### 2. Login Challenge

**Request:**

```http
GET /login-challenge?email=user@example.com
```

**Response:**

```json
{
  "challenge": "base64Challenge",
  "allowCredentials": [
    { "id": "base64CredentialId1", "type": "public-key" },
    { "id": "base64CredentialId2", "type": "public-key" }
  ]
}
```

The backend sends only the credentials that belong to this user.

### 3. Signing the Challenge

```javascript
const assertion = await navigator.credentials.get({
  publicKey: {
    challenge: Uint8Array.from(atob(challenge), c => c.charCodeAt(0)),
    allowCredentials: allowCredentials.map(c => ({
      id: Uint8Array.from(atob(c.id), ch => ch.charCodeAt(0)),
      type: "public-key"
    }))
  }
});
```

The TPM automatically uses the correct private key.

---

## Credential ID Explained

### What is `credentialId`?

- A reference to the private key inside the TPM
- **Not secret**
- Safe to store in the database
- Used to find the correct public key

### How It Is Created

```javascript
const credentialId = btoa(
  String.fromCharCode(...new Uint8Array(assertion.rawId))
);
```

**Example:**

Input bytes: `[1, 2, 3, 255, 0, 128]`  
Encoded result: `AQID/wCA`

This value is stored as `credential_id`.

---

## Login Request

**Endpoint:**

```http
POST /login
```

**Request:**

```json
{
  "email": "user@example.com",
  "credentialId": "AQID/wCA",
  "challenge": "base64Challenge",
  "signedChallenge": "base64Signature"
}
```

**Backend Process:**

1. Finds the public key by `credentialId`
2. Verifies the signature
3. Logs the user in

---

## What This Solves

✅ Private keys never leave the device  
✅ Devices cannot be cloned  
✅ Multiple users per device are allowed  
✅ Device limits are enforced reliably  
✅ No dependence on MAC, IP, or fingerprints

---

## Summary

This approach uses standard WebAuthn + TPM behavior to safely limit the number of devices per user while keeping the system secure and user-friendly.
