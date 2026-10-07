# Valt

Valt is Letterboxd for video games: log what you have played, rate and review it,
follow people, and read what they thought. A native iOS app on a TypeScript
backend, built solo.

The loop runs end to end on a physical device against a deployed backend and a
live IGDB integration. The API runs on EC2 behind Caddy, over HTTPS, kept up by
systemd, in the same VPC as the database — which admits the API's security group
and one development address, and nothing else.

**The source is private and available on request.** This repo is the write up:
what was built, how the decisions were measured, and what is not finished.

https://github.com/user-attachments/assets/cdea8a0c-c4fd-4995-8df8-9c62eb0ed926

## What works

Sign in, search a game, rate and review it, follow someone, see their review in
your feed, open it, comment on it.

- **Auth** with email and password: argon2id, JWT via jose, `requireAuth` and
  `optionalAuth`. A new account confirms its address with an emailed 8 digit code
  before it can use the app, and one left unconfirmed for 24 hours is deleted. A
  forgotten password is reset with an emailed code of its own, and the reset ends
  every session the account had, on every device. Sign in with Apple is intended,
  not built.
- **Signup consent**, recorded in the statement that creates the account: when
  the terms and the privacy policy were accepted, and which text of each. Both
  documents open from the agreement and from Settings.
- **IGDB game data** proxied and cached in Postgres. The client never talks to
  IGDB and never holds the key.
- **Game search** that forgives how people type: accents, hyphens, punctuation
  and numerals ("pokemon", "spiderman", "gta 5"), with popular main games ranked
  above fan games, DLC and editions.
- **Reviews** with half star ratings, 0.5 to 5, and optional text. Edited and
  deleted from the view they are read in.
- **A rating histogram** per game, public, so it renders before you sign up.
- **Follows, profiles and people search** that answers whether you already follow
  each result in the statement that finds them. Tapping either count on a profile
  opens who follows them and whom they follow, each row with its own Follow
  button; across a block both lists are empty.
- **A feed** of the people you follow, cursor paged and cached to disk, so it
  draws on launch and refreshes behind.
- **Comments** on any review, deletable by their author and by the review's
  author.
- **Blocking and reporting.** A block hides each person from the other, and the
  seven routes that return other people's content filter it inside the query. A
  block is undone from the profile or from a list in Settings. A person, a
  review, a comment, a profile photo or a list can be reported, and a report
  emails the support inbox.
- **Lists**, Letterboxd's way: up to 250 games, ranked or not, private until
  published, a note on each, reordered by dragging and saved in one transaction
  refused if the list changed elsewhere. A list page shows "You've logged X of
  Y", a numbered cover grid or a detailed view, and a profile's Lists tab shows
  each as a stack of covers. Public lists can be reported; blocks hide them.
- **A paid tier** unlocking per review backdrop art and profile header art,
  resolved at read time so a lapsed subscription hides a choice rather than
  destroying it.
- **A profile** that is a portrait rather than a dashboard: header art, a
  Favorites shelf of four games picked by search and reorderable by drag, an
  archive of every review, and a rating histogram of how that person scores,
  with tappable buckets.
- **Discover**, built on IGDB rather than on our own reviews, which numbered 27
  when it was built: three rails and 23 genre pages with infinite scroll, behind
  a process wide rate limiter that queues every IGDB call 260 ms apart to stay
  under its 4/s ceiling.
- **Account deletion** that means it: one statement, twelve cascading foreign
  keys, and the caller's comments leave other people's reviews.

| Feed | Game page | Review and comments | Profile |
|---|---|---|---|
| ![Home feed](screenshots/01-feed.png) | ![Game page with ratings and community reviews](screenshots/02-game-page.png) | ![Review with its comment thread](screenshots/03-review.png) | ![Profile](screenshots/04-profile.png) |

## Stack

**Server** TypeScript, Express 5, Prisma, Postgres on RDS, deployed to EC2
behind Caddy, with mail sent through Amazon SES. Seven runtime dependencies.

**iOS** Swift and SwiftUI, deployment target 26.5, `@Observable` screen models,
async/await over URLSession. No third party dependencies at all: the image
pipeline, the disk caches and the Keychain wrapper are hand written.

**Layout** A monorepo, one backend and one iOS client. `shared/` was meant to
hold the API contract and is empty, see below.

## How decisions were made

### Latency was round trip count, not slow code

The laptop is in California and the database is in Virginia, so **every SQL
statement costs about 82 ms**, and an endpoint's latency is that times its number
of statements. Application code never exceeded **5 ms** on any endpoint
measured. (Those are development numbers. The deployed API sits in the same
region as the database and has not been re-measured.)

Search was the worst of them at **2,034 ms** for twenty results, of which IGDB
was 190 ms and the rest was twenty cache writes with the client blocked. It now
answers from the IGDB payload and caches afterwards, so latency stopped scaling
with result count, which had tracked linearly at 89 ms each. The `.catch` on
that detached write is load bearing, because Node terminates the process on an
unhandled rejection. Tested by pointing the database at a dead port.

### Search was measured against IGDB before it was changed

Friends found search too strict, so IGDB's behaviour was measured first. Its
`search` ranks by name alone and is rarely short of results, just wrong ones:
"pokemon" returned fifty fan games and no Pokémon Red, "zelda" had no Breath of
the Wild. A query on IGDB's ASCII slugs sorted by rating count finds those, but
misses what `search` finds through alternative names ("gta 5" is "GTA V" there).
So each search sends both, and the server merges and reranks them: how well the
name matches, then how many people rated the game, with DLC, expansions and
editions under their main game. One measured trap shaped the ranking: an
alternative name may count as an exact match but never as "starts with", or
"Zelda: Ocarina of Time" puts Ocarina of Time above Breath of the Wild.

