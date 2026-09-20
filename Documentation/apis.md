# Hydrium --- REC ROOM 20231207 API Reconstruction

> **Target build:** Rec Room `20231207`\
> **Steam:** App `471710` · Depot `471711` · Manifest
> `1151455856673601091`\
> **Purpose:** preservation/interoperability documentation for the
> Hydrium backend.\
> **Rule:** this file separates confirmed behavior from structural
> candidates. Do not invent DTO fields or use `{}` as a universal
> fallback.

## 1. Current boot chain

The 20231207 client contains the following player endpoints:

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

Observed/implemented boot flow:

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
/matchmake/dorm
    ↓
room join/reporting
```

### Photon state --- confirmed/frozen

Current account used in testing: `178933005`.

Photon application IDs:

``` text
Realtime: 390700d6-7387-4fa5-b6d7-7ee39e46ad4a
Voice:    91c066ff-4be0-4c77-9733-d9e5831051a5
Chat:     5daaf075-d152-4c42-a4f7-47391cbff9be
Region:   us
```

`/photon/auth` now resolves the requested `accountId` dynamically. The
previous hard-coded Photon UserId mismatch is fixed. Exclusive login
accepts the current identity. Do not change this path without new
evidence.

## 2. Matchmaking

Routes present in the 20231207 client include:

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

Known-good Dorm matchmaking response uses:

``` text
errorCode: 0
roomInstanceId: 1
gameSessionId: 20181
roomId: 1
roomSceneId: 1
photonRegionId: "us"
photonRoomId: "1"
location: "76d98498-60a1-430c-ab76-b54a29b7a163"
roomSceneLocationId: "76d98498-60a1-430c-ab76-b54a29b7a163"
private: true
maxCapacity: 1
```

Dorm:

``` text
RoomId:      1
RoomSceneId: 1
UUID:        76d98498-60a1-430c-ab76-b54a29b7a163
Scene:       dormroom2
```

Rec Center:

``` text
RoomId:      2
RoomSceneId: 1
UUID:        cbad71af-0831-44d8-b8ef-69edafa841f6
Scene:       reccenter
```

`/rooms/bulk` is known-good and should remain frozen.

The client also contains room-instance operations including
`reportjoinresult`, `inprogress`, `markprivate`, and `matchpolicy`. A
historical join attempt sent:

``` json
{"result":"2","error_code":"8707"}
```

to `/roominstance/1/reportjoinresult`.

## 3. Response root-type matrix

This is the most important rule while reconstructing the API: **do not
return `{}` for every unknown endpoint**. A valid JSON document with the
wrong root type can deserialize incorrectly and produce misleading boot
failures.

  ---------------------------------------------------------------------------------------------------------
  Endpoint                                        Expected root / minimal shape     Status
  ----------------------------------------------- --------------------------------- -----------------------
  `/api/avatar/v1/defaultbaseavataritems`         `[]`                              High confidence

  `/api/avatar/v1/defaultunlocked`                `[]`                              High confidence

  `/api/avatar/v4/items`                          `[]`                              High confidence

  `/api/avatar/v2/gifts`                          `[]`                              High confidence

  `/api/equipment/v2/getUnlocked`                 `[]`                              Confirmed

  `/api/consumables/v2/getUnlocked`               `[]`                              Confirmed

  `/api/images/v2/named`                          `[]`                              Confirmed

  `/api/gamerewards/v1/pending`                   `[]`                              Confirmed

  `/api/roomkeys/v1/mine`                         `[]`                              Confirmed

  `/api/storefronts/v4/balance/:currencyType`     `[]`                              Confirmed root

  `/api/communityboard/v2/current`                `{}` / object DTO                 Confirmed root

  `/api/subscriptionseasons/v1/seasons/current`   `{}` / object DTO                 Confirmed root

  `/api/quickPlay/v1/getandclear`                 `{}` / object DTO                 Confirmed root

  `/announcements/v2/mine/unread`                 `{}` / object DTO                 Confirmed root

  `/voice/config`                                 `{}` / object DTO                 Parser-confirmed root

  `/voice/requiresModeration`                     JSON boolean                      Strong structural
                                                                                    evidence

  `/api/playerevents/v1/all`                      `{"Created":[],"Responses":[]}`   Confirmed

  `/api/config/v1/azurespeech`                    object DTO                        Confirmed root

  `/api/config/v1/amplitude`                      object DTO                        Confirmed root

  `/api/config/v1/backtrace`                      object DTO                        Confirmed root

  `/api/config/v2`                                object DTO                        Confirmed

  `/rooms/bulk`                                   array                             Known-good

  `/rooms/createdby/me`                           `[]`                              Confirmed

  `/club/mine/member`                             `[]`                              Confirmed/frozen

  `/club/home/me`                                 object DTO                        Root known; schema
                                                                                    incomplete

  `/Player`                                       array                             Known-good/frozen

  `/thread`                                       array where applicable            `{}`→`[]` fix confirmed

  `/api/PlayerReporting/v1/voteToKickReasons`     `[]`                              Confirmed

  `/api/ugcPurchasables/v1/items/room/:roomId`    `[]`                              Confirmed

  `/outfits/me/saved`                             `[]`                              Confirmed

  room consumables/currencies                     `[]`                              Confirmed

  `/iam/me/channels/Announcements`                `[]`                              Confirmed
  ---------------------------------------------------------------------------------------------------------

The player log still contains at least one deserialization error
equivalent to:

``` text
expected: '['
actual: '{'
```

Therefore at least one remaining Hydrium route is returning an object
where the client expects an array. The responsible endpoint has not yet
been mapped.

## 4. Config API

Routes found in the 20231207 client:

``` text
api/config/
api/config/v1/amplitude
api/config/v1/azurespeech
api/config/v1/backtrace
api/config/v1/freegiftbutton
api/config/v2
```

### `/api/config/v2`

Current working structure:

``` js
router.get("/api/config/v2", (req, res) => {
    return res.status(200).json({
        LevelProgressionMaps: [],
        DailyObjectives: [],
        ServerMaintenance: null,

        AutoMicMutingConfig: {
            MicSpamSamplePercentageForForceMute: 0.0,
            MicSpamSamplePercentageForForceMuteToEnd: 0.0,
            MicSpamSamplePercentageForWarning: 0.0,
            MicSpamSamplePercentageForWarningToEnd: 0.0,
            MicSpamVolumeThreshold: 0.0,
            MicSpamWarningStateVolumeMultiplier: 0.0,
            MicVolumeSampleInterval: 0.0,
            MicVolumeSampleRollingWindowLength: 0.0
        },

        StorefrontConfig: {
            AwardCurrencyCooldownSeconds: 0.0
        },

        RoomKeyConfig: {
            MaxKeysPerRoom: 0
        },

        RoomCurrencyConfig: {
            MinPlayerLevelForGifting: 0
        },

        ShareBaseUrl: "",

        MessageOfTheDay: "Welcome To Hydra Net",
        CdnBaseUri: "",

        MatchmakingParams: {
            PreferFullRoomsFrequency: 1.0,
            PreferEmptyRoomsFrequency: 0.0
        },

        PhotonConfig: {
            CloudRegion: "us",
            CrcCheckEnabled: true
        },

        ConfigTable: []
    });
});
```

The eight `AutoMicMutingConfig` property names are recovered from the
20231207 binary and correspond to the eight-float configuration
structure associated with the AudioManager path. The **official numeric
values have not been recovered yet**. The zero values above are test
values, not historical defaults.

Previously, `AutoMicMutingConfig: {}` did **not** remove:

``` text
JBHPHMDFHAF..ctor (JBDHBAKEONF DIIHIAJGIIB)
RecRoom.Audio.AudioManager.Initialize()
System.NullReferenceException
```

The next experiment is to determine whether supplying all eight
properties changes or removes that stack.

### Backtrace

Current route uses numeric `CaptureNativeCrashes`, because the December
2023 client parser rejected the boolean interpretation used by older
documentation:

``` js
router.get("/api/config/v1/backtrace", (req, res) => {
    return res.status(200).json({
        ANRThresholdMs: 0,
        CaptureNativeCrashes: 0,
        FilterType: 0,
        LogLineCount: 0,
        MessageCount: 0,
        MessageRegex: "",
        ReportBudget: 0,
        SampleRate: 0.0,
        VersionRegex: ""
    });
});
```

Freeze unless new December-2023 evidence contradicts it.

### Azure Speech

Structural candidate:

``` json
{
  "Enabled": false,
  "Key": "",
  "Region": ""
}
```

The exact December values remain unconfirmed.

### Amplitude

Known structural candidate:

``` json
{
  "AmplitudeKey": "",
  "RudderStackKey": "",
  "UseRudderStack": false,
  "StatSigKey": ""
}
```

Again, structure and historical values must be distinguished.

## 5. Avatar API

Routes recovered from the client include:

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

Known structural information:

-   `defaultbaseavataritems` → array.
-   `defaultunlocked` → array.
-   `v4/items` → array.
-   `defaultbaseavataritems` and `defaultunlocked` use
    unlocked-avatar-item-style DTOs.
-   Known item fields include `AvatarItemType` (integer),
    `AvatarItemDesc` (string), and `AvatarItemId` (integer).
-   Do not invent default items merely to populate the avatar. IDs must
    be valid and unique.

Related equipment/consumable routes:

``` text
api/equipment/v1/update
api/equipment/v2/getUnlocked
api/consumables/v1/consume
api/consumables/v1/transfer
api/consumables/v1/updateActive
api/consumables/v2/getUnlocked
```

## 6. Other boot-visible API routes

Confirmed as present in the 20231207 client:

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

Known special shape:

``` json
{
  "Created": [],
  "Responses": []
}
```

for `/api/playerevents/v1/all`.

## 7. Rooms API inventory

Recovered route family includes:

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

Room operations found include:

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

Presence in the client does **not** by itself establish HTTP method or
response DTO. Those must be reconstructed separately.

## 8. Thread/chat inventory

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

A previous Hydrium fix changed an incorrectly object-shaped `/thread`
response from `{}` to `[]`. Response shape can vary by specific thread
operation; do not generalize blindly to every route in the family.

## 9. Announcements inventory

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

## 10. Voice

Routes present:

``` text
voice/config
voice/requiresModeration
```

`/voice/config` must have an object root. Returning `[]` produced an
explicit parser error: expected object, actual array.

`/voice/requiresModeration` is structurally expected to return a bare
JSON boolean; use `false` as the disabled compatibility response unless
December evidence shows otherwise.

ToxMod/moderation failures seen later in boot occur **after** the first
AudioManager/Core Systems failure, so they are not currently treated as
the trigger for that first NRE.

## 11. Strings/localization

Observed request:

``` text
GET /strings?locale=pt-BR&scopePrefix=client.&eTag=20231207
```

Current Hydrium `{}` response is wrong/incomplete. The client reports
deserialization/localization failures and queues scopes. Exact response
schema has not yet been reconstructed.

Do **not** guess a replacement root until the consuming DTO is
recovered.

## 12. Known malformed-response evidence

Player logs contain:

``` text
Malformed Response: '{}'
```

multiple times.

They also contain:

``` text
expected: '['
actual: '{'
```

before the first Core Systems AudioManager failure.

This means backend schema cleanup is still required independently of the
AudioManager investigation.

Known later issues include:

``` text
Unable to GetMyHomeClub
ToxMod initialization failure
```

These occur later and should not be conflated with the first
AudioManager crash.

## 13. Implementation policy

For every route, classify in this order:

``` text
route exists
    ↓
