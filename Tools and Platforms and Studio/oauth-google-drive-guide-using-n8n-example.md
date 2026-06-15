# Complete Guide: Connecting n8n to Google Drive via OAuth

> A beginner-friendly deep-dive into OAuth, Google Cloud Console, Apps, Projects, Credentials, and how they all connect.

---

## Table of Contents

1. [What is OAuth?](#1-what-is-oauth)
2. [What is an "App" in Google Cloud?](#2-what-is-an-app-in-google-cloud)
3. [Managed vs Custom Credentials in n8n](#3-managed-vs-custom-credentials-in-n8n)
4. [Why Choose Custom Over Managed?](#4-why-choose-custom-over-managed)
5. [Step-by-Step: Creating Your Own Google Cloud App](#5-step-by-step-creating-your-own-google-cloud-app)
6. [Project vs App vs Credentials — The Hierarchy](#6-project-vs-app-vs-credentials--the-hierarchy)
7. [Understanding Scopes (Permissions)](#7-understanding-scopes-permissions)
8. [Testing Mode, Publishing & Verification](#8-testing-mode-publishing--verification)
9. [The Two Credentials — App vs User](#9-the-two-credentials--app-vs-user)
10. [Who Sees Your Consent Screen?](#10-who-sees-your-consent-screen)
11. [Common Errors & Fixes](#11-common-errors--fixes)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. What is OAuth?

**Question:** What is OAuth? I don't have much idea about it.

**Answer:** OAuth is a permission system. When a tool like n8n wants to access your Google Drive, Google won't just let it in — it needs YOUR permission first.

```
┌─────────────────────────────────────────────────────────────┐
│                     OAuth Flow (Simple)                     │
│                                                             │
│   n8n says: "I want to read this person's Drive"            │
│       │                                                     │
│       ▼                                                     │
│   Google says: "Let me ask them first"                      │
│       │                                                     │
│       ▼                                                     │
│   Google shows YOU a screen:                                │
│   "n8n wants to view your Drive files. Allow?"              │
│       │                                                     │
│       ▼                                                     │
│   You click "Allow"                                         │
│       │                                                     │
│       ▼                                                     │
│   Google gives n8n a TOKEN (temporary key)                  │
│       │                                                     │
│       ▼                                                     │
│   n8n uses that token to read your Drive                    │
│   (n8n NEVER sees your Google password)                     │
└─────────────────────────────────────────────────────────────┘
```

**Analogy:** It's like giving a valet a car key that only opens the door but can't open the trunk. You control what access you give.

---

## 2. What is an "App" in Google Cloud?

**Question:** What is an app? I keep hearing this term.

**Answer:** When software (like n8n) wants to talk to Google, Google requires it to **register** first. This registration is called an "App."

```
┌──────────────────────────────────────────────────┐
│              Google's Front Desk                  │
│                                                   │
│   Software arrives: "Hi, I'm n8n"                │
│                                                   │
│   Google: "Show me your ID"                      │
│                                                   │
│   Software registers → gets a VISITOR BADGE:     │
│     • Client ID     (like a username)            │
│     • Client Secret  (like a password)           │
│                                                   │
│   Now Google knows this software exists           │
│   and can track what it does.                    │
└──────────────────────────────────────────────────┘
```

**Key point:** The "app" is NOT a piece of software you build. It's just a **configuration/registration** inside Google Cloud Console — a name, some permissions, and credentials.

---

## 3. Managed vs Custom Credentials in n8n

**Question:** When connecting Google Drive in n8n, I see "Managed" and "Custom". What are they?

**Answer:**

| Feature | Managed | Custom |
|---------|---------|--------|
| Who created the app? | n8n (the company) | You |
| Setup required? | None — just click "Sign in" | You create app in Google Cloud Console |
| Works on n8n Cloud? | Yes | Yes |
| Works on self-hosted? | No | Yes |
| Whose Client ID is used? | n8n's | Yours |
| Consent screen says | "n8n" | Your app name (e.g., "n8n-automations") |

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   MANAGED:                                                  │
│   n8n's Client ID → n8n's consent screen → your login      │
│   (n8n already registered the app for you)                  │
│                                                             │
│   CUSTOM:                                                   │
│   Your Client ID → your consent screen → your login        │
│   (you register the app yourself)                           │
│                                                             │
│   In BOTH cases: YOU log in with YOUR Google account        │
│   In BOTH cases: n8n accesses YOUR Drive                    │
└─────────────────────────────────────────────────────────────┘
```

**Analogy:**
- **Managed** = Hotel concierge already has a taxi arranged. Just get in.
- **Custom** = Airbnb stay — you call and book your own taxi.

---

## 4. Why Choose Custom Over Managed?

**Question:** If Managed is easier, why would I use Custom? What extra control do I get?

**Answer:**

| Concern | Managed | Custom | Custom + Self-hosted |
|---------|---------|--------|----------------------|
| Control what permissions Google grants | ❌ No | ✅ Yes | ✅ Yes |
| Own API quota (not shared with others) | ❌ No | ✅ Yes | ✅ Yes |
| Revoke/rotate credentials yourself | ⚠️ Partial | ✅ Yes | ✅ Yes |
| Data stays off n8n's servers | ❌ No | ❌ No | ✅ Yes |

**Key differences explained:**

1. **Scopes:** Managed requests whatever permissions n8n decided. Custom lets YOU choose (e.g., read-only instead of full access).
2. **Rate limits:** Managed shares Google API quotas with ALL n8n Cloud users. Custom gives you your own quota.
3. **Credentials:** Custom lets you rotate or delete the Client Secret anytime.
4. **Data flow:** Regardless of Managed/Custom, n8n Cloud still processes your data on their servers. Only Custom + self-hosted gives you full data control.

---

## 5. Step-by-Step: Creating Your Own Google Cloud App

**Question:** Walk me through creating an app. Explain what's happening at each step.

### Step 1: Go to Google Cloud Console
- Open [console.cloud.google.com](https://console.cloud.google.com/)
- This is Google's admin panel for developers. It's free.

### Step 2: Create a New Project
- Click project dropdown → "New Project" → Name it (e.g., `n8n-automations`) → Create

```
What's happening:
A "project" is a CONTAINER that groups together:
  • Which APIs you've enabled
  • Which credentials you've created  
  • Your usage/quota
  • Billing
Think of it as a folder for organizing everything.
```

### Step 3: Enable the Google Drive API
- Go to APIs & Services → Library → Search "Google Drive API" → Enable

```
What's happening:
By default, your project can't talk to any Google service.
You explicitly turn on each API you need.
This is a SECURITY measure — your app can only access 
services you've consciously enabled.

⚠️ APIs are PROJECT-level. Enabling Drive API in Project A 
   does NOT enable it in Project B.
```

### Step 4: Configure OAuth Consent Screen
- Go to APIs & Services → OAuth consent screen
- Fill in: App name, your email, choose "External"

```
What's happening:
This configures the popup that appears when someone 
clicks "Sign in with Google":

  ┌──────────────────────────────────┐
  │                                  │
  │  "n8n-automations" wants to      │
  │  access your Google Account      │
  │                                  │
  │  [Cancel]         [Allow]        │
  └──────────────────────────────────┘

• "App name" = what appears on this popup
• "External" = anyone with a Google account can authorize
• Your app starts in "Testing" mode automatically
```

### Step 5: Add Scopes (⚡ THIS IS WHERE YOU CONTROL ACCESS)
- Click "Add or Remove Scopes" → Search for `drive`

```
Available Google Drive scopes:

┌──────────────────────────────────────┬────────────────────────────────────┐
│ Scope                                │ What it allows                     │
├──────────────────────────────────────┼────────────────────────────────────┤
│ drive.metadata.readonly              │ See file names, dates, sizes ONLY  │
│                                      │ (cannot read file contents)        │
├──────────────────────────────────────┼────────────────────────────────────┤
│ drive.readonly                       │ READ files (cannot edit/delete)    │
│                                      │ ⚠️ Marked as "restricted" because │
│                                      │ it can read file CONTENTS          │
├──────────────────────────────────────┼────────────────────────────────────┤
│ drive.file                           │ Access ONLY files created by your  │
│                                      │ app or explicitly opened by user   │
├──────────────────────────────────────┼────────────────────────────────────┤
│ drive                                │ FULL access — read, write, create, │
│                                      │ delete ANY file. Most permissive.  │
└──────────────────────────────────────┴────────────────────────────────────┘

Recommendation: Start with drive.readonly for learning.
You can always add more scopes later.
```

**Why is `drive.readonly` under "Restricted Scopes"?**

Google classifies scopes into 3 sensitivity levels:

| Level | Meaning | Example |
|-------|---------|---------|
| Non-sensitive | Basic profile info | `openid`, `email` |
| Sensitive | Access to user data, but limited | `drive.metadata.readonly` |
| Restricted | Access to actual file contents | `drive.readonly`, `drive` |

Even read-only is "restricted" because it can read the CONTENTS of files (documents, photos, etc.).

### Step 6: Add Yourself as a Test User
- Enter your Gmail address and save

```
What's happening:
Since your app is in "Testing" mode, ONLY listed test users 
can authorize it. If you skip this, you'll get an error.
Test users are at PROJECT level (tied to the consent screen).
```

### Step 7: Create OAuth Credentials
- Go to Credentials → Create Credentials → OAuth client ID
- Type: Web application
- Add redirect URI from n8n (e.g., `https://app.n8n.cloud/rest/oauth2-credential/callback`)

```
What's happening:

• Client ID = public identifier for your app (like a username)
• Client Secret = private key (like a password — keep it safe!)
• Redirect URI = where Google sends the token after you click "Allow"
  (must match EXACTLY what n8n expects — most common source of errors)
```

### Step 8: Paste into n8n & Connect
- In n8n: Create credential → OAuth2 (Custom) → Paste Client ID & Secret
- Click "Sign in with Google" → Allow → Done!

```
What happens behind the scenes:

  n8n                       Google                    You
   │                           │                        │
   │── Client ID + scopes ────►│                        │
   │                           │── Login page ─────────►│
   │                           │                        │
   │                           │◄── Email + password ───│
   │                           │                        │
   │                           │── Consent screen ─────►│
   │                           │   "Allow drive.readonly?"
   │                           │                        │
   │                           │◄── "Allow" ────────────│
   │                           │                        │
   │◄── Access token ─────────│                        │
   │    (expires ~1 hour)      │                        │
   │◄── Refresh token ────────│                        │
   │    (long-lived, auto-     │                        │
   │     refreshes tokens)     │                        │
   │                           │                        │
   │── "List files" + token ──►│                        │
   │◄── Here are the files ───│                        │

You do this ONCE. n8n stores the tokens and auto-refreshes them.
You never have to sign in again unless you revoke access.
```

---

## 6. Project vs App vs Credentials — The Hierarchy

**Question:** What's the difference between Project, App, and Credentials? I'm confused about where everything lives.

**Answer:** Everything lives under the Project.

```
Google Cloud Account (your Google login)
  │
  └── Project: "n8n-automations" ◄── TOP-LEVEL CONTAINER
        │
        ├── Enabled APIs ◄── Project level
        │     ├── Google Drive API ✅
        │     ├── Google Sheets API ✅
        │     └── Gmail API ❌ (not enabled)
        │
        ├── Billing & Quotas ◄── Project level
        │     └── 10,000 API calls/day (shared across all credentials)
        │
        ├── App (OAuth Consent Screen) ◄── ONE per project
        │     ├── Name: "n8n-automations"
        │     ├── Scopes: drive.readonly, sheets.readonly
        │     ├── Test Users: you@gmail.com
        │     └── Publishing Status: Testing
        │
        └── Credentials ◄── Multiple per project
              ├── Client ID #1 (for n8n)
              └── Client ID #2 (for a script)
              
              Both share the SAME consent screen,
              SAME scopes, SAME quota pool.
```

### Key relationships:

| Entity | Level | How many? | Purpose |
|--------|-------|-----------|---------|
| **Project** | Top | Unlimited per Google account | Container for everything |
| **App** (Consent Screen) | Inside project | **ONE per project** | Identity shown to users |
| **Credentials** (Client ID/Secret) | Inside project | Multiple per project | Authentication keys |
| **APIs** | Inside project | Multiple per project | Which services are available |
| **Scopes** | Inside app | Multiple per app | What permissions to request |
| **Test Users** | Inside app | Up to 100 | Who can authorize (in testing mode) |
| **Billing/Quotas** | Inside project | One per project | Usage tracking & limits |

### The "one consent screen" limitation:

```
⚠️ You CANNOT have different permissions per credential within the same project.

❌ This is NOT possible:
   Project
     ├── Client ID #1 → only Drive access
     └── Client ID #2 → only Sheets access

✅ To achieve this, create SEPARATE projects:
   Project A: "n8n-drive"
     └── Client ID #1 → drive.readonly
   
   Project B: "n8n-sheets"  
     └── Client ID #2 → sheets
```

---

## 7. Understanding Scopes (Permissions)

**Question:** Where do I control what n8n can and can't do with my Drive?

**Answer:** Scopes are set at the **App level** (consent screen configuration). They define the MAXIMUM permissions any credential in that project can request.

```
Scope Flow:

  Project enables API  →  App configures scopes  →  Consent screen shows scopes  →  User approves
        │                        │                          │                            │
  "Drive API is ON"     "We'll ask for          "This app wants to         "I allow this"
                         read-only access"       view your files"
```

**The scope is a CONTRACT between three parties:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Developer   │     │   Google     │     │    User      │
│  (you)       │     │              │     │  (you, in    │
│              │     │              │     │  this case)  │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ Configures   │────►│ Validates    │────►│ Sees what's  │
│ which scopes │     │ the scopes   │     │ being asked  │
│ to request   │     │ and shows    │     │ and decides  │
│              │     │ consent      │     │ to allow or  │
│              │     │ screen       │     │ deny         │
└──────────────┘     └──────────────┘     └──────────────┘
```

---

## 8. Testing Mode, Publishing & Verification

**Question:** What is testing mode? What's the difference between publishing and verification?

### The Three States of an App:

```
┌─────────────┐   one-click   ┌─────────────┐   submit to    ┌─────────────┐
│             │   (instant)   │             │   Google       │             │
│   TESTING   │──────────────►│  PUBLISHED  │──────────────►│  VERIFIED   │
│             │               │             │  (days/weeks)  │             │
├─────────────┤               ├─────────────┤               ├─────────────┤
│ • Max 100   │               │ • Anyone    │               │ • Anyone    │
│   test users│               │   can use   │               │   can use   │
│             │               │             │               │             │
│ • Tokens    │               │ • Tokens    │               │ • Tokens    │
│   expire    │               │   DON'T     │               │   DON'T     │
│   in 7 days │               │   expire    │               │   expire    │
│             │               │             │               │             │
│ • Warning   │               │ • Warning   │               │ • NO        │
│   shows     │               │   STILL     │               │   warning   │
│             │               │   shows     │               │             │
│ • Free,     │               │ • Free,     │               │ • Free,     │
│   automatic │               │   instant   │               │   takes     │
│   (default) │               │   (1 click) │               │   days/weeks│
└─────────────┘               └─────────────┘               └─────────────┘
  YOU ARE HERE                                                
```

### Testing Mode (your current state):
- **Set automatically** when you chose "External" — you didn't do anything special.
- Only test users you listed can authorize.
- Shows "Google hasn't verified this app" warning — **this is normal, not an error**.
- Tokens expire in 7 days.

### Publishing:
- **You do it** — one click in Google Console (OAuth consent screen → Publish App).
- Google does NOT review anything. It's instant.
- Removes the 100-user limit and 7-day token expiry.
- The "unverified" warning STILL shows.

### Verification:
- **Google's team manually reviews** your project configuration (NOT your code).
- YOU must submit it — it's never automatic.
- Requires: privacy policy, website, domain verification, YouTube demo video, justification.
- Removes the "unverified" warning entirely.
- Only needed for public-facing apps (Zapier, Notion, etc.) — **NOT for personal use**.

### The "Google hasn't verified this app" warning:

```
When you see this, just click through:

  "Google hasn't verified this app"
           │
           ▼
     Click "Advanced" (small text at bottom-left)
           │
           ▼
     Click "Go to n8n-automations (unsafe)"
           │
           ▼
     Shows scopes → Click "Allow"
           │
           ▼
     Done ✅
```

### What does Google "review"?
They review your **project's consent screen configuration** — NOT any software:
- What scopes did you configure?
- Do you have a privacy policy?
- Is your domain verified?
- Does a video demo show why you need those permissions?

---

## 9. The Two Credentials — App vs User (⚡ KEY CONCEPT)

**Question:** I'm confused. When I connect to Google Drive, whose credentials are used? If someone uses my website, whose credentials are used?

**Answer:** There are TWO completely separate credentials involved in every OAuth flow:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   CREDENTIAL #1: App Identification                                │
│   ┌───────────────────────────────────────────────┐                │
│   │ Client ID + Client Secret                      │                │
│   │                                                │                │
│   │ • Set ONCE during setup                        │                │
│   │ • The user NEVER sees this                     │                │
│   │ • Purpose: tells Google WHICH APP is asking    │                │
│   │ • Does NOT determine whose data is accessed    │                │
│   └───────────────────────────────────────────────┘                │
│                                                                     │
│   CREDENTIAL #2: User Login                                        │
│   ┌───────────────────────────────────────────────┐                │
│   │ Google email + password                        │                │
│   │                                                │                │
│   │ • Entered by the PERSON granting access        │                │
│   │ • Different every time a new person connects   │                │
│   │ • Purpose: tells Google WHO is granting access │                │
│   │ • DETERMINES whose data is accessed            │                │
│   └───────────────────────────────────────────────┘                │
│                                                                     │
│   These are INDEPENDENT. Don't mix them up.                        │
└─────────────────────────────────────────────────────────────────────┘
```

### Example 1: You connecting Google Drive in n8n

```
Your Client ID (app credential) ──► Google knows: "app n8n-automations is asking"
Your Google login (user credential) ──► Google knows: "you@gmail.com is granting"
Result: n8n gets token for YOUR Drive
```

### Example 2: Someone else using a website you built (hypothetical)

```
Your Client ID (app credential) ──► Google knows: "app n8n-automations is asking"
Bob's Google login (user credential) ──► Google knows: "bob@gmail.com is granting"
Result: your website gets token for BOB's Drive (NOT yours)
```

### Summary table:

| | App credential (Client ID) | User credential (Google login) |
|---|---|---|
| **Who provides it** | The developer (you) | The person granting access |
| **Set when** | Once, during setup | Every time someone connects |
| **Purpose** | Tells Google "this is app X" | Tells Google "this is person Y" |
| **Whose data is accessed** | Nobody's — just identifies the app | This person's data |
| **Visible to the user?** | No (hidden in the URL) | Yes (they type it on Google's login page) |

---

## 10. Who Sees Your Consent Screen?

**Question:** You said "anyone who gets your consent screen URL." How would anyone see my consent screen?

**Answer:** For YOUR use case (personal n8n automation) — **nobody else will ever see it.**

### What the consent screen URL looks like:

```
https://accounts.google.com/o/oauth2/v2/auth
  ?client_id=YOUR_CLIENT_ID          ← identifies your app
  &redirect_uri=https://app.n8n.cloud/rest/oauth2-credential/callback
  &scope=https://www.googleapis.com/auth/drive.readonly
  &response_type=code
```

### Your scenario (personal n8n):

```
Your n8n account (private, only you can log in)
         │
         ▼
You click "Sign in with Google" (only you see this button)
         │
         ▼
n8n generates the URL with your Client ID
         │
         ▼
Opens in YOUR browser → Google shows consent screen to YOU
         │
         ▼
You click Allow → n8n gets access to YOUR Drive
         │
         ▼
Nobody else is involved. Ever.
```

### Hypothetical: If you built a public website (NOT your case):

```
You build "myapp.com" with a "Connect Google Drive" button
         │
         ▼
User visits myapp.com → clicks the button
         │
         ▼
Your website generates the URL with YOUR Client ID
         │
         ▼
Google shows YOUR consent screen on THEIR screen
         │
         ▼
They log in with THEIR Google account
         │
         ▼
They click Allow → your website gets access to THEIR Drive
(NOT your Drive — because THEY logged in, not you)
```

### Key point:

Nobody can see your consent screen unless they have a URL containing your Client ID. Since your Client ID only lives inside your private n8n credential settings, **no one else will ever encounter it**.

---

## 11. Common Errors & Fixes

### Error: "OAuth Authorization Error — Status 520"

**Cause:** Cloudflare/n8n server issue (NOT a Google error).

**Fixes:**
1. Check that the **redirect URI** in Google Console matches EXACTLY what n8n shows
2. Try again in a few minutes (could be transient)
3. Try in an incognito/private browser window
4. Check [status.n8n.io](https://status.n8n.io) for outages

### Error: "Google hasn't verified this app"

**Cause:** Normal behavior for apps in "Testing" mode. NOT an error.

**Fix:** Click Advanced → "Go to [app name] (unsafe)" → Allow

### Tokens keep expiring (every 7 days)

**Cause:** App is still in "Testing" mode.

**Fix:** In Google Console → OAuth consent screen → Publishing status → Click "Publish App"

---

## 12. Quick Reference Cheat Sheet

### The Complete Picture:

```
┌──────────────────────────────────────────────────────────────────┐
│                     GOOGLE CLOUD ACCOUNT                         │
│                     (your Google login)                           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    PROJECT                                 │  │
│  │              (top-level container)                          │  │
│  │                                                            │  │
│  │  ┌──────────────────┐  ┌─────────────────────────────────┐│  │
│  │  │   Enabled APIs   │  │   Billing & Quotas              ││  │
│  │  │  • Drive API ✅  │  │   • 10,000 calls/day            ││  │
│  │  │  • Sheets API ✅ │  │   • Shared across all           ││  │
│  │  │  • Gmail API ❌  │  │     credentials                 ││  │
│  │  └──────────────────┘  └─────────────────────────────────┘│  │
│  │                                                            │  │
│  │  ┌────────────────────────────────────────────────────────┐│  │
│  │  │              APP (Consent Screen)                       ││  │
│  │  │              ⚡ ONE per project                         ││  │
│  │  │                                                        ││  │
│  │  │  Name: "n8n-automations"                               ││  │
│  │  │  Scopes: drive.readonly                                ││  │
│  │  │  Test Users: you@gmail.com                             ││  │
│  │  │  Status: Testing → Published → Verified                ││  │
│  │  │                                                        ││  │
│  │  │  This is the "real player" — the configuration model   ││  │
│  │  │  that defines what permissions are asked for and       ││  │
│  │  │  what users see on the consent popup.                  ││  │
│  │  └────────────────────────────────────────────────────────┘│  │
│  │                                                            │  │
│  │  ┌────────────────────────────────────────────────────────┐│  │
│  │  │              CREDENTIALS                               ││  │
│  │  │              (can have multiple)                        ││  │
│  │  │                                                        ││  │
│  │  │  Client ID #1 ──► used by n8n                          ││  │
│  │  │  Client ID #2 ──► used by another tool                 ││  │
│  │  │                                                        ││  │
│  │  │  All share same consent screen & scopes.               ││  │
│  │  │  For different scopes → create separate projects.      ││  │
│  │  └────────────────────────────────────────────────────────┘│  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### The OAuth Flow (Complete):

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│   n8n   │         │ Google  │         │   You   │
│(the app)│         │         │         │(the user│
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     │ 1. Client ID +    │                   │
     │    scopes         │                   │
     │──────────────────►│                   │
     │                   │ 2. Login page     │
     │                   │──────────────────►│
     │                   │                   │
     │                   │ 3. Email +        │
     │                   │    password       │
     │                   │◄──────────────────│
     │                   │                   │
     │                   │ 4. Consent screen │
     │                   │──────────────────►│
     │                   │ "Allow read-only  │
     │                   │  access to Drive?"│
     │                   │                   │
     │                   │ 5. "Allow"        │
     │                   │◄──────────────────│
     │                   │                   │
     │ 6. Access token + │                   │
     │    Refresh token  │                   │
     │◄──────────────────│                   │
     │                   │                   │
     │ 7. API call +     │                   │
     │    access token   │                   │
     │──────────────────►│                   │
     │                   │                   │
     │ 8. Drive data     │                   │
     │◄──────────────────│                   │
     │                   │                   │
```

### Decision Tree: Which option should I use?

```
Are you using n8n Cloud?
  │
  ├── YES
  │     │
  │     ├── Just want it to work fast? → Use MANAGED
  │     │
  │     └── Want to control permissions/learn? → Use CUSTOM
  │           │
  │           └── Create app in Google Cloud Console
  │               Use Testing mode (click through warning)
  │               Publish app if tokens expire after 7 days
  │
  └── NO (self-hosted)
        │
        └── You MUST use CUSTOM (managed not available)
              │
              └── Create app in Google Cloud Console
                  Publish app (so tokens don't expire)
```

---

> **Created from a learning conversation. Feel free to share!**
> 
> Last updated: June 6, 2026
