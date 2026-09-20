# Hydrium --- 20231207 Client Contract Reconstruction

> **Target:** Rec Room `20231207`\
> **Steam App:** `471710`\
> **Depot:** `471711`\
> **Manifest:** `1151455856673601091`\
> **Inputs analyzed:** `GameAssembly.dll` and
> `RecRoom_Data/il2cpp_data/Metadata/global-metadata.dat` from the
> target build, plus observed 20231207 client behavior already captured
> during Hydrium testing.
>
> **Goal:** document what the 20231207 client expects from its backend
> so Hydrium can implement the boot-critical protocol without guessing
> response shapes.

------------------------------------------------------------------------

## 1. Evidence rules

This document deliberately separates four kinds of evidence.

**CONFIRMED** --- observed working against the 20231207 client, or the
client parser directly rejected the opposite shape.

**BINARY-CONFIRMED** --- route/property/error string is present in the
supplied 20231207 binary/metadata.

**HIGH CONFIDENCE** --- binary structure plus runtime behavior strongly
constrain the contract, but a complete DTO has not yet been recovered.

**UNKNOWN** --- route exists, but HTTP method, response root, DTO, or
values still need reconstruction.

A route string by itself proves that the client knows the route. It does
**not** prove its HTTP method or response type.

------------------------------------------------------------------------

# 2. Major result of the binary sweep

The 20231207 metadata contains a large client API catalogue. It includes
the exact boot families already observed in Hydrium:

``` text
player/*
matchmake/*
roominstance/*
rooms/*
api/config/*
api/avatar/*
api/equipment/*
api/consumables/*
api/communityboard/*
api/gameconfigs/*
api/gamerewards/*
api/images/*
api/playerevents/*
api/quickPlay/*
api/roomkeys/*
api/storefronts/*
api/subscriptionseasons/*
thread/*
announcements/*
voice/*
club/*
/strings
```

The same metadata contains client-side diagnostic strings including:

``` text
Malformed Response
Malformed Response: '
Deserialization returned null
Failed to retrieve strings
Unable to GetMyHomeClub
```

This is important: the client itself distinguishes transport success
from schema/deserialization success. Returning HTTP 200 with `{}` is
therefore not sufficient.

------------------------------------------------------------------------

# 3. Boot contract --- current reconstruction

The known boot path is:

``` text
/player/login
      ↓
/player/qos
      ↓
/player/connection-info
      ↓
Photon authentication
      ↓
/player/heartbeat
      ↓
boot/config/social/content GETs
      ↓
/matchmake/dorm
      ↓
room instance join
      ↓
/roominstance/{id}/reportjoinresult
      ↓
Dorm
```

The client binary explicitly contains:

``` text
player/avoidjuniors
player/connection-info
player/exclusivelogin
player/gameserverregionpings
player/heartbeat
player/login
player/logout
player/notifydisconnect
player/photonregionpings
player/qos
player/statusvisibility
player/vrmovementmode
```

### Photon

Current Hydrium Photon identity flow is working and should be considered
frozen unless new evidence appears.

``` text
Current AccountId: 178933005
Region: us

Realtime AppId: 390700d6-7387-4fa5-b6d7-7ee39e46ad4a
Voice AppId:    91c066ff-4be0-4c77-9733-d9e5831051a5
Chat AppId:     5daaf075-d152-4c42-a4f7-47391cbff9be
```

`/photon/auth` dynamically returns the requested account identity. The
old hard-coded Photon UserId mismatch has been eliminated.

------------------------------------------------------------------------

# 4. Response-root matrix

