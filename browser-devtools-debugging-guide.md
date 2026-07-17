# Browser DevTools debugging guide

This guide uses Chrome/Edge (Chromium DevTools) panel names, since they're the most common. Firefox is very similar but names two panels differently: **Storage** instead of Application, **Debugger** instead of Sources. Everything else lines up closely enough to follow along.

## Opening DevTools

| Action | Windows/Linux | Mac |
|---|---|---|
| Open DevTools | `F12` or `Ctrl+Shift+I` | `Cmd+Option+I` |
| Open Console directly | `Ctrl+Shift+J` | `Cmd+Option+J` |
| Inspect element mode | `Ctrl+Shift+C` | `Cmd+Option+C` |
| Right-click any element | Inspect | Inspect |

## Quick decision guide

- No error anywhere, but something looks wrong on screen → **Elements**
- Red text in the console → **Console**, then **Sources** to trace it
- A button/feature calls an API and something's off → **Network**
- Backend is confirmed fixed but the user still sees old data → **Application**
- console.log guessing isn't cutting it and you need to watch values change → **Sources**

---

## 1. Console

**What it's for:** JS errors and warnings the browser catches on its own, your own log output, and a live JS prompt against the running page.

### Reading error types

| Error | What it usually means |
|---|---|
| `Uncaught TypeError: Cannot read properties of undefined (reading 'x')` | You called a method or accessed a property on `undefined`/`null` — the value you expected never arrived |
| `Uncaught ReferenceError: x is not defined` | A variable/function is used before it exists or is out of scope |
| `Uncaught SyntaxError` | Malformed JS — rare in shipped code, common when parsing JSON or template strings |
| `Uncaught (in promise) ...` | An unhandled promise rejection — somewhere a `.catch()` is missing |
| `Failed to fetch` / `NetworkError when attempting to fetch resource` | The request never completed at all: CORS block, offline, wrong URL, or mixed content (calling `http://` from an `https://` page) |
| `Access to fetch at '...' has been blocked by CORS policy` | The *server's* response is missing the right CORS headers — not something you can fix from the client |
| `Refused to ... because it violates the following Content Security Policy directive` | CSP is blocking a script or resource from loading |

**Reading a stack trace:** the top line is the error message; each line below it is a call, most recent first. Every line is clickable and jumps straight to Sources. Skip past minified vendor bundle frames and find the first frame that's your own code — that's where to start looking, even if it's not literally where the crash happened.

### Useful console commands

```js
console.log(value)                 // basic output
console.table(arrayOfObjects)      // renders an array of objects as an actual table — great for API responses
console.error('msg'); console.warn('msg')   // colored, filterable by severity
console.group('label'); /* logs */ console.groupEnd()   // nests related logs together
console.trace()                    // prints a stack trace from wherever you call it — find out who's calling a function
console.assert(condition, 'msg')   // only logs if condition is false
console.count('label')             // counts how many times this line runs — catches "why did this fire 5 times"
console.time('label'); /* code */ console.timeEnd('label')  // measure how long something takes

$0          // the element currently selected in the Elements panel
$_          // the value of the last expression you evaluated
$$('div.card')   // shorthand for document.querySelectorAll, returns a real array
copy(someValue)  // copies a JS value to your clipboard as text
monitorEvents($0, 'click')  // logs every click event fired on the selected element (Chrome/Edge only)
```

### Example walkthrough

Console shows:
```
Uncaught TypeError: Cannot read properties of undefined (reading 'map')
    at UserList.jsx:24
```
1. Click the `UserList.jsx:24` link → jumps to Sources at that exact line: `users.map(u => ...)`
2. `users` is undefined. Set a breakpoint on that line, or add `console.log(users)` just above it and reload.
3. The logged value shows the API response now looks like `{ data: { users: [...] } }` instead of `{ users: [...] }`.

The console error pointed at the frontend, but the actual bug is an API contract change — the console just told you *where* to start looking, not *whose* code is at fault.

---

## 2. Network