HTTP method
    ↓
root JSON type
    ↓
DTO/member names and primitive types
    ↓
required vs optional members
    ↓
historical/default values
```

Root types:

``` text
List / Array     → []
DTO / Class      → {...}
Boolean          → true / false
String           → ""
Composite DTO    → exact documented structure
No response body → empty response where confirmed
```

### Do not

-   Do not use `{}` as a universal fallback.
-   Do not invent DTO fields.
-   Do not copy March-2023 structures into December without checking.
-   Do not modify frozen Photon, `/rooms/bulk`, `/Player`,
    `/club/mine/member`, or other known-good routes without new
    evidence.
-   Do not treat a route string found in metadata as proof of its HTTP
    method or response schema.
-   Do not treat zero test values as historical Rec Room defaults.

### Preferred workflow

1.  Implement one endpoint at a time.
2.  Test the 20231207 client after each change.
3.  Compare the first changed error/stack.
4.  Freeze routes once their root/schema is experimentally confirmed.
5.  Preserve logs showing both successful and failed deserialization.

## 14. Priority reconstruction queue

Current priorities:

1.  Test the complete eight-field `AutoMicMutingConfig`.
2.  Identify the endpoint responsible for `expected '[' actual '{'`.
3.  Reconstruct `/strings`.
4.  Reconstruct the exact `/voice/config` DTO.
5.  Reconstruct boot-visible object DTOs that currently return `{}`.
6.  Recover valid default avatar item responses.
7.  Continue mapping GET response roots from the 20231207 metadata
    before implementing the broader API inventory.

## 15. Confidence legend

-   **Confirmed / frozen** --- observed working with the 20231207 client
    or directly constrained by its parser.
-   **High confidence** --- supported by 20231207 metadata plus observed
    behavior/type information.
-   **Structural candidate** --- DTO/route evidence exists, but
    December-specific behavior or values remain unverified.
-   **Unknown** --- route exists, but method/response contract has not
    yet been reconstructed.

------------------------------------------------------------------------

### Current objective

The immediate boot blocker remains:

``` text
JBHPHMDFHAF..ctor(JBDHBAKEONF DIIHIAJGIIB)
    at RecRoom.Audio.AudioManager.Initialize()
    → System.NullReferenceException
    → Error initializing Core Systems
```

The working hypothesis is now testable: provide the complete eight-float
`AutoMicMutingConfig` object and observe whether this stack disappears
or changes. Independently, continue eliminating wrong response root
types so unrelated malformed responses do not obscure the boot sequence.
