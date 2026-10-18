# Paskonect — Task Checklist

Everything we need to do to ship v1, grouped by sprint. Tick boxes as work lands.
Details for each item are in [`SPEC.md`](SPEC.md). Visual rules are in
[`ART_DIRECTION.md`](ART_DIRECTION.md).

---

## Sprint 0 — Design setup

- [x] Get a 3D illustration set: Microsoft Fluent Emoji 3D (MIT), 29 assets in `public/illustrations/`
- [x] Record sources and license in `public/illustrations/CREDITS.md`
- [ ] Build the `<HeroCluster>` component for composing heroes from several illustrations (max 128px each)
- [ ] Compose the landing, join and reveal hero clusters
- [ ] Add color, radius, shadow and font tokens to the Tailwind config
- [ ] Load Plus Jakarta Sans with `next/font/google`
- [ ] Build base components: button (primary gradient, secondary outline), pill chip, card, input, icon button
- [ ] Build the curved red hero header with the soft arc behind it
- [ ] Build the mobile bottom nav with the active red pill
- [ ] Build the centered phone-width desktop frame on the blush gradient background
- [ ] Check color contrast for all text-on-red combinations

## Sprint 1 — Foundation and accounts

**Goal:** an organizer can sign up and create a room.

- [ ] Verify the Resend sending domain (do this first; DNS takes time)
- [ ] Create the Next.js (App Router) project with TypeScript and Tailwind
- [ ] Set up Drizzle and connect to Neon Postgres
- [ ] Add Auth.js with the Drizzle adapter
- [ ] Add email magic link sign-in (Resend)
- [ ] Add Google OAuth sign-in
- [ ] Create `users`, `accounts`, `sessions`, `verification_tokens` tables
- [ ] Create the `rooms` table
- [ ] Sign up, sign in and sign out flows
- [ ] Landing page using the onboarding layout ("Ho-HO-HO!" + "Create a room")
- [ ] Create room form: name, party date, budget amount, currency (default PHP)
- [ ] Store `party_date` and compute `expires_at` (end of day, party date + 14 days)
- [ ] Room code generator with an unambiguous alphabet (no `0/O`, `1/I/l`), format `PASKO-XXXX`
- [ ] Organizer's room list
- [ ] Deploy to Vercel with environment variables and the live database

**Done when:** you can sign up on the deployed URL, create a room, sign out, sign back in, and see your room.

## Sprint 2 — Joining

**Goal:** people can join a room without an account.

- [ ] Create the `participants` table, including the `unique (room_id, user_id)` partial index
- [ ] Join page at `/join/[code]`: room name, budget, current roster, join form
- [ ] Room code entry for people who have the code but not the link
- [ ] Name and wishlist entry
- [ ] Token generation: 32 bytes from `crypto.randomBytes`, base64url
- [ ] Set the httpOnly, secure, SameSite=Lax cookie scoped to the room path
- [ ] Personal link screen with a copy button and a "save this link" prompt
- [ ] Optional email field for recovery
- [ ] Duplicate name handling: prompt, don't block
- [ ] Organizer joins their own room; `participants.user_id` links the row
- [ ] Dashboard shows the organizer's own personal link automatically
- [ ] Wishlist editing via the personal link
- [ ] Rate limit join attempts per IP
- [ ] Reject joins when the room isn't open
- [ ] Add `left_at` to `participants`
- [ ] "Leave room" button before the draw, with confirmation; leaver drops off the roster

**Done when:** five people on five devices join from one link and all appear in the roster with wishlists.

- [ ] Test on a real phone, in a real group chat, with someone who isn't you

## Sprint 3 — The draw

**Goal:** assignments generated, validated, locked.

Write the tests first.

- [ ] Unit test: no self-assignment across 10,000 runs
- [ ] Unit test: every participant is a giver exactly once and a receiver exactly once
- [ ] Unit test: the result is a single complete cycle
- [ ] Unit test: exclusion rules are respected
- [ ] Unit test: unsatisfiable exclusions return an error naming the failing pair
- [ ] Unit test: fewer than four participants is rejected
- [ ] Create the `assignments` table with `unique (room_id, giver_id)` and `unique (room_id, receiver_id)`
- [ ] Create the `assignment_snapshots` and `room_events` tables
- [ ] The first draw saves snapshot version 1 and a `drawn` event
- [ ] Sattolo shuffle using `crypto.randomInt` (`j < i`, not `j ≤ i`)
- [ ] Exclusion re-roll loop with a 10,000-attempt cap
- [ ] Exclusion rules UI for marking pairs
- [ ] Enforce a minimum of four participants
- [ ] Draw server action returns only `{ ok, participantCount, cycleValid, exclusionsRespected }`
- [ ] Confirm no pairing data ends up in any response payload
- [ ] Irreversible-action confirmation dialog before drawing
- [ ] Lock the room: `draw_state = 'drawn'`, set `drawn_at`, reject joins, freeze names
- [ ] Keep wishlists editable after the draw

**Done when:** the test suite passes and the draw action's response contains no participant identifiers.

## Sprint 4 — Reveal

