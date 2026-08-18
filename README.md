# Equipment Check-Out / Check-In System

A single-page app for scanning equipment out to associates and back in, live
across every shift. It's one self-contained `index.html` file backed by
Firestore, so any device on any shift that loads the page sees the same
real-time data.

## How it works

- Every piece of equipment gets a printed barcode/QR label with a unique code.
- Every associate gets a printed badge/card with their own unique code.
- **Check out:** scan an available item's code, then scan the associate's
  badge. The item is now assigned to that person.
- **Check in:** scan a checked-out item's code by itself. It's returned to
  "available" automatically — no need to scan the associate again.
- All scans are logged with a timestamp in the History tab.
- The Manage tab lets you add/remove equipment and associates, change a
  device's status (Available / In Repair / Lost), and print their QR labels.
  Marking something In Repair or Lost logs who made the change (whoever's
  signed in) and lets you attach an optional ticket number, both of which
  show up in the History tab.
- The Audit tab lets you do a physical walk-through: start an audit, scan
  every item you actually find on the floor, and finish it to generate a PDF
  report — flagging anything missing, anything found that the system still
  shows as checked out, and anything recovered that had been marked lost.
  Past audits are saved (with who ran them) and can be re-downloaded as a PDF
  any time.
- The whole app requires signing in. The Accounts tab (visible once you're
  logged in) is where you add new logins and deactivate old ones.

Scanning is done with a USB or Bluetooth barcode/QR scanner, which behaves
like a keyboard: it "types" the code into the scan box and sends Enter. No
camera or app install needed — just plug the scanner into whatever device
(PC, tablet) is sitting at the check-out station for that shift.

## One-time setup

### 1. Firebase project (Firestore)

Since you're already using Firestore for another project, you can reuse that
same Firebase project or create a new one — a new project keeps this data
cleanly separated, which is recommended.

1. Go to the [Firebase console](https://console.firebase.google.com/).
2. Create a project (or open your existing one).
3. In the left nav, go to **Build → Firestore Database** → **Create
   database** → start in production mode (rules below lock it down anyway).
4. Go to **Project settings → General → Your apps**, click the web icon
   (`</>`) to register a new web app (no Hosting needed here, just the
   config).
5. Copy the `firebaseConfig` object it gives you.

### 2. Paste your config into index.html

Open `index.html`, find this block near the bottom of the file, and replace
the placeholder values with your real config:

```js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
```

### 3. Turn on email/password sign-in

The app now requires a login. In the Firebase console, go to **Build →
Authentication → Sign-in method**, click **Email/Password**, enable it, and
save. (If you haven't opened Authentication before, it may ask you to "Get
started" first — that's fine, just enable Email/Password once you're in.)

### 4. Firestore security rules (replace entirely — don't just add to the old ones)

The rules changed shape to support logins: instead of "anyone can read/write,"
they now require you to be a signed-in, active user. In Firestore → Rules,
replace whatever is there with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isActiveUser() {
      return request.auth != null &&
        exists(/databases/$(database)/documents/users/$(request.auth.uid)) &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.active == true;
    }

    match /equipment/{id} { allow read, write: if isActiveUser(); }
    match /associates/{id} { allow read, write: if isActiveUser(); }
    match /logs/{id} { allow read, write: if isActiveUser(); }
    match /audits/{id} { allow read, write: if isActiveUser(); }

    match /users/{uid} {
      allow read: if request.auth != null && (request.auth.uid == uid || isActiveUser());
      allow write: if isActiveUser();
    }
  }
}
```

Click **Publish**.

### 5. Bootstrap your first account (one-time, manual)

This is the one step the app can't do for itself: with logins required, you
need at least one account to exist before anyone can sign in and use the
Accounts tab to add everyone else. Do this once, directly in the Firebase
console:

1. Go to **Build → Authentication → Users → Add user**. Enter your email and
   a password, then click **Add user**. Copy the **User UID** shown next to
   your new entry — you'll need it in the next step.
2. Go to **Build → Firestore Database → Data**. Click **Start collection**
   (or **+ Add collection** if you already have others), name it `users`,
   and for the document ID paste in the UID you copied. Add these fields to
   the document:
   - `email` (string) — your email address
   - `displayName` (string) — your name
   - `active` (boolean) — `true`
3. Save the document.

Now you can open the live page, sign in with that email/password, and use
the **Accounts** tab to add everyone else properly — new accounts you add
there don't need any of this manual Firestore-console work; the "Add User"
form handles it.

If you want tighter control later (e.g. role-based permissions, or a
"forgot password" flow — there isn't one built in yet, so keep track of
passwords you hand out), that's a straightforward next step — just ask.

### 6. Host it (GitHub Pages)

Since you already use GitHub for another project, the easiest path:

1. Create a new repo (e.g. `equipment-checkout`), or a folder in an existing
   one.
2. Commit `index.html` to it.
3. In the repo, go to **Settings → Pages**, set the source to your main
   branch (root), and save.
4. GitHub gives you a URL like `https://yourorg.github.io/equipment-checkout/`
   — that's the link every shift bookmarks on their check-out station.