**What it's for:** every HTTP request the page makes — exactly what was sent, what came back, and how long each part took.

### How to check a request

1. Open the Network tab **before** triggering the action (or turn on **Preserve log** if the action navigates or reloads the page — otherwise the log clears).
2. Trigger the action in the app.
3. Find the request. Filter by **Fetch/XHR** to hide image/CSS/font noise, or search by a URL fragment.
4. Click it and check each sub-tab:
   - **Headers** — request headers (is the auth token actually attached? right `Content-Type`?) and response headers (CORS headers present? `Cache-Control`?)
   - **Payload / Request** — exactly what your client sent. Compare it against what the backend actually expects.
   - **Response / Preview** — exactly what came back. Preview renders JSON as a collapsible tree; Response shows the raw text.
   - **Timing** — the waterfall: Queueing → Stalled → DNS → Initial connection → SSL → Request sent → Waiting (TTFB) → Content download. Tells you whether the slowness is DNS, a slow backend, or a genuinely huge payload.

### Status codes — what to actually infer

| Code | Meaning | What it tells you |
|---|---|---|
| 200 | OK | Worked. If the UI is still wrong, the bug is client-side, not network/backend. |
| 201 / 204 | Created / No Content | Write succeeded (204 has no response body — don't expect one). |
| 301 / 302 | Redirect | Check the `Location` header — could be an auth redirect you didn't expect. |
| 304 | Not Modified | The browser served its own cached copy. If you expected fresh data, this is your stale-data culprit. |
| 400 | Bad Request | Your request payload is malformed or missing required fields. |
| 401 | Unauthorized | No, invalid, or expired credentials. |
| 403 | Forbidden | Authenticated, but not allowed — a permissions issue. |
| 404 | Not Found | Wrong URL/route, or the resource genuinely doesn't exist. |
| 409 | Conflict | E.g. duplicate resource, version mismatch. |
| 422 | Unprocessable Entity | Payload is well-formed but fails validation. |
| 429 | Too Many Requests | Rate limited. |
| 500 | Internal Server Error | Unhandled exception on the server — this is now a backend bug, go read server logs. |
| 502 / 504 | Bad Gateway / Gateway Timeout | The server behind a proxy or load balancer crashed or didn't respond in time — an infra issue, not your app code. |
| 503 | Service Unavailable | Server intentionally down or overloaded (deploy in progress, failing health check). |
| *(no status / "failed")* | Never reached a server | DNS failure, CORS blocked before it left, mixed content blocked, or `net::ERR_CONNECTION_REFUSED`. |

### Useful features

- **Preserve log** — keeps entries across page loads/navigations
- **Disable cache** (checkbox, only active while DevTools is open) — forces fresh requests, ruling out browser cache as the cause
- **Throttling** dropdown — simulate slow 3G to catch race conditions and loading-state bugs
- Right-click a request → **Copy → Copy as cURL** — reproduce the exact request outside the browser to isolate frontend from backend
- Right-click → **Copy as fetch** — same idea, as a JS snippet

### Example walkthrough

A login button appears to do nothing. Network tab shows `POST /api/login` → status `401` → Response tab shows `{"error":"invalid credentials"}`. The frontend correctly sent the request, so this isn't a frontend bug at all — you've eliminated Browser and jumped straight to "check the backend/auth logic," without guessing.

---

## 3. Elements (DOM)

**What it's for:** inspecting what's actually rendered and what styles are actually applied — for things that aren't visible, are misaligned, or don't react to clicks.

### How to check

- Right-click the element on the page → **Inspect**, or use the selector tool (cursor icon, top-left of DevTools).
- **Is it in the DOM at all?** If it's missing entirely, the bug is upstream — conditional render logic, a failed API call, an early return.
- **Styles pane** — every CSS rule applying, in cascade order, with a strikethrough on any rule that's overridden. Look for the strikethrough on the rule you expected to win.
- **Computed tab** — the final resolved value for every property. Check for `display: none`, `visibility: hidden`, `opacity: 0`, a `width`/`height` of `0`, or a parent with `overflow: hidden` / a low `z-index` clipping it.
- **Box model diagram** (bottom of Computed tab) — visualizes padding/border/margin, useful for layout bugs.
- **Event Listeners tab** (right-hand panel) — shows exactly which handlers are attached to the selected element. If you expect a `click` handler and it's not listed, JS never attached it.
- **Break on → subtree modifications / attribute modifications** (right-click an element) — pauses execution the instant something changes that element, and shows the JS call stack responsible.

### Example walkthrough

A button is visible but clicking it does nothing. Elements → select the button → Event Listeners tab → no `click` entry at all. That means the JS that should attach the handler never ran — back in Console, you find an earlier uncaught error that stopped the script before it reached the `addEventListener` call.

---

## 4. Application (storage)

**What it's for:** everything the browser is persisting for the site — local/session storage, cookies, cached responses, service workers.

### How to check

- **Local Storage / Session Storage** (left sidebar) — every key/value pair for the origin. Edit a value directly to see how the app reacts, or delete a key to simulate a fresh user.
- **Cookies** — check the actual value and the flags: `HttpOnly` (invisible to JS — if your JS is trying to read it, that's the bug), `Secure` (won't send over plain HTTP), `SameSite` (can block it from being sent on cross-site requests).
- **Clear site data** button — wipes storage/cookies/cache in one click so you can retest from a genuinely clean state, ruling out corrupted local state.
- **Service Workers** — shows what's registered and its status. An old worker still serving a cached page or API response is a very common cause of "I fixed it but the user still sees the old version."
- **Cache Storage** — the actual cached request/response pairs a service worker is serving from.

### Example walkthrough

A backend fix is confirmed deployed, but a user still sees old data. Application → Service Workers shows an old version stuck "waiting to activate." Application → Cache Storage shows the stale response still cached. Unregistering the worker (or bumping its cache version) resolves it — a cache-layer bug wearing a "backend is broken" disguise.

---

## 5. Sources (the actual debugger)

**What it's for:** setting breakpoints and stepping through your real running code — the most precise tool once `console.log` guessing stops being enough.

### How to check

- Find your file in the left-hand file tree, or `Ctrl/Cmd+P` to fuzzy-search by filename.
- Click a line number to set a breakpoint — execution pauses there the next time that line runs.
- Right-click a line number → **Add conditional breakpoint** — only pauses when an expression is true (e.g. `userId === 42`), useful when a bug only shows up for specific data.
- Add a `debugger;` statement directly in your source as a breakpoint that travels with the code (remove before shipping).
- While paused, use the right-hand panels: **Watch** (track specific expressions), **Scope** (every variable in scope and its current value), **Call Stack** (the exact chain of calls that got you here — click any frame to inspect that level).
- Step controls: **Step over** (next line, skip into function calls), **Step into** (enter the function call), **Step out** (finish the current function, return to caller), **Resume** (continue to the next breakpoint).
- The `{}` **pretty-print** icon at the bottom of the code pane un-minifies bundled code so line numbers and variable names are actually readable.

### Example walkthrough

An order total is calculated wrong. Set a breakpoint on the `return total` line inside `calculateTotal()`. Reload, trigger the calculation, execution pauses. The Scope panel shows `taxRate` is `undefined` instead of `0.08`. Step out to the caller — the Call Stack shows `taxRate` was never passed as an argument by the parent component. Bug found: a missing prop, three function calls away from where the wrong number actually showed up.

---

## General tips

- Dock DevTools to the side rather than the bottom on wide screens — more room for the Network waterfall and Sources side-by-side.
- Device toolbar (`Ctrl/Cmd+Shift+M`) reproduces mobile-only bugs, including touch events and viewport-specific CSS.
- "Disable cache" only takes effect while DevTools is open — it resets the moment you close the panel.
- A bug that disappears "when I open DevTools" is usually timing-sensitive — DevTools slows execution slightly, which is a strong hint you're dealing with a race condition rather than a logic error.
