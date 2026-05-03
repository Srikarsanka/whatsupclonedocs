# Message Encryption — How It Works

This document explains how messages are encrypted before saving to MongoDB and decrypted when fetched. This ensures that even if someone accesses your database directly, they cannot read the messages.

---

## Simple Explanation

```
User types: "Hello bro!"
        |
Backend ENCRYPTS it: "a3f2b1c4...:9e8d7c6b..."
        |
Saved in MongoDB as encrypted gibberish
        |
When fetched, backend DECRYPTS it back to: "Hello bro!"
        |
User sees: "Hello bro!"
```

The user never knows encryption is happening. It all happens automatically on the backend.

---

## What is AES-256-CBC?

**AES** = Advanced Encryption Standard (used by banks, governments, WhatsApp itself)

**256** = the key is 256 bits long (very hard to crack)

**CBC** = Cipher Block Chaining (a mode that makes encryption stronger by chaining blocks together)

Think of it like a lock:
- The **key** is your password (from `.env` file)
- The **IV** (Initialization Vector) is a random number that makes each encryption unique
- Even if you encrypt "Hello" twice, the result will be different each time because of the random IV

---

## The Two Functions

### encrypt(text) — Turns readable text into encrypted gibberish

```javascript
const crypto = require("crypto")

const encrypt = (text) => {
    // Step 1: Generate a random 16-byte IV (different every time)
    const iv = crypto.randomBytes(IV_LENGTH)

    // Step 2: Create an encryption tool using our secret key + IV
    const cipher = crypto.createCipheriv("aes-256-cbc", Buffer.from(ENCRYPTION_KEY), iv)

    // Step 3: Feed the text into the encryption tool
    let encrypted = cipher.update(text)
    encrypted = Buffer.concat([encrypted, cipher.final()])

    // Step 4: Return IV + encrypted text joined by ":"
    // We need to store the IV because we need it to decrypt later
    return iv.toString("hex") + ":" + encrypted.toString("hex")
}
```

**Example:**
```
Input:  "Hello bro!"
Output: "a3f2b1c4d5e6f7a8b9c0d1e2f3a4b5c6:9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b"
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
         This is the IV (random)            This is the encrypted message
```

---

### decrypt(text) — Turns encrypted gibberish back into readable text

```javascript
const decrypt = (text) => {
    // Step 1: Split the stored string by ":" to get IV and encrypted text
    const parts = text.split(":")
    const iv = Buffer.from(parts[0], "hex")
    const encryptedText = Buffer.from(parts[1], "hex")

    // Step 2: Create a decryption tool using the SAME key + SAME IV
    const decipher = crypto.createDecipheriv("aes-256-cbc", Buffer.from(ENCRYPTION_KEY), iv)

    // Step 3: Feed the encrypted text into the decryption tool
    let decrypted = decipher.update(encryptedText)
    decrypted = Buffer.concat([decrypted, decipher.final()])

    // Step 4: Return the original readable text
    return decrypted.toString()
}
```

**Example:**
```
Input:  "a3f2b1c4d5e6f7a8b9c0d1e2f3a4b5c6:9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b"
Output: "Hello bro!"
```

---

## Where is it Used?

### When SENDING a message (in sendMessage):

```javascript
// User types "Hello bro!"
const { message } = req.body   // message = "Hello bro!"

// Encrypt BEFORE saving to database
const encryptedMessage = encrypt(message)
// encryptedMessage = "a3f2....:9e8d..."

// Save the ENCRYPTED version to MongoDB
const newMessage = new Message({ sender, receiver, message: encryptedMessage })
await newMessage.save()

// But return the ORIGINAL text to the sender's screen
const responseData = { ...newMessage.toObject(), message: message }
return res.status(201).json({ message: "Message sent", data: responseData })
```

### When FETCHING messages (in getMessages):

```javascript
// Fetch encrypted messages from MongoDB
const messages = await Message.find({ ... })

// Decrypt EACH message before sending to frontend
const decryptedMessages = messages.map((msg) => {
    const msgObj = msg.toObject()
    try {
        msgObj.message = decrypt(msgObj.message)  // "a3f2...:9e8d..." -> "Hello bro!"
    } catch (e) {
        // If decryption fails (old message saved before encryption was added)
        // just keep the original text — don't crash
        msgObj.message = msgObj.message
    }
    return msgObj
})

return res.status(200).json(decryptedMessages)
```

---

## What Does MongoDB See?

Without encryption:
```json
{
    "sender": "664abc...",
    "receiver": "664def...",
    "message": "Hello bro!"
}
```

With encryption:
```json
{
    "sender": "664abc...",
    "receiver": "664def...",
    "message": "a3f2b1c4d5e6f7a8b9c0d1e2f3a4b5c6:9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b"
}
```

Even if someone hacks into your MongoDB, they see only gibberish. They cannot read the actual messages without your secret key.

---

## Where Does the Key Come From?

```javascript
const ENCRYPTION_KEY = process.env.scretkey
      .padEnd(32, '0')   // pad to 32 characters if too short
      .slice(0, 32)      // trim to exactly 32 characters (AES-256 needs 32 bytes)
```

It uses the same `scretkey` from your `.env` file that you already use for JWT tokens. No extra setup needed.

---

## Why is the IV Random Every Time?

If we used the same IV for every message, then encrypting "Hello" would always produce the same output. An attacker could notice patterns like:
- "Oh, this encrypted string appears 50 times, it's probably 'Hello'"

With a random IV, encrypting "Hello" produces a DIFFERENT encrypted string every single time. This is why we store the IV alongside the encrypted text (separated by ":").

---

## Summary

| Step | What Happens |
|---|---|
| User sends "Hello" | Frontend sends plain text to backend |
| Backend encrypts | `encrypt("Hello")` -> `"iv:encrypted_text"` |
| Saved to MongoDB | Stored as encrypted gibberish |
| User opens chat | Frontend requests messages |
| Backend decrypts | `decrypt("iv:encrypted_text")` -> `"Hello"` |
| User sees "Hello" | Frontend displays plain text |

No extra packages needed — uses Node.js built-in `crypto` module.
