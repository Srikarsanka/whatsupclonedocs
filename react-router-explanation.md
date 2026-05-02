# 🗺️ React Router DOM — Full Explanation

This document explains what `react-router-dom` is and how it is used to manage navigation in this WhatsApp Clone project.

---

## 📌 What is React Router DOM?

In traditional websites, clicking a link (like `/about`) asks the server for an entirely new HTML page. The screen flashes white, and the new page loads from scratch.

React applications are usually **Single Page Applications (SPAs)**. This means the server only ever sends *one* HTML file (`index.html`). 

**React Router DOM** is a library that hijacks the browser's URL. When you click a link or type a new URL, React Router stops the browser from refreshing. Instead, it looks at the URL and instantly loads the correct React component right on the same page.

---

## 🚦 How it's Set Up (`main.jsx` & `App.jsx`)

At the highest level of our application, we wrap everything in `<BrowserRouter>`. This gives our entire app the ability to read the URL.

```javascript
// main.jsx
import { BrowserRouter } from 'react-router-dom'

ReactDOM.createRoot(document.getElementById('root')).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
)
```

Inside `App.jsx`, we define our **Routes** (the rules of navigation).

```javascript
// App.jsx
import { Routes, Route } from 'react-router-dom';
import Register from './pages/Register';
import Login from './pages/Login';
import ChatInterface from './pages/ChatInterface';

function App() {
  return (
    <Routes>
      {/* If the URL is exactly "/register", show the Register component */}
      <Route path="/register" element={<Register />} />

      {/* If the URL is exactly "/login", show the Login component */}
      <Route path="/login" element={<Login />} />

      {/* If the URL is exactly "/", show the main Chat component */}
      <Route path="/" element={<ChatInterface />} />
    </Routes>
  );
}
```

---

## 🚀 How We Navigate (`useNavigate`)

To move between pages *without* clicking a standard `<a>` link, we use a hook called `useNavigate()`.

### Example 1: After Successful Login

When a user successfully logs in, we don't want them to stay on the login page. We want to send them to the main chat interface.

```javascript
// Login.jsx
import { useNavigate } from 'react-router-dom';

export default function Login() {
      const navigate = useNavigate(); // Get the navigator function

      const handleLogin = async (e) => {
            // ... wait for axios to log in ...
            if (response.status === 200) {
                  // User logged in! Take them to the homepage!
                  navigate("/");
            }
      }
}
```

### Example 2: The "Register here" Button

At the bottom of the Login form, there's a button to go to the Register page.

```javascript
// Login.jsx
<span 
      onClick={() => navigate("/register")} 
      className="register-link"
>
      Register here
</span>
```

---

## 🔄 React Router vs Express Router

You might be wondering: *"Wait, don't we have routers in the backend too?"*

Yes! But they do completely different things:

| | `react-router-dom` (Frontend) | Express Router (Backend) |
|---|---|---|
| **Where it runs** | In the user's Browser | On the Node.js Server |
| **What it controls** | What **UI Components** are shown on screen | What **Data/Actions** are performed via APIs |
| **Example** | `navigate("/login")` shows the Login Form | `axios.post("/api/user/login")` checks the DB password |
| **URL Example** | `http://localhost:5173/login` | `http://localhost:3000/api/user/login` |

In short: **React Router handles the Views. Express Router handles the Data.**
