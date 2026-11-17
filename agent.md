# agent.md

Guide for AI coding agents working on **Bunutan** (Filipino gift draw / Secret Santa).

## What this is

A single-page app with no build step. Everything lives in `index.html` (markup, CSS, and one `<script type="module">`). Firebase is loaded from the CDN. Hosting is static (GitHub Pages).

| File | Purpose |
|---|---|
| `index.html` | The whole app |
| `favicon.svg`, `gift-icon.svg`, `gift-icon.png` | Static icons |
| `demo.html` | Offline test copy of the app (mock backend in localStorage, fake accounts, sample draw). Not for production |

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
          escrow?: { pub, s, iv, ct },   // date is 'YYYY-MM-DD' when escrow exists
          assign?: { [giverPid]: gifteePid } }   // admin-set assigned draws, PLAINTEXT (the one exception, see "Assigned draws")
  p/{pid}: { name, claimedBy, claimedName, claimedAt,
             pw:{s,v},            // salt + verifier for the person's password
             taken,               // blind HMAC tag marking a name as already drawn by someone
             enc:{iv,ct},         // pick, AES-GCM with the person's password key
             encA,                // pick, RSA-OAEP for the admin (only if escrow exists)
             drewAt, wishlist{}, giftee /* legacy plaintext, old draws only */ }
  chat/{pid}/{mid}: { from:'giver'|'giftee', t, i, at }
bunutan_users/{uid}/draws/{sid}: { title, key, createdAt }   // creator's own list
bunutan_users/{uid}/joined/{sid}: { title, name, joinedAt }   // draws where the person holds a name (no pick, no key); written by rememberJoin() when they claim, or on first open if they claimed earlier
```

Admin = whoever has the `k` value in the URL hash (`#s=<sid>&k=<key>`); `sha(k)` must equal `info.adminHash`. Participants use `#s=<sid>`.

## Privacy model (do not weaken)

1. The database must only ever hold ciphertext for picks. Never write a plaintext giver-to-giftee pair. **The only exception is `info.assign` (admin-set assigned draws, see "Assigned draws").** Real picks (`enc`, `encA`) stay encrypted, including for people who were assigned.
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

