# Messaging Logic — Complete Guide

This document explains the entire messaging system from backend to frontend — how messages are encrypted, stored, fetched, sent in real-time, and displayed on screen.

---

## Part 1: BACKEND (Server Side)

Everything that happens on the server before any data reaches the user's browser.

---

### 1.1 Message Schema — The Blueprint

Think of a schema like a form template. Every message saved to the database must follow this structure.

```javascript
const MessageSchema = new mongoose.Schema({
      sender: { type: mongoose.Schema.Types.ObjectId, ref: "Login", required: true },
      receiver: { type: mongoose.Schema.Types.ObjectId, ref: "Login", required: true },
      message: { type: String, required: true }
}, { timestamps: true })
```

| Field | What It Stores | Real-Life Example |
|---|---|---|
| `sender` | Who sent it | Like the "From" on an envelope |
| `receiver` | Who it's for | Like the "To" on an envelope |
| `message` | The actual text (encrypted) | The letter inside the sealed envelope |
| `createdAt` | When it was sent (auto-generated) | The postmark date on the envelope |

---

### 1.2 JWT Secret Key Generation — How We Created the Key

Before encryption can work, we need a secret key. We generated ours using Node.js built-in `crypto` module:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

**What this does line by line:**
- `require('crypto')` — Loads Node.js built-in security toolbox (no install needed)
- `.randomBytes(64)` — Generates 64 random bytes (like rolling 64 dice at once)
- `.toString('hex')` — Converts those random bytes into a readable string of letters and numbers

**Real-life example:** It's like a locksmith cutting a completely unique key that no one in the world has ever seen before. This key is then stored in the `.env` file as `scretkey`.

**This same key is used for TWO things:**
1. Signing JWT tokens (for login authentication)
2. Encrypting messages (for secure chat)

---

### 1.3 Message Encryption — Locking the Message

Before any message is saved to MongoDB, it gets encrypted using AES-256-CBC.

**Real-life example:** Imagine writing a letter, putting it in a box, locking it with a key, and then mailing the locked box. Only someone with the same key can open it.

```javascript
const crypto = require("crypto")
```
- `crypto` — Node.js built-in security toolbox. Like a safe manufacturer that comes pre-installed.

```javascript
const ENCRYPTION_KEY = process.env.scretkey.padEnd(32, '0').slice(0, 32)
```
- Takes your secret key from `.env` and makes it exactly 32 characters long (AES-256 requires exactly 32 bytes). Like cutting a key to fit a specific lock.

```javascript
const IV_LENGTH = 16
```
- IV (Initialization Vector) is 16 random bytes. Like adding a random ingredient to a recipe so the same dish tastes different every time.

#### The encrypt function:

```javascript
const encrypt = (text) => {
    const iv = crypto.randomBytes(IV_LENGTH)
```
- Generates 16 random bytes. Like shaking a dice 16 times. This ensures "Hello" encrypted now looks different from "Hello" encrypted 5 minutes ago.

```javascript
    const cipher = crypto.createCipheriv("aes-256-cbc", Buffer.from(ENCRYPTION_KEY), iv)
```
- Creates an encryption machine using your key + IV. Like loading a key into a lock.

```javascript
    let encrypted = cipher.update(text)
    encrypted = Buffer.concat([encrypted, cipher.final()])
```
- Feeds the message through the machine. What comes out is gibberish. Like putting a letter through a paper shredder, except this shredder can reverse itself.

```javascript
    return iv.toString("hex") + ":" + encrypted.toString("hex")
}
```
- Combines the IV and encrypted text with a `:` separator. We need to save the IV because it's the only way to decrypt later. Like taping the combination to the outside of the safe (but only you know how to read it).

#### The decrypt function:

```javascript
const decrypt = (text) => {
    const parts = text.split(":")
```
- Splits the stored string back into IV and encrypted text. Like separating the combination from the safe.

```javascript
    const iv = Buffer.from(parts[0], "hex")
    const encryptedText = Buffer.from(parts[1], "hex")
```
- Converts the hex strings back into usable bytes. Like translating a code back into numbers.

```javascript
    const decipher = crypto.createDecipheriv("aes-256-cbc", Buffer.from(ENCRYPTION_KEY), iv)
    let decrypted = decipher.update(encryptedText)
    decrypted = Buffer.concat([decrypted, decipher.final()])
    return decrypted.toString()
}
```
- Runs the encryption in reverse. The gibberish turns back into the original message. Like the shredder reassembling the letter.

---

### 1.4 Sending a Message — Backend Controller

When the frontend calls `POST /api/message/send/:receiverId`:

```javascript
const sendMessage = async (req, res) => {
    const { message } = req.body
```
- Grabs the message text from the request. Like reading what someone wrote.

```javascript
    const sender = req.user.id
```
- Gets YOUR user ID from the JWT token. Like checking who is making the phone call.