This is the minimum schema information required before implementing a
route.

  -----------------------------------------------------------------------------------------------
  Route                                           Expected response root  Evidence
  ----------------------------------------------- ----------------------- -----------------------
  `/api/avatar/v1/defaultbaseavataritems`         array `[]`              HIGH CONFIDENCE

  `/api/avatar/v1/defaultunlocked`                array `[]`              HIGH CONFIDENCE

  `/api/avatar/v4/items`                          array `[]`              HIGH CONFIDENCE

  `/api/avatar/v2/gifts`                          array `[]`              CONFIRMED

  `/api/equipment/v2/getUnlocked`                 array `[]`              CONFIRMED

  `/api/consumables/v2/getUnlocked`               array `[]`              CONFIRMED

  `/api/images/v2/named`                          array `[]`              CONFIRMED

  `/api/gamerewards/v1/pending`                   array `[]`              CONFIRMED

  `/api/roomkeys/v1/mine`                         array `[]`              CONFIRMED

  `/api/storefronts/v4/balance/:currencyType`     array `[]`              CONFIRMED ROOT

  `/api/communityboard/v2/current`                object                  CONFIRMED ROOT

  `/api/subscriptionseasons/v1/seasons/current`   object                  CONFIRMED ROOT

  `/api/quickPlay/v1/getandclear`                 object                  CONFIRMED ROOT

  `/announcements/v2/mine/unread`                 object                  CONFIRMED ROOT

  `/voice/config`                                 object                  PARSER-CONFIRMED

  `/voice/requiresModeration`                     JSON boolean            HIGH CONFIDENCE

  `/api/playerevents/v1/all`                      object with `Created`   CONFIRMED
                                                  and `Responses` arrays  

  `/api/config/v1/azurespeech`                    object                  CONFIRMED ROOT

  `/api/config/v1/amplitude`                      object                  CONFIRMED ROOT

  `/api/config/v1/backtrace`                      object                  CONFIRMED ROOT

  `/api/config/v2`                                object                  CONFIRMED

  `/rooms/bulk`                                   array                   CONFIRMED / FROZEN

  `/rooms/createdby/me`                           array `[]`              CONFIRMED

  `/club/mine/member`                             array `[]`              CONFIRMED / FROZEN

  `/club/home/me`                                 object                  CONFIRMED ROOT; DTO
                                                                          UNKNOWN

  `/Player`                                       array                   CONFIRMED / FROZEN

  `/api/PlayerReporting/v1/voteToKickReasons`     array `[]`              CONFIRMED

  `/api/ugcPurchasables/v1/items/room/:roomId`    array `[]`              CONFIRMED

  `/outfits/me/saved`                             array `[]`              CONFIRMED

  `/iam/me/channels/Announcements`                array `[]`              CONFIRMED
  -----------------------------------------------------------------------------------------------

There is still an observed client error:

``` text
expected: '['
actual: '{'
```

Therefore at least one remaining Hydrium boot response is currently `{}`
while its consumer expects an array.

------------------------------------------------------------------------

# 5. Config v2 --- strongest reconstructed DTO

The 20231207 metadata directly contains these property names:

``` text
LevelProgressionMaps
DailyObjectives
ServerMaintenance
AutoMicMutingConfig
StorefrontConfig
RoomKeyConfig
RoomCurrencyConfig
ShareBaseUrl
AwardCurrencyCooldownSeconds
MaxKeysPerRoom
MinPlayerLevelForGifting
```

The binary also contains all eight microphone-spam configuration
property names:

``` text
MicSpamSamplePercentageForForceMute
MicSpamSamplePercentageForForceMuteToEnd
MicSpamSamplePercentageForWarning
MicSpamSamplePercentageForWarningToEnd
MicSpamVolumeThreshold
MicSpamWarningStateVolumeMultiplier
MicVolumeSampleInterval
MicVolumeSampleRollingWindowLength
```

Structural reconstruction of the object passed into the AudioManager
path indicates eight float fields.

The boot crash currently observed is:

``` text
JBHPHMDFHAF..ctor(JBDHBAKEONF DIIHIAJGIIB)
RecRoom.Audio.AudioManager.Initialize()
System.NullReferenceException
```

`AutoMicMutingConfig: {}` did not remove it.

### Current test contract

