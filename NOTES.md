# 📝 FinHub Development Notes

**Last Updated:** December 2024  
**Purpose:** Documentation for future developers/AI assistants working on this codebase

---

## 🎯 Overview

FinHub is a single-page web application built with vanilla HTML/CSS/JavaScript and Firebase Realtime Database. It features a progressive unlock system, real-time chat, friends system, and collaborative painting.

**Main File:** `index.html` (contains all HTML, CSS, and JavaScript)

---

## 🔓 Unlock System

### Current Implementation (Updated Dec 2024)

The unlock system uses a progressive flow where features unlock in sequence:

1. **FINHUB** → Click the FinHub logo to unlock
   - Automatically unlocks **CHAT** when FINHUB is unlocked
   - Location: Logo click handler at line ~1439

2. **CHAT** → Automatically unlocked when FINHUB is unlocked
   - Can also be unlocked by sending 1 message (backup method)
   - Location: `checkUnlock()` function at line ~1449

3. **FRIENDS** → Unlocked by sending 1 chat message
   - Tracks `messageCount` in Firebase stats
   - Location: `sendMessage()` function checks unlock at line ~1774

4. **PAINT** → Unlocked by getting 1 friend
   - Tracks `friendCount` in Firebase stats
   - Location: `acceptFriendRequest()` function checks unlock at line ~2019

### Key Functions

- **`initUnlockSystem()`** (line ~1401): Initializes unlock system on page load
  - Loads unlocked features from Firebase
  - If FINHUB is unlocked, automatically unlocks CHAT
  - Sets up logo click handler

- **`checkUnlock(type)`** (line ~1449): Checks if a feature should be unlocked
  - FINHUB: Always unlocks on click
  - CHAT: Unlocks if FINHUB is unlocked OR messageCount >= 1
  - FRIENDS: Unlocks if messageCount >= 1
  - PAINT: Unlocks if friendCount >= 1

- **`unlockFeature(type)`** (line ~1475): Actually unlocks a feature
  - Adds to `unlockedFeatures` Set
  - Saves to Firebase at `users/{usernameLower}/unlocks/{type}`
  - Shows unlock animation
  - **Special:** If FINHUB is unlocked, automatically unlocks CHAT

- **`updateUIUnlocks()`** (line ~1572): Updates UI to show/hide locked features
  - CHAT button: Unlocked if FINHUB is unlocked OR CHAT is unlocked
  - FRIENDS button: Unlocked if FRIENDS feature is unlocked
  - PAINT button: Unlocked if PAINT feature is unlocked

### Unlock Data Structure

**Firebase Path:** `users/{usernameLower}/unlocks/{featureType}`
- Values: `true` (unlocked) or missing (locked)
- Feature types: `'finhub'`, `'chat'`, `'friends'`, `'paint'`

**Local Storage:**
- `unlockedFeatures`: Set containing unlocked feature types
- Persisted in Firebase, loaded on page init

### Unlock Animation

- Overlay appears with animated card
- Shows icon, name, and description
- Animates from target element (if available) to center
- Auto-hides after 1.8 seconds

---

## 💬 Chat System

### Global Chat
- **Firebase Path:** `messages`
- Real-time listener updates chat automatically
- Messages stored with: `username`, `message`, `timestamp`
- Location: `sendMessage()` at line ~1749, `loadGlobalChat()` at line ~1781

### Private Chat
- **Firebase Path:** `privateChats/{chatId}/messages`
- Chat ID: Alphabetically sorted lowercase usernames joined with `_`
- Example: `user1_user2` (always same regardless of who created it)
- Location: `openPrivateChat()` at line ~2065, `sendPrivateMessage()` at line ~2145

### Message Tracking
- `messageCount` tracked in `users/{usernameLower}/stats/messageCount`
- Incremented when sending global chat message
- Used for FRIENDS unlock trigger

---

## 👥 Friends System

### Friend Requests
- **Firebase Path:** `friendRequests/{targetUsernameLower}/{fromUsernameLower}`
- Structure: `{ username: string, displayName: string, timestamp: number }`
- Location: `sendFriendRequest()` at line ~1837

### Friends List
- **Firebase Path:** `friends/{usernameLower}/{friendUsernameLower}`
- Structure: `{ username: string, displayName: string, addedAt: number }`
- Location: `loadFriends()` at line ~2000

### Friend Count Tracking
- `friendCount` tracked in `users/{usernameLower}/stats/friendCount`
- Incremented when accepting friend request
- Used for PAINT unlock trigger