```javascript
    const receiver = req.params.receiverId
```
- Gets the receiver's ID from the URL. Like reading the phone number you dialed.

```javascript
    if (!message) {
        return res.status(400).json({ message: "Message cannot be empty" })
    }
```
- Blocks empty messages. Like a post office refusing to send a blank envelope.

```javascript
    const encryptedMessage = encrypt(message)
```
- Encrypts the message before saving. `"Hello bro"` becomes `"a3f2...:9e8d..."`. Like sealing the letter in a locked box.

```javascript
    const newMessage = new Message({ sender, receiver, message: encryptedMessage })
    await newMessage.save()
```
- Saves the ENCRYPTED version to MongoDB. The database never sees your actual message. Like the post office only handles locked boxes, never the letters inside.

```javascript
    const responseData = { ...newMessage.toObject(), message: message }
    return res.status(201).json({ message: "Message sent", data: responseData })
```
- Returns the ORIGINAL (unencrypted) message back to the sender's screen. Like giving you a copy of your letter before mailing the locked box.

---

### 1.5 Fetching Messages — Backend Controller

When the frontend calls `GET /api/message/:receiverId`:

```javascript
const messages = await Message.find({
    $or: [
        { sender: sender, receiver: receiver },
        { sender: receiver, receiver: sender }
    ]
}).sort({ createdAt: 1 })
```
- `$or` — Finds messages in BOTH directions (you to them AND them to you). Like pulling ALL letters between two pen pals from a mailbox.
- `.sort({ createdAt: 1 })` — Sorts oldest first. Like arranging letters by date.

```javascript
const decryptedMessages = messages.map((msg) => {
    const msgObj = msg.toObject()
    try {
        msgObj.message = decrypt(msgObj.message)
    } catch (e) {
        msgObj.message = msgObj.message
    }
    return msgObj
})
```
- Loops through every message and decrypts it before sending to the frontend.
- `try/catch` — If a message was saved before encryption was added, decryption will fail. The catch block keeps the original text instead of crashing. Like having a backup plan.

---

### 1.6 Socket.IO — Real-Time Delivery

```javascript
const io = new Server(server, { cors: { origin: "http://localhost:5173" } })
```
- Creates a WebSocket server. Like installing a walkie-talkie system in the building.

```javascript
socket.on("setup", (userId) => {
    socket.join(userId)
})
```
- When a user connects, they join a private room named after their ID. Like each person getting their own mailbox.

```javascript
socket.on("send_message", (data) => {
    socket.to(data.receiverId).emit("receive_message", data)
})
```
- When User A sends a message, the server pushes it directly to User B's private room. Like a mailman delivering a letter directly to someone's door instead of waiting for them to check their mailbox.

---

## Part 2: FRONTEND (Browser Side)

Everything that happens in the user's browser — how data is stored, displayed, and updated.

---

### 2.1 State Variables — Where Data Lives in the Browser

React stores all data in "state" variables. When state changes, the screen updates automatically.

```javascript
const [activeChat, setActiveChat] = useState(null)
```
- Which contact is currently open. Like which conversation tab is selected.

```javascript
const [messages, setMessages] = useState([])
```
- Array of all messages in the current chat. Like the chat history you see on screen.

```javascript
const [newMessage, setNewMessage] = useState('')
```
- What you're currently typing in the input box. Like a draft before hitting send.

```javascript
const [socket, setSocket] = useState(null)
```
- The WebSocket connection object. Like keeping the walkie-talkie turned on.

```javascript
const [unreadCounts, setUnreadCounts] = useState({})
```
- Tracks how many unread messages each contact has. Like the number badges on WhatsApp.

---

### 2.2 Loading Conversations on Page Open

```javascript
useEffect(() => {
    fetchConversations();
}, []);
```
- Runs ONCE when the page loads. Like opening WhatsApp and seeing your recent chats already there.

```javascript
const fetchConversations = async () => {
    const response = await axios.get(
        "http://localhost:3000/api/message/conversations/all",
        { withCredentials: true }
    );
    setSearchResults(response.data);
}
```
- Asks the backend: "Who have I chatted with before?" The backend checks all messages, finds unique users, and returns their details. The left contact panel gets populated.

---

### 2.3 Opening a Chat — Fetching History