``` json
{
  "LevelProgressionMaps": [],
  "DailyObjectives": [],
  "ServerMaintenance": null,

  "AutoMicMutingConfig": {
    "MicSpamSamplePercentageForForceMute": 0.0,
    "MicSpamSamplePercentageForForceMuteToEnd": 0.0,
    "MicSpamSamplePercentageForWarning": 0.0,
    "MicSpamSamplePercentageForWarningToEnd": 0.0,
    "MicSpamVolumeThreshold": 0.0,
    "MicSpamWarningStateVolumeMultiplier": 0.0,
    "MicVolumeSampleInterval": 0.0,
    "MicVolumeSampleRollingWindowLength": 0.0
  },

  "StorefrontConfig": {
    "AwardCurrencyCooldownSeconds": 0.0
  },

  "RoomKeyConfig": {
    "MaxKeysPerRoom": 0
  },

  "RoomCurrencyConfig": {
    "MinPlayerLevelForGifting": 0
  },

  "ShareBaseUrl": ""
}
```

The **property names and primitive structure** are the important
recovered information. The zero values are compatibility-test values and
must not be represented as historical Rec Room defaults.

------------------------------------------------------------------------

# 6. Config v1 DTO evidence

## Amplitude

The 20231207 metadata contains:

``` text
AmplitudeKey
RudderStackKey
UseRudderStack
StatSigKey
```

Candidate disabled/minimal response:

``` json
{
  "AmplitudeKey": "",
  "RudderStackKey": "",
  "UseRudderStack": false,
  "StatSigKey": ""
}
```

Property presence is binary-confirmed. Exact historical values are not.

## Backtrace

The binary contains:

``` text
ANRThresholdMs
CaptureNativeCrashes
FilterType
LogLineCount
MessageCount
MessageRegex
ReportBudget
SampleRate
VersionRegex
```

The December parser behavior showed that `CaptureNativeCrashes` must not
simply be copied as a boolean from older documentation; the current
Hydrium numeric representation avoided the previous token-type failure.

Current compatibility object:

``` json
{
  "ANRThresholdMs": 0,
  "CaptureNativeCrashes": 0,
  "FilterType": 0,
  "LogLineCount": 0,
  "MessageCount": 0,
  "MessageRegex": "",
  "ReportBudget": 0,
  "SampleRate": 0.0,
  "VersionRegex": ""
}
```

## Azure Speech

Known structural candidate:

``` json
{
  "Enabled": false,
  "Key": "",
  "Region": ""
}
```

Exact December contract still requires structural confirmation before
freezing.

------------------------------------------------------------------------

# 7. Strings/localization --- major unresolved contract

The binary contains `/strings`, and separately contains the
localization/autolocalization path:

``` text
autoloc/strings
autoloc/v2/strings
```

It also contains:

``` text
scopePrefix
locale
eTag
stringsList
Failed to retrieve strings
Deserialization returned null
```

Observed Hydrium request:

``` text
GET /strings?locale=pt-BR&scopePrefix=client.&eTag=20231207
```

Current `{}` response is not considered valid. Runtime evidence includes
failed string retrieval/deserialization.

### What is already constrained

The request has at least:

``` text
locale
scopePrefix
eTag
```

The client has a concept named `stringsList`.

### What is NOT yet proven

We do not yet have enough structural evidence to claim whether the
response is:

``` text
[]
```

or:

``` text
{ eTag, strings: [...] }
```

or a dictionary/map-style object.

Do not implement a guessed envelope yet.

------------------------------------------------------------------------

# 8. Avatar contract

The exact 20231207 route catalogue contains:

``` text
api/avatar/v1/defaultbaseavataritems
api/avatar/v1/defaultunlocked
api/avatar/v1/lockeditems
api/avatar/v2/gifts
api/avatar/v2/gifts/consume/
api/avatar/v2/gifts/generate
api/avatar/v3/gifts/generate
api/avatar/v4/items
```

The binary contains avatar DTO property names including:

``` text
AvatarItemType
AvatarItemDesc
AvatarItemId
AvatarItemMaterialId
AvatarVersion
```

The empty-array root is acceptable for several retrieval routes, but
eventually the client needs valid base/unlocked avatar data if Hydrium
is to reconstruct the normal default avatar rather than merely pass
deserialization.

Do not invent item IDs. A valid reconstruction needs the
historical/default item records or another source of IDs accepted by
this build.

------------------------------------------------------------------------

# 9. Equipment and consumables

Exact route strings include:

``` text
api/equipment/v1/update
api/equipment/v2/getUnlocked

api/consumables/v1/consume
api/consumables/v1/transfer
api/consumables/v1/updateActive
api/consumables/v2/getUnlocked
```

