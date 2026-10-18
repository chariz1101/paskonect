# Paskonect

Run Manito/Manita gift exchanges with people who aren't in the same room.

An organizer signs up and creates a room. Everyone else joins through one shared
link, writes a wishlist, and gets their assignment privately. The draw happens in
code, so nobody, including the organizer, sees the full list of pairings.

> **Status:** pre-build. The full specification is in [`docs/SPEC.md`](docs/SPEC.md).

## How it works

1. **Create a room.** The organizer signs in (email magic link or Google) and sets
   the room name and budget. They get a join link and a short room code like
   `PASKO-4RT9`.
2. **Join.** Participants open the link and enter their name and wishlist. No
   account needed. Each person gets a private personal link.
3. **Draw.** The organizer clicks Draw. The room locks and everyone is assigned
   exactly one person to give to.
4. **Reveal.** Each participant opens their personal link to see who they're giving
   to and that person's wishlist.

The organizer joins their own room as a regular participant and sees only their
own assignment.

## Privacy guarantees

- **Nobody sees the full pairing list.** The organizer dashboard never reads
  assignments. The draw only reports whether the result is valid.
- **One link, one assignment.** A personal link shows its holder's receiver and
  nothing else.
- **The draw is fair and checked.** It forms a single loop through everyone
  (Sattolo's algorithm with `crypto.randomInt`): nobody draws themselves, there are
  no two-person swaps, and exclusion rules are respected. The database rejects any
  draw where someone gives or receives twice.
- **Lost links can be recovered** through a browser cookie, email (if provided), or
  a fresh link from the organizer. A new link doesn't change the assignment.

## Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js (App Router) |
| Hosting | Vercel |
| Database | Postgres on Neon |
| ORM | Drizzle |
| Auth | Auth.js + Drizzle adapter |
| Styling | Tailwind CSS |
| Email | Resend |

## Roadmap

| Sprint | Goal |
|---|---|
| 1. Foundation and accounts | An organizer can sign up and create a room |
| 2. Joining | People can join a room without an account |
| 3. The draw | Assignments are generated, validated and locked |
| 4. Reveal | Each participant sees their assignment and nobody else's |
| 5. Organizer tools and polish | The organizer can run the exchange without seeing pairings |

See [`docs/SPEC.md`](docs/SPEC.md) for each sprint's tasks, done criteria and
risks.

## Getting started

There's no code yet. Setup instructions will be added in Sprint 1, once the
Next.js project, database and auth are in place.

## Documentation

- [`docs/SPEC.md`](docs/SPEC.md): full project specification (scope, principles,
  architecture, data model, user flows, sprints, testing, open questions)