### 7. Add your equipment and associates

Open the live page → **Manage / Labels** tab:

- Add each piece of equipment (pick a short code, e.g. `EQ-001`, or scan a
  blank barcode you've generated elsewhere and let the scanner fill in the
  code field).
- Add each associate the same way (e.g. `ASSOC-014`).
- When adding an associate, you can optionally give them a **Shift** (e.g.
  `A1`, `A2`, `B1`) — handy for filtering labels later.
- Adding equipment or an associate now checks the code first — if that code
  is already taken, it tells you who it belongs to instead of silently
  overwriting them.
- Need to add a lot of associates at once? Use **Bulk Import Associates** in
  the Manage tab: paste one per line as `Name, Shift` (shift is optional),
  click **Import List**, and badge codes (`ASSOC-0001`, `ASSOC-0002`, …) are
  assigned automatically. Names that already exist are skipped and reported
  so nobody gets duplicated.
- Switch the "Print Labels" dropdown between Equipment/Associates, then use
  the **Search** box to find one specific person or item by name or code
  (or the **Shift** dropdown, for associates, to print just one shift's
  badges) instead of printing everything. Click **Print This Set** and
  print. Stick equipment labels on the gear and hand out/laminate associate
  badges.

## Everyday use

Leave the page open on a shared PC/tablet at the check-out station with a
scanner plugged in, on the **Scan** tab. Nothing needs to be clicked between
scans — the input box stays focused automatically. If someone clicks
elsewhere on the page by accident, clicking anywhere on the Scan tab
re-focuses it.

Since the app now requires a login, whoever sets up the shared check-out
station just needs to sign in **once** — the browser stays signed in across
reloads and between shifts, so this doesn't slow down day-to-day scanning.
You'd only need to sign in again if someone explicitly clicks "Log Out," or
if it's a brand-new browser/device that's never signed in before.

## About the account system's limits

This runs entirely client-side (no server), so a couple of things work a
little differently than a full-blown user-management system:

- **Deactivating someone doesn't delete their login**, it just blocks them
  from using the app (checked right after sign-in, and enforced again by the
  Firestore rules). True deletion of a login would need a small server-side
  piece (a Cloud Function with admin privileges) — worth adding later if you
  need it, but not required for day-to-day use.
- **There's no "forgot password" flow yet.** If someone forgets theirs,
  you'd need to add them again with a new temporary password from the
  Accounts tab, or add a password-reset email flow later.
- **Passwords you set in "Add User" are shared as plain text between you and
  that person** (there's no email delivery of credentials) — hand them off
  securely and ask people to keep track of their own password after that.

## Notes / things to decide later if useful

- **Overdue alerts** — right now nothing flags an item that's been out too
  long. This could be added (e.g. highlight in red after N hours, or a daily
  summary) if that becomes useful.
- **Photos/categories** — the data model has a spot for a category; photos
  of equipment could be added too.
- **Multiple locations** — if different sites need separate equipment pools,
  add a `location` field and filter by it; the current design is single-site.
- **Reporting/export** — the History tab shows the last 300 scans; exporting
  to CSV or building simple utilization reports off the `logs` collection is
  a natural next step.
