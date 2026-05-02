# 🔄 Authentication & Full App Flow

This document explains the end-to-end journey of a user using the WhatsApp Clone, from registering an account to seeing their recent chats.

---

## 1️⃣ Registration (`/api/user/register`)

When a new user visits the app, they start at the `Register.jsx` page.

### The Flow:
1. User enters their `username`, `email`, `phone`, `date of birth`, and `password`.
2. Frontend sends an Axios `POST` request to `/api/user/register`.
3. **Backend Logic (`register.js`)**:
   - Checks if all fields are provided.
   - Hashes the password using `bcrypt` for security (so raw passwords are never saved).
   - Saves the new user into the MongoDB `Login` collection.
4. User is redirected to the Login page.

---

## 2️⃣ Login (`/api/user/login`)

After registering, the user must authenticate themselves to use the app.

### The Flow:
1. User enters their `email` and `password` on `Login.jsx`.
2. Frontend sends an Axios `POST` request to `/api/user/login`.
3. **Backend Logic (`login.js`)**:
   - Finds the user in MongoDB by their email.
   - Uses `bcrypt.compare` to check if the entered password matches the hashed password in the database.
   - **JWT Creation**: If successful, creates a JSON Web Token (JWT) containing the user's `_id`.
   - **Cookie**: Sends this JWT back to the browser securely inside an HTTP-only cookie.
4. Frontend redirects the user to the main `ChatInterface.jsx`.

---

## 3️⃣ App Initialization (`ChatInterface.jsx`)

When the main chat page loads, it needs to figure out who the user has talked to previously to populate the left-hand contact panel.

### The Flow:
1. `useEffect` runs once on page load.
2. It calls `fetchConversations()`, making a `GET` request to `/api/message/conversations/all`.
3. **Backend Logic (`conversations.js`)**:
   - Looks at all messages where the logged-in user is either the `sender` or `receiver`.
   - Collects all the unique user IDs from those messages.
   - Fetches the user profiles for those IDs.
   - Checks if each user is in our `friends` array (using `.some()` on the ObjectIds) to decide whether to show their `username` (friend) or `phone number` (stranger).
4. Frontend saves this list into the `searchResults` state, making the contacts appear on the screen!

---

## 4️⃣ Searching for Users (`/api/user/search/:searchTerm`)

If the user wants to start a new chat, they use the search bar.

### The Flow:
1. User types in the search bar. The `handleSearchQueryChange` function is triggered.
2. **If empty**: It immediately calls `fetchConversations()` again to bring back the recent chats list.
3. **If typing**: It sends a `GET` request to `/api/user/search/searchTerm`.
4. **Backend Logic (`searchuser.js`)**:
   - Uses MongoDB's `$regex` operator to search the `username`, `email`, or `phone` fields.
   - Ignores the user's own profile so they don't search themselves.
   - Again, checks if the found users are friends to determine if it should send back their username or phone number.
5. Frontend updates `searchResults`, showing the matching people.

---

## 5️⃣ Adding / Removing Friends

When a user clicks the heart icon next to a contact, it toggles their friend status.

### The Flow:
1. **Optimistic UI**: The frontend immediately changes the heart icon locally using `setSearchResults` mapping so the app feels incredibly fast.
2. An API call is sent to either `/api/user/addfriend` or `/api/user/removefriend`.
3. **Backend Logic**:
   - Adds or splices the `friendId` from the user's `friends` array in MongoDB.
   - Uses `.some()` to safely compare Mongoose `ObjectId` types to strings to find the right friend to remove.
4. If the backend fails for any reason, the frontend catches the error and flips the heart back to its original state.
