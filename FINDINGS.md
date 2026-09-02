# What actually leaves the phone

Written 2026-09-02 by reading the code, not by assuming. Every claim below names the file it came
from, so it can be re-checked when the code changes. This is the evidence the privacy policy in
index.html is built on; if the code changes, change this file first and the policy second.

Sources read: Runtime/Meta/Analytics.cs, OnlineRuns.cs, OnlineBoards.cs, OnlineNames.cs,
PlayerIdentity.cs, PlayerProfile.cs, ProfileStore.cs, Accounts.cs, PlayGamesLink.cs,
GameNotifications.cs, Runtime/Net/DuelNet.cs, DuelSession.cs, DuelLink.cs, QuickMatch.cs,
DuelRoom.cs, Runtime/Store/Billing.cs, CoinPurchases.cs, CoinPacks.cs, Iap/GooglePlayBilling.cs,
Server/handler.js, firestore-store.js, vocabulary.js, index.js, server.js, README.md,
Packages/manifest.json, ProjectSettings/ProjectSettings.asset.

---

## 1. The four destinations, and nothing else

Every byte that leaves the phone goes to exactly one of these. There is no fifth.

| Destination | Who runs it | What for |
|---|---|---|
| https://europe-west1-memora-bf520.cloudfunctions.net/scores | Ben own Google Cloud function, Firestore behind it, EU (europe-west1) | leaderboards, ghost runs, name registry, the game own analytics |
| Unity Gaming Services (Authentication, Lobby, Relay) | Unity Technologies | the account, the 1v1 room, the 1v1 connection |
| Google Play Games | Google | signing the player in on Android |
| Google Play Billing | Google | buying coin packs |

**No third-party SDK of any other kind is in the build.** Packages/manifest.json has no ad network,
no attribution SDK, no crash reporter, no Firebase Analytics, no Unity Analytics service package.
The only Unity service packages are authentication, core, lobby, relay. The entry
com.unity.modules.unityanalytics is the legacy built-in module, not the service, and nothing in the
code calls it. Grepping Runtime/ for UnityWebRequest and http returns hits in exactly four files:
Analytics.cs, OnlineRuns.cs, OnlineBoards.cs, OnlineNames.cs - all of them the one server above.

Notifications are scheduled locally on the device (GameNotifications.cs, Unity mobile notifications
package). Nothing about them is sent anywhere; there is no push service.

---

## 2. Meowtropolis own server

Firebase project memora-bf520, Firestore, region europe-west1 (Belgium). Confirmed in
OnlineRuns.DefaultBaseUrl and Server/README.md. The game holds no database credentials and never
touches Firestore directly; the rules are "allow read, write: if false" for every client (README,
checked in console 2026-08-29). Everything goes through the HTTP routes.

### 2.1 A finished match - POST /runs

Sent as the results screen appears, once per match, at most one per 20 seconds per player, only if
the final mass is 50 or more (handler.js MIN_FINAL_MASS, UPLOAD_COOLDOWN_SECONDS). Built in
OnlineRuns.Upload. Fields:

- playerId - the id from the save file (see section 6)
- name - the display name
- arenaId, catId - slugs the game minted
- curve - 25 numbers, the mass of the cat sampled through the match
- abilityIds (up to 8), abilityCasts - which skills were equipped and how often each was cast
- skill, aggression, castEagerness, recklessness, roam, orbit, castPhase - seven 0..1 numbers
  describing how this player moved and fought, so the recorded run can be replayed as a believable
  opponent
- tier, wins, rating - rank badge step, lifetime wins, this map rating

Stored in collection meowtropolis_runs, document id <playerId>_<slot>, plus a server-added "at"
(upload time in ms). **Five slots per player, rotating** - the sixth upload overwrites the first
(firestore-store.js put; SLOTS_PER_PLAYER = 5 is in handler.js:49).

The server rebuilds every field against a closed schema and **drops anything it does not
recognise** (handler.js validate). It refuses names outside the allowed set, unknown arenas and
malformed curves.

### 2.2 The arena record board - written from the same upload

meowtropolis_best_<arenaId>, one document per player per arena: playerId, name, catId, arenaId,
tier, wins, rating, mass, at. Mass only ever rises; rating is overwritten every time. Read back by
GET /top.

### 2.3 The duel ladder - POST /duel

Posted after a 1v1 that had a result (OnlineBoards.PublishDuel): playerId, name, catId, rating,
wins, losses, bestStreak, tier, plus server-added at. Collection meowtropolis_duel, one document
per player, **overwritten** each time - a standing, not a history.

### 2.4 The name registry - GET /name, POST /name, and opportunistic claims

meowtropolis_names, one document per claimed name, id = the name lower-cased, holding playerId,
name, at. Written when a player claims a name on the rename screen (OnlineNames.Claim) and also
silently on every run and duel upload for anybody who has not claimed yet (handler.js,
store.claimName). A rename releases the old name if the player owned it (releaseName).

This is the one table that maps **name -> playerId**, which is what makes a deletion request
answerable from a name alone.

### 2.5 Reads - the player id travels in the URL