Retrieval endpoints currently use array roots where confirmed.

Mutation routes require separate reconstruction of request bodies and
success/error response semantics.

------------------------------------------------------------------------

# 10. Game/config/content routes

The 20231207 catalogue contains:

``` text
api/communityboard/v2/current
api/gameconfigs/v1/all
api/gamerewards/v1/pending
api/gamerewards/v1/request
api/gamerewards/v1/select
api/images/v2/named
api/playerevents/v1/all
api/quickPlay/v1/getandclear
api/roomkeys/v1/mine
api/subscriptionseasons/v1/seasons/current
```

`/api/playerevents/v1/all` must not be `{}`. Known-good minimal shape:

``` json
{
  "Created": [],
  "Responses": []
}
```

------------------------------------------------------------------------

# 11. Matchmaking

The binary contains:

``` text
matchmake/chatinvite/{0}/{1}
matchmake/club/{0}
matchmake/code/{0}/{1}
matchmake/dorm
matchmake/event/{0}
matchmake/instance/{0}
matchmake/invite/{0}
matchmake/none
matchmake/player/{0}
matchmake/room/{0}
matchmake/room/{0}/{1}
```

Known-good Dorm values:

``` json
{
  "errorCode": 0,
  "roomInstanceId": 1,
  "gameSessionId": 20181,
  "roomId": 1,
  "roomSceneId": 1,
  "photonRegionId": "us",
  "photonRoomId": "1",
  "location": "76d98498-60a1-430c-ab76-b54a29b7a163",
  "roomSceneLocationId": "76d98498-60a1-430c-ab76-b54a29b7a163",
  "private": true,
  "maxCapacity": 1
}
```

Dorm location:

``` text
RoomId: 1
RoomSceneId: 1
Scene: dormroom2
UUID: 76d98498-60a1-430c-ab76-b54a29b7a163
```

Rec Center:

``` text
RoomId: 2
RoomSceneId: 1
Scene: reccenter
UUID: cbad71af-0831-44d8-b8ef-69edafa841f6
```

------------------------------------------------------------------------

# 12. Room-instance protocol

Binary-confirmed paths:

``` text
roominstance/{0}/inprogress
roominstance/{0}/markprivate
roominstance/{0}/matchpolicy
roominstance/{0}/reportjoinresult
```

Observed request:

``` text
/roominstance/1/reportjoinresult
```

with:

``` json
{
  "result": "2",
  "error_code": "8707"
}
```

This happened during an earlier failed join attempt and was followed by
another Dorm matchmaking request. Later runs did not always produce it.

Still required:

``` text
HTTP method for each roominstance operation
request DTOs
success response/body rules
meaning of result/error codes
state transitions after a successful join
```

------------------------------------------------------------------------

# 13. Rooms API catalogue

The client contains:

``` text
rooms/base
rooms/bulk
rooms/carousel/{0}&skip={1}&take={2}
rooms/cheeredby/me
rooms/contestwinners
rooms/contributedby/me
rooms/contributedby/{0}
rooms/createdby/me
rooms/createdby/{0}
rooms/curated_playlists
rooms/favoritedby/me
rooms/fromcreators
rooms/hot
rooms/magic_door
rooms/moderatedby/me
rooms/ownedby/me
rooms/ownedby/{0}
rooms/recommendations
rooms/rro_ids
rooms/search
rooms/topcreators
rooms/visitedby/me
rooms/visitedby/{0}
rooms/{0}
```

Operations include:

``` text
rooms/{0}/accessibility
rooms/{0}/allow_new_users
rooms/{0}/automute
rooms/{0}/bans
rooms/{0}/bans/import
rooms/{0}/bans/{1}
rooms/{0}/clone
rooms/{0}/cloning
rooms/{0}/comments
rooms/{0}/creator
rooms/{0}/description
rooms/{0}/image
rooms/{0}/interactionby/me
rooms/{0}/interactionby/me/cheer
rooms/{0}/interactionby/me/favorite
rooms/{0}/leaderboards/{1}
rooms/{0}/loadscreen
rooms/{0}/max_player_calculation_mode
rooms/{0}/min_level
rooms/{0}/modify
rooms/{0}/name
rooms/{0}/playerdata/me
rooms/{0}/promo_external
rooms/{0}/promo_images
rooms/{0}/requestManualModeration
rooms/{0}/restrictions
rooms/{0}/roles
```

