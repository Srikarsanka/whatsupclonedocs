# 💬 Message Sending & Receiving Logic

This document breaks down exactly how messages are sent, received, stored, and displayed in real-time, including how the "Active Chat" logic determines what you see on your screen.

---

## 1️⃣ Opening a Chat (The `activeChat` Logic)

When you click on a contact in the left panel, several things happen:

1. **Set Active Chat**: The state `setActiveChat(user)` is triggered. The app now knows exactly who you are trying to talk to.
2. **Reset Unread Badge**: If that user had unread messages (the green circle), clicking their name resets `setUnreadCounts` for their ID back to `0`.
3. **Fetch History**: The `useEffect` listening to `[activeChat]` triggers.
   - It makes an Axios `GET` request to `/api/message/:receiverId`.
   - The backend looks up all messages in MongoDB where you are the sender and they are the receiver, OR they are the sender and you are the receiver.
   - It sorts them by time and sends them back.
4. **Display**: The `messages` state is updated, and the chat window instantly populates with your history.

---

## 2️⃣ Sending a Message

When you type a message and press the Send button, the `handlemessage(activeChat._id)` function fires.

### Step-by-Step Flow:
1. **Save to Database**: 
   - We immediately send an Axios `POST` request to `/api/message/send/:receiverId` containing the message text.
   - The backend saves this in the MongoDB `Message` collection and returns the created message object (which includes a MongoDB generated `createdAt` timestamp).
2. **Emit to Socket**:
   - We package the message data (including the exact `createdAt` timestamp from the database) and send it over the real-time WebSocket connection: `socket.emit('send_message', messageData)`.
3. **Update Local Screen**:
   - We do NOT wait for the socket to echo the message back to us.
   - Instead, we manually append our new message directly to our own `messages` array state so it appears on our screen instantly.
4. **Clear Input**: The text box is cleared.

---

## 3️⃣ Receiving a Message (The Real-Time Magic)

When your friend is online and sends you a message, the Socket.IO server pushes that message to your browser instantly.

The listener is set up in `ChatInterface.jsx`:
```javascript
newSocket.on("receive_message", (incomingMessage) => { ... })
```

### The "Stale Closure" Problem & Solution
When `newSocket.on` is created on page load, it "freezes" the variables it knows about. It doesn't actually know if you change your `activeChat` later! To fix this, we put a hidden HTML element on the page that always holds the real-time ID of the open chat:
`<div id="active-chat-id" data-id={activeChat?._id || ""} />`

### The Receiving Logic:
When a message arrives over the socket, the listener reads the sender's ID (`whoSentIt = incomingMessage.senderId`).

**Scenario A: You are already chatting with them!**
- The hidden HTML element's ID matches the `whoSentIt` ID.
- We append the incoming message directly into our `messages` array.
- The new message bubble pops up on screen instantly.

**Scenario B: You are talking to someone else (or looking at the empty screen)**
- The hidden HTML element does NOT match the sender.
- We do NOT put the message in the main chat window (because that would mix up conversations!).
- Instead, we update the `unreadCounts` state.
- `setUnreadCounts(prev => ({ ...prev, [whoSentIt]: prev[whoSentIt] + 1 }))`
- This makes the green badge appear next to their name in the contact panel!

---

## 4️⃣ Rendering the Messages (UI)

Inside the `#chat-content` div, we map over the `messages` array to render each bubble.

1. **Alignment**:
   - If `msg.sender === login_id`, it is **YOUR** message. We apply the `.sent` class (green background, aligned to the right, bottom-right tail).
   - Otherwise, it is **THEIR** message. We apply the `.received` class (dark background, aligned to the left, top-left tail).
2. **Timestamp Formatting**:
   - We pass the `msg.createdAt` timestamp to a formatting function.
   - `new Date(msg.createdAt).toLocaleTimeString(...)` turns `2026-05-02T17:21:21.959Z` into a clean format like `10:51 PM`.
3. **Display**:
   - The message bubble is rendered using CSS Flexbox. The text sits at the top, and the timestamp sits tucked away in the bottom right corner with a slightly faded color, perfectly mimicking WhatsApp.
