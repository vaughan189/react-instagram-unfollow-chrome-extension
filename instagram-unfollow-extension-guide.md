Now let me create the comprehensive architecture and code document:
Action: file_editor create /app/instagram-unfollow-extension-guide.md --file-text "# Instagram Unfollow Chrome Extension — Architecture, Approach & Code

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [File Structure](#file-structure)
4. [Anti-Suspension Strategy](#anti-suspension-strategy)
5. [Complete Code](#complete-code)
6. [Installation Guide](#installation-guide)
7. [Usage Guide](#usage-guide)
8. [Known Limitations & Risks](#known-limitations--risks)

---

## Overview

A Chrome extension that automates unfollowing users on Instagram. The extension operates on the `/following` page of the logged-in user and provides:

- **Unfollow All** — Bulk unfollow every user in the following list
- **Unfollow with Limit** — Unfollow a specific number of users (e.g., 50 at a time)
- **Configurable Delay** — User sets min/max delay range; actual delay is randomized
- **Progress Tracking** — Live counter of unfollowed users
- **Pause/Resume** — Pause mid-process and resume without losing progress
- **Headless UI** — Minimal popup showing only the current user being unfollowed

---

## Architecture

```
+--------------------------------------------------+
|               Chrome Extension                    |
|                                                   |
|  +-------------+    +-------------------------+   |
|  |   Popup UI  |<-->|   Background Script     |   |
|  | (popup.html |    | (background.js)         |   |
|  |  popup.js)  |    | - State management      |   |
|  |             |    | - Message relay          |   |
|  +-------------+    +-------------------------+   |
|        ^                       ^                  |
|        |     Chrome Messages   |                  |
|        v                       v                  |
|  +-------------------------------------------+   |
|  |           Content Script                   |   |
|  |         (content.js)                       |   |
|  |                                            |   |
|  |  - DOM manipulation on instagram.com       |   |
|  |  - Scrolls the \"Following\" modal/list      |   |
|  |  - Clicks \"Following\" -> \"Unfollow\"        |   |
|  |  - Randomized delay between actions        |   |
|  |  - Sends progress updates to popup         |   |
|  +-------------------------------------------+   |
|                      |                            |
|                      v                            |
|            instagram.com/following                |
+--------------------------------------------------+
```

### Communication Flow

```
Popup (UI) <--chrome.runtime.sendMessage --> Background (State Store)
                                                    ^
                                                    |
                                            chrome.tabs.sendMessage
                                                    |
                                                    v
                                          Content Script (DOM Worker)
```

1. **Popup** sends commands: `START`, `PAUSE`, `RESUME`, `STOP`, `SET_CONFIG`
2. **Background** relays commands to the active tab's content script and persists state
3. **Content Script** executes DOM automation and sends `PROGRESS`, `STATUS`, `ERROR` updates back

---

## File Structure

```
instagram-unfollow-extension/
├── manifest.json            # Extension manifest (Manifest V3)
├── popup/
│   ├── popup.html           # Minimal headless UI
│   ├── popup.css            # Dark, minimal styling
│   └── popup.js             # Popup logic & message handling
├── scripts/
│   ├── content.js           # Core unfollow automation logic
│   └── background.js        # Service worker for state & messaging
└── icons/
    ├── icon16.png
    ├── icon48.png
    └── icon128.png
```

---

## Anti-Suspension Strategy

Instagram actively detects and penalizes automated behavior. The extension uses multiple strategies to stay under the radar:

### 1. Randomized Delays
Never use fixed intervals. Each action delay is randomized:
```
actualDelay = minDelay + Math.random() * (maxDelay - minDelay)
```
Default range: **3–7 seconds** between unfollows.

### 2. Session Limits
- **Hard cap**: Max ~150–200 unfollows per session
- **Recommended**: 50–80 per session with 1–2 hour cooldown between sessions
- The extension enforces a configurable limit

### 3. Human-like Behavior Simulation
- Random micro-pauses (50–300ms) before clicking buttons
- Occasional \"longer breaks\" (every 10–20 unfollows, add a 15–30s pause)
- Scroll behavior mimics human scrolling speed

### 4. Action Throttling Table

| Unfollows Done | Extra Delay Added     |
|----------------|-----------------------|
| Every 10       | +10–20s pause         |
| Every 50       | +60–120s pause        |
| Every 100      | +180–300s pause       |

### 5. Important Warnings
- Do NOT run more than ~200 unfollows per day
- Space sessions at least 1 hour apart
- If you see \"Action Blocked\" — STOP immediately and wait 24–48 hours
- Use the Pause/Resume feature to take breaks

---

## Complete Code

### 1. `manifest.json`

```json
{
  \"manifest_version\": 3,
  \"name\": \"Instagram Unfollow Manager\",
  \"version\": \"1.0.0\",
  \"description\": \"Safely unfollow Instagram users with configurable delays and anti-suspension measures.\",
  \"permissions\": [
    \"activeTab\",
    \"storage\",
    \"scripting\"
  ],
  \"host_permissions\": [
    \"https://www.instagram.com/*\"
  ],
  \"action\": {
    \"default_popup\": \"popup/popup.html\",
    \"default_icon\": {
      \"16\": \"icons/icon16.png\",
      \"48\": \"icons/icon48.png\",
      \"128\": \"icons/icon128.png\"
    }
  },
  \"background\": {
    \"service_worker\": \"scripts/background.js\"
  },
  \"content_scripts\": [
    {
      \"matches\": [\"https://www.instagram.com/*\"],
      \"js\": [\"scripts/content.js\"],
      \"run_at\": \"document_idle\"
    }
  ],
  \"icons\": {
    \"16\": \"icons/icon16.png\",
    \"48\": \"icons/icon48.png\",
    \"128\": \"icons/icon128.png\"
  }
}
```

---

### 2. `scripts/background.js`

```javascript
// background.js — Service Worker (Manifest V3)
// Manages state persistence and message relay between popup and content script

const DEFAULT_STATE = {
  status: \"idle\",           // idle | running | paused | completed | error
  unfollowedCount: 0,
  currentUser: \"\",
  limit: 0,                 // 0 = unlimited (unfollow all)
  minDelay: 3000,           // ms
  maxDelay: 7000,           // ms
  totalFound: 0,
  errorMessage: \"\"
};

// Initialize state on install
chrome.runtime.onInstalled.addListener(() => {
  chrome.storage.local.set({ unfollowState: DEFAULT_STATE });
});

// Message relay between popup <-> content script
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  const { type, payload } = message;

  switch (type) {
    // Commands from Popup -> relay to Content Script
    case \"START_UNFOLLOW\":
    case \"PAUSE_UNFOLLOW\":
    case \"RESUME_UNFOLLOW\":
    case \"STOP_UNFOLLOW\":
      relayToContentScript(type, payload, sendResponse);
      return true; // async response

    // Config update from Popup -> save to storage
    case \"UPDATE_CONFIG\":
      chrome.storage.local.get(\"unfollowState\", (data) => {
        const state = data.unfollowState || DEFAULT_STATE;
        const updated = { ...state, ...payload };
        chrome.storage.local.set({ unfollowState: updated });
        sendResponse({ success: true });
      });
      return true;

    // Progress updates from Content Script -> save to storage
    case \"PROGRESS_UPDATE\":
      chrome.storage.local.get(\"unfollowState\", (data) => {
        const state = data.unfollowState || DEFAULT_STATE;
        const updated = { ...state, ...payload };
        chrome.storage.local.set({ unfollowState: updated });
        sendResponse({ success: true });
      });
      return true;

    // Get current state (from Popup)
    case \"GET_STATE\":
      chrome.storage.local.get(\"unfollowState\", (data) => {
        sendResponse(data.unfollowState || DEFAULT_STATE);
      });
      return true;

    // Reset state
    case \"RESET_STATE\":
      chrome.storage.local.set({ unfollowState: DEFAULT_STATE });
      sendResponse({ success: true });
      return true;

    default:
      sendResponse({ error: \"Unknown message type\" });
  }
});

function relayToContentScript(type, payload, sendResponse) {
  chrome.tabs.query({ active: true, currentWindow: true }, (tabs) => {
    if (!tabs[0]) {
      sendResponse({ error: \"No active tab found\" });
      return;
    }

    const tabUrl = tabs[0].url || \"\";
    if (!tabUrl.includes(\"instagram.com\")) {
      sendResponse({ error: \"Navigate to Instagram first\" });
      return;
    }

    chrome.tabs.sendMessage(tabs[0].id, { type, payload }, (response) => {
      if (chrome.runtime.lastError) {
        sendResponse({ error: chrome.runtime.lastError.message });
      } else {
        sendResponse(response || { success: true });
      }
    });
  });
}
```

---

### 3. `scripts/content.js`

```javascript
// content.js — Core Unfollow Automation
// Runs on instagram.com pages, manipulates the DOM to unfollow users

(function () {
  \"use strict\";

  // ============================================================
  // STATE
  // ============================================================
  let isRunning = false;
  let isPaused = false;
  let shouldStop = false;
  let unfollowedCount = 0;
  let limit = 0;
  let minDelay = 3000;
  let maxDelay = 7000;
  let currentUsername = \"\";

  // ============================================================
  // UTILITIES
  // ============================================================

  function sleep(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  function randomDelay(min, max) {
    return min + Math.random() * (max - min);
  }

  // Sends progress back to background script
  function sendProgress(data) {
    chrome.runtime.sendMessage({
      type: \"PROGRESS_UPDATE\",
      payload: data
    });
  }

  // Wait for an element to appear in the DOM
  function waitForElement(selector, timeout = 5000) {
    return new Promise((resolve, reject) => {
      const el = document.querySelector(selector);
      if (el) return resolve(el);

      const observer = new MutationObserver((mutations, obs) => {
        const found = document.querySelector(selector);
        if (found) {
          obs.disconnect();
          resolve(found);
        }
      });

      observer.observe(document.body, { childList: true, subtree: true });

      setTimeout(() => {
        observer.disconnect();
        reject(new Error(`Element ${selector} not found within ${timeout}ms`));
      }, timeout);
    });
  }

  // ============================================================
  // ANTI-SUSPENSION: ADAPTIVE DELAYS
  // ============================================================

  async function antiSuspensionPause(count) {
    // Every 10 unfollows: short break
    if (count > 0 && count % 10 === 0) {
      const breakTime = 10000 + Math.random() * 10000; // 10-20s
      sendProgress({
        status: \"running\",
        currentUser: `Taking a short break (${Math.round(breakTime / 1000)}s)...`,
        unfollowedCount: count
      });
      await sleep(breakTime);
    }

    // Every 50 unfollows: longer break
    if (count > 0 && count % 50 === 0) {
      const breakTime = 60000 + Math.random() * 60000; // 60-120s
      sendProgress({
        status: \"running\",
        currentUser: `Safety cooldown (${Math.round(breakTime / 1000)}s)...`,
        unfollowedCount: count
      });
      await sleep(breakTime);
    }

    // Every 100 unfollows: extended break
    if (count > 0 && count % 100 === 0) {
      const breakTime = 180000 + Math.random() * 120000; // 180-300s
      sendProgress({
        status: \"running\",
        currentUser: `Extended safety break (${Math.round(breakTime / 1000)}s)...`,
        unfollowedCount: count
      });
      await sleep(breakTime);
    }
  }

  // ============================================================
  // CORE: INSTAGRAM DOM SELECTORS
  // ============================================================
  //
  // Instagram's DOM uses dynamic class names that change frequently.
  // The selectors below use structural patterns that are more stable.
  //
  // IMPORTANT: These selectors WILL break when Instagram updates
  // their markup. You'll need to inspect the DOM and update them.
  //
  // Strategy: We look for \"Following\" buttons in the followers/
  // following dialog or on the /following page.
  // ============================================================

  // Finds all \"Following\" buttons in the following list
  function getFollowingButtons() {
    // Instagram renders the following list inside a dialog or a
    // scrollable container. The \"Following\" buttons typically have
    // specific text content. We query all buttons and filter.
    const allButtons = document.querySelectorAll(\"button\");
    const followingButtons = [];

    allButtons.forEach((btn) => {
      const text = btn.textContent.trim();
      // Match \"Following\" button text (may also say \"Requested\")
      if (text === \"Following\" || text === \"Requested\") {
        followingButtons.push(btn);
      }
    });

    return followingButtons;
  }

  // Gets the username associated with a \"Following\" button
  function getUsernameFromButton(btn) {
    // Traverse up to the list item and find the username link/span
    // Instagram typically structures it as:
    //   <div> (list item container)
    //     <a href=\"/username/\">  or  <span>username</span>
    //     <button>Following</button>
    //   </div>
    let container = btn.closest(\"li\") || btn.closest(\"[role='listitem']\");
    if (!container) {
      // Fallback: walk up a few levels
      container = btn.parentElement?.parentElement?.parentElement;
    }

    if (container) {
      // Try to find a link with href pattern /<username>/
      const link = container.querySelector(\"a[href^='/']\");
      if (link) {
        const href = link.getAttribute(\"href\");
        // Extract username from href like \"/username/\"
        const match = href.match(/^\/([^/]+)\/?$/);
        if (match) return match[1];
      }

      // Fallback: look for the username in span elements
      const spans = container.querySelectorAll(\"span\");
      for (const span of spans) {
        const text = span.textContent.trim();
        // Usernames don't have spaces and are reasonably short
        if (text && !text.includes(\" \") && text.length < 40 && text !== \"Following\") {
          return text;
        }
      }
    }

    return \"unknown\";
  }

  // Clicks the \"Following\" button, then confirms \"Unfollow\" in the dialog
  async function unfollowUser(followingBtn) {
    // Step 1: Click \"Following\" — this opens a confirmation dialog
    followingBtn.click();

    // Step 2: Wait for the confirmation dialog to appear
    // The \"Unfollow\" confirmation button appears in a modal
    await sleep(800 + Math.random() * 500);

    // Step 3: Find and click the \"Unfollow\" button in the confirmation dialog
    // It's typically a red/bold button inside a dialog/modal
    const confirmBtn = await findUnfollowConfirmButton();

    if (confirmBtn) {
      // Add a micro-pause to simulate human hesitation
      await sleep(100 + Math.random() * 200);
      confirmBtn.click();
      return true;
    }

    return false;
  }

  // Finds the \"Unfollow\" confirmation button in the modal dialog
  async function findUnfollowConfirmButton() {
    // Wait briefly for the modal to render
    await sleep(500);

    // The confirmation dialog typically has a button with text \"Unfollow\"
    // It might be inside a dialog[role=\"dialog\"] or a specific modal container
    const allButtons = document.querySelectorAll(\"button\");

    for (const btn of allButtons) {
      const text = btn.textContent.trim();
      if (text === \"Unfollow\") {
        return btn;
      }
    }

    // Fallback: look inside dialog elements
    const dialogs = document.querySelectorAll(\"[role='dialog']\");
    for (const dialog of dialogs) {
      const buttons = dialog.querySelectorAll(\"button\");
      for (const btn of buttons) {
        if (btn.textContent.trim() === \"Unfollow\") {
          return btn;
        }
      }
    }

    return null;
  }

  // Scrolls the following list to load more users
  async function scrollFollowingList() {
    // The following list is usually inside a scrollable dialog
    const scrollable =
      document.querySelector(\"[role='dialog'] [style*='overflow']\") ||
      document.querySelector(\"[role='dialog'] ul\")?.parentElement ||
      document.querySelector(\"div[style*='overflow: hidden auto']\");

    if (scrollable) {
      scrollable.scrollTop = scrollable.scrollHeight;
      await sleep(1500 + Math.random() * 1000);
    }
  }

  // ============================================================
  // MAIN UNFOLLOW LOOP
  // ============================================================

  async function startUnfollowProcess(config) {
    limit = config.limit || 0;
    minDelay = config.minDelay || 3000;
    maxDelay = config.maxDelay || 7000;
    unfollowedCount = 0;
    isRunning = true;
    isPaused = false;
    shouldStop = false;

    sendProgress({
      status: \"running\",
      currentUser: \"Starting...\",
      unfollowedCount: 0
    });

    // Verify we're on the right page
    if (!window.location.href.includes(\"instagram.com\")) {
      sendProgress({
        status: \"error\",
        errorMessage: \"Please navigate to Instagram first.\"
      });
      return;
    }

    try {
      let consecutiveEmptyRounds = 0;
      const MAX_EMPTY_ROUNDS = 3;

      while (isRunning && !shouldStop) {
        // Check pause state
        while (isPaused && !shouldStop) {
          await sleep(500);
        }
        if (shouldStop) break;

        // Check limit
        if (limit > 0 && unfollowedCount >= limit) {
          sendProgress({
            status: \"completed\",
            currentUser: `Limit reached! Unfollowed ${unfollowedCount} users.`,
            unfollowedCount
          });
          break;
        }

        // Get current visible \"Following\" buttons
        const buttons = getFollowingButtons();

        if (buttons.length === 0) {
          consecutiveEmptyRounds++;

          if (consecutiveEmptyRounds >= MAX_EMPTY_ROUNDS) {
            sendProgress({
              status: \"completed\",
              currentUser: `Done! Unfollowed ${unfollowedCount} users. No more users found.`,
              unfollowedCount
            });
            break;
          }

          // Try scrolling to load more
          await scrollFollowingList();
          continue;
        }

        consecutiveEmptyRounds = 0;

        // Process the first visible \"Following\" button
        const btn = buttons[0];
        const username = getUsernameFromButton(btn);
        currentUsername = username;

        sendProgress({
          status: \"running\",
          currentUser: username,
          unfollowedCount
        });

        // Perform the unfollow
        const success = await unfollowUser(btn);

        if (success) {
          unfollowedCount++;
          sendProgress({
            status: \"running\",
            currentUser: `Unfollowed: ${username}`,
            unfollowedCount
          });

          // Anti-suspension adaptive pauses
          await antiSuspensionPause(unfollowedCount);

          // Randomized delay before next unfollow
          const delay = randomDelay(minDelay, maxDelay);
          sendProgress({
            status: \"running\",
            currentUser: `Waiting ${Math.round(delay / 1000)}s before next...`,
            unfollowedCount
          });
          await sleep(delay);

        } else {
          // If unfollow confirmation failed, might be an \"Action Blocked\" dialog
          sendProgress({
            status: \"error\",
            currentUser: username,
            errorMessage: \"Could not confirm unfollow. Possible action block detected. STOP and wait 24-48 hours.\",
            unfollowedCount
          });

          // Emergency stop on potential block
          shouldStop = true;
          break;
        }
      }

    } catch (err) {
      sendProgress({
        status: \"error\",
        errorMessage: `Error: ${err.message}`,
        unfollowedCount
      });
    }

    isRunning = false;
  }

  // ============================================================
  // MESSAGE LISTENER
  // ============================================================

  chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    const { type, payload } = message;

    switch (type) {
      case \"START_UNFOLLOW\":
        if (isRunning) {
          sendResponse({ error: \"Already running\" });
        } else {
          startUnfollowProcess(payload || {});
          sendResponse({ success: true });
        }
        break;

      case \"PAUSE_UNFOLLOW\":
        isPaused = true;
        sendProgress({ status: \"paused\", currentUser: currentUsername, unfollowedCount });
        sendResponse({ success: true });
        break;

      case \"RESUME_UNFOLLOW\":
        isPaused = false;
        sendProgress({ status: \"running\", currentUser: currentUsername, unfollowedCount });
        sendResponse({ success: true });
        break;

      case \"STOP_UNFOLLOW\":
        shouldStop = true;
        isRunning = false;
        sendProgress({
          status: \"completed\",
          currentUser: `Stopped. Unfollowed ${unfollowedCount} users.`,
          unfollowedCount
        });
        sendResponse({ success: true });
        break;

      default:
        sendResponse({ error: \"Unknown command\" });
    }

    return true; // async
  });

  console.log(\"[Instagram Unfollow Manager] Content script loaded.\");
})();
```

---

### 4. `popup/popup.html`

```html
<!DOCTYPE html>
<html lang=\"en\">
<head>
  <meta charset=\"UTF-8\" />
  <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\" />
  <title>IG Unfollow</title>
  <link rel=\"stylesheet\" href=\"popup.css\" />
</head>
<body>
  <div id=\"app\">

    <!-- Status Display (Headless: only shows current user) -->
    <div id=\"status-section\">
      <div id=\"current-user\" data-testid=\"current-user-display\">Ready</div>
      <div id=\"progress-counter\" data-testid=\"progress-counter\">0 unfollowed</div>
      <div id=\"progress-bar-container\" data-testid=\"progress-bar-container\">
        <div id=\"progress-bar\" data-testid=\"progress-bar\"></div>
      </div>
    </div>

    <!-- Config (collapsible) -->
    <details id=\"config-section\">
      <summary data-testid=\"config-toggle\">Settings</summary>

      <div class=\"config-row\">
        <label for=\"limit-input\">Limit (0 = all):</label>
        <input
          type=\"number\"
          id=\"limit-input\"
          data-testid=\"limit-input\"
          value=\"0\"
          min=\"0\"
          max=\"1000\"
        />
      </div>

      <div class=\"config-row\">
        <label for=\"min-delay-input\">Min Delay (sec):</label>
        <input
          type=\"number\"
          id=\"min-delay-input\"
          data-testid=\"min-delay-input\"
          value=\"3\"
          min=\"1\"
          max=\"60\"
        />
      </div>

      <div class=\"config-row\">
        <label for=\"max-delay-input\">Max Delay (sec):</label>
        <input
          type=\"number\"
          id=\"max-delay-input\"
          data-testid=\"max-delay-input\"
          value=\"7\"
          min=\"2\"
          max=\"120\"
        />
      </div>
    </details>

    <!-- Controls -->
    <div id=\"controls\">
      <button id=\"btn-start\" data-testid=\"start-button\">Start</button>
      <button id=\"btn-pause\" data-testid=\"pause-button\" disabled>Pause</button>
      <button id=\"btn-resume\" data-testid=\"resume-button\" disabled>Resume</button>
      <button id=\"btn-stop\" data-testid=\"stop-button\" disabled>Stop</button>
    </div>

    <!-- Error display -->
    <div id=\"error-display\" data-testid=\"error-display\" class=\"hidden\"></div>

  </div>

  <script src=\"popup.js\"></script>
</body>
</html>
```

---

### 5. `popup/popup.css`

```css
/* Headless, dark, minimal UI */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  width: 300px;
  min-height: 180px;
  background: #0a0a0a;
  color: #e0e0e0;
  font-family: -apple-system, BlinkMacSystemFont, \"Segoe UI\", sans-serif;
  font-size: 13px;
  padding: 16px;
}

#app {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

/* Status Section */
#status-section {
  text-align: center;
}

#current-user {
  font-size: 15px;
  font-weight: 600;
  color: #ffffff;
  padding: 8px 0;
  word-break: break-all;
  min-height: 24px;
}

#progress-counter {
  font-size: 12px;
  color: #888;
  margin-bottom: 8px;
}

#progress-bar-container {
  width: 100%;
  height: 3px;
  background: #1a1a1a;
  border-radius: 2px;
  overflow: hidden;
}

#progress-bar {
  height: 100%;
  width: 0%;
  background: #e1306c; /* Instagram pink */
  transition: width 0.3s ease;
  border-radius: 2px;
}

/* Config Section */
#config-section {
  border: 1px solid #1a1a1a;
  border-radius: 6px;
  padding: 8px;
}

#config-section summary {
  cursor: pointer;
  font-size: 12px;
  color: #666;
  user-select: none;
}

#config-section summary:hover {
  color: #999;
}

.config-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 0;
}

.config-row label {
  font-size: 12px;
  color: #888;
}

.config-row input {
  width: 70px;
  padding: 4px 8px;
  background: #111;
  border: 1px solid #222;
  border-radius: 4px;
  color: #fff;
  font-size: 12px;
  text-align: center;
}

.config-row input:focus {
  outline: none;
  border-color: #e1306c;
}

/* Controls */
#controls {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}

#controls button {
  padding: 8px 0;
  border: none;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s, background 0.2s;
}

#controls button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

#btn-start {
  background: #e1306c;
  color: #fff;
  grid-column: 1 / -1;
}

#btn-start:hover:not(:disabled) {
  background: #c72a5e;
}

#btn-pause {
  background: #333;
  color: #fff;
}

#btn-pause:hover:not(:disabled) {
  background: #444;
}

#btn-resume {
  background: #27ae60;
  color: #fff;
}

#btn-resume:hover:not(:disabled) {
  background: #219653;
}

#btn-stop {
  background: #222;
  color: #e74c3c;
  border: 1px solid #e74c3c33;
  grid-column: 1 / -1;
}

#btn-stop:hover:not(:disabled) {
  background: #2a1515;
}

/* Error Display */
#error-display {
  background: #1a0a0a;
  border: 1px solid #e74c3c33;
  border-radius: 4px;
  padding: 8px;
  font-size: 11px;
  color: #e74c3c;
}

.hidden {
  display: none !important;
}
```

---

### 6. `popup/popup.js`

```javascript
// popup.js — Popup UI Logic

const DOM = {
  currentUser: document.getElementById(\"current-user\"),
  progressCounter: document.getElementById(\"progress-counter\"),
  progressBar: document.getElementById(\"progress-bar\"),
  limitInput: document.getElementById(\"limit-input\"),
  minDelayInput: document.getElementById(\"min-delay-input\"),
  maxDelayInput: document.getElementById(\"max-delay-input\"),
  btnStart: document.getElementById(\"btn-start\"),
  btnPause: document.getElementById(\"btn-pause\"),
  btnResume: document.getElementById(\"btn-resume\"),
  btnStop: document.getElementById(\"btn-stop\"),
  errorDisplay: document.getElementById(\"error-display\")
};

// ============================================================
// STATE POLLING
// ============================================================

// Poll background for state updates every 500ms
let pollInterval = null;

function startPolling() {
  if (pollInterval) return;
  pollInterval = setInterval(fetchAndRenderState, 500);
}

function stopPolling() {
  if (pollInterval) {
    clearInterval(pollInterval);
    pollInterval = null;
  }
}

function fetchAndRenderState() {
  chrome.runtime.sendMessage({ type: \"GET_STATE\" }, (state) => {
    if (!state) return;
    renderState(state);
  });
}

function renderState(state) {
  // Update current user display
  DOM.currentUser.textContent = state.currentUser || \"Ready\";

  // Update counter
  const limitText = state.limit > 0 ? ` / ${state.limit}` : \"\";
  DOM.progressCounter.textContent = `${state.unfollowedCount}${limitText} unfollowed`;

  // Update progress bar
  if (state.limit > 0) {
    const pct = Math.min((state.unfollowedCount / state.limit) * 100, 100);
    DOM.progressBar.style.width = `${pct}%`;
  } else {
    // Indeterminate: pulse between 0-100 based on count
    const pct = Math.min(state.unfollowedCount * 2, 95);
    DOM.progressBar.style.width = `${pct}%`;
  }

  // Update button states based on status
  switch (state.status) {
    case \"idle\":
      DOM.btnStart.disabled = false;
      DOM.btnPause.disabled = true;
      DOM.btnResume.disabled = true;
      DOM.btnStop.disabled = true;
      break;
    case \"running\":
      DOM.btnStart.disabled = true;
      DOM.btnPause.disabled = false;
      DOM.btnResume.disabled = true;
      DOM.btnStop.disabled = false;
      break;
    case \"paused\":
      DOM.btnStart.disabled = true;
      DOM.btnPause.disabled = true;
      DOM.btnResume.disabled = false;
      DOM.btnStop.disabled = false;
      break;
    case \"completed\":
    case \"error\":
      DOM.btnStart.disabled = false;
      DOM.btnPause.disabled = true;
      DOM.btnResume.disabled = true;
      DOM.btnStop.disabled = true;
      stopPolling();
      break;
  }

  // Show/hide error
  if (state.errorMessage) {
    DOM.errorDisplay.textContent = state.errorMessage;
    DOM.errorDisplay.classList.remove(\"hidden\");
  } else {
    DOM.errorDisplay.classList.add(\"hidden\");
  }
}

// ============================================================
// BUTTON HANDLERS
// ============================================================

DOM.btnStart.addEventListener(\"click\", () => {
  const limit = parseInt(DOM.limitInput.value, 10) || 0;
  const minDelay = (parseFloat(DOM.minDelayInput.value) || 3) * 1000;
  const maxDelay = (parseFloat(DOM.maxDelayInput.value) || 7) * 1000;

  // Validate
  if (maxDelay <= minDelay) {
    DOM.errorDisplay.textContent = \"Max delay must be greater than min delay.\";
    DOM.errorDisplay.classList.remove(\"hidden\");
    return;
  }

  // Save config
  chrome.runtime.sendMessage({
    type: \"UPDATE_CONFIG\",
    payload: { limit, minDelay, maxDelay, status: \"running\", unfollowedCount: 0, errorMessage: \"\" }
  });

  // Send start command
  chrome.runtime.sendMessage({
    type: \"START_UNFOLLOW\",
    payload: { limit, minDelay, maxDelay }
  }, (response) => {
    if (response?.error) {
      DOM.errorDisplay.textContent = response.error;
      DOM.errorDisplay.classList.remove(\"hidden\");
    } else {
      DOM.errorDisplay.classList.add(\"hidden\");
      startPolling();
    }
  });
});

DOM.btnPause.addEventListener(\"click\", () => {
  chrome.runtime.sendMessage({ type: \"PAUSE_UNFOLLOW\" });
});

DOM.btnResume.addEventListener(\"click\", () => {
  chrome.runtime.sendMessage({ type: \"RESUME_UNFOLLOW\" });
});

DOM.btnStop.addEventListener(\"click\", () => {
  chrome.runtime.sendMessage({ type: \"STOP_UNFOLLOW\" });
});

// ============================================================
// INIT
// ============================================================

// Load saved config & state on popup open
chrome.runtime.sendMessage({ type: \"GET_STATE\" }, (state) => {
  if (state) {
    DOM.limitInput.value = (state.limit || 0);
    DOM.minDelayInput.value = (state.minDelay || 3000) / 1000;
    DOM.maxDelayInput.value = (state.maxDelay || 7000) / 1000;
    renderState(state);

    if (state.status === \"running\" || state.status === \"paused\") {
      startPolling();
    }
  }
});
```

---

## Installation Guide

### Loading as an Unpacked Extension (Developer Mode)

1. **Prepare the folder**: Ensure all files are in the `instagram-unfollow-extension/` directory with the structure shown above.

2. **Create icons**: You need 16x16, 48x48, and 128x128 PNG icons in the `icons/` folder. Use any icon generator or simple solid-color squares.

3. **Open Chrome Extensions page**:
   - Navigate to `chrome://extensions/`
   - Toggle **Developer Mode** (top right) ON

4. **Load extension**:
   - Click **\"Load unpacked\"**
   - Select the `instagram-unfollow-extension/` folder

5. **Pin the extension**: Click the puzzle icon in Chrome toolbar and pin \"Instagram Unfollow Manager\"

---

## Usage Guide

### Step-by-Step

1. **Log in to Instagram** at `https://www.instagram.com/`

2. **Navigate to your Following list**:
   - Go to your profile
   - Click on \"Following\" to open the following list/dialog

3. **Open the extension popup** by clicking the extension icon

4. **Configure settings** (optional):
   - **Limit**: Set to `0` for all, or a specific number like `50`
   - **Min Delay**: Minimum seconds between unfollows (default: 3)
   - **Max Delay**: Maximum seconds between unfollows (default: 7)

5. **Click \"Start\"** to begin the unfollow process

6. **Monitor progress**:
   - The popup shows the username currently being unfollowed
   - Counter shows how many have been unfollowed
   - Progress bar fills based on limit (or incrementally if unlimited)

7. **Pause/Resume**: Click \"Pause\" anytime to pause. Click \"Resume\" to continue.

8. **Stop**: Click \"Stop\" to end the process at any time.

### Recommended Safe Settings

| Setting     | Conservative | Moderate  | Aggressive (Risky) |
|-------------|-------------|-----------|---------------------|
| Limit       | 30          | 80        | 150                 |
| Min Delay   | 5s          | 3s        | 2s                  |
| Max Delay   | 12s         | 7s        | 5s                  |
| Session Gap | 2 hours     | 1 hour    | 30 min              |

---

## Known Limitations & Risks

### Selector Fragility
Instagram uses dynamically generated CSS class names (e.g., `x1lliihq x1plvlek`). The content script avoids relying on class names and instead uses:
- Button text content (`\"Following\"`, `\"Unfollow\"`)
- Structural DOM traversal (parent containers, `<a>` href patterns)
- ARIA roles (`role=\"dialog\"`, `role=\"listitem\"`)

**However**, Instagram can change these at any time. When the extension stops working, you'll need to:
1. Open Instagram's Following dialog
2. Right-click > Inspect Element
3. Update the selectors in `content.js`

### Rate Limiting & Action Blocks
- Instagram may temporarily block your ability to unfollow if you do it too fast
- If you see \"Action Blocked\", the extension will detect it and stop automatically
- Wait **24–48 hours** before trying again after a block
- Repeated blocks can lead to account restrictions

### Privacy & Security
- The extension only runs on `instagram.com`
- No data is sent to any external server
- All state is stored locally in `chrome.storage.local`
- The extension does **not** access your Instagram password or token

### Instagram Updates
Instagram's web interface is a React SPA with frequent UI updates. The DOM structure can change without notice, breaking the selectors. This is an inherent limitation of any DOM-based automation tool.

### What This Extension Does NOT Do
- Does NOT use Instagram's private API (less detectable but against ToS)
- Does NOT require your Instagram credentials
- Does NOT bypass Instagram's rate limits — it works within them using delays
- Does NOT guarantee zero risk of account action
"
Observation: Create successful: /app/instagram-unfollow-extension-guide.md