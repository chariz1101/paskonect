# Paskonect — Project Specification

A web app for running Manito/Manita gift exchanges across distance. Organizers sign
up and create a room; participants join through a single shared link, write a
wishlist, and receive their assignment privately.

**Version:** 1.0
**Status:** Pre-build

---

## 1. Problem

Manito/Manita traditionally requires everyone in one room drawing names from a bowl.
Groups spread across cities or countries can't do that. The workarounds all leak:
one person draws for everybody and knows every pairing, or the group uses a
spreadsheet nobody trusts.

This app removes the need for a trusted intermediary. The draw happens in code, the
organizer participates on equal terms, and no human sees the full pairing list.

## 2. Scope

**In scope**

- Organizer accounts (email magic link, Google OAuth)
- Rooms with a shareable join link and short room code
- Anonymous, token-based participation — no account needed to join
- Self-serve joining: participants enter their own name and wishlist
- Exclusion rules (pairs who must not be matched)
- Single-cycle draw via Sattolo shuffle
- Private reveal page, one assignment per token
- Wishlists visible only to the assigned giver, editable after the draw
- Organizer dashboard: join status, view status, validation, link re-issue
- Emergency reveal behind an explicit confirmation
- Weekly themes, budget display

**Out of scope for v1**

- Native mobile apps
- Payments or gift purchasing
- Group chat or messaging inside the app
- Multi-round exchanges within one room
- Anonymous giver→receiver messaging

## 3. Core principles

These constrain implementation and should not be traded away for convenience.

**The organizer is a participant, not an administrator with a peek.** Their account
session grants room management. Their participant token grants exactly one
assignment. These are separate access paths and neither reaches the other.

**Pairings never leave the server except to the one person entitled to them.** No
dashboard query touches the `assignments` table. The draw action returns a boolean
validation summary, never the pairing array — not to the client, not into a
server-side variable that gets serialized into a response payload.

**Correctness is proven, not inspected.** The organizer confirms the draw is valid
by reading a checklist, not by reading the pairings.

**Losing your link is expected, not exceptional.** Three independent recovery paths
exist because at least one person in every group will lose theirs.

## 4. Architecture

### Stack

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js (App Router) | Server actions keep pairing logic off the client |
| Hosting | Vercel | Free tier covers this scale |
| Database | Postgres on Neon | Free tier, branching for safe migration testing |
| ORM | Drizzle | Typed schema, lightweight migrations |
| Auth | Auth.js + Drizzle adapter | Magic link + Google, no per-user pricing ceiling |
| Styling | Tailwind | Fast, no design system overhead |
| Email | Resend | Magic links and optional recovery mail |

Clerk is a reasonable substitute for Auth.js if you want auth fully handled — it
costs a per-MAU ceiling and a second source of user data, but saves a day of setup.

### Access model

There are exactly three ways to reach data:

1. **Session** (organizer) — rooms they own, participant roster, join/view status,
   validation results, room settings. Never assignments.
2. **Participant token** (anyone, including the organizer) — their own name and
   wishlist, and after the draw, their receiver's name and wishlist. Nothing else.
3. **Room code / join link** (public) — room name, budget, participant count,
   join form. Only while the room is open.

Every read is written against one of these three and no query spans two.

### Data model