### Friend Request Flow
1. User enters username in "Add Friend" input
2. System checks: not self, not already friends, no pending request
3. Creates entry in `friendRequests/{target}/{sender}`
4. Target user sees request in "Friend Requests" tab
5. Accept/Decline updates `friends/` and removes from `friendRequests/`

---

## 🎨 Paint System

### Paint Sessions
- **Firebase Path:** `paintSessions/{sessionId}`
- Structure:
  ```javascript
  {
    createdBy: string,
    createdAt: number,
    participants: { [usernameLower]: true },
    strokes: {
      [strokeId]: {
        id: string,
        username: string,
        color: string,
        brushSize: number,
        texture: string,
        points: [[x, y], ...],
        timestamp: number
      }
    }
  }
  ```

### Paint Invites
- **Firebase Path:** `paintInvites/{targetUsernameLower}/{fromUsernameLower}`
- Structure: `{ sessionId: string, fromUsername: string, timestamp: number }`
- Location: `sendPaintInvites()` at line ~1990, `acceptPaintInvite()` at line ~1940

### Paint Features
- Real-time collaborative drawing
- Multiple brush textures (solid, dots, lines, grid)
- Color picker
- Brush size slider
- Touch support for mobile devices

### Texture Unlocks
- **Dots:** Unlocked after 1 color change
- **Lines:** Unlocked after 3 color changes
- **Grid:** Unlocked after 5 color changes
- Tracked in `users/{usernameLower}/stats/colorChangeCount`

---

## 🔥 Firebase Structure

### Database Schema

```
firebase-database/
├── messages/
│   └── {messageId}/
│       ├── username: string
│       ├── message: string
│       └── timestamp: number
│
├── privateChats/
│   └── {chatId}/  (e.g., "user1_user2")
│       └── messages/
│           └── {messageId}/
│               ├── username: string
│               ├── message: string
│               └── timestamp: number
│
├── friendRequests/
│   └── {targetUsernameLower}/
│       └── {fromUsernameLower}/
│           ├── username: string
│           ├── displayName: string
│           └── timestamp: number
│
├── friends/
│   └── {usernameLower}/
│       └── {friendUsernameLower}/
│           ├── username: string
│           ├── displayName: string
│           └── addedAt: number
│
├── paintSessions/
│   └── {sessionId}/
│       ├── createdBy: string
│       ├── createdAt: number
│       ├── participants: { [usernameLower]: true }
│       └── strokes/
│           └── {strokeId}/
│               ├── id: string
│               ├── username: string
│               ├── color: string
│               ├── brushSize: number
│               ├── texture: string
│               ├── points: [[x, y], ...]
│               └── timestamp: number
│
├── paintInvites/
│   └── {targetUsernameLower}/
│       └── {fromUsernameLower}/
│           ├── sessionId: string
│           ├── fromUsername: string
│           └── timestamp: number
│
└── users/
    └── {usernameLower}/
        ├── unlocks/
        │   ├── finhub: boolean
        │   ├── chat: boolean
        │   ├── friends: boolean
        │   └── paint: boolean
        └── stats/
            ├── messageCount: number
            ├── friendCount: number
            └── colorChangeCount: number
```

### Firebase Helper Functions

Located around line ~2600:
- `writeToFirebase(path, data)`: Write data to path
- `readFromFirebase(path)`: Read data from path (returns Promise)
- `updateFirebase(path, data)`: Update data at path
- `deleteFromFirebase(path)`: Delete data at path
- `listenToFirebase(path, callback)`: Real-time listener (returns listener object)
- `pushToFirebase(path, data)`: Push data with auto-generated key

---

## 🎨 UI Components

### Pages
- **Home Page:** Login/Register interface
- **Main Page:** Main application interface with sidebar buttons

### Sidebar Buttons
- 💬 Chat (unlocks with FINHUB)
- 👥 Friends (unlocks with 1 message)
- 🎨 Paint (unlocks with 1 friend)

### Visual Effects
- **FinHub Logo:** Mouse proximity effect (letters scale/rotate/glow based on cursor distance)
- **Unlock Animations:** Animated overlay cards with icons
- **Locked Features:** Grayed out buttons with alert messages

---

## 🔧 Important Implementation Details

### Username Handling
- **Always use lowercase** for Firebase paths: `usernameLower`
- Display names can have original casing
- Stored in `localStorage`: `loggedInUsername` and `loggedInUsernameLower`

### State Management
- Uses global variables for current state
- `currentPrivateChatUser`: Currently open private chat
- `currentPaintSession`: Current paint session ID
- `unlockedFeatures`: Set of unlocked feature types
- Firebase listeners stored in variables for cleanup