`/rooms/bulk` is already known-good and should not be altered while
reconstructing unrelated routes.

------------------------------------------------------------------------

# 14. Thread/chat contract catalogue

Binary-confirmed:

``` text
thread/club/{0}
thread/message/{0}/moderate
thread/party
thread/withmembers
thread/{0}
thread/{0}/favorite
thread/{0}/joinpartychat
thread/{0}/leave
thread/{0}/member/{1}
thread/{0}/message
thread/{0}/message/{1}/read
thread/{0}/rename
thread/{0}/snooze
```

A previous root-shape correction from `{}` to `[]` fixed one thread
retrieval path. Do not infer that every thread operation returns an
array; mutation/detail routes may use other contracts.

------------------------------------------------------------------------

# 15. Announcements

Exact route catalogue:

``` text
announcements/club/{0}
announcements/club/{0}/{1}
announcements/club/{0}/{1}/read
announcements/mine
announcements/subscription/mine
announcements/v2/mine/unread
announcements/v2/mine/unread?sendAnnouncements=true
announcements/v2/subscription/mine/unread
announcements/v2/subscription/mine/unread?sendAnnouncements=true
```

`/announcements/v2/mine/unread` has an object root.

------------------------------------------------------------------------

# 16. Voice

The binary contains exactly:

``` text
voice/config
voice/requiresModeration
```

It also contains nearby voice-related property/concept strings:

``` text
voiceConnectionInfo
voiceServerId
VoiceSettings
```

Runtime parser evidence is stronger than string proximity:

-   `/voice/config` must return an **object**. An array produced an
    explicit expected-object/actual-array failure.
-   `/voice/requiresModeration` is expected to be a bare JSON boolean
    with high confidence.

The exact `/voice/config` member list remains unresolved. Do not infer
that nearby `voiceConnectionInfo` and `voiceServerId` necessarily belong
to that DTO without a structural link.

------------------------------------------------------------------------

# 17. Club/social boot data

Known:

``` text
club/mine/created
club/mine/member
club/home/me
```

`club/mine/member` is confirmed as an array and is frozen.

`club/home/me` expects an object root. The current `{}` avoids a
root-type error but is not a reconstructed DTO.

The binary contains the diagnostic:

``` text
Unable to GetMyHomeClub
```

Therefore this path has client-visible failure handling and remains a
reconstruction target.

------------------------------------------------------------------------

# 18. Storefront/economy route evidence

The client catalogue contains broad economy families, including:

``` text
api/storefronts/
api/roomconsumables
api/roomcurrencies
api/roomkeys/
api/ugcPurchasables
econ/roomGiftDropShops
econ/roomInventory
econ/roomOffer
purchasecampaign/allcurrent/v2
purchasecampaign/shown
```

It also contains numerous storefront operations, including purchase and
objective routes.

These are not boot priority unless the client blocks on one of them.
Empty retrieval arrays are preferable only where the consumer type is
actually known to be a collection.

------------------------------------------------------------------------

# 19. Additional exact API routes recovered

The catalogue also includes, among many others:

``` text
api/catalog/v1/all?onlyAvailableSkus=true
api/challenge/v2/getCurrent
api/challenge/v2/updateProgress
api/chatreport/createChatReport
api/checklist/v1/complete
api/checklist/v1/current
api/clubreporting/v1/report
api/externalfriendinvite/v1/createplatforminvite
api/externalfriendinvite/v1/getplatformreferrers
api/externalfriendinvite/v1/gettextmessagereferrers
api/externalfriendinvite/v1/sendtextmessageinvite
api/freegifts/v1/sendmultiple
api/incentivizedreferrals/claim
api/incentivizedreferrals/progress
api/incentivizedreferrals/referrals
api/screensharereports/v1/report
api/sanitize/v1
api/sanitize/v1/isPure
```