Two queries make a fresh search slower than one, and IGDB's request limit is
shared by every user, so results are cached in memory for an hour, keyed by the
normalised query: a repeat makes no IGDB request at all, and a failure is never
cached. Timed on the deployed server itself, from localhost, a fresh search took
0.57 to 0.61 s and a cached repeat 4 ms. That leaves out the phone's own network,
so it is not an end to end figure.

### A pagination bug that only appeared if you edited mid scroll

Ordering the feed on `updatedAt` looked fine and was measurably wrong. Prisma
re-reads the cursor row's timestamp by subselect at query time, so editing **the
cursor row itself** between pages moved it to now and the predicate collapsed to
"everything at or below now": the whole feed served again, duplicating everything
already scrolled past. Observed as **4 items returned where 3 were correct**.

`createdAt` is immutable here, and with `id` as the tiebreak the pair is a total
order. Verified against a deliberate three way tie.

### The image optimisation that measured worse

`AsyncImage` has no decoded image cache, so covers decoded again on every scroll
back into view, at full size regardless of display size. The replacement
downsamples at decode, off the main thread.

The part worth reading is the experiment that was reverted. Requesting a smaller
IGDB variant for small rows is obviously correct, and it shipped, and it came
back out after looking visibly softer. Downsampling to the same output size from
the **larger** source keeps **23% more high frequency detail** on a feed card and
**14% even on a 44 pt row**: a **1.05x** downscale softens edges while averaging
almost nothing, where a 2.1x reduction averages about four source pixels into
each output pixel. Fetch meaningfully larger than the target, then downsample.

### A picker that was measured and then not built

IGDB shipped multiple covers per game, which sounds like a picker. Measured
first: **3.3%** of the cached catalogue has more than one cover, and the **median
is 1 in every sample taken**, so the control would not have rendered on **96.7%**
of the catalogue. Not built.

The same pass found a live defect in the backdrop picker that *did* ship. Its
filter judged images on shape alone, so a 1280x720 logo plate passed as landscape
art, and three of the four that slipped through were a game's default hero. Fixed
with an exclude list rather than an include list, so the image types IGDB adds
later stay usable instead of silently shrinking every picker.

### A control that lies is worse than no control

An activity bell, a "Now playing" shelf, two review sections on the game page, a
backdrop picker in Edit Profile, and three onboarding steps that pretended to
connect game platforms, scan a library and suggest friends were all deleted under
that rule. "Now playing" is the clearest: `playStatus` is only ever `COMPLETED`
in practice, so that list would have been permanently empty for every user.

Where the answer has merely not arrived yet, the corollary is to draw nothing
rather than guess. The follow button waits as a shape with no tap target, sized
so nothing moves when the real one lands, because a button that renders a state
and then flips it invites a tap against a fabricated baseline. Two of the deleted
things came back later, once there was real data behind them.

### The fix that compiles is not always the right one

Xcode reported ten Swift 6 warnings; seven had a single cause, and it was a build
setting rather than seven bugs. `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` had
silently bound the image pipeline's request and cache types to the main actor,
and the loader is an `actor`, so every access became a cross-actor violation.

Adding `await` at each site silences all seven and compiles cleanly. It also
moves every decoded-image cache read onto the main thread, which is the exact
property the pipeline exists to provide. The correct fix was to mark the two
types `nonisolated` — the compiler's complaint was right, and its obvious remedy
was wrong. Both types are `Sendable` already; the isolation bought nothing and
cost the one thing the cache is for.

### Fixing a bug is not the same as proving it fixed

Adding comment counts surfaced an older bug. The feed decided "nothing changed"
by comparing review **ids**, and an id does not move when someone edits their
text or a count changes, so every one of those refreshes arrived, matched, and
was silently discarded.

The fix is a value comparison, then proved by A/B on the simulator: same cache,
same server, same everything else, the old comparison renders no count at all and
the new one renders 6.

## What is not built

- **No payments.** The paid tier is a boolean set by hand in the database. The
  development only route that flipped it, and the switch that called it, were
  deleted before any payment path exists; the Settings row that names the tier
  shows only to Pro accounts.
- **No test suite.** Verification is query logs, `EXPLAIN`, shell scripts that
  walk the permission matrices with curl, and checks on device.
- **Rate limits live in memory.** Comments are limited to 60 an hour and reports
  to 20 an hour per account, beside the per IP limits on register, login and
  password reset. The counts are held by the one server process, so a restart or
  a deploy resets them. Follows are the exception: at most 200 in any 24 hours per
  account, counted in the database so a deploy does not reset them, and an
  unfollow gives no slot back.
- **Moderation is manual.** A report arrives by email and is acted on with SQL
  from a runbook. There is no admin tool.
- **Comments cannot be edited**, only deleted; the row has no `updatedAt`.
  Reviews can be edited and deleted.
- **Lists have no likes or comments yet**, and nobody is told when a list is
  published. The schema leaves room for both.
- **`shared/` is empty.** The API contract lives in the server's types, mirrored
  by hand in Swift.

## What I would do next

- Add `publishedAt` to `Review`. A bare log that later gains a rating keeps its
  original `createdAt`, so it sorts back dated below every follower's cursor and
  is invisible.
- Measure Prisma's `relationJoins`. Every `include` is its own round trip today,
  so the feed's is three statements where it reads like one.
- Tests around the two things verified by hand and easiest to regress: cursor
  pagination and the comment permission matrix.
