# Test Coverage Analysis — Nilea-site

## Current State

The codebase has **zero automated tests** and **no testing infrastructure**. There is no `package.json`, no test framework, no test runner, and no CI pipeline. All JavaScript is embedded inline within HTML files.

The project consists of 7 files total — 4 HTML pages, a CNAME, robots.txt, and sitemap.xml. The only file with meaningful logic is `app/index.html`, which contains ~300 lines of JavaScript handling authentication, session management, data fetching, and UI state.

---

## Where Tests Would Have the Most Impact

### 1. Authentication Logic (High Priority)

**File:** `app/index.html`, lines 205–234

The `handleAuthRedirect()` function handles two distinct auth flows (code-based and hash-based) and cleans the URL afterward. This is the entry point for every authenticated session.

**What to test:**
- Code parameter is extracted and exchanged for a session via `exchangeCodeForSession`
- Hash-based auth parameters (`access_token`, `refresh_token`, `token_hash`, `type`) are detected correctly
- URL is cleaned after redirect handling (both query params and hash removed)
- Error from `exchangeCodeForSession` sets a user-visible status message
- No action taken when neither code nor hash params are present

**Why it matters:** A bug here silently breaks login for all users. The two-path logic (code vs. hash) is the kind of branching that's easy to regress.

---

### 2. `usernameFromUser()` (High Priority)

**File:** `app/index.html`, lines 198–203

This pure function extracts a display name from a Supabase user object with multiple fallback layers.

**What to test:**
- Returns `user_metadata.name` when present and non-empty
- Falls back to email local part (before `@`) when no metadata name
- Falls back to `"there"` when both email and metadata are missing
- Handles `null`/`undefined` user, missing email, email without `@`
- Trims whitespace-only metadata names (should fall through to email)

**Why it matters:** This is a pure function with no side effects — the easiest kind of code to unit test, and it has real edge cases (whitespace names, missing fields).

---

### 3. `buildTypeformUrl()` (High Priority)

**File:** `app/index.html`, lines 261–267

Constructs the Typeform embed URL with the user's ID appended as a query parameter.

**What to test:**
- Appends `?user_id=<id>` when base URL has no existing query string
- Appends `&user_id=<id>` when base URL already has query params
- Returns `""` when base URL is empty or whitespace
- Properly encodes special characters in userId via `encodeURIComponent`

**Why it matters:** If this URL is malformed, the intake form breaks silently. The `?` vs `&` separator logic is a classic source of bugs.

---

### 4. `getIntakeCompleted()` (Medium Priority)

**File:** `app/index.html`, lines 269–287

Queries Supabase for the user's most recent intake submission and returns a status object.

**What to test:**
- Returns `{ completed: true, lastCreatedAt: <timestamp> }` when a submission exists
- Returns `{ completed: false, lastCreatedAt: null }` when no submissions exist
- Returns `{ completed: false, lastCreatedAt: null }` on query error (fail-closed behavior)
- Queries with the correct `user_id`, ordering, and limit

**Why it matters:** This function controls what the user sees on their dashboard. The fail-closed error handling is a deliberate design choice that should be asserted.

---

### 5. UI State Transitions in `render()` (Medium Priority)

**File:** `app/index.html`, lines 289–336

The `render()` function validates the session, fetches intake status, and toggles visibility on multiple DOM elements. It orchestrates several sub-states.

**What to test:**
- Unauthenticated: calls `hardResetToLoggedOut()`, hides loader/loggedIn, shows loggedOut
- Authenticated + no intake: shows "Start intake" button, status text "Intake not submitted yet"
- Authenticated + intake completed: shows "Add more" button, status text "Intake received" with timestamp
- Welcome message includes the correct username and email

**Why it matters:** With multiple elements being shown/hidden in specific combinations, it's easy for a change to leave the UI in an inconsistent state (e.g., showing both the login form and the dashboard).

---

### 6. `hardResetToLoggedOut()` (Medium Priority)

**File:** `app/index.html`, lines 236–259

Resets all UI elements to the logged-out state. This is called on logout, auth errors, and session expiry.

**What to test:**
- All expected elements are hidden (loader, loggedIn, logoutBtn, embedSection, closeIntakeBtn, intakeState)
- loggedOut section is shown
- Iframe src is reset to `about:blank`
- Button labels and disabled states are reset
- Status message is cleared

**Why it matters:** If this function misses resetting a single element, users could see stale data from a previous session — a privacy/security issue.

---

### 7. Sign-in Flow (Event Handler) (Lower Priority)