Known limits: admin-set assigned draws (`info.assign`) are stored readable, so anyone who inspects the database of that draw can see them (accepted for friends' draws, chosen by the owner). The date check uses the device clock and anyone holding the admin link could bypass it with developer tools, and Rules allow any signed-in user to read and write a draw. The cryptography is what protects the picks, not the UI.

## Assigned draws (admin only, behind the skull)

Lets the draw creator decide that specific people must draw specific people (for example Person A draws Person C, Person B draws Person F), or arrange a loop or chain (A draws B, B draws C, C draws A). It is meant to be unnoticeable: participants see no notice, no extra screen and no different wording. Do **not** add any "set by the draw creator" disclaimer, the owner explicitly does not want one.

**Status:** implemented in `index.html` (skull tab, `viewSkull`, `assignProblem`, `saveAssign`, loop/chain builder, and the `performDraw` changes). This section is still the spec; keep the code and this text in step.

Implementation notes:
- `performDraw` has a local `take(c)` helper (blind tag, wait, verify). Assigned givers call it for their giftee and throw `conflict` if they lose it; `draw()` shows the neutral "No names are available" message for both `empty` and `conflict`.
- `assignProblem` skips the drawn/taken checks for rules that are unchanged, so old rules never block an unrelated edit. `makeLoop` also refuses to overwrite an existing rule for the same giver (remove it first).
- The giver dropdown lists undrawn people with no rule; the giftee dropdown lists people who are not taken and not already assigned. Loop/chain name buttons list everyone and the validator reports any problem.
- Logic was checked with a Node simulation (3 to 9 people, random rules, random draw order): no one drew themselves, assigned people always got their giftee, and no draw dead-ended.

### Access
- A skull icon button (`SKULL`, outline SVG, same style as `LOCK`/`EYE`) is the last item of the admin tab row in `viewTabs()`. `viewTabs()` is only rendered for the admin, so participants never see it.
- Tapping it toggles `skullOpen`. The panel (`viewSkull()`) renders under the tabs on both admin tabs. Its visible title is "Debugger" (shown only after unlocking); the locked screen has no title and there is no intro paragraph, so nothing on screen says "assigned".
- The panel is locked until the password **`Bunutan`** (case-sensitive) is entered. The check is `await sha(input) === SKULL_HASH`, where `SKULL_HASH = 'e98ef513dc73f745f7df16c5488405b45d210c532d2239e7da6c916e2c9cbc91'` (SHA-256 of `Bunutan`). This is only a UI gate, not security; do not describe it as encryption.
- Unlocked state is `skullOk`, in memory only. Reset `skullOpen`, `skullOk`, `loopSel` and `loopClose` in `boot()`, and on "Lock and close" or when the skull is toggled closed. Enter in the password field (`id="skp"`) submits.

### Data
- `info.assign = { [giverPid]: gifteePid }`, written with `dbPut(sid + '/info/assign', map)`, or `dbDel(sid + '/info/assign')` when the map is empty. Always write the whole map. `saveInfo` uses `dbPatch` on named fields only, so it does not touch `assign`; keep it that way. `createSession` does not write `assign`.
- Deleting the draw deletes it with the rest of `bunutan/{sid}`.

### Editor (all inside `viewSkull()`)
- Rows `Giver → Giftee` with a "Remove" button, hidden once the giver has drawn (then the row says "already drawn").
- Two `<select>` boxes (`id="asg"` giver, `id="ast"` assigned giftee) and "Add assignment". The giver list holds people who have not drawn and have no rule. The giftee list holds people who are not taken (`taken`/legacy `takenBy`) and not already assigned.
- "Loop or chain": the admin taps names in order (`loopSel`, shown as "1. Ann", "2. Bo"...) and sees a preview such as `Bo → Di → Ed → Bo`. `loopClose` (default Yes) adds the last-to-first rule. Names[i] → names[i+1]; two names with loop-back is a swap, three names is a 3-way loop. "Create loop" saves all rules or none.
- "Remove all undrawn assignments" and "Lock and close".
- `keepInputs`/`restoreInputs` must include `select` elements, and the 4-second poll must not re-render while a `select` has focus.

### Validation (`assignProblem(map)`), reject with a plain message
- A person cannot draw themselves.
- A giftee can be assigned to one giver only.
- A giver who already drew cannot get a new or changed rule.
- A giftee who is already taken (drawn by someone) cannot be assigned.
- The rest of the draw must stay solvable. After applying the rules, let G = people who have not drawn and have no rule, and R = people who are not taken and not assigned. G and R always have the same size. It is impossible only when exactly one person is left in each and they are the same person ("would leave X with only themselves to draw"). With two or more left, a valid matching always exists.

### Drawing (`performDraw`)
- Read `info/assign` (`dbGet(sid + '/info/assign')`) at draw time. `reserved` = its values.
- If the drawer has a rule: the pick is that giftee, taken with the same blind tag + verify step as a random pick. If the giftee is already taken by someone else, throw `conflict` (shown with the same neutral "No names are available to draw right now" message). Keep the spinner behaviour: it still spins and lands on the real pick.
- If the drawer has no rule: pool = people who are not themselves, not taken, **not reserved**. Dead-end check uses only non-forced people: givers = undrawn, no rule, not me; receivers = not taken, not reserved, not the candidate. Reject a candidate if exactly one giver and one receiver would remain and they are the same person.
- Everything else is unchanged: password lock (`enc`), blind `taken` tag, `encA` for the exchange-date view (still written for assigned givers too), resume of an interrupted draw.

### Tests to run after changing this
- Admin sees the skull, participants do not and see no "assigned" wording anywhere.
- Wrong password and lowercase `bunutan` are rejected; `Bunutan` opens the editor.
- Validation cases above (self, duplicate giftee, drawn giver, only-themselves).
- A real multi-person draw with a specific rule and a 3-way loop: assigned people get exactly their giftee, everyone else gets a valid draw, nobody draws themselves. Try Cy-first order (a leftover person who must not take the last free name).

## Code layout inside the script

Roughly top to bottom: config and helpers, state, DB helpers, drum-roll audio, password lock and escrow crypto, image compression, notifications, auth, themes, views (`viewHome`, `viewDash`, `viewMe`, ...), chat, actions (`createSession`, `claim`, `performDraw`, `draw`, `setPassword`, `unlockPick`, `adminReveal`, `adminShowAll`, ...), the delegated click, change and keydown handlers, then `load` (polls every 4 seconds) and `boot`. The assigned-draws helpers (`assigns`, `assignProblem`, `saveAssign`, `viewSkull`, `unlockSkull`, `addAssign`, `makeLoop`, `delAssign`, `clearAssign`) sit just above `viewTabs`.

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

## demo.html (testing copy)

`demo.html` is `index.html` with Firebase removed. `callRaw` is replaced by a Firebase-like tree stored in `localStorage` (`bunutan_demo_db`), and sign-in is replaced by a demo bar with fake accounts (Host, Ann, Bo, Cy, Di, Ed, Flo). It seeds one draw (`sampledraw`, admin key `demoadminkey0001`, 6 names, exchange date = today so "Show all picks" works). The skull password is `Bunutan`. After changing `index.html`, rebuild `demo.html` from it rather than editing the two by hand, and never deploy `demo.html`.

## Home list

`loadHome` shows two lists: draws the person created (`draws`, opens with the admin key) and "Draws you joined" (`joined`, opens the participant link). Joined entries whose draw no longer exists (`info/title` is null) are dropped on load. Only the draw title and the claimed name are stored, never the pick.

## Hide-name eye

The eye on the drawn screen (`hidePick`) masks the giftee's name everywhere it appears: the big name, the wishlist title ("••••••'s wishlist"), the "Chat with ..." title and bubble label, and the unread-reply notification. If you add a new place that shows the giftee's name, mask it the same way.
