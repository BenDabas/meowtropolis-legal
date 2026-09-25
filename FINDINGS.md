# What actually leaves the phone

Written 2026-09-02 by reading the code, not by assuming. **Updated 2026-09-25** for Friends, the
friend-online push, the friend link, the live seated modes and two new analytics events, read against
Meowtropolis main at **71c63f66** (2026-09-25 06:13). Every claim below names the file it came
from, so it can be re-checked when the code changes. **Sections 2.6, 10 (items 1 and 4) and 12 were
brought level on 2026-09-25 with the analytics server deployed at 07:00 that day (Meowtropolis
b310a915)**, read from that code and checked against the live function. **Section 13 (account deletion) and
the deletion answers in 12 were added on 2026-09-25 for a flow that is BUILT BUT NOT DEPLOYED**: the policy
text that describes it must not go live before the server deploy and an APK with the button. This is the evidence the privacy policy in
index.html is built on; if the code changes, change this file first and the policy second.

Sources read (2026-09-02): Runtime/Meta/Analytics.cs, OnlineRuns.cs, OnlineBoards.cs, OnlineNames.cs,
PlayerIdentity.cs, PlayerProfile.cs, ProfileStore.cs, Accounts.cs, PlayGamesLink.cs,
GameNotifications.cs, Runtime/Net/DuelNet.cs, DuelSession.cs, DuelLink.cs, QuickMatch.cs,
DuelRoom.cs, Runtime/Store/Billing.cs, CoinPurchases.cs, CoinPacks.cs, Iap/GooglePlayBilling.cs,
Server/handler.js, firestore-store.js, vocabulary.js, index.js, server.js, README.md,
Packages/manifest.json, ProjectSettings/ProjectSettings.asset.

Sources added (2026-09-25): Runtime/Meta/Friends.cs, FriendsPush.cs, MatchHealth.cs, Analytics.cs
(again), Runtime/Net/FriendsUgs.cs, FriendsRunner.cs, FriendLinks.cs, FriendsProbe.cs, NetProbe.cs,
OnlineNoticeboard.cs, ArenaQueue.cs, DuelNet.cs and DuelSession.cs (again),
Runtime/UI/GameUi.FriendsParty.cs, GameUi.FriendsWide.cs, Assets/Plugins/Android/MeowPush.java,
MeowPushService.java, MeowPush.androidlib/AndroidManifest.xml,
FriendLinkManifest.androidlib/AndroidManifest.xml, mainTemplate.gradle, server-push/handler.js,
store.js, fcm.js, ugs.js, index.js, firebase.json, package.json, server-friend-link/firebase.json,
public/index.html, docs/FRIENDS.md, FRIENDS-DESIGN.md, FRIENDS-PUSH.md, HANDOFF-2026-09-25.md.

---

## 1. The destinations, and nothing else

Every byte that leaves the phone goes to one of these. The scores function, Unity, Play Games and
Play Billing were the whole list on 2026-09-02; the push function and Firebase Cloud Messaging are
new on 2026-09-25, and Unity now also runs Friends.

| Destination | Who runs it | What for |
|---|---|---|
| https://europe-west1-memora-bf520.cloudfunctions.net/scores | Ben own Google Cloud function, Firestore behind it, EU (europe-west1) | leaderboards, ghost runs, name registry, the game own analytics |
| https://europe-west1-memora-bf520.cloudfunctions.net/push | Ben own Google Cloud function (codebase meowtropolis-push), a separate **named** Firestore database "meowtropolis" behind it | friend-online pushes, and the friend-code lookup behind friend links (section 9) |
| Unity Gaming Services (Authentication incl. Player Names, Lobby, Relay, **Friends**) | Unity Technologies | the account, the friend code, the rooms and connections for every live mode, the friends list, presence and friend messages |
| Google Play Games | Google | signing the player in on Android |
| Google Play Billing | Google | buying coin packs |
| Firebase Cloud Messaging | Google | Android only: minting the phone's notification token and delivering the push |

**Still no ad, attribution, crash-reporting or analytics SDK.** Packages/manifest.json adds exactly
one Unity service since 2026-09-02, com.unity.services.friends 1.2.0 (beside authentication, core,
lobby, relay). Assets/Plugins/Android/mainTemplate.gradle adds exactly one Google library,
com.google.firebase:firebase-messaging 24.1.2 - not the Firebase Unity SDK and not Firebase Analytics
(grep of mainTemplate.gradle for "firebase|analytics" returns that one line). The entry
com.unity.modules.unityanalytics is still the legacy built-in module, and nothing calls it.