This proves that the backend surface is substantially larger than the
boot API. It does **not** mean all of these need to be implemented
before the client can reach Dorm.

------------------------------------------------------------------------

# 20. Malformed response investigation

The 20231207 binary contains:

``` text
Malformed Response
Malformed Response: '
```

Runtime logs have produced repeated:

``` text
Malformed Response: '{}'
```

and:

``` text
expected: '['
actual: '{'
```

These must be treated as backend-contract bugs until mapped.

### Reconstruction strategy

For every malformed response:

``` text
timestamp
    ↓
immediately preceding HTTP request
    ↓
current Hydrium response
    ↓
client expected root
    ↓
client expected DTO
```

This is substantially more reliable than globally replacing `{}` with
`[]`.

------------------------------------------------------------------------

# 21. What the client still needs before boot reconstruction is "complete"

## Tier 1 --- immediate boot blockers

1.  Test the full eight-field `AutoMicMutingConfig`.
2.  Identify the endpoint responsible for `expected '[' actual '{'`.
3.  Reconstruct `/strings`.
4.  Reconstruct the actual `/voice/config` DTO.
5.  Resolve any remaining `Malformed Response: '{}'` before the first
    Dorm join.

## Tier 2 --- Dorm entry

6.  Reconstruct room-instance state transitions.
7.  Determine exact success contract for `reportjoinresult`.
8.  Verify whether `inprogress`, `markprivate`, or `matchpolicy` are
    required in the normal Dorm flow.
9.  Preserve the already-working Photon identity and Dorm matchmaking
    contracts.

## Tier 3 --- normal player presentation

10. Reconstruct valid default avatar items.
11. Reconstruct `club/home/me` sufficiently to avoid its failure path.
12. Reconstruct announcements/quick-play/community-board object DTOs if
    empty objects trigger later null access.

## Tier 4 --- broader functionality

13. Storefront/economy.
14. Chat/thread mutations.
15. Clubs.
16. Room creation/editing.
17. Events/rewards.
18. UGC/inventions.
19. Reporting/moderation.
20. Remaining non-boot API catalogue.

------------------------------------------------------------------------

# 22. Recommended Hydrium implementation rule

Do **not** implement a generic handler like:

``` js
return res.json({});
```

for unknown API paths.

During reconstruction, a 404 is diagnostically cleaner than returning a
syntactically valid but semantically incorrect JSON root.

For each endpoint, record:

``` text
PATH
METHOD
QUERY
REQUEST ROOT
REQUEST DTO
RESPONSE STATUS
RESPONSE ROOT
RESPONSE DTO
NULLABILITY
EMPTY-COLLECTION BEHAVIOR
ERROR RESPONSE
SOURCE OF EVIDENCE
```

Then freeze it after a successful 20231207 client test.

------------------------------------------------------------------------

# 23. Current confidence map

### Frozen / do not modify casually

``` text
Photon identity/authentication
/player login flow
/player/qos
/player/connection-info
/player/heartbeat
/matchmake/dorm core response
/rooms/bulk
/Player
/club/mine/member
/api/playerevents/v1/all
```

### Structurally understood, needs final values/DTO completion

``` text
/api/config/v2
/api/config/v1/amplitude
/api/config/v1/backtrace
/api/config/v1/azurespeech
/avatar retrieval family
/voice/config
/voice/requiresModeration
```

### Major unknowns

``` text
/strings response DTO
club/home/me DTO
identity of expected '[' actual '{' endpoint
remaining malformed '{}' responses
normal roominstance post-matchmake sequence
historical/default avatar records
```

------------------------------------------------------------------------

# 24. The reconstruction target

The minimum viable reconstructed server is not "every route in the
executable."

It is:

``` text
authentication
    +
player bootstrap
    +
configuration
    +
localization
    +
avatar bootstrap
    +
Photon
    +
matchmaking
    +
room metadata
    +
room-instance join protocol
```

Once those are correct, the client should have enough backend state to
progress through normal startup and attempt a genuine Dorm session.
Everything else can be added after the boot contract is stable.

------------------------------------------------------------------------

# 25. Next concrete work

The next binary/runtime analysis should focus on only three questions:

