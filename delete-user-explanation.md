# Delete User Account — How It Works

This document explains how the **Delete Account** feature works, from the button click on the frontend all the way to the database cleanup on the backend.

---

## The Flow (Simple Version)

```
User clicks "Delete Account" in Profile modal
        |
Confirmation dialog appears: "Are you sure?"
        |
Frontend sends DELETE /api/user/delete
        |
Backend deletes ALL messages (sent and received)
        |
Backend removes user from everyone's friends list
        |
Backend deletes the user document from MongoDB
        |
Backend clears the auth cookie
        |
Frontend clears sessionStorage and redirects to Login
```

---

## Files Involved

| File | Role |
|---|---|
| `frontend/src/pages/ChatInterface.jsx` | Delete button inside Profile modal |
| `backend/routes/deleteuserroute.js` | Defines the DELETE route |
| `backend/controllers/users/deleteuser.js` | The main deletion logic |
| `backend/Auth/middleware.js` | Verifies JWT token before allowing deletion |
| `backend/model/loginschema.js` | User schema (the document being deleted) |
| `backend/model/messageschema.js` | Message schema (messages being cleaned up) |

---

## Frontend — The Delete Button

Inside the Profile modal, there is a red "Delete Account" button:

```javascript
<button
      onClick={async () => {
            // Step 1: Ask the user to confirm
            const confirmed = window.confirm(
                  "Are you sure you want to delete your account? This action cannot be undone."
            );
            if (!confirmed) return;

            try {
                  // Step 2: Send DELETE request to the backend
                  await axios.delete(
                        "http://localhost:3000/api/user/delete",
                        { withCredentials: true }
                  );

                  // Step 3: Clear local session data
                  sessionStorage.clear();

                  // Step 4: Notify and redirect to login
                  alert("Account deleted successfully.");
                  navigate("/login");
            } catch (e) {
                  console.log(e);
                  alert("Failed to delete account.");
            }
      }}
>
      Delete Account
</button>
```

### Why `window.confirm()`?
This is a safety net. Deleting an account is irreversible, so we show a browser confirmation dialog before proceeding. If the user clicks "Cancel", the function exits immediately with `return`.

### Why `sessionStorage.clear()`?
After the backend deletes the user, the frontend still has `userId` and `username` stored in sessionStorage. If we don't clear it, the `ProtectedRoute` in `App.jsx` would still think the user is logged in.

---

## Route — Where the Request Goes

```javascript
// backend/routes/deleteuserroute.js

const express = require("express")
const deleteUser = require("../controllers/users/deleteuser")
const auth = require("../Auth/middleware")
const router = express.Router()

router.delete("/user/delete", auth, deleteUser)

module.exports = router
```

The `auth` middleware runs first — it reads the JWT cookie, verifies the token, and attaches `req.user = { id, name }` so the controller knows which user to delete.

---

## Backend Controller — The Main Logic

Here is the full code with a step-by-step breakdown:

```javascript
const User = require("../../model/loginschema")
const Message = require("../../model/messageschema")

const deleteUser = async (req, res) => {
      try {
            const userId = req.user.id
```

### Step 1: Delete All Messages

```javascript
            await Message.deleteMany({
                  $or: [
                        { sender: userId },
                        { receiver: userId }
                  ]
            })
```

**What this does:**
Finds every message in the database where the user is either the `sender` or the `receiver` and deletes all of them in one query.

**Why delete messages?**
If we only delete the user document but leave their messages, the chat interface would show messages from a user that no longer exists. This keeps the database clean.

**Example:**
If User A (being deleted) chatted with User B and User C:
```
Before deletion:
  { sender: "A", receiver: "B", message: "Hi B" }      <- deleted
  { sender: "B", receiver: "A", message: "Hey A" }      <- deleted
  { sender: "A", receiver: "C", message: "Hello C" }    <- deleted
  { sender: "B", receiver: "C", message: "Hi C" }       <- NOT deleted (doesn't involve A)
```

---

### Step 2: Remove User from Everyone's Friends List

```javascript
            await User.updateMany(
                  { friends: userId },
                  { $pull: { friends: userId } }
            )
```

**What this does:**
- `{ friends: userId }` — finds every user who has this person in their `friends` array
- `{ $pull: { friends: userId } }` — removes this person's ID from their `friends` array

**Why do this?**
Each user has a `friends` array storing ObjectIds of their friends. If we delete User A without cleaning this up, other users would have a "ghost" ID in their friends list pointing to a user that no longer exists.

**Example:**
```
Before:
  User B: { friends: ["A", "C"] }
  User C: { friends: ["A"] }

After deleting User A:
  User B: { friends: ["C"] }
  User C: { friends: [] }
```

---

### Step 3: Delete the User Document

```javascript
            await User.findByIdAndDelete(userId)
```

**What this does:**
Finds the user document by its `_id` and permanently removes it from the `Login` collection.

After this, the user's username, email, phone, password, and all other data are gone from the database.

---

### Step 4: Clear the Auth Cookie

```javascript
            res.clearCookie("token")

            return res.status(200).json({ message: "Account deleted successfully" })
```

**What `clearCookie` does:**
The user's JWT token is stored in an HTTP-only cookie called `token`. Even though the user document is deleted, the cookie would still exist in the browser until it expires. `clearCookie` tells the browser to immediately delete it.

---

## Order of Operations Matters

The deletion steps are executed in a specific order for a reason:

```
1. Delete messages     <- clean up related data first
2. Clean friends lists <- remove references from other users
3. Delete user         <- delete the main document last
4. Clear cookie        <- clean up the browser session
```

If we deleted the user first (Step 3), and then Steps 1 or 2 failed, we would have orphaned messages and ghost friend references with no way to trace them back.

---

## Error Handling

```javascript
      } catch (err) {
            console.log("Error deleting user:", err)
            return res.status(500).json({ message: "Internal Server error while deleting user" })
      }
}
```

If any step fails (database error, network issue), the entire operation stops and returns a 500 error. The frontend catches this and shows an alert: "Failed to delete account."

---

## Summary

| Step | What Happens | MongoDB Operation |
|---|---|---|
| 1 | Delete all user's messages | `Message.deleteMany()` |
| 2 | Remove user from all friends lists | `User.updateMany()` with `$pull` |
| 3 | Delete the user document | `User.findByIdAndDelete()` |
| 4 | Clear auth cookie | `res.clearCookie("token")` |
| 5 | Frontend clears session | `sessionStorage.clear()` |
| 6 | Redirect to login page | `navigate("/login")` |