### Listener Cleanup
- Always call `.off('value')` on Firebase listeners before creating new ones
- Prevents memory leaks and duplicate listeners
- Example: `if (globalChatListener) globalChatListener.off('value');`

### Error Handling
- Firebase operations use `.then()` and `.catch()`
- User-facing errors shown via `alert()`
- Missing data handled with `||` defaults

---

## 🚀 Recent Changes (Dec 2024)

### Unlock System Update
- **Changed:** Clicking logo now unlocks both FINHUB and CHAT simultaneously
- **Changed:** CHAT unlock description updated to reflect logo click
- **Changed:** `updateUIUnlocks()` now checks FINHUB status for CHAT button
- **Changed:** `checkUnlock()` for CHAT now checks if FINHUB is unlocked
- **Changed:** `unlockFeature()` automatically unlocks CHAT when FINHUB unlocks
- **Changed:** `initUnlockSystem()` automatically unlocks CHAT if FINHUB is already unlocked

**Files Modified:**
- `index.html` (lines ~1415-1421, ~1449-1465, ~1475-1485, ~1572-1580)

---

## 🐛 Known Issues / Future Improvements

### Potential Issues
- No pagination for chat messages (could get slow with many messages)
- No rate limiting on friend requests
- Paint canvas doesn't resize on window resize
- No offline support

### Future Enhancements
- Add more unlock features
- Add user profiles/avatars
- Add notifications for friend requests
- Add paint session history
- Add chat message search
- Add emoji support in chat
- Add file/image sharing

---

## 📚 Code Organization

### File Structure
```
Finleyhub/
├── index.html          # All code (HTML, CSS, JavaScript)
├── README.md          # GitHub Pages setup guide
├── FIREBASE_SETUP_GUIDE.md  # Firebase configuration guide
└── NOTES.md           # This file
```

### Code Sections (approximate line numbers)
- **Lines 1-1000:** CSS styles
- **Lines 1000-1300:** HTML structure
- **Lines 1300-1400:** Initialization and setup
- **Lines 1400-1650:** Unlock system
- **Lines 1650-2200:** Chat and Friends system
- **Lines 2200-2600:** Paint system
- **Lines 2600-2900:** Firebase helper functions
- **Lines 2900-3100:** Authentication and initialization

---

## 🔐 Authentication

### Login/Register
- Simple username-based authentication
- No password required (for simplicity)
- Username stored in `localStorage`
- Firebase path: `users/{usernameLower}`

### Session Management
- Username persists in `localStorage`
- No session expiration
- Logout clears `localStorage` and redirects to home

---

## 📝 Development Notes for Future Work

### When Adding New Features
1. Check if it should be part of unlock system
2. Add to `UNLOCK_TYPES` and `UNLOCK_INFO` if needed
3. Update `checkUnlock()` with unlock conditions
4. Add UI button/component
5. Update `updateUIUnlocks()` if it's a locked feature
6. Document Firebase path structure

### When Modifying Unlock System
1. Remember: FINHUB unlock automatically unlocks CHAT
2. Check `initUnlockSystem()` for initialization logic
3. Update `updateUIUnlocks()` for UI changes
4. Test unlock flow: Logo → Chat → Friends → Paint

### When Working with Firebase
1. Always use lowercase usernames for paths
2. Clean up listeners with `.off('value')`
3. Handle missing data with defaults (`|| {}`, `|| 0`, etc.)
4. Use `pushToFirebase()` for arrays/lists with auto-IDs
5. Use `writeToFirebase()` for specific paths

### Code Style
- Functions use camelCase
- Constants use UPPER_SNAKE_CASE
- CSS classes use kebab-case
- Firebase paths use camelCase for keys

---

## 🎯 Quick Reference

### Unlock Flow
```
User clicks logo → FINHUB unlocked → CHAT unlocked
User sends message → FRIENDS unlocked
User gets friend → PAINT unlocked
```

### Key Variables
- `unlockedFeatures`: Set of unlocked feature strings
- `messageCount`: Number of messages sent
- `friendCount`: Number of friends
- `colorChangeCount`: Number of paint color changes
- `usernameLower`: Current user's lowercase username

### Key Functions
- `initUnlockSystem()`: Initialize unlock system
- `checkUnlock(type)`: Check if feature should unlock
- `unlockFeature(type)`: Unlock a feature
- `updateUIUnlocks()`: Update UI based on unlocks
- `sendMessage()`: Send chat message (triggers FRIENDS unlock)
- `acceptFriendRequest()`: Accept friend (triggers PAINT unlock)

---

**End of Notes**