``` text
A. Which request causes expected '[' actual '{'?
B. What is the exact /strings response DTO?
C. Which members does /voice/config actually deserialize?
```

Those three answers are currently more valuable than discovering another
hundred route strings.

After that, repeat the same DTO reconstruction process for every
object-shaped boot endpoint that still returns `{}`.

------------------------------------------------------------------------

## Bottom line

The 20231207 client already gives us enough evidence to stop
reconstructing the server by random endpoint guessing.

We now have:

-   the boot endpoint families;
-   a large exact route catalogue;
-   many confirmed response roots;
-   the Config v2 property set;
-   the complete eight-name AutoMicMutingConfig candidate;
-   known Photon and Dorm contracts;
-   explicit client deserialization diagnostics;
-   a short list of high-value unknown DTOs.

The remaining reconstruction is primarily **schema recovery and runtime
correlation**, not endpoint discovery.


---

# 26. Focused reconstruction pass — `[]` mismatch, `/strings`, `/voice/config`

This pass correlates the supplied 20231207 binaries with the retained Hydrium server logs and 20231207 Player logs. Older March-2023 material is used only as a structural cross-check; December runtime behavior remains authoritative.

## 26.1 `expected:'[', actual:'{'` — strongest current attribution

The 20231207 Player logs repeatedly show:

```text
MAOBNNJNPHF: expected:'[', actual:'{', at offset:0
```

The error is a generated JSON collection reader rejecting an object at byte zero. Therefore the wire response for that request must have an **array root**.

Two boot requests are especially important because Hydrium has historically answered them with `{}`:

```text
GET /strings?locale=pt-BR&scopePrefix=client.&eTag=20231207
GET /api/gameconfigs/v1/all
```

The March-2023 static client contract independently proves that `GET api/gameconfigs/v1/all` returns a JSON array of rows shaped as:

```text
Key: String
Value: String
StartTime: Nullable<DateTime>
EndTime: Nullable<DateTime>
```

The 20231207 binary contains the same route. Consequently Hydrium must **not** answer `/api/gameconfigs/v1/all` with `{}`. A safe empty response while no game configs are defined is:

```json
[]
```

### `/strings` correlation

The runtime correlation is even stronger for `/strings`:

```text
GET /strings?locale=pt-BR&scopePrefix=client.&eTag=20231207
```

is followed in the Player log by localization download/deserialization failure, queued localization tables, and then the asynchronous:

```text
expected:'[', actual:'{'
```

Hydrium was returning `{}` for this request.

Therefore the strongest current hypothesis is:

> **`/strings` itself is one source of the `expected '[' actual '{'` failure and its top-level response is an array.**

This is not yet promoted to fully frozen because the exception is asynchronous and the log does not print the request URL inside the exception. However, the temporal correlation plus the known `{}` response makes `/strings` the first route to test.

There may be **more than one** wrong-root endpoint. Fixing `/strings` does not remove the independent requirement to change `/api/gameconfigs/v1/all` to `[]`.

### Immediate root fixes to test

```text
/strings                    {} → []   [strong runtime hypothesis]
/api/gameconfigs/v1/all     {} → []   [strong structural evidence]
```

Do not globally replace objects with arrays.

---

## 26.2 `/strings` — corrected findings

Previous notes associated the metadata string `stringsList` with localization. That association was wrong.

Direct inspection of the 20231207 metadata places `stringsList` inside the metrics subsystem next to:

```text
MetricIdLookup
metricSourceName
metricUnit
nameLookup
namesList
GetMetricNameInfos
metricString
lookupTable
TryGetMetricIdxFromString
stringsList
GetOrRegisterMetricIdxFromString
```

Therefore **`stringsList` is not evidence for the `/strings` HTTP response DTO** and must not be used to design the localization response.

What is genuinely confirmed for the localization request is:

```text
GET /strings
query:
    locale
    scopePrefix
    eTag
```

Observed concrete request:

```text
GET /strings?locale=pt-BR&scopePrefix=client.&eTag=20231207
```

The client binary contains localization diagnostics including:

```text
Failed to retrieve strings
Something went wrong in trying to download strings
Processing queued table
```