```
users                    Auth.js standard
accounts                 Auth.js standard
sessions                 Auth.js standard
verification_tokens      Auth.js standard

rooms
  id                     uuid pk
  code                   text unique        -- PASKO-4RT9, unambiguous alphabet
  name                   text
  budget_amount          numeric nullable
  budget_currency        text default 'PHP'
  exclusion_rules        jsonb default '[]' -- [[participant_id, participant_id]]
  draw_state             text               -- 'open' | 'drawn'
  drawn_at               timestamptz nullable
  revealed_at            timestamptz nullable  -- emergency reveal audit
  owner_user_id          uuid → users.id
  created_at             timestamptz

participants
  id                     uuid pk
  room_id                uuid → rooms.id
  user_id                uuid → users.id, nullable   -- links organizer's own row
  display_name           text
  email                  text nullable
  token                  text unique, indexed
  wishlist               text default ''
  wishlist_updated_at    timestamptz nullable
  joined_at              timestamptz
  viewed_at              timestamptz nullable
  unique (room_id, user_id) where user_id is not null

assignments
  room_id                uuid → rooms.id
  giver_id               uuid → participants.id
  receiver_id            uuid → participants.id
  unique (room_id, giver_id)
  unique (room_id, receiver_id)

themes
  id                     uuid pk
  room_id                uuid → rooms.id
  week_no                int
  label                  text
  scheduled_date         date nullable
```

The two unique constraints on `assignments` make a malformed draw impossible to
persist. A duplicate receiver fails at the database, not in review.

### The draw

Sattolo's algorithm produces a single cycle through all participants: A gives to B,
B to C, and so on back to A. This guarantees no self-assignment and no mutual pairs,
which people find anticlimactic.

```
for i from n−1 down to 1:
    j ← random integer with 0 ≤ j < i
    swap a[i], a[j]
```

Note `j < i` rather than `j ≤ i` — that single difference is what makes it a cycle
rather than an ordinary shuffle.

With exclusion rules, re-run until the result satisfies them. Under 50 participants
this resolves in milliseconds. Cap at 10,000 attempts, then return an error telling
the organizer their constraints are unsatisfiable, with the specific pair that keeps
failing.

Minimum four participants. Three works mathematically but the cycle is transparent —
if A has B, then B has C and C has A, and everyone can deduce it.

Use `crypto.randomInt`, not `Math.random`.

### Tokens

32 bytes from `crypto.randomBytes`, base64url encoded. Stored in an httpOnly,
secure, SameSite=Lax cookie scoped to the room path, and also displayed on screen
with a copy button and a "save this link" prompt.

Re-issue mints a fresh token and nulls the old one in the same transaction. The
assignment is untouched — only the door key changes.

## 5. User flows

### Organizer creates a room

Sign up → create room (name, budget, currency) → receive join link and room code →
join their own room as a participant like everyone else → share link in group chat.

### Participant joins

Open link → see room name, budget, who has joined already → enter display name and
wishlist → receive personal link, cookie set, "save this" prompt → optionally
provide email for recovery.

### Draw

Organizer reviews roster → sets exclusions if any → clicks Draw → confirms → room
locks. No further joins, no name changes. Wishlists stay editable.

### Reveal

Participant reopens their personal link — same URL as before — and now sees their
receiver's name, wishlist, and last-updated timestamp. `viewed_at` stamps on first
load.

### Recovery

Cookie handles most people. Email recovery for those who provided one. Organizer
re-issue for everyone else — visible as a button next to each participant on the
dashboard, revealing nothing about assignments.

## 6. Sprints

Each sprint ends with something that runs. No sprint leaves the app in a state where
you can't demo it.

---

### Sprint 1 — Foundation and accounts

**Goal:** an organizer can sign up and create a room.

- Next.js project, Tailwind, Drizzle, Neon connection
- Auth.js with magic link and Google, Drizzle adapter
- `users`, `accounts`, `sessions`, `verification_tokens`, `rooms` tables
- Sign up, sign in, sign out
- Create room form: name, budget, currency
- Room code generator using an unambiguous alphabet (no `0/O`, `1/I/l`) — people
  will read these aloud over video calls
- Organizer's room list
- Deploy to Vercel with a live database

**Done when:** you can sign up on the deployed URL, create a room, sign out, sign
back in, and see your room.

**Risks:** magic link email deliverability. Verify the Resend domain on day one of
this sprint, not day four — DNS propagation will eat an afternoon.

---

### Sprint 2 — Joining

**Goal:** people can join a room without an account.