- GET /runs?arena=...&count=...&mine=<playerId>   (OnlineRuns.Fetch)
- GET /top?arena=...&count=...&mine=<playerId>&as=<display name>   (OnlineBoards.FetchArena)
- GET /duel?count=...&mine=<playerId>&as=<display name>   (OnlineBoards.FetchDuel)
- GET /name?name=<typed name>&mine=<playerId>   (OnlineNames.Check)

Worth knowing: the id and the name therefore appear in **query strings**, so they will be present
in Google Cloud own request logs for the function - infrastructure logging outside this codebase
control. The app own code logs no IP address and stores none.

**Nothing the server returns carries anybody player id.** /runs and /top return a boolean "yours"
flag instead, which tells the caller only what the caller already told the server (handler.js, GET
routes). So one player can never learn another player id.

### 2.6 Analytics - POST /events

Analytics.cs. Batched, written to a disk outbox first, sent at app open, match end and match quit;
kept and retried if the send fails; bounded at 500 held events. Batch body is
{playerId, events:[...]}. Every event carries t (unix seconds) and seq (a per-install counter).

| Event | Fields |
|---|---|
| app_open | app version, matches played so far, coin balance |
| match_start | arena, cat, equipped skills and their levels, mode (online/bots), matches played |
| match_end | arena, cat, skills, mode, mass, tier, place, rivals, kills, coins earned, stars, duration in whole seconds, end reason, matches played, coin balance |
| match_quit | arena, cat, mode, mass, tier, duration |
| purchase | kind (cat, skill, level, coinpack), item id, price in coins, resulting balance, amount |
| claim | kind (gift, mission, achievement), id, amount, resulting balance |
| tutorial | which step was completed |
| notification_open | which reminder was tapped - **see the bug in section 8** |

Stored in meowtropolis_events, one document per batch, auto-id, plus at.

**There is no GET route for events and never was** (handler.js, README). Nothing can read this back
over HTTP; it is read in the Firebase console behind Ben Google account.

Strings are sanitised on the client to a slug alphabet before they are even built into JSON
(Analytics.EventBuilder.Str), and re-validated field by field on the server against
EVENT_STRING_FIELDS / EVENT_INT_FIELDS. **There is no free-text field anywhere in the analytics
schema.**

### 2.7 What is downloaded from the server

Other players runs (name, arena, cat, 25-number curve, skills, behaviour numbers) and board rows
(rank, name, cat, mass, rating, tier, wins). Those names are shown in the game. No ids.

### 2.8 Retention - the honest answer

**The code sets no retention period and no automatic deletion, anywhere.** There is no Firestore
TTL policy (firebase.json deliberately has no firestore section at all - README explains why), no
cron, no expiry field. What actually happens:

- runs: capped at 5 per player, older ones overwritten by newer ones, kept indefinitely otherwise
- arena record rows: one per player per arena, kept indefinitely; deleted only if the mass is above
  what the re-paced arena can still produce (purgeBestAbove)
- duel rows and name rows: one per player, kept indefinitely, overwritten in place
- analytics batches: appended, **never overwritten, never deleted by any code**

So the policy must say: kept until we delete it, and deletion happens on request or when it is
cleared by hand. Anything else would be invented.

---

## 3. Unity Gaming Services

Accounts.EnsureAsync() is called at launch for **every player**, from MatchController line 742 -
not only for players who press 1v1.

- **Authentication**: SignInAnonymouslyAsync(). Unity mints an anonymous account and caches its
  token per install. The Unity account id is stored back into the local save as BoundAccountId only
  once Play Games links successfully (Accounts.RecordBinding). This id is **never** sent to
  Meowtropolis own server.
- **Lobby**: used for the 1v1 room. The room holds the Relay join code and the city seed
  (DuelNet.HostAsync). Friend rooms are private; quick-match rooms are public under the fixed name
  "quickduel" (DuelRoom.cs) so two strangers can find each other.
- **Relay**: carries the match traffic between the two phones, DTLS-encrypted
  (DuelNet.ConnectionType).

**What crosses between two players during a 1v1** (DuelSession.SendHello, DuelIntro): protocol
version, the eat rule, duel rating, cat id, up to three skill ids with levels - then positions,
inputs, eaten-object indices and match events. DuelIntro own comment states it and the wire format
confirms it: **the display name is deliberately not sent, and neither is the player id.** One
player never learns who the other is.

Unity retention of the account, lobby and relay data is Unity, and nothing in this codebase can
state it. Project id cb47c491-3f7e-4db6-9fa5-25889ec17ca1, organisation bendabas1
(ProjectSettings.asset).

## 4. Google Play Games (Android only, new today)

PlayGamesLink.ServerAuthCodeAsync() runs at launch, silently, after the anonymous Unity sign-in. It
calls PlayGamesPlatform.Authenticate and then RequestServerSideAccess(false, ...), which returns a
**short-lived server authorisation code**. That code is handed to
AuthenticationService.LinkWithGooglePlayGamesAsync (Accounts.TryLinkPlayGamesAsync).

Three things this means, all verified in code:

- A real Google identity is now involved: Google knows the player launched the game, and Unity
  holds a link between its account and that Play Games account.
- **The game itself never receives or stores the Google account name, email, or Play Games id.**
  It receives one auth code and passes it straight to Unity. Grep confirms no other Play Games call.
- Every failure path is silent and harmless: no Play Games account, declined, offline, broken Play
  services, or a 20-second timeout all return null and the game carries on locally.

Not in the picture on iOS - PlayGamesLink.Available is Android-only.

## 5. Google Play Billing (the coin store, new today)

Runtime/Store/. Six consumable coin packs, $0.99 to $19.99 (CoinPacks.All).

- The purchase itself is entirely Google: launchBillingFlow (GooglePlayBilling.cs). **The game
  never sees, handles or stores a card number, an address or any payment detail.**
- One thing the game does hand Google: setObfuscatedAccountId(CoinPurchases.ObfuscatedAccountId())
  - the first 16 bytes of SHA-256("meow:" + playerKey) as hex. It ties purchases to the account
  without giving Google the key itself.
- The receipt is filed **on the phone only**, in coin-purchases.json: order id, product id, pack id,
  coins, purchase token, purchase time, and the sandbox/credited/consumed/verified flags. Verified
  is always false - **there is no receipt-verification route on the server and no code anywhere
  uploads a receipt.**
- The only thing that leaves the phone about a purchase is one analytics event: purchase with kind
  coinpack, the pack id, the coins granted and the resulting balance (CoinPurchases.Credit). No
  price, no order id, no token.

## 6. How a player is identified - the brief claim, re-checked

**Mostly true, with one exception the brief did not mention.**

PlayerIdentity.Ensure() mints Guid.NewGuid().ToString("N") into the save file on first launch. No
advertising id, no hardware id, no IMEI, no Android id - grep finds none of them anywhere in
Runtime/. Uninstalling deletes the save file and genuinely breaks the link. The class comment
saying so is still accurate on this point.

The exception: **the player can replace that id by hand.** GameUi.OnPairSavePressed writes an
8-character typed pairing code straight into ProfileStore.Current.PlayerId, so two phones can share
one identity. So the id is "random unless the player chose to pair devices, in which case it is the
short code they typed on both". (It is also currently broken - see section 8.)

Accounts.AdoptKey can also change the id, but its only caller path is disabled today: the "already
linked to another account" case deliberately stays on the local key until a server-side merge route
exists.

## 7. Names - the brief second claim, re-checked

**Half true, and the stale half is in the very comment the brief points at.**

PlayerIdentity class comment says "The name is generated, not typed" and calls a free text field a
moderation problem. That comment is now **out of date**: further down the same file, TrySetName /
Reject accept typed names, and Server/vocabulary.js has a matching rejectTypedName. Ben asked for it
on 2026-08-18.

What a typed name may be, enforced identically on both sides:

- 3 to 14 characters, ASCII letters and digits only - no spaces, no punctuation, no Unicode
  lookalikes, no direction overrides
- must contain at least one letter
- checked against a 50-word blocklist after leetspeak folding (5H1T becomes shit), matched as
  substrings

The generated two-word cat name (32 adjectives x 32 nouns) is still the default and is always
accepted. Names are unique since 2026-09-02: the registry in section 2.4 decides.

So the policy has to say plainly that **the name is public** - it appears on other players
leaderboards and on recorded ghost opponents - and that a player who types their real name into it
has published it. Names are never checked against anything but the blocklist.

## 8. Bugs found while reading (not privacy, but Ben should know)

1. **notification_open events are silently thrown away, and they take the whole batch with them.**
   Analytics.NotificationOpen is live (called from GameUi.cs:4984), but the server EVENT_NAMES set
   in handler.js does not contain notification_open. An unknown name makes validateEvents refuse the
   batch with 400, and the client treats a 400 as "malformed, discard" - so every event batched
   alongside a notification tap is lost too. One word added to the server set fixes it, plus a
   redeploy.
2. **Pairing two phones breaks every upload from that phone.** OnPairSavePressed sets PlayerId to
   the 8-character pairing code, but the server requires 32 hex characters on /runs, /duel, /events
   and /name. A paired player is refused with 400 everywhere and silently disappears from the
   boards, and their analytics stops arriving - while the panel says "paired - your records are
   shared now".
3. **Billing.SandboxAllowedOnDevice is still true**, as its own comment says it must not be at
   release, alongside ProfileStore.TesterCoinGrant. Not a privacy matter; it is a release blocker
   sitting next to one.

## 9. Things the policy must NOT say

- No named regulation. Nothing in the code implements an age gate, a consent flow, a data-export
  route or a documented retention schedule, so claiming any framework by name would be a claim the
  code does not support.
- No retention period in days. There is none in the code.
- Not "we never see your name" - the display name is uploaded and shown to other players.
- Not "no data is ever shared" - Unity and Google are both genuinely in the picture, and the Play
  Games link is new as of today.
- Not "anonymous". The id is random and not derived from the device, which is better than most, but
  a random id that follows a player across matches is still an identifier.
