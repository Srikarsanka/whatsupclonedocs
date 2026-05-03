# Animations Used in the Project

This document explains all the animations and transitions used across the Whatsupp Clone frontend.

---

## Libraries Used

| Library | Where Used | What It Does |
|---|---|---|
| **Framer Motion** | Login, Register pages | Provides smooth entry animations, hover effects, and exit animations for React components |
| **CSS Transitions** | Login, Register, ChatInterface | Smooth color changes on hover for buttons, inputs, and contact rows |
| **CSS @keyframes** | Register page | Continuous spinning animation for the background glow circle |

---

## Framer Motion Animations

Framer Motion is a React animation library. Instead of writing `<div>`, you write `<motion.div>` and pass animation props directly.

### 1. Page Entry Animation (Login and Register)

```jsx
<motion.div
      initial={{ opacity: 0, scale: 0.98 }}
      animate={{ opacity: 1, scale: 1 }}
      transition={{ duration: 0.5 }}
>
```

| Prop | What It Does |
|---|---|
| `initial` | The starting state — page starts invisible (`opacity: 0`) and slightly shrunken (`scale: 0.98`) |
| `animate` | The ending state — page fades in (`opacity: 1`) and scales to normal size (`scale: 1`) |
| `transition` | How long the animation takes — `0.5` seconds |

**Result:** When you open the Login or Register page, it smoothly fades in and slightly zooms into place instead of appearing instantly.

---

### 2. Mobile Phone Frame (Login Page)

```jsx
<motion.div
      className="mobile-frame"
      initial={{ y: 50, opacity: 0 }}
      animate={{ y: 0, opacity: 1 }}
      transition={{ type: "spring", delay: 0.3 }}
>
```

| Prop | What It Does |
|---|---|
| `initial={{ y: 50 }}` | Starts 50px below its final position |
| `animate={{ y: 0 }}` | Slides up to its correct position |
| `type: "spring"` | Uses a spring/bounce effect instead of a linear slide |
| `delay: 0.3` | Waits 0.3 seconds before starting (so it appears after the page fades in) |

**Result:** The phone mockup on the login page slides up with a natural bouncy spring effect, delayed slightly after the page appears.

---

### 3. Chat Bubble Animation (Login Page)

```jsx
<AnimatePresence>
      {showReply && (
            <motion.div
                  className="chat-bubble friend-message"
                  initial={{ opacity: 0, scale: 0.8, y: 10 }}
                  animate={{ opacity: 1, scale: 1, y: 0 }}
                  exit={{ opacity: 0, scale: 0.8, y: 5 }}
            >
                  All good here!
            </motion.div>
      )}
</AnimatePresence>
```

| Prop | What It Does |
|---|---|
| `AnimatePresence` | A wrapper that enables exit animations (normally React just removes elements instantly) |
| `initial` | Starts invisible, small, and shifted down |
| `animate` | Pops in to full size and position |
| `exit` | When removed, shrinks and fades out (instead of disappearing instantly) |

**Result:** On the login page, a chat bubble appears and disappears every 3 seconds with a smooth pop-in/pop-out effect, simulating a live conversation.

---

### 4. Button Hover and Tap (Login and Register)

```jsx
<motion.button
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.98 }}
>
      Log In
</motion.button>
```

| Prop | What It Does |
|---|---|
| `whileHover` | When the mouse hovers over the button, it slightly enlarges (2% bigger) |
| `whileTap` | When the button is clicked/pressed, it slightly shrinks (2% smaller) |

**Result:** The Login and Register buttons have a satisfying press/click feel — they grow on hover and compress on click.

---

## CSS Transitions

CSS transitions create smooth changes when a property value changes (like on hover).

### 5. Contact Row Hover (ChatInterface)

```css
.contact-row {
      transition: background-color 0.2s;
}
.contact-row:hover {
      background-color: #202c33;
}
```

**Result:** When you hover over a contact in the left panel, the background color smoothly transitions to a lighter shade instead of snapping instantly.

---

### 6. Input Focus Effects (Login and Register)

```css
.input-container input {
      transition: all 0.2s;
}
.input-container input:focus {
      border-color: #00a884;
}
```

**Result:** When you click into an input field, the border color smoothly changes to green.

---

### 7. Button Hover Glow (Login and Register)

```css
.login-button {
      transition: all 0.2s;
}
.login-button:hover {
      box-shadow: 0 0 20px rgba(0, 168, 132, 0.4);
}
```

**Result:** When you hover over the login button, a green glow appears around it smoothly.

---

## CSS @keyframes Animation

### 8. Spinning Glow Circle (Register Page)

```css
.glow-orb {
      animation: spin 40s linear infinite;
}

@keyframes spin {
      from { transform: rotate(0deg); }
      to { transform: rotate(360deg); }
}
```

| Property | What It Does |
|---|---|
| `animation: spin` | Uses the `spin` keyframe |
| `40s` | One full rotation takes 40 seconds (very slow) |
| `linear` | Constant speed (no acceleration) |
| `infinite` | Never stops |

**Result:** A large glowing circle on the Register page slowly rotates forever, giving the page a dynamic, premium feel.

---

## Summary

| Animation | Type | Where | Effect |
|---|---|---|---|
| Page fade-in | Framer Motion | Login, Register | Smooth page entrance |
| Phone slide-up | Framer Motion (spring) | Login | Bouncy slide from bottom |
| Chat bubble pop | Framer Motion + AnimatePresence | Login | Appears/disappears every 3s |
| Button press | Framer Motion (whileHover/Tap) | Login, Register | Grows on hover, shrinks on click |
| Contact row hover | CSS transition | ChatInterface | Smooth background color change |
| Input focus | CSS transition | Login, Register | Smooth border color change |
| Button glow | CSS transition | Login, Register | Green glow on hover |
| Spinning circle | CSS @keyframes | Register | Slow infinite rotation |