**Goal:** each participant sees their assignment and nobody else's.

- [ ] `getReceiverForToken(token)`: the only function that reads across participants
- [ ] Reveal page in the product-detail layout: red hero with the receiver's name, white sheet with their wishlist
- [ ] Show `wishlist_updated_at` ("updated 2 hours ago")
- [ ] Budget reminder on the reveal page
- [ ] "Their wishlist" / "My wishlist" tabs; own wishlist editable from the same page
- [ ] Stamp `viewed_at` on first load
- [ ] First-view reveal animation, with a reduced-motion fallback
- [ ] Invalid token state with its own message
- [ ] Revoked token state with its own message
- [ ] Pre-draw token state with its own message
- [ ] Audit: grep every reference to `assignments` and `assignment_snapshots` and confirm none is ever read into a session response
- [ ] Confirm the token cookie is httpOnly

**Done when:** two people on two devices each see exactly one assignment, and swapping tokens between browsers shows the other person's.

## Sprint 5 — Organizer tools and polish

**Goal:** the organizer can run the exchange without seeing pairings.

- [ ] Dashboard roster cards with Joined and Viewed status badges
- [ ] Roster filter chips: All / Joined / Viewed / Not viewed
- [ ] Validation panel restating the four guarantees in plain language
- [ ] Re-issue link per participant: new token and old token nulled in one transaction, assignment untouched
- [ ] Email recovery for participants who gave an address
- [ ] Emergency reveal behind a typed confirmation, stamping `revealed_at`
- [ ] Create the `themes` table
- [ ] Weekly theme schedule as a vertical timeline, visible to the whole room
- [ ] Santa speech-bubble announcements (e.g. "Names drawn!")
- [ ] Budget display on join and reveal pages

**Leaving after the draw**

- [ ] "Leave room" after the draw stamps `left_at` and flags the room as having a gap
- [ ] Dashboard banner asking the organizer to close the gap or redraw
- [ ] Close the gap: the leaver's giver takes over the leaver's receiver, server-side, boolean summary only
- [ ] Redraw everyone still in the room
- [ ] Block both when fewer than four remain; offer emergency reveal or delete instead
- [ ] Block close the gap if it breaks an exclusion rule; suggest redraw
- [ ] Affected giver's page: "Your person left, the organizer is sorting it out"
- [ ] Stamp `assignment_changed_at` and clear `viewed_at` for anyone whose receiver changed
- [ ] "Your assignment changed" notice on the reveal page

**History and restore**

- [ ] Save a snapshot and a `room_events` row before every change to `assignments`
- [ ] History panel listing versions from `room_events` only (no pairings)
- [ ] Restore a version: save the current state first, copy the version back, bring back anyone who left after it
- [ ] Restore confirmation warning that people may have already bought gifts

**Deleting a room**

- [ ] Delete room behind a typed confirmation (type the room name)
- [ ] On delete: stamp `deleted_at`, scrub assignments, snapshots, themes, wishlists, names and emails
- [ ] "The organizer deleted this room" page for personal links and the join link
- [ ] Purge deleted-room leftovers after 30 days

**Expiry**

- [ ] Let the organizer change the party date; `expires_at` moves with it
- [ ] Daily Vercel Cron job that expires due rooms (stamp `expired_at`, scrub data)
- [ ] Expiry warnings 7, 3 and 1 day before, by email, tracked in `expiry_warnings_sent`
- [ ] Expiry countdown banner on the dashboard
- [ ] "This exchange has ended" page for links to expired rooms
- [ ] Purge expired-room leftovers after 30 days

**Polish**

- [ ] Optional: "your recipient updated their wishlist" notification
- [ ] Empty states with illustrations
- [ ] Error states
- [ ] Mobile layout pass on every screen
- [ ] Accessibility pass: contrast, 44px tap targets, focus rings, alt text

**Done when:** you can run a full exchange with real people and never need to open the database.

## Testing

- [ ] Integration test: a session can't reach an assignment through any route
- [ ] Integration test: a token reaches exactly one assignment
- [ ] Integration test: a revoked token reaches none
- [ ] Integration test: a pre-draw token reaches none
- [ ] Integration test: a session can't read `assignment_snapshots` through any route
- [ ] Integration test: tokens for left, deleted and expired rooms show the right page and no assignment
- [ ] Unit test: close the gap keeps a single complete cycle and changes exactly one giver
- [ ] Unit test: restore brings back the exact earlier version
- [ ] Unit test: every change saves a snapshot first
- [ ] Manual run: full exchange with five people on five devices
- [ ] Manual run: one person deliberately loses their link and recovers it
- [ ] Manual run: one person leaves after the draw, the gap is closed, then the earlier version is restored

## Decisions made

- [x] Leaving before the draw: allowed
- [x] Leaving after the draw: organizer chooses close the gap or redraw
- [x] Dropping out mid-exchange: version history with restore
- [x] Deleting a room: allowed; links show a "room deleted" page
- [x] Expiry: two weeks after the party date, with 7/3/1-day warnings

Details are in section 5 and section 8 of [`SPEC.md`](SPEC.md).
