# agent.md

Guide for AI coding agents working on **Bunutan** (Filipino gift draw / Secret Santa).

## What this is

A single-page app with no build step. Everything lives in `index.html` (markup, CSS, and one `<script type="module">`). Firebase is loaded from the CDN. Hosting is static (GitHub Pages).

| File | Purpose |
|---|---|
| `index.html` | The whole app |
| `favicon.svg`, `gift-icon.svg`, `gift-icon.png` | Static icons |

There is no package.json, bundler, or test suite. Do not add one unless asked.

## Running and checking

- Serve the folder over https or `localhost` (Web Crypto needs a secure context): `python3 -m http.server 8000`.
- Sign-in needs the domain listed in Firebase > Authentication > Authorized domains.
- Quick syntax check of the script: copy the `<script type="module">` body to a `.mjs` file, remove the `import` lines, run `node --check`.
- Test crypto changes in Node (`globalThis.crypto.subtle`) by copying the helper functions.

## Backend

Firebase Auth (Google sign-in) plus Realtime Database through the **REST API** (`callRaw`, `dbGet/dbPut/dbPatch/dbDel`), authenticated with the user's ID token. `FIREBASE_CONFIG` is at the top of the script. The web API key is public by design; access is controlled by database Rules.

Data model:

```
bunutan/{sid}/
  info: { title, date, budget, giftTheme, subthemes{}, createdAt, adminHash, owner, theme,
          escrow?: { pub, s, iv, ct } }   // date is 'YYYY-MM-DD' when escrow exists
  p/{pid}: { name, claimedBy, claimedName, claimedAt,
             pw:{s,v},            // salt + verifier for the person's password
             taken,               // blind HMAC tag marking a name as already drawn by someone
             enc:{iv,ct},         // pick, AES-GCM with the person's password key
             encA,                // pick, RSA-OAEP for the admin (only if escrow exists)
             drewAt, wishlist{}, giftee /* legacy plaintext, old draws only */ }
  chat/{pid}/{mid}: { from:'giver'|'giftee', t, i, at }
bunutan_users/{uid}/draws/{sid}: { title, key, createdAt }   // creator's own list
```

Admin = whoever has the `k` value in the URL hash (`#s=<sid>&k=<key>`); `sha(k)` must equal `info.adminHash`. Participants use `#s=<sid>`.

## Privacy model (do not weaken)

1. The database must only ever hold ciphertext for picks. Never write a plaintext giver-to-giftee pair.
2. Each participant sets a password; keys come from PBKDF2 (200k) then HKDF (`deriveKeys`). The password and keys stay in memory only (`myKeys`).
3. **Exchange-date view (optional per draw, only when a gift exchange date is set at creation):**
   - A fresh RSA-OAEP key pair is made in the browser. The public key goes in `info.escrow.pub`. The private key is encrypted with an AES key derived from the admin link secret (`k` in `#s=...&k=...`), with the draw id as AES-GCM additional data. There is no admin password.
   - On drawing, each person's browser also writes `encA` (the giftee id encrypted to the public key).
   - On or after `info.date` (ISO `YYYY-MM-DD`, device local date), the Admin dashboard's "All picks" card (`viewRevealAll`) offers "Show all picks". `adminShowAll()` unlocks the private key using `k`, and `refreshAll()` decrypts picks into `allPicks`. Both are in memory only and cleared on "Hide all", reload, or if the date no longer applies.
   - Before the date, the admin can only open one person's pick with that person's own password (point 4). Other participants can never see anyone's pick.
   - The date is enforced by the app, not by the cryptography. To stop easy early peeking, the exchange date of a draw with escrow can only be moved later, never earlier (`saveInfo`).
   - Participants are told about this in `privacyNotice()`, the set-password screen and the drawn screen (`seeNote()`). Keep those notices truthful if you change this.
   - Draws without `info.escrow` (no date at creation, and all older draws) stay fully private, and their date remains free text.
4. The older per-person reveal (admin enters that person's password) still exists and is independent.

Known limits: the date check uses the device clock and anyone holding the admin link could bypass it with developer tools, and Rules allow any signed-in user to read and write a draw. The cryptography is what protects the picks, not the UI.

## Code layout inside the script

Roughly top to bottom: config and helpers, state, DB helpers, drum-roll audio, password lock and escrow crypto, image compression, notifications, auth, themes, views (`viewHome`, `viewDash`, `viewMe`, ...), chat, actions (`createSession`, `claim`, `performDraw`, `draw`, `setPassword`, `unlockPick`, `adminReveal`, `adminShowAll`, ...), the delegated click, change and keydown handlers, then `load` (polls every 4 seconds) and `boot`.

## Conventions

- Plain JS, no frameworks. Views are string-built HTML; always escape user data with `esc()`.
- Buttons use `data-act` (and `data-v`), handled in the single click listener on `#app`.
- `render()` rebuilds the DOM; form values survive via `keepInputs`/`restoreInputs`, which match inputs by `id`. Give every input a unique `id`.
- Reuse existing CSS classes (`card`, `row`, `lk`, `notice`, `btn`, `mut`, ...) and the theme variables (`--gold`, `--card`, ...). Both themes (forest, halloween) must keep working.
- Sensitive state (keys, decrypted picks) lives in memory only. Reset it in `boot()`.
- The app is used on phones; keep layouts to a single column and tap targets large.
- Keep copy plain and friendly. Match the existing tone.

## When changing things

- Edit `index.html` in place with small, targeted changes. Do not reformat unrelated code.
- After edits: syntax-check the script and test any crypto change end to end (create with a date, claim, draw, reveal).
- Changing the data model? Keep old draws working (see the legacy `giftee` handling) and update this file.