**File:** `app/index.html`, lines 339–363

The magic link sign-in handler validates input, disables the button during the request, calls `signInWithOtp`, and shows success/error status.

**What to test:**
- Empty email shows "Please enter your email" status
- Button is disabled and shows "Sending..." during the API call
- Button is re-enabled after completion (success or failure)
- Error from Supabase shows "Couldn't send link" message
- Success shows "Check your email" message
- `emailRedirectTo` uses `window.location.origin + "/app/"`

---

### 8. Logout Flow (Lower Priority)

**File:** `app/index.html`, lines 417–437

Handles logout with a 1.2-second timeout fallback in case `signOut` hangs.

**What to test:**
- Calls `supabase.auth.signOut({ scope: "global" })`
- Falls back to `hardResetToLoggedOut()` after 1200ms timeout
- Clears the timeout on successful logout
- Shows error status if signOut fails
- Button is disabled and shows "Logging out..." during the process

---

### 9. Intake Embed Open/Close (Lower Priority)

**File:** `app/index.html`, lines 366–414

Controls the Typeform iframe: opening it with the correct URL, closing it, and handling load/error events.

**What to test:**
- Opening sets iframe src to the built Typeform URL
- Opening shows the embed section and close button
- If user is no longer authenticated, redirects to logged-out state
- Closing hides the embed and resets iframe to `about:blank`
- Iframe load event clears the loading state
- Iframe error event shows a status message
- Does not reload if the embed is already open with the same URL

---

## Recommended Approach

### Step 1: Extract JavaScript from HTML

The biggest barrier to testing is that all logic is embedded inline in `app/index.html` inside a `<script type="module">` tag. Before adding tests:

- Create an `app/app.js` (or `src/app.js`) file with the extracted logic
- Export pure functions (`usernameFromUser`, `buildTypeformUrl`) and any testable modules
- Keep the HTML file importing the extracted module

### Step 2: Initialize a Minimal Build/Test Setup

```
npm init -y
npm install --save-dev vitest jsdom
```

Vitest with jsdom is lightweight and sufficient for this project. No bundler is needed for tests — Vitest handles ES modules natively.

### Step 3: Start with Pure Function Tests

Write tests for `usernameFromUser` and `buildTypeformUrl` first. These require no mocking and provide immediate value:

```js
// Example: app.test.js
import { usernameFromUser, buildTypeformUrl } from './app.js';
import { describe, it, expect } from 'vitest';

describe('usernameFromUser', () => {
  it('returns metadata name when present', () => {
    const user = { email: 'a@b.com', user_metadata: { name: 'Alice' } };
    expect(usernameFromUser(user)).toBe('Alice');
  });

  it('falls back to email local part', () => {
    const user = { email: 'a@b.com' };
    expect(usernameFromUser(user)).toBe('a');
  });

  it('returns "there" when no info available', () => {
    expect(usernameFromUser(null)).toBe('there');
  });
});

describe('buildTypeformUrl', () => {
  it('appends user_id with ? when no existing params', () => {
    expect(buildTypeformUrl('abc')).toContain('?user_id=abc');
  });

  it('appends user_id with & when params exist', () => {
    // after setting TYPEFORM_EMBED_URL to include ?foo=bar
    expect(buildTypeformUrl('abc')).toContain('&user_id=abc');
  });
});
```

### Step 4: Add Integration Tests with Mocked Supabase

Mock the Supabase client to test `getIntakeCompleted`, `handleAuthRedirect`, and `render` without hitting a real backend.

### Step 5: Consider E2E Tests (Optional, Long-term)

Playwright could cover the full login/dashboard flow against a staging Supabase instance, but this is a larger investment and only worthwhile once the product stabilizes.

---

## Summary Table

| Area | Priority | Test Type | Difficulty | Risk if Untested |
|---|---|---|---|---|
| `usernameFromUser()` | High | Unit | Easy | Wrong display name, crashes |
| `buildTypeformUrl()` | High | Unit | Easy | Broken intake form |
| `handleAuthRedirect()` | High | Integration | Medium | Login completely broken |
| `getIntakeCompleted()` | Medium | Integration | Medium | Wrong dashboard state |
| `render()` UI states | Medium | Integration | Medium | Inconsistent UI |
| `hardResetToLoggedOut()` | Medium | Integration | Medium | Stale session data shown |
| Sign-in event handler | Lower | Integration | Medium | UX bugs on login |
| Logout flow | Lower | Integration | Medium | Can't log out |
| Intake embed open/close | Lower | Integration | Low | Embed doesn't load |
