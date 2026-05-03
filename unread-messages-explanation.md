# Unread Messages and Auto-Scroll — How It Works

This document explains the "unseen/unread message" badge system and how the chat automatically scrolls to the latest message.

---

## Simple Explanation

```
You are chatting with Alice
        |
Bob sends you a message
        |
You are NOT in Bob's chat, so:
   -> Message does NOT appear on screen
   -> Green badge "1" appears next to Bob's name
        |
Bob sends another message
   -> Badge updates to "2"
        |
You click on Bob's name
   -> Badge resets to 0
   -> Chat history loads
   -> Screen auto-scrolls to the latest message
```

Real-life example: It works exactly like WhatsApp — you see a green circle with a number next to contacts who sent you messages you haven't read yet.

---

## Part 1: How Unread Counts Are Tracked

There is NO backend involved in unread counts. It all happens in the browser using React state.

### The State Variable

```javascript
const [unreadCounts, setUnreadCounts] = useState({})
```

- This is an object (like a dictionary) that maps user IDs to their unread count.
- Like a notebook where you write: "Bob: 3 unread, Carol: 1 unread"

**Example of what it looks like inside:**
```javascript
{
    "664abc123": 3,    // Bob has 3 unread messages
    "664def456": 1     // Carol has 1 unread message
}
```

---

## Part 2: When Does the Count Go UP?

The count goes up when a message arrives via Socket.IO and you are NOT in that person's chat.

### The Socket Listener

```javascript
newSocket.on("receive_message", (incomingMessage) => {
```
- This fires every time someone sends you a message in real-time. Like your phone buzzing when a message arrives.

```javascript
    const whoSentIt = incomingMessage.senderId;
```
- Grabs the sender's ID from the incoming message. Like checking "who sent this?"

```javascript
    const currentChatId = document.getElementById("active-chat-id")?.dataset?.id;
```
- Reads the hidden HTML element to check which chat you currently have open. Like checking which conversation tab is on your screen right now.

**Why not just use `activeChat` directly?** Because of React's "stale closure" problem — when the socket listener was created, it captured the OLD value of `activeChat`. Even if you switch chats later, the listener still remembers the old one. Reading from the DOM always gives the CURRENT value.

### The Decision: Show message OR show badge?

```javascript
    if (currentChatId === whoSentIt) {
        // You ARE chatting with this person — show the message on screen
        setMessages((prevMessages) => [...prevMessages, incomingMessage]);
    } else {
        // You are NOT in this person's chat — increase their badge count
        setUnreadCounts((prev) => ({
            ...prev,
            [whoSentIt]: (prev[whoSentIt] || 0) + 1,
        }));
    }
```

**Scenario A: You are chatting with the sender**
- The message gets added directly to the chat window. You see it pop up instantly.
- Like receiving a message while you already have that chat open on WhatsApp.

**Scenario B: You are NOT chatting with the sender**
- The message is NOT shown on screen (that would mix up conversations).
- Instead, `unreadCounts` is updated: their count goes from 0 to 1, or 1 to 2, etc.
- Like receiving a message from someone while you're reading a different chat.

### How `(prev[whoSentIt] || 0) + 1` works:

```javascript
// First message from Bob:
prev[whoSentIt] = undefined    // Bob has no entry yet
(undefined || 0) + 1 = 1      // Result: 1

// Second message from Bob:
prev[whoSentIt] = 1            // Bob already has 1
(1 || 0) + 1 = 2              // Result: 2

// Third message from Bob:
prev[whoSentIt] = 2
(2 || 0) + 1 = 3              // Result: 3
```

The `|| 0` is a safety net — if the user has no entry yet, treat it as 0 instead of `undefined`.

---

## Part 3: How the Badge Appears on Screen

### The JSX (inside each contact row):

```jsx
{unreadCounts[user._id] > 0 && (
    <div className="unread-badge">
        {unreadCounts[user._id]}
    </div>
)}
```

- `unreadCounts[user._id] > 0` — Only show the badge if this user has unread messages. Like WhatsApp hiding the badge when count is 0.
- `&&` — This is "short-circuit rendering". If the condition is false, React renders nothing. If true, it renders the badge div.
- `{unreadCounts[user._id]}` — Displays the actual number (1, 2, 3, etc.)

### The CSS Styling:

```css
.unread-badge {
    min-width: 20px;
    height: 20px;
    background-color: #00a884;     /* WhatsApp green */
    color: white;
    border-radius: 50%;            /* Makes it a circle */
    font-size: 12px;
    font-weight: 700;              /* Bold number */
    display: flex;
    justify-content: center;       /* Center number horizontally */
    align-items: center;           /* Center number vertically */
    padding: 0 5px;
    margin-left: 8px;
}
```

- `border-radius: 50%` — Makes the div a perfect circle. Like the green bubble on WhatsApp.
- `min-width: 20px` — Ensures single digit numbers (1-9) still look circular. Double digits (10+) will stretch slightly.
- `background-color: #00a884` — The exact WhatsApp green color.

---

## Part 4: When Does the Count Reset to ZERO?

The count resets when you click on that contact's name (opening their chat).

```javascript
onClick={() => {
    setActiveChat(user);
    // Reset unread count for this user
    setUnreadCounts((prev) => ({ ...prev, [user._id]: 0 }));
}}
```

- `setActiveChat(user)` — Opens this person's chat. Like tapping a contact on WhatsApp.
- `setUnreadCounts` with `[user._id]: 0` — Sets their count back to 0. The badge disappears because `0 > 0` is false.
- Like opening an unread chat on WhatsApp — the badge disappears immediately.

---

## Part 5: Auto-Scroll to Latest Message

When new messages arrive (or when you open a chat), the screen should automatically scroll to the bottom so you see the latest message.

### The Ref

```javascript
const messageEndRef = useRef(null);
```

- Creates a reference to a DOM element. Like putting a bookmark at the very bottom of the chat.

### The Invisible Anchor

```jsx
{/* This invisible div sits at the very bottom of all messages */}
<div ref={messageEndRef} />
```

- This empty div is placed AFTER all the message bubbles. It's invisible but acts as a target to scroll to. Like placing a flag at the bottom of a page.

### The Auto-Scroll Effect

```javascript
useEffect(() => {
    messageEndRef.current?.scrollIntoView({ behavior: "smooth" });
}, [messages]);
```

- `[messages]` — This effect runs every time the `messages` array changes (new message sent, received, or chat history loaded).
- `messageEndRef.current` — Gets the actual DOM element (the invisible div at the bottom).
- `?.` — Optional chaining. If the ref isn't ready yet, don't crash.
- `.scrollIntoView()` — Scrolls the page until this element is visible on screen.
- `{ behavior: "smooth" }` — Smooth animated scroll instead of jumping instantly.

**Real-life example:** Like WhatsApp automatically scrolling down when you receive a new message so you always see the latest one without manually scrolling.

**When does it trigger?**

| Trigger | What Happens |
|---|---|
| You open a chat | `messages` gets set with history -> scroll to bottom |
| You send a message | New message added to `messages` -> scroll to bottom |
| You receive a message (in current chat) | New message added to `messages` -> scroll to bottom |

---

## Complete Flow Diagram

```
MESSAGE ARRIVES VIA SOCKET
        |
        v
Is the sender's chat currently open?
        |
   YES  |  NO
   |         |
   v         v
Add to       Increment badge:
messages     unreadCounts[sender] + 1
array             |
   |              v
   v         Green circle "3"
Message      appears next to
appears      their name
on screen         |
   |              v
   v         User clicks their name
Auto-scroll       |
to bottom         v
             Badge resets to 0
             Chat history loads
             Auto-scroll to bottom
```

---

## Summary

| What | How | Where |
|---|---|---|
| Track unread counts | `useState({})` object mapping user IDs to counts | ChatInterface.jsx line 28 |
| Increment count | `setUnreadCounts` in socket listener when chat is NOT open | ChatInterface.jsx line 213 |
| Show green badge | Conditional render `{count > 0 && <badge>}` | ChatInterface.jsx line 337 |
| Style the badge | `.unread-badge` CSS class (green circle) | chatinterface.css line 269 |
| Reset count | `setUnreadCounts({...prev, [id]: 0})` on contact click | ChatInterface.jsx line 305 |
| Auto-scroll ref | `useRef(null)` pointing to invisible bottom div | ChatInterface.jsx line 229 |
| Trigger scroll | `useEffect` watching `[messages]` changes | ChatInterface.jsx line 232 |