and the runtime queues at least:

```text
RecNetErrors
TitleScreen
LoadingScreen
TutorialPrompts
Subtitles
```

### Current `/strings` contract state

```text
HTTP method:       GET                         CONFIRMED
host family:       strings CDN                 CONFIRMED by runtime routing
query locale:      String                      CONFIRMED
query scopePrefix: String                      CONFIRMED
query eTag:        String                      CONFIRMED
response root:     ARRAY                       STRONG RUNTIME HYPOTHESIS
element DTO:       UNKNOWN
empty [] valid?:   MUST BE TESTED
```

The next safe compatibility experiment is therefore an empty JSON array, not `{}`:

```json
[]
```

If `[]` removes both the localization deserializer failure and one `expected '[' actual '{'` exception, the array root is experimentally confirmed. It does **not** prove that an empty array is sufficient for normal localization; the game may still fall back to built-in tables or report missing entries.

The exact per-entry fields remain unrecovered from the current static evidence. Do not invent `Key`, `Value`, `Scope`, `Locale`, or `ETag` members until a structural link to the localization response type is found.

---

## 26.3 `/api/gameconfigs/v1/all`

The structural 2023 contract gives:

```text
GET api/gameconfigs/v1/all

Response:
[
  {
    Key: String,
    Value: String,
    StartTime: Nullable<DateTime>,
    EndTime: Nullable<DateTime>
  }
]
```

The 20231207 binary retains this route and Hydrium calls it repeatedly during boot.

Until December-specific evidence proves a DTO change, the correct minimal response is:

```json
[]
```

Returning `{}` is known to be incompatible with a collection consumer.

**Recommended status:** change to array and freeze the root after a December client test.

---

## 26.4 `/voice/config` — real structural result

The client requires an **object root**. A previous `[]` experiment produced the inverse parser failure: expected object, actual array.

The March-2023 static client contract provides a useful structural cross-check:

```text
GET voice/config
Response DTO: object containing exactly two String properties
```

The exact JSON property names were not recoverable in that March analysis because the properties are obfuscated/attribute-driven.

The 20231207 binary contains:

```text
voice/config
voice/requiresModeration
voiceConnectionInfo
voiceServerId
VoiceSettings
VoiceServerId
```

but string proximity alone does **not** prove that `voiceConnectionInfo` and `voiceServerId` are the two wire keys.

Therefore the current December-safe conclusion is:

```text
GET /voice/config

root:       OBJECT        CONFIRMED
properties: 2 x String    HIGH CONFIDENCE cross-build structural evidence
wire names: UNKNOWN
```

Do not freeze a guessed response such as:

```json
{
  "voiceConnectionInfo": "",
  "voiceServerId": ""
}
```

until the December formatter/call site links those names to the response DTO.

### `/voice/requiresModeration`

Cross-build static evidence and the generic response type agree:

```text
GET /voice/requiresModeration
→ bare JSON Boolean
```

Disabled compatibility response:

```json
false
```

This route is substantially better understood than `/voice/config`.

---

## 26.5 New priority order

The next server tests should be performed independently, one change at a time:

```text
1. /strings
   {} → []

2. /api/gameconfigs/v1/all
   {} → []

3. rerun 20231207 client
   count every expected '[' actual '{'
   count every Malformed Response '{}'
   check localization errors

4. keep /voice/config as an object
   do NOT switch it to []
   recover its two String wire keys before populating it
```

The most important diagnostic result will be whether step 1 removes the localization error and one array/object exception. If it does, `/strings` root becomes CONFIRMED.

---

# 27. Corrected unresolved list

After this pass:

```text
LIKELY SOLVED AT ROOT LEVEL
- /strings                         likely []
- /api/gameconfigs/v1/all          []

CONFIRMED ROOT, DTO INCOMPLETE
- /voice/config                    object; likely two String members

CONFIRMED SCALAR
- /voice/requiresModeration        Boolean

STILL UNKNOWN
- exact /strings element DTO
- exact /voice/config wire property names
- whether additional endpoints also produce expected '[' actual '{'
```

The old statement that `stringsList` supported the localization DTO has been withdrawn.
