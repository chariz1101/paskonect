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
- [ ] Create room form: name, budget amount, currency (default PHP)
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
- [ ] Audit: grep every reference to `assignments` and confirm each is reachable only through a token
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
- [ ] Manual run: full exchange with four people on four devices
- [ ] Manual run: one person deliberately loses their link and recovers it

## Open questions

Decide these before the sprint noted.

- [ ] Can participants leave before the draw? Does leaving after require a re-draw? *(before Sprint 5)*
- [ ] What happens if someone drops out mid-exchange: emergency reveal plus manual reassignment, or automated re-draw of the affected segment? *(before Sprint 5)*
- [ ] Can the organizer delete a room, and what happens to live participant links?
- [ ] Do rooms expire, or persist indefinitely?