- `participants` table
- Join page at `/join/[code]` — room name, budget, current roster, join form
- Name and wishlist entry
- Token generation, cookie set, personal link display with copy button
- Duplicate name handling — prompt rather than block, since two Marias is normal
- Organizer joins their own room, `participants.user_id` links the row, dashboard
  surfaces their personal link automatically
- Wishlist editing via personal link
- Rate limit join attempts per IP so room codes can't be brute-forced

**Done when:** you can send the link to five people on five different devices and
all five appear in the roster with wishlists.

**Risks:** this is the sprint where you learn how people actually behave with links.
Test on a real phone, in a real group chat, with someone who isn't you.

---

### Sprint 3 — The draw

**Goal:** assignments generated, validated, locked.

- `assignments` table with both unique constraints
- Sattolo shuffle using `crypto.randomInt`
- Exclusion rules: UI for marking pairs, re-roll loop, 10,000-attempt cap,
  unsatisfiable-constraint error naming the offending pair
- Minimum four participants enforced
- Draw server action returning only `{ ok, participantCount, cycleValid,
  exclusionsRespected }`
- Room locks: `draw_state = 'drawn'`, joins rejected, names frozen
- Confirmation dialog before drawing — this is irreversible
- Unit tests: no self-assignment across 10,000 runs, every participant appears
  exactly once as giver and once as receiver, exclusions hold, cycle is single and
  complete

**Done when:** the test suite passes and the draw action's response contains no
participant identifiers.

**Risks:** the highest-value sprint to get right and the hardest to fix later. Write
the tests before the UI.

---

### Sprint 4 — Reveal

**Goal:** each participant sees their assignment and nobody else's.

- `getReceiverForToken(token)` — the single function in the codebase that reads
  across participants
- Reveal page: receiver name, wishlist, `wishlist_updated_at`, budget reminder
- `viewed_at` stamp on first load
- Invalid, revoked, and pre-draw token states each handled with their own message
- Wishlist stays editable from the same page
- Audit the codebase: grep for every reference to `assignments` and confirm each one
  is reachable only through a token, never a session

**Done when:** two people on two devices each see exactly one assignment, and
swapping tokens between browsers shows the other person's — proving the token is the
only thing that matters, then confirming the cookie is httpOnly so no script can
read it.

**Risks:** the grep audit is not optional. This is the last point where a leak is
cheap to fix.

---

### Sprint 5 — Organizer tools and polish

**Goal:** the organizer can run the exchange without seeing pairings.

- Dashboard: roster with joined and viewed status
- Validation panel restating the four guarantees in plain language
- Re-issue link per participant — new token, old one nulled, same assignment
- Email recovery for participants who provided an address
- Emergency reveal behind a typed confirmation, stamping `revealed_at`
- `themes` table and weekly theme schedule, visible to the whole room
- Budget display on reveal and join pages
- Optional: "your recipient updated their wishlist" notification
- Empty states, error states, mobile layout pass

**Done when:** you can run a full exchange with real people and never need to open
the database.

**Risks:** scope creep lives here. Everything in this sprint is comfort — ship
Sprints 1–4 and use them before deciding what in Sprint 5 you actually want.

---

## 7. Testing

**Unit** — the draw, over thousands of iterations: no self-assignment, single
complete cycle, exclusions respected, unsatisfiable constraints error cleanly.

**Integration** — a session cannot reach an assignment through any route. A token
reaches exactly one. A revoked token reaches none. A pre-draw token reaches none.

**Manual, before real use** — run a full exchange with four people on four devices,
including one person who deliberately loses their link and recovers it.

## 8. Open questions

- Should participants be able to leave a room before the draw, and does that
  require a re-draw if it happens after?
- What happens if someone drops out mid-exchange? Emergency reveal plus manual
  reassignment, or an automated re-draw of the affected segment?
- Should the organizer be able to delete a room, and what happens to participants
  holding live links?
- Do you want rooms to expire, or persist indefinitely?

None of these block Sprint 1. The first two want answers before Sprint 5.