```javascript
useEffect(() => {
    if (!activeChat) return;
    const fetchMessages = async () => {
        const response = await axios.get(
            `http://localhost:3000/api/message/${activeChat._id}`,
            { withCredentials: true }
        );
        setMessages(response.data);
    };
    fetchMessages();
}, [activeChat]);
```
- Runs every time you click a different contact.
- `activeChat` changes -> this effect fires -> fetches all messages between you and that person -> stores them in `messages` state -> screen updates automatically.
- Like tapping a contact name on WhatsApp and the chat history appearing.

---

### 2.4 Sending a Message — Frontend Flow

```javascript
const handlemessage = async (receiverId) => {
    if (!receiverId || !newMessage.trim()) return;
```
- Blocks empty messages. Like WhatsApp graying out the send button when the input is empty.

```javascript
    const response = await axios.post(
        `http://localhost:3000/api/message/send/${receiverId}`,
        { message: newMessage },
        { withCredentials: true }
    );
```
- Sends the message to the backend API. The backend encrypts it, saves to MongoDB, and returns the original text with a timestamp.

```javascript
    const messageData = {
        receiverId: activeChat._id,
        senderId: login_id,
        message: newMessage,
        createdAt: response.data.data.createdAt,
    };
    if (socket) {
        socket.emit('send_message', messageData);
    }
```
- Sends the same message through the WebSocket so the receiver gets it INSTANTLY without refreshing. Like shouting across the room instead of mailing a letter.

```javascript
    setMessages((prevMessages) => [
        ...prevMessages,
        { sender: login_id, message: newMessage, createdAt: response.data.data.createdAt }
    ]);
```
- Adds the message to YOUR OWN screen immediately. We don't wait for the socket to echo it back. Like seeing your sent message appear in the chat bubble right away.

```javascript
    setNewMessage("");
```
- Clears the input box. Like WhatsApp clearing the text field after you hit send.

---

### 2.5 Receiving a Message — Real-Time Listener

```javascript
newSocket.on("receive_message", (incomingMessage) => {
    const whoSentIt = incomingMessage.senderId;
    const currentChatId = document.getElementById("active-chat-id")?.dataset?.id;
```
- Listens for incoming messages from the socket.
- Reads the hidden DOM element to check who you're currently chatting with (workaround for React's stale closure problem).

**If you are chatting with the sender:**
```javascript
    if (currentChatId === whoSentIt) {
        setMessages((prev) => [...prev, incomingMessage]);
    }
```
- Adds the message directly to your chat window. You see it pop up instantly.

**If you are NOT chatting with the sender:**
```javascript
    else {
        setUnreadCounts((prev) => ({
            ...prev,
            [whoSentIt]: (prev[whoSentIt] || 0) + 1,
        }));
    }
```
- Increments the green badge number next to their name. Like the unread count bubble on WhatsApp.

---

### 2.6 Displaying Messages — The UI

```javascript
messages.map((msg, index) => {
    const messageClass = msg.sender === login_id ? "sent" : "received";
    const timeString = msg.createdAt
        ? new Date(msg.createdAt).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        : "";

    return (
        <div key={index} className={messageClass}>
            <span className="message-text">{msg.message}</span>
            <span className="message-time">{timeString}</span>
        </div>
    )
})
```

- `msg.sender === login_id` — If YOU sent it, apply `"sent"` class (green, right side). If THEY sent it, apply `"received"` class (dark, left side). Like how WhatsApp shows your messages on the right and theirs on the left.
- `toLocaleTimeString` — Converts `2026-05-02T17:21:21.959Z` into `10:51 PM`. Like formatting a raw timestamp into a human-readable time.

---

### 2.7 Auto-Scroll to Latest Message

```javascript
const messageEndRef = useRef(null);

useEffect(() => {
    messageEndRef.current?.scrollIntoView({ behavior: "smooth" });
}, [messages]);
```

- `useRef` points to an invisible `<div>` at the very bottom of the chat.
- Every time `messages` changes (new message sent or received), it scrolls to that invisible div.
- `behavior: "smooth"` — Smooth scroll instead of jumping. Like WhatsApp automatically scrolling down when a new message arrives.

---

## Complete Flow Diagram

```
YOU type "Hello bro!" and press Send
        |
   [FRONTEND]
        |
   1. axios.POST sends "Hello bro!" to backend
   2. socket.emit sends it via WebSocket
   3. setMessages adds it to YOUR screen
   4. Input box clears
        |
   [BACKEND]
        |
   5. encrypt("Hello bro!") -> "a3f2...:9e8d..."
   6. Save encrypted text to MongoDB
   7. Return original text + timestamp to sender
   8. Socket pushes message to receiver's room
        |
   [RECEIVER'S FRONTEND]
        |
   9. socket.on("receive_message") fires
   10. If chat is open -> message appears on screen
       If chat is closed -> green badge count + 1
        |
   [LATER - RECEIVER OPENS CHAT]
        |
   11. axios.GET fetches all messages
   12. Backend decrypts each one: "a3f2...:9e8d..." -> "Hello bro!"
   13. Messages display in chronological order
   14. Auto-scroll to bottom
```

---

## What MongoDB Sees vs What You See

| MongoDB (Database) | Your Screen |
|---|---|
| `"a3f2b1c4...:9e8d7c6b..."` | "Hello bro!" |
| `"7d9e3f2a...:1b4c8d5e..."` | "What's up?" |
| `"f1a2b3c4...:5d6e7f8a..."` | "Nothing much, you?" |

Even if someone hacks the database, they only see encrypted gibberish. The messages are unreadable without the secret key from your `.env` file.