Grepping Runtime/ for UnityWebRequest now returns eight files (at 71c63f66). Four are the scores function as before
(Analytics.cs, OnlineRuns.cs, OnlineBoards.cs, OnlineNames.cs). FriendsPush.cs and FriendLinks.cs are
the push function. NetProbe.cs is a diagnostics probe that does nothing unless a file named
netprobe.txt is placed in the app's storage by hand, and then reads the scores board with the literal
player id "netprobe". GameUi.cs matches only in a comment. The only literal URLs in Runtime/ and
Plugins/ are the two function base URLs above.

**Uncertain:** what Google's firebase-messaging library itself sends to Google in order to mint and
refresh a token (it registers a Firebase installation). That is inside Google's library and not
visible in this code.

The daily reminders are still scheduled locally on the device (GameNotifications.cs, Unity mobile
notifications package); nothing about them is sent anywhere. **The friend-online alert is different:
it is a real push service, section 9.**

---

## 2. Meowtropolis own server

Firebase project memora-bf520, Firestore, region europe-west1 (Belgium). Confirmed in
OnlineRuns.DefaultBaseUrl and Server/README.md. The game holds no database credentials and never
touches Firestore directly; the rules are "allow read, write: if false" for every client (README,
checked in console 2026-08-29). Everything goes through the HTTP routes.

**Added 2026-09-25, from docs/FRIENDS-PUSH.md and HANDOFF-2026-09-25.md, not from code:** Memora's own
backup script (scripts/snapshot-firestore-json.mjs, in the Memora repo) walks every collection of this
project's `(default)` database, which holds the meowtropolis_* collections above, into a file Memora
commits to git. Memora's last backup (2026-08-24) held no game documents, so nothing has leaked; the
next run would include them. The push data was put in a named database for exactly this reason. Ben
plans to move the game to its own Firebase project, which ends it.

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

Analytics.cs. Batched, written to a disk outbox first, sent at app open, match end, match quit and
after an online match's match_health; kept and retried if the send fails, **except** a batch the
server refuses as malformed (HTTP 400), which the client discards and never resends (Analytics.Flush);
bounded at 500 held events. Batch body is
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
| notification_open | which reminder was tapped: the deep-link slug the notification carried, as `target` (GameUi.cs). Refused by the server until 2026-09-25, stored since |
| match_health (sent since 2026-09-20, **stored since 2026-09-25**) | ONLINE-mode matches only, of 5 s and 30 frames or more (MatchHealth.cs): arena, live or bots, **device model, OS version string, GPU name, RAM in MB, CPU core count** (SystemInfo.deviceModel, operatingSystem, graphicsDeviceName, systemMemorySize, processorCount), frame-rate median and 5th percentile (overall and while refereeing), network silences (count, worst, total), reconnects, refereeing time, whether the referee changed, snapshot counts and timing, stranded/alone/reached the whistle, seats, queue window, wire version, duration |
| item (sent since 2026-09-05, **stored since 2026-09-25**) | which belt item was used in a match (`id`) and on which leg of the run (`leg`) (MatchController.cs:290, Prowl.Leg) |

match_health's device fields describe a kind of handset: no device id, serial, IMEI or Android id is
read (grep of Runtime/ for them still returns nothing). **The OS string includes the maker's firmware
build number** (Android's SystemInfo.operatingSystem reads like "Android OS 13 / API-33
(TP1A.220624.014/A725FXXU6FWB1)"): a version shared by every phone of that model on that firmware, not
an identifier of one phone, but more specific than "Android 13", and the policy says so. The client's
slug filter strips the spaces, brackets and slashes before sending (Analytics.EventBuilder.Str), so
what is stored reads like "AndroidOS13API-33TP1A.220624.014A725FXXU6FWB1", "samsungSM-A725F",
"AdrenoTM618".

Stored in meowtropolis_events, one document per batch, auto-id, plus at.

**There is no GET route for events and never was** (handler.js, README). Nothing can read this back
over HTTP; it is read in the Firebase console behind Ben Google account.

Strings are sanitised on the client to a slug alphabet before they are even built into JSON
(Analytics.EventBuilder.Str), and re-validated field by field on the server against
EVENT_STRING_FIELDS / EVENT_INT_FIELDS. device, os and gpu have their own server list (EVENT_DEVICE_FIELDS:
96 characters of letters, digits, space and ._()/+,:;=?!#$%&*@[]{}|~^-); a value outside it drops that
field, not the batch. **There is no free-text field anywhere in the analytics schema.**

**Since 2026-09-25 an unknown event NAME is skipped, not refused** (handler.js EVENT_NAMES): it stores
nothing and the rest of its batch is stored. Before that, one unknown name refused the whole batch,
which the client then discarded (section 10, items 1 and 4).

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
  Meowtropolis own scores server. **Changed 2026-09-24/25:** it is now the key of everything in
  sections 8 and 9, it is sent (as the verified `sub` of the UGS access token) to the push function,
  and the account's **Player Name is set to the friend code** (FriendsUgs.EnsureCodeAsync,
  UpdatePlayerNameAsync) - e.g. "K7M2QX9A#16522", where the #number is Unity's.
- **Lobby**: used for the 1v1 room. The room holds the Relay join code and the city seed
  (DuelNet.HostAsync). Friend rooms are private; quick-match rooms are public under the fixed name
  "quickduel" (DuelRoom.cs) so two strangers can find each other. **Since 2026-09-15 also the meeting
  list for the live seated modes** - ONLINE (8 seats), Catball and Cats vs Dogs (6) - see 3.1.
- **Relay**: carries the match traffic between the phones, DTLS where Relay offers it, the first
  endpoint otherwise (DuelNet.ToServerData, ArenaQueue.ToServerData: "DTLS where it is offered, the
  first endpoint otherwise"). **Uncertain:** whether that fallback endpoint is unencrypted; the code
  does not say and it has not been observed.
- **Friends** (new 2026-09-24): relationships, requests, blocks, presence and friend messages. Section 8.

**What crosses between two players during a 1v1** (DuelSession.SendHello, DuelIntro): protocol
version, the eat rule, duel rating, cat id, **the display name**, up to three skill ids with levels -
then positions, inputs, eaten-object indices and match events. **Stale on 2026-09-02, corrected
2026-09-25:** the name has been in the hello since wire version 6 (commit a86acffd, 2026-09-02, "Two
households played a 1v1 and both fought a bot called Rival"); DuelIntro.Name's own comment records the
reversal. The save-file player id is still not sent. The opponent's **UGS player id** is read from the
shared lobby (DuelNet.LearnPeerAsync, PeerPlayerId) - every lobby member can see every other member's
Player.Id, that is how Lobby works - and is kept past the match for "recently played" (section 8.4).

### 3.1 The live seated modes - what goes into the Unity lobby (ArenaQueue.cs)

Per player, member-visible (only phones in the same list can read it): `name` (the display name),
`cat`, and for ONLINE also `kit` (equipped skill ids and levels), `rank` (rank points) and `fit` (a
number from DeviceFitness: how well this phone could referee, derived from hardware and a
self-measurement). Per list, **public** (readable by any phone querying lobbies): a protocol `tag`, the
host's **UGS player id** (`host`, indexed S2, so a lone host can say "any list but mine"), the list's
birth time and the host's **UTC offset in minutes + 720** (`tz`, indexed N2, so nearby players are
tried first). Member-visible per list: relay code, seed, roster, deadline, arena. The seat's UGS id is
kept for "recently played" and the results screen's ADD (ArenaQueue.Seat.PlayerId: "Never shown to
anyone").

OnlineNoticeboard.cs writes referee claims (seat numbers and relay codes) into the same lobby during a
match: technical, nothing about the person.

Unity retention of the account, lobby, relay and friends data is Unity, and nothing in this codebase
can state it. Project id cb47c491-3f7e-4db6-9fa5-25889ec17ca1, organisation bendabas1
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
short code they typed on both". (It is also currently broken - see section 10.)

Accounts.AdoptKey can also change the id, but its only caller path is disabled today: the "already
linked to another account" case deliberately stays on the local key until a server-side merge route
exists.

**Since 2026-09-24 there are two more identifiers, and neither is the save-file id** (re-checked
2026-09-25: OnPairSavePressed still writes the typed code into PlayerId, GameUi.cs:8956):

- **The UGS player id** - Unity's id for the anonymous account. The key of the friends list, of
  presence, of friend messages, of the push records and of the friend-code table. It does not survive a
  reinstall or a new phone (FRIENDS-DESIGN.md names this as a known gap; the Play Games link is the
  planned fix).
- **The friend code** - 8 characters from PlayerIdentity's unambiguous alphabet, drawn with
  UnityEngine.Random (PlayerIdentity.NewPairCode), stored in the phone's friends.json, set as the Unity
  Player Name (Unity appends #number), and claimed on the push function for friend links (section 9.3).
  Random; not derived from the player, the device or the save-file id.

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

**Where else the name goes, as of 2026-09-25:** to the opponent in a 1v1 (section 3), to every phone in
a live seated list (3.1), into Unity Friends presence for friends to see (8.2), and into the other
players' "recently played" lists on their own phones (8.4). Incoming names pass through
PlayerIdentity.FromWire on arrival. The share-sheet message and the friend link carry **no name**
(8.1, 9.3).

## 8. Friends (new 2026-09-24/25) - UGS Friends

FriendsUgs.cs behind the Friends.cs facade; started by FriendsRunner.cs once the home screen is shown
and a UGS session exists (Accounts.ServicesReady). Runs for every signed-in player on every platform -
it is not gated on having friends. The product spec is docs/FRIENDS.md, the design FRIENDS-DESIGN.md.

### 8.1 Adding a friend

- By the long code: AddFriendByNameAsync("K7M2QX9A#16522") - a Unity call.
- By the short code alone, or by opening a friend link: FriendLinks.ResolveAsync asks the push
  function for the UGS id behind the 8 characters, then AddFriendAsync(id) (section 9.3).
- From "recently played" or the results screen: AddFriendAsync(ugsId).
- **Mutual accept**: a request is only a request until the other side accepts (the service's own rule).
- **Share**: the Android share sheet with the text "Play Meowtropolis with me! Add my friend code:
  <code>" (GameUi.FriendsWide.ShareFriendCode at 71c63f66). An uncommitted change in flight on
  2026-09-25 makes it "... Tap to be my friend: <link> (or add my code: <code>)". Neither carries the name.
- Client-side limits only (FRIENDS-DESIGN.md calls it an honest deviation): 20 requests an hour, one
  invite per friend per 30 s, one quick message a second. Every unanswered request this phone sent is
  withdrawn after 7 days (ExpireOldRequests), when the game next starts.

### 8.2 Presence - what a friend sees (FriendsRunner.Publish, FriendsUgs.Activity)

SetPresenceAsync on state changes only (home, queue/match, results, pause, resume, quit):
availability ONLINE / AWAY / OFFLINE plus an activity JSON
`{ s: home|match|results, m: solo|online|duel|catball|hunt, n: display name, c: cat id, lv: level,
tz: UTC offset in minutes, v: wire version }`. Unity supplies last-seen. Presence is only delivered to
friends (FriendsUgs.OnRelationshipAdded's comment). `tz` is sent but nothing in the client or the push
server reads it back (the push uses the IANA zone from registration, section 9).

### 8.3 Messages - no free text, by construction

Every message is an Envelope `{ k, i, id, m, l, t }` (FriendsUgs.Envelope):
- `q`: a quick message, `i` indexes Friends.Quick - exactly eight fixed phrases: LET'S PLAY!, GOOD GAME!,
  ONE MORE?, WAIT FOR ME, NICE!, SORRY!, THANKS!, BRB. The words never travel; a receiver drops an index
  it does not have.
- `inv`, `ans`, `go`, `cxl`, `lv`, `pm`, `list`: invite, answer, "go" (carrying a 1v1 lobby join code in
  `l`), cancel, leave, party line-up (UGS ids), the leader's list id.

There is no text field. OnMessage drops anything from a sender who is not on the friends list. Block
(FriendsUgs.Block) removes the friendship first, then AddBlockAsync, and drops queued invites from that id.

### 8.4 Recently played - on the phone only

FriendsUgs.NoteRecent, called from GameUi.FriendsParty.NoteRecentPlayers at the results screen: the
last **25** people (and bots, with invented ids "b:..." that never reach Unity) as
`{ id: UGS id, name, cat, mode, at }`. Stored in **friends.json in the app's own storage** with the
friend code, the requests this phone sent (id, name, cat, time), the online-toast stamps and the
request timestamps (FriendsUgs.Store, StoreFile). **Nothing in it is uploaded.** Deleted on uninstall.

The same is true the other way: **other players' phones keep this player's UGS id, name and cat** in
their own recently-played list after a live match.

Also on the phone (PlayerPrefs / SharedPreferences): `meow_friend_name_<ugsId>` - the last real name
seen for each friend, so an offline friend's row is not a code (FriendsUgs.Apply, commit 243ea787);
`push.friendids`, `push.alerts`, `push.asked`; and `meow_push_names` (section 9.2).

## 9. Friend-online pushes and friend codes (new 2026-09-25) - server-push/

### 9.1 What the phone sends (FriendsPush.cs, MeowPush.java) - Android only

**Changed 2026-09-25 (Meowtropolis 7e205601, deployed 06:47): see 10.5.** Once per session, after the
home screen and a UGS session, Android only. The token is registered **only while ALERTS is on and
Friends lists at least one friend** (FriendsPush.ShouldRegister); "I'm online" is sent every session
regardless, because it is the friends' alert, not this phone's:

1. Firebase is initialised by hand from options in code (FriendsPush: app id, API key, project id
   memora-bf520, sender id) and FirebaseMessaging.getToken() mints the **FCM token**.
2. POST /push/register `{ fcm, tz, alerts, platform }` - the token, the phone's **IANA time zone**
   (TimeZone.getDefault().getID(), e.g. "Asia/Jerusalem"), the ALERTS toggle, "android".
3. POST /push/online `{}` - "I'm online", which fans out to friends.
4. POST /push/heartbeat `{}` every 5 minutes while focused, and on resume (or /push/online again after
   more than 10 minutes away).

Every call carries `Authorization: Bearer <UGS access token>`. The server verifies it against Unity's
JWKS (issuer, project cb47c491-..., expiry; ugs.js) and takes the caller's id from `sub` only - no body
field names a player. **No name and no text is sent.** FriendsPush's own summary: "The server gets a
token, a time zone and a boolean."

**Superseded by 7e205601:** switching ALERTS off now calls /push/forget, which deletes the whole
record, and the server also deletes the record when any client registers with `alerts: false`. What
follows was true of the code audited at 71c63f66: switching ALERTS off re-registered with
`alerts: false`; **the token stayed stored** - the client never
called /push/unregister (grep of Runtime/ and Plugins/Android/ for "unregister": no match, while
"/push/register" matches FriendsPush.cs:389 as the control). Turning ALERTS off stops
this phone *receiving* alerts; it does not stop /push/online from alerting this player's friends.

iOS: FriendsPush is compiled out (`#if UNITY_ANDROID`). There is no APNs key.

### 9.2 What the server does and stores (handler.js, store.js, fcm.js)

Named Firestore database **"meowtropolis"** (store.js; not `(default)`, so Memora's committed JSON
backup never sweeps it). Location **eur3** (Europe multi-region) per FRIENDS-PUSH.md's creation record -
the code does not state it; the function itself runs in europe-west1 (index.js).

- `push_players/{ugsId}`: `tokens` (at most **3**, newest wins, each `{t, at, platform}`), `tz`,
  `alerts`, `lastSeen`, `lastOnlineCall`. **From 7e205601:** `tokens`, `tz`, `lastSeen` only, and the
  record exists only while ALERTS is on and there is a friend. /push/heartbeat and /push/online update
  an existing record and never create one (store.js touchPlayer), verified live on the deployed
  function: a heartbeat and an "online" from a player with no record left none (Firestore read 404).
- `push_rate/{ugsId}`: `at`, the 10-minute announce limit (new in 7e205601; it used to be
  `lastOnlineCall` on the player record, which is why "online" created records).
- `push_pairs/{to}_{from}`: `lastSent`. **This is a record of which pairs of UGS ids have alerted each
  other**, so it does reveal that two ids are friends, even though the friend list itself is never
  stored (handler.js: friendsOf is fetched from UGS per call and not kept).

The fan-out on /push/online: for each id UGS lists as the caller's FRIEND (asked of Unity with the
caller's own token, ugs.js createFriendsOf), skip it unless it has a doc, alerts on, a token, was not
seen in the last 10 minutes, is not in 22:00-08:00 in its own IANA zone (an unknown zone counts as
quiet), and the pair has not been alerted in 3 h (a transaction). Then FCM sends a **data-only** message
`{ kind: "friend_online", sender: <caller's UGS id> }`, high priority, 30-minute TTL (fcm.js).
/push/online is rate-limited to once per caller per 10 minutes, server-side.

**The friend's name is cached on the phone only - verified.** fcm.js's payload has two keys, kind and
sender, and handler.js passes only `{ kind, from: me.id }`. MeowPushService.onMessageReceived builds
"<name> is online / Tap to play together" from MeowPush.nameFor(), a SharedPreferences file
`meow_push_names` (id -> name, names cut to 32 characters, cleared when it reaches 200 entries) written
by FriendsPush.RememberName from the friends list, which skips codes and "A FRIEND" (IsStandIn, commit
243ea787). An unknown id reads "A friend is online". Precisely: the name reaches the phone through
**Unity presence** (8.2), not through the push server.

**Retention from 7e205601 (deployed and checked live 2026-09-25 06:47-06:52 local):**
- The push record is deleted by /push/forget (ALERTS off), by a register with `alerts: false`, and by
  the daily scheduled function `pushExpiry` once `lastSeen` is older than **60 days**.
- `push_pairs` and `push_rate` rows are deleted by the same function once older than **1 day**.
- Live check: a planted `push_rate` row with `at: 1` was deleted by one run, which logged
  `push expiry: {"players":0,"pairs":0,"rates":1}`, and the row read 404 after.
- The friend codes and lookup counters (9.3) are not covered: they are still kept until deleted by
  hand.

**Retention as audited at 71c63f66 (superseded):** no TTL, no expiry, no deletion code. FCM tokens that FCM reports as
unregistered or invalid are removed (removeTokens); the fourth phone pushes out the oldest token.
Everything else - the player doc, the pair docs, the friend codes (9.3) and the lookup counters - is
kept until deleted by hand.

### 9.3 Friend codes and friend links - **in the tree, not deployed**

Commits 41a30ed6, 01ab03fd, 5b56f9c3 and 71c63f66 landed on main between 06:00 and 06:13 on 2026-09-25.
Their own messages say "(not deployed)"; a read-only probe at 06:2x returned **404** for
https://meowtropolis-friends.web.app/f/K7M2QX9A while the scores function's /health returned 200 as the
control. So: code present, server routes and page not live.

- **POST /friend/claim `{ code }`** (handler.js claimCode, store.js claimCode): binds the 8-character code
  to the caller's UGS id in `friend_codes/{code}` = `{ owner, at }`, in a transaction, first owner wins,
  never moved. FriendsUgs.EnsureCodeAsync calls it on start until it succeeds (a taken code, 409, draws a
  new one).
- **POST /friend/resolve `{ code }`**: returns `{ id: <owner's UGS id> }`. Needs a valid UGS token;
  counted per caller in `friend_lookups/{ugsId}` = `{ from, n }`, at most 60 an hour, so codes cannot be
  walked. The resolved id goes straight into AddFriendAsync. **This is the one route in either server
  that returns another player's id** (section 2.5's rule is about the scores server and still holds
  there): a UGS id, never the save-file id, and only to a caller who already holds that player's code.
- **The link** `https://meowtropolis-friends.web.app/f/<CODE>` (FriendLinks.LinkFor) - the code and
  nothing else, **no name** (the page's own comment: "a kids' game should not put a nickname into a URL
  that messengers and browsers log"). The page (Firebase Hosting, site meowtropolis-friends,
  server-friend-link/) has no third-party scripts, no cookies, no analytics, sends `Referrer-Policy:
  no-referrer`, and on Android hands to `meowtropolis://friend/<CODE>` with a Play Store fallback.
  FriendLinkManifest.androidlib adds the intent filter; FriendLinks.TakeLaunchCode reads the code and
  FriendsUgs.OpenLink sends the request.
- As with any web page, Google's hosting sees the visitor's request, the code in the path included.
  That is infrastructure logging outside this codebase.

## 10. Bugs found while reading (not privacy, but Ben should know)

1. **FIXED 2026-09-25 (b310a915, deployed 07:00; see item 4 for the measurements).**
   **notification_open events are silently thrown away, and they take the whole batch with them.**
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
   sitting next to one. (Re-checked 2026-09-25: both still there, Billing.cs:36 and ProfileStore.cs:43.)
4. **FIXED 2026-09-25 (Meowtropolis b310a915, deployed 06:57-07:00 by agent 2 with Ben's go).**
   Confirmed first against the LIVE function, not only the tree, with batches that could not be
   stored: {app_open, t:-5} -> 400 "bad t" (the control: the validator runs and says why);
   {match_health}, {item}, {notification_open} -> 400 "event 0 has an unknown name" each. After the
   deploy: a valid match_end + match_health batch -> 200 {stored: 2}; {keylog} -> 200 {stored: 0,
   skipped: 1}; the control unchanged. The server now stores the three events with their field
   whitelists and SKIPS any unknown name instead of refusing the batch. Batches phones already
   discarded are gone; the loss stopped at the deploy. The windows, from the client commits that
   added each event: notification_open from 2026-09-02 (6c731442), item from 2026-09-05 (d1d0f907),
   match_health from 2026-09-20 (e81e6197); any batch holding one of them in those windows was lost. The original finding, as written:
   **match_health and item events are refused too, the same way as bug 1** (found 2026-09-25).
   Server/handler.js EVENT_NAMES is still the seven names of 2026-09-02 (control: 'app_open' matches);
   the last commit to Server/handler.js is 2026-09-02, and no commit on any branch adds match_health to
   Server/. match_health calls Flush() at once, so the batch it rides in - usually carrying that
   ONLINE match's match_end - is refused with 400 and discarded by the client (Analytics.cs:364-373).
   **If the deployed function matches the tree**, no ONLINE match has been recorded in analytics since
   2026-09-20, and any batch holding a belt-item event is lost as well. Uncertain only in that the
   deployed function was not probed; adding the names to EVENT_NAMES (and their fields to the
   whitelists) plus a redeploy fixes it. Privacy note: the device fields are sent and refused, so today
   they are transmitted but not stored; once the server accepts them they are stored.
5. **Fixed in Meowtropolis 7e205601, deployed 2026-09-25 06:47:** registration only with ALERTS on
   and a friend, deletion on ALERTS off, and a 60-day expiry (see 9.1 and 9.2). The original finding:
   **The push record outlives its purpose.** Every Android player gets a push_players doc whether or
   not they have friends, the ALERTS toggle keeps the token, and no code ever deletes a doc (section
   9.2). Not a bug; a data-minimisation choice Ben may want: register only once there is a friend, and
   call /push/unregister when ALERTS goes off.

## 11. Things the policy must NOT say

- No named regulation. Nothing in the code implements an age gate, a consent flow, a data-export
  route or a documented retention schedule, so claiming any framework by name would be a claim the
  code does not support.
- No retention period in days. There is none in the code.
- Not "we never see your name" - the display name is uploaded and shown to other players.
- Not "no data is ever shared" - Unity and Google are both genuinely in the picture, and the Play
  Games link is new as of today.
- Not "anonymous". The id is random and not derived from the device, which is better than most, but
  a random id that follows a player across matches is still an identifier.

Added 2026-09-25:

- Not "no chat and no messages" or "players cannot contact each other outside a match". Friends
  can send each other eight fixed phrases and invites at any time. The true claim is **no free-text
  chat**, and **only accepted friends** can reach a player.
- Not "your name is not sent in a 1v1" - it has been since wire version 6 (section 3).
- Not "there is no push service" - there is one, on Android (section 9).
- Not "nothing identifies your phone" without a qualifier - the FCM token addresses this install on
  this phone. No advertising id and no hardware id is still true.
- Not "the game never receives your location" without a qualifier - it reads the time-zone setting
  and sends it. No location permission and no coordinates is still true.
- Not "the name never leaves the phone" for alerts - it never passes through *our* server, but it
  reaches a friend's phone through Unity presence.
- **No Firebase project id and no hosting domain in the policy.** Ben plans to move the game off the
  shared memora-bf520 project to its own; the policy already names neither, so keep it that way.
- Nothing that promises a Unity-side deletion Ben cannot perform. **Settled 2026-09-25 (section 13):** he
  can. The Unity account is deleted by the game itself (DeleteAccountAsync) or by the admin API / the
  dashboard's Player Management; Cloud Save and Friends relationships must be deleted separately, and are
  (Unity: "deleting a Player Account doesn't automatically delete player data in Cloud Save, active
  Leaderboards, Friends, etc."). The policy now promises that, and only once the flow is deployed.
- Not "deleted immediately". Deletion is scheduled 7 days out (with DELETE NOW in the game), and a web
  request that cannot reach Unity through a service-account key leaves the Unity half to Ben by hand.
- Not the 7-day request expiry as if it were a retention period. It is a client action run when the
  sender's game starts, not a server deletion.

## 12. Google Play Data Safety answers

Ready to copy into Play Console -> App content -> Data safety. Derived from sections 1-11, for the
build at 71c63f66 **with the friend link deployed**. Unity and Google act as service providers
processing data for the game, which Play does not count as "sharing"; that assumption is behind every
"Shared: No" below. **Uncertain** items are marked; decide them before submitting.

**Overview questions**

- Does your app collect or share any of the required user data types? **Yes.**
- Is all of the user data collected by your app encrypted in transit? **Yes** - HTTPS to both
  functions, to Unity and to Google. *Uncertain:* the Relay fallback when DTLS is not offered
  (section 3). Answer Yes only if Relay's plain endpoint is still encrypted, or once the fallback is
  removed.
- Which account creation methods does your app support? **None the player performs.** Unity's
  anonymous account and the Play Games link are created automatically. *Uncertain* whether Play treats
  that as account creation (which would require a web deletion link); if it does, the deletion email
  below is the path.
- Do you provide a way for users to request that their data be deleted? **Yes.** **From section 13, once
  deployed:** in the app (Settings -> YOUR ACCOUNT -> DELETE ACCOUNT) and on the web at
  https://bendabas.github.io/meowtropolis-legal/delete.html (form, or email). Before that deploy, the
  honest answer is email only.
- **Account deletion (the Data deletion section of the form).** Play treats the Unity account plus the Play
  Games link as an account the app creates, so it asks for the delete-account URL: use
  https://bendabas.github.io/meowtropolis-legal/delete.html. "Do you provide a way for users to request
  that some or all of their data is deleted without requiring them to delete their account?" **Yes**:
  friends can be removed and blocked in the game, ALERTS off deletes the push record, and the email route
  takes partial requests. *Stated there too:* the 7-day grace, and what is kept (section 13).

**Data types collected** (Collected: Yes; Shared: No; Processed ephemerally: No, unless stated)

| Play category -> type | What it is here | Required or optional | Purposes |
|---|---|---|---|
| Personal info -> Name | The cat name (a nickname; generated by default, may be typed). Uploaded with runs, duels and names; in Unity presence and lobbies | Required (a name is always assigned and uploaded) | App functionality |
| Personal info -> User IDs | The random save-file id; the UGS player id; the friend code | Required | App functionality, Analytics, Account management |
| Device or other IDs | The FCM token (Android only) | **Optional** (from 7e205601: registered only while ALERTS is on and the player has a friend; deleted when ALERTS goes off, or after 60 days unused) | App functionality |
| Location -> Approximate location | *Judgement call.* The IANA time-zone name (push; optional, stored only with the FCM token under the same rule) and the UTC offset (lobby, presence; required). Never coordinates, no location permission | Required (the UTC offset); the push time zone is optional | App functionality |
| Financial info -> Purchase history | The purchase analytics event: coin pack id, coins granted, new balance; coin spends | Required | Analytics |
| Messages -> Other in-app messages | The eight fixed quick phrases and invites, relayed by Unity Friends between friends | Optional (only if the player uses Friends) | App functionality. *Uncertain* whether Unity stores them or relays them ephemerally |
| App activity -> App interactions | Analytics events (app open, match start/end/quit, claims, tutorial, reminder taps, items) and match results for boards and ghost runs | Required | Analytics, App functionality |
| App activity -> Other actions | Friends list, requests, blocks and presence (online, mode, level, last seen), held by Unity | Required (presence is published for every signed-in player; only friends can read it) | App functionality |
| App info and performance -> Diagnostics | match_health: device model, OS version string (with the firmware build number), GPU name, RAM, cores, frame rate, network quality (ONLINE-mode matches of 5 s or more) | Required | Analytics. Stored since the 2026-09-25 deploy (b310a915) |

The Play Console asks the same four questions for each type above, and the answers are the same for all:
collected Yes; shared No; ephemeral No (except as noted); deletable on request Yes for everything on
Ben's servers. For Unity-held data (friends, presence, player name): **from section 13, once deployed**,
DELETE ACCOUNT removes the relationships, the Cloud Save slot and the Unity account itself; before that,
only removing friends and blocking in the game.

**Not collected** - answer No: email address, phone number, physical address, race or ethnicity,
political or religious beliefs, sexual orientation, other personal info; precise location; payment info
(Google Play Billing handles it; the game never sees it), credit score, other financial info; health,
fitness; emails, SMS or MMS; photos, videos, voice or sound recordings, music, other audio; files and
docs; calendar; contacts; in-app search history, installed apps, other user-generated content, web
browsing history; crash logs.

**Not in this list, deliberately:** the daily reminders (scheduled on the phone, nothing sent), the
purchase receipts (on the phone only), and "recently played" (on the phone only).

**Also for Ben before release:** FRIENDS-PUSH.md flags that Play's policy on notifications in apps for
children needs checking. That is a Families-policy question, not a Data Safety one.

## 13. Account deletion (built 2026-09-25, NOT deployed)

Meowtropolis docs/ACCOUNT-DELETION.md is the design; the code is server-push/account*.js, ugs-account.js
and Runtime/Meta/AccountDeletion.cs. Read from the code as committed on 2026-09-25, before any deploy.

**Where every piece of a player's data lives, and what deletes it:**

| Where | Keyed by | Deleted by |
|---|---|---|
| (default) meowtropolis_runs, _best_<arena>, _duel, _names, _events | save-file playerId | /account/erase or the daily accountSweep (account-store.js eraseScores); boards found by listing collections |
| meowtropolis DB push_players, push_rate, friend_lookups, friend_codes, push_pairs | UGS id | the same (erasePush); pairs matched on the document id |
| UGS Friends relationships (friends, requests, blocks) | UGS id | /account/erase with the player's own token only (Friends has no service-account route) |
| UGS Cloud Save (agent 4's profile copy, when it ships) | UGS id | /account/erase with the player's token; the sweep with an admin key; the client's own hook as a fallback |
| UGS Authentication account (and its Play Games link) | UGS id | the game (DeleteAccountAsync) last; the sweep with an admin key; else Ben in the dashboard |
| The phone: profile, friends.json, outboxes, receipts, PlayerPrefs, SharedPreferences | - | Android clearApplicationUserData() after the server steps |

**New data this flow itself stores** (named database meowtropolis): account_deletions/{ugsId} =
{ugs, player, requestedAt, dueAt, source, ref?, contact?} for 7 days; an unmatched web request
{code, name, contact?, requestedAt, ref} for up to 30 days; account_web_rate/{day} = {n}; and
account_manual/{ugsId} = {at, needs} when the Unity half is left for a human. The optional contact email
is new personal data, collected only on the web form, and deleted with the request.

**Not deleted, stated in the policy:** Google Play purchase history; the Play Games link on Google's side;
Cloud Logging request logs (default retention 30 days, not checked on this project); other players'
"played with you" lists on their phones; Unity's own gateway IP logs (30 days, Unity's Friends privacy page).

**Uncertain until deployed and tested:** that the push function's service account may delete in the
(default) database (same project, so expected); that Unity's Friends and Cloud Save client REST routes
accept DELETE with a player token as documented; that a deleted Unity account ends the Friends retention
of its id as Unity's privacy page implies for the sweep path, where relationships cannot be removed first.
