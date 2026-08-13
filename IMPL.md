# GPX Cues — Implementation Plan

## 1. Overview

GPX Cues is an Android application that:

1. Loads a GPX file from local storage.
2. The user selects cue delivery options (on-screen, TTS, sound) for this ride or walk.
3. Extracts a predetermined route and cue points from the GPX.
4. Tracks the user's GPS location while riding or walking.
5. Determines whether the user is on or off the route.
6. Detects when the user passes cue locations, including cases where the cue is passed between GPS fixes.
7. Delivers cues sequentially via the selected channels: on-screen (visual), TTS (audio), or sounds (audio).
8. Continues operating reliably with the screen off and the phone in a pocket.
9. Requires no network connection.

The application should be implemented as a **foreground Android location service backed by a stateful route-tracking engine**.

The most important architectural principle is:

> Route progress, rather than geographic proximity alone, is the primary representation of where the rider is on the GPX route.

This is necessary because the route can cross itself or pass through the same latitude/longitude multiple times.

---

# 2. Architecture

```text
┌─────────────────────────────────────────┐
│                UI Layer                 │
│                                         │
│ GPX picker                              │
│ Configuration                           │
│ START / STOP                            │
│ Ride status                             │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│            RideController               │
│                                         │
│ Session lifecycle                       │
│ Configuration                           │
│ Route state                             │
└────────────┬────────────────┬───────────┘
             │                │
             ▼                ▼
┌──────────────────┐  ┌──────────────────┐
│ Location Service │  │ Cue Playback     │
│                  │  │ Queue            │
│ GPS fixes        │  │                  │
│ Background       │  │ TTS              │
│ Foreground svc   │  │ SoundPool        │
│                  │  │ On-screen display│
└────────┬─────────┘  │ Audio focus      │
         │            └──────────────────┘
         ▼
┌─────────────────────────────────────────┐
│              Route Engine               │
│                                         │
│ GPX route                               │
│ Track points                            │
│ Route geometry                          │
│ Route progress                          │
│ On/off-route state                      │
│ Route recovery                          │
│ Cue crossing detection                  │
└─────────────────────────────────────────┘
```

The `RouteEngine` should be independent of Android APIs.

It should accept GPS samples and produce domain events such as:

```text
WentOnRoute
WentOffRoute
CueTriggered
OnTrackHeartbeat
```

This makes the hardest part of the application straightforward to unit test.

---

# 3. Project Structure

Suggested Kotlin project structure:

```text
app/
  src/main/java/.../gpxcues/

    MainActivity.kt

    ui/
      MainScreen.kt
      ConfigurationScreen.kt
      RideScreen.kt
      CueOverlay.kt

    service/
      RideForegroundService.kt

    route/
      GpxParser.kt
      Route.kt
      TrackPoint.kt
      Cue.kt
      RouteEngine.kt
      RouteMatcher.kt
      CueDetector.kt

    location/
      LocationManager.kt
      LocationSample.kt

    audio/
      CuePlaybackQueue.kt
      TtsPlayer.kt
      SoundPlayer.kt
      AudioFocusManager.kt

    persistence/
      Preferences.kt
      SessionState.kt
```

Use Kotlin.

Jetpack Compose is appropriate for the UI unless there is a reason to use traditional Android Views.

---

# 4. Core Data Model

Represent the GPX route internally as track points:

```kotlin
data class TrackPoint(
    val latitude: Double,
    val longitude: Double,
    val elevation: Double?,
    val cue: Cue?
)

data class Cue(
    val text: String?,
    val sound: String?,
    val times: Int = 1
)

data class Route(
    val trackPoints: List<TrackPoint>
)
```

After loading, calculate derived route information:

```kotlin
data class RoutePoint(
    val latitude: Double,
    val longitude: Double,
    val distanceFromStartMeters: Double,
    val cue: Cue?
)
```

For example:

```text
trackpoint 0       0 m
trackpoint 1      14 m
trackpoint 2      31 m
trackpoint 3      49 m
...
trackpoint 200  3821 m
```

`distanceFromStartMeters` gives the route a one-dimensional coordinate system.

This is essential for handling loops and repeated geographic coordinates.

---

# 5. GPX Parsing

Use an Android XML parser such as `XmlPullParser` rather than introducing a third-party GPX library.

Parse:

```xml
<trkpt lat="..." lon="...">
    ...
</trkpt>
```

and look for the cue extensions.

Text cue:

```xml
<extensions>
    <gpxcues:cue>
        <gpxcues:text>turn right next intersection</gpxcues:text>
    </gpxcues:cue>
</extensions>
```

Sound cue:

```xml
<extensions>
    <gpxcues:cue>
        <gpxcues:sound>chime.wav</gpxcues:sound>
    </gpxcues:cue>
</extensions>
```

Combined text, sound, and repetition:

```xml
<extensions>
    <gpxcues:cue>
        <gpxcues:text>turn left in 100 feet</gpxcues:text>
        <gpxcues:sound>chime.wav</gpxcues:sound>
        <gpxcues:times>3</gpxcues:times>
    </gpxcues:cue>
</extensions>
```

The `times` element specifies how many times the sound should be played.
When a cue has both text and sound, the text is spoken first, then the
sound is played `times` times.

## XML namespace handling

Do not depend on the literal `gpxcues:` prefix.

XML namespace prefixes are arbitrary.

The parser should compare:

- namespace URI
- local element name

rather than the textual prefix.

The stable namespace URI for the GPX Cues extension is:

```text
http://example.com/gpxcues
```

Use this URI consistently in your GPX-generation tooling.

---

# 6. GPX File Selection

Use Android Storage Access Framework.

Launch:

```text
ACTION_OPEN_DOCUMENT
```

with:

```text
CATEGORY_OPENABLE
```

and an appropriate GPX MIME type.

Request persistable read permission where supported.

The resulting `content://` URI should be parsed using `ContentResolver`.

For robustness, copy the selected GPX into application-private storage after successful import.

The app still never modifies the user's GPX file. The private copy is only an internal cache.

Flow:

```text
User selects GPX
      ↓
ContentResolver opens URI
      ↓
Parse GPX
      ↓
Validate route
      ↓
Copy GPX into app-private storage
      ↓
Construct Route
      ↓
Show route summary
      ↓
Enable START
```

---

# 7. GPX Validation

Before START is enabled, validate:

- at least two trackpoints
- valid latitude values
- valid longitude values
- valid route geometry
- valid cue structure

A cue may contain one or both of:

```text
text
```

```text
sound
```

A cue may also contain a `times` element specifying how many times the sound should
be played (default 1). The `times` element is only meaningful when `sound` is present.
If `times` is present, it must be a positive integer.

Malformed input should fail explicitly rather than being silently repaired.

Example:

```text
GPX error:
Cue at trackpoint 237 contains neither text nor sound.
```

Also warn if referenced sound files are not yet in the sound registry (see Section 47).

---

# 8. Route Coordinate System

Do not repeatedly perform route geometry calculations directly using latitude/longitude.

For typical bike routes, convert the GPX coordinates into a local meter-based coordinate system.

A local approximation can use:

```text
x = longitude difference × meters per degree longitude
y = latitude difference × meters per degree latitude
```

relative to a suitable origin.

Represent converted coordinates as:

```kotlin
data class XY(
    val x: Double,
    val y: Double
)
```

This makes:

- point-to-segment distance
- route projection
- cue radius calculations
- movement-segment intersection

simple and fast.

---

# 9. Route Position

The GPS position needs to be associated with a position along the route.

Represent it as:

```kotlin
data class RoutePosition(
    val trackPointIndex: Int,
    val distanceAlongRouteMeters: Double,
    val distanceFromRouteMeters: Double
)
```

The important field is:

```text
distanceAlongRouteMeters
```

not simply the nearest trackpoint index.

For example:

```text
0m ─── 1000m ─── 2000m ─── 3000m ─── 4000m
                      ↑
                   rider
```

---

# 10. Route Matching

Do not implement route matching as:

```text
GPS location
    ↓
globally nearest GPX trackpoint
```

That fails when the route overlaps or crosses itself.

Instead, maintain route progress:

```kotlin
lastRouteIndex
lastRouteDistance
```

The matcher should be stateful.

---

# 11. Route State Machine

Use:

```kotlin
enum class RouteState {
    NOT_STARTED,
    OFF_ROUTE,
    ON_ROUTE
}
```

After START:

```text
NOT_STARTED
     │
     ▼
Find route
     │
 ┌───┴────┐
 │        │
near    not near
 │        │
 ▼        ▼
ON      OFF
```

During the ride:

```text
ON_ROUTE
    │
    │ exceeds off-route threshold
    ▼
OFF_ROUTE
    │
    │ route reacquired
    ▼
ON_ROUTE
```

---

# 12. Determining Whether the Rider Is On Route

The configured `routeDeviationThreshold` should represent the maximum distance
from the **route geometry** (not from an individual trackpoint) at which the
rider is still considered ON_ROUTE.

For example:

```text
routeDeviationThreshold = 30m
```

means:

```text
GPS position
     ↓
nearest route segment
     ↓
distance ≤ 30m
     ↓
ON_ROUTE
```

Calculate the minimum distance from the GPS position to route segments.

This works even when GPX trackpoints are relatively far apart.

---

# 13. Efficient Route Matching

Start with a simple implementation that checks all route segments.

For each GPS fix:

```text
for every route segment:
    calculate distance
take minimum
```

This is likely sufficient for typical routes.

For example, even a route with tens of thousands of trackpoints and GPS updates every several seconds represents a relatively small computational workload for a modern phone.

Do not initially introduce an R-tree, geohash index, or other spatial index.

If profiling later demonstrates a performance problem, optimize then.

---

# 14. Stateful Route Progress

Once the rider is on route, search primarily around the previous route position.

Example:

```text
last route index = 500

search:
400 ... 600
```

The search must still allow recovery from GPS noise and deviations.

Use an expanding search:

```text
1. Search around current route progress.
2. If no plausible match, expand the window.
3. If still unsuccessful, perform a global search.
```

The matcher should strongly prefer nearby route progress over a geographically similar point much farther along the route.

---

# 15. Forward Progress

The intended use case is following the route forward.

GPS noise can produce small backward movements:

```text
previous progress = 5000m
new progress      = 4997m
```

This should not cause the engine to believe the rider has substantially moved backward.

Allow a small backwards tolerance.

For example:

```text
progressTolerance = 30m
```

Then small changes within this range are treated as GPS noise.

A large backward jump should be treated as suspicious and should not automatically reset route progress.

Version 1 does not need to explicitly support intentionally riding backward along the route.

---

# 16. Initial Route Interception

The user may press START away from the route.

Initially there is no previous route position, so perform a global route search.

```text
START
  ↓
GPS position
  ↓
nearest route segment
  ↓
within routeDeviationThreshold?
```

If yes:

```text
ON_ROUTE
```

If no:

```text
OFF_ROUTE
```

Continue receiving location updates until the route is intercepted.

---

# 17. Initial Off-Route Notification

If START occurs away from the route:

```text
START
 ↓
OFF_ROUTE
 ↓
off-track cue (built-in)
```

When the route is subsequently intercepted:

```text
OFF_ROUTE
 ↓
ON_ROUTE
 ↓
on-track cue (built-in)
```

This gives immediate confirmation of the route state without requiring the user to look at the screen.

---

# 18. Route Recovery After a Deviation

When the user leaves the route:

```text
last known route progress = 1500m
```

When searching for the route again, do not immediately perform an unrestricted global nearest-point search.

Prefer a route position near the last known progress.

Example:

```text
last known progress = 1500m

search:
1000m ... 2000m
```

If no route match is found, expand the search.

Suggested strategy:

```text
1. Search ±500m around previous route progress.
2. If not found, search ±2km.
3. If still not found, search globally.
```

This supports temporary deviations and prevents a self-intersecting route from causing an incorrect jump to another occurrence of the same geographic location.

---

# 19. Cue Model

Every cue should have a stable ID based on its position in the GPX.

```kotlin
data class RouteCue(
    val id: Int,
    val routeDistanceMeters: Double,
    val latitude: Double,
    val longitude: Double,
    val cue: Cue
)
```

Example:

```text
Cue #0
route distance = 1234m
text = "Turn right"

Cue #1
route distance = 1890m
sound = "chime.wav"

Cue #2
route distance = 2100m
text = "Cross the bridge"

times is optional (defaults to 1).
When present, the sound is repeated that many times.
A cue can have text, sound, or both.
Example with repetition:

```text
Cue #3
route distance = 2500m
text = "Sharp curve"
sound = "chime.wav"
times = 3
```
```

The cue's route position is more important than its geographic coordinate.

---

# 20. Cue Detection

Do not trigger cues only by checking:

```kotlin
distance(currentLocation, cue) < cueRadius
```

This can miss a cue when the user passes through its activation circle between two GPS fixes.

Example:

```text
GPS #1
    ●

      ○ cue radius

              ●
             GPS #2
```

Neither GPS fix may be inside the circle, while the rider passed directly through it.

---

# 21. Segment-to-Circle Cue Detection

For each pair of GPS fixes:

```text
previousLocation
currentLocation
```

treat the movement as a segment.

A cue has been passed when:

```text
distance(
    segment(previousLocation, currentLocation),
    cuePosition
) <= cueRadius
```

This catches cues passed between GPS fixes.

---

# 22. Limit Cue Checks to Relevant Route Progress

Do not compare every GPS segment against every cue.

If:

```text
previous progress = 1000m
current progress  = 1100m
```

only inspect cues near:

```text
970m ... 1130m
```

with an appropriate tolerance.

This improves performance and makes the cue detector follow route progress rather than arbitrary geographic proximity.

---

# 23. Cue Crossing Algorithm

For each GPS fix:

```text
1. Receive GPS location.
2. Match location to route.
3. Determine ON_ROUTE/OFF_ROUTE.
4. If OFF_ROUTE:
       handle off-route state.
       suspend normal cue processing.
5. If ON_ROUTE:
       determine route progress.
6. Compare previous and current route progress.
7. Find cues between the two positions.
8. For each candidate cue:
       test movement segment against cue radius.
9. Queue qualifying cues.
10. Update route state.
```

---

# 24. Cue Delivery Exactly Once

Maintain:

```kotlin
val deliveredCueIds: MutableSet<Int>
```

When a cue is queued, immediately mark it as delivered.

Do not wait for playback to finish.

This prevents duplicate delivery when:

- multiple GPS fixes remain inside the cue radius
- GPS oscillates around a cue
- the audio takes several seconds
- several cues are queued at once

A cue the rider passes while `OFF_ROUTE` is dropped: it is never delivered
retroactively when the route is reacquired, because the rider did not actually
traverse that section of the route. The `deliveredCueIds` set combined with cue
detection being suspended while off-route (Section 25) ensures each cue is
delivered at most once, and only along the portion of the route actually
followed.

---

# 25. Cue Processing While Off Route

Normal route cues should **not** be triggered while the route engine is `OFF_ROUTE`.

Otherwise, during a road closure or temporary deviation, the user might pass geographically near a later cue without actually following that section of the route.

Use:

```text
ON_ROUTE
    ↓
cue detection enabled

OFF_ROUTE
    ↓
cue detection suspended

ON_ROUTE
    ↓
cue detection resumes
```

This matches the assumption that the user intends to follow the GPX route.

Cues passed while `OFF_ROUTE` are dropped, not delivered later upon reacquiring the
route. Detection resumes from the recovered route progress, so only cues ahead of
the last known on-track position are considered going forward.

---

# 26. Cue Playback Queue

Never play audio directly from the GPS callback.

Use:

```text
GPS
 ↓
RouteEngine
 ↓
CueEvent
 ↓
CuePlaybackQueue
 ↓
Audio
```

The playback queue must be serial.

It is initialized with the ride's `DeliveryOptions` (the checkbox selection from the
main loop) and applies them when expanding each `CueTriggered` event into
`PlaybackItem`s — see Section 27.

Example:

```text
Cue A
  ↓
playing
  ↓
finished
  ↓
Cue B
  ↓
finished
  ↓
Cue C
```

A new cue must never interrupt an already-playing cue.

---

# 27. Playback Model

Use:

```kotlin
data class DeliveryOptions(
    val visual: Boolean,
    val tts: Boolean,
    val sound: Boolean
)

sealed class PlaybackItem {
    data class Text(val text: String) : PlaybackItem()
    data class Sound(val filename: String, val times: Int = 1) : PlaybackItem()
}
```

The queue owns a single worker.

Each cue can be delivered through the channels enabled in `DeliveryOptions`:
- visual — on-screen display (high brightness, timeout)
- tts — verbal text through TextToSpeech
- sound — sound playback through SoundPool

When a CueTriggered event is received, the Cue is expanded into PlaybackItems
according to the active DeliveryOptions and document order (text before sound):

- If `text` is present:
    - if `tts` is enabled, add a Text(text) item delivered through TTS
    - if `visual` is enabled, the cue text is also shown on-screen (Section 27.1)
- If `sound` is present and `sound` delivery is enabled, add a Sound(filename, times)
  item delivered through SoundPool.

Items are added in document order (text before sound).

```text
Cue(text="turn left", sound="chime.wav", times=3)
    ↓
[Text("turn left"), Sound("chime.wav", times=3)]
    ↓
queue
```

A new cue must never interrupt an already-playing cue.

Conceptually:

```text
while running:
    item = queue.receive()

    acquireAudioFocus()

    play(item)

    wait until complete

    releaseAudioFocus()
```

For a Sound item with times > 1, play plays the sound times times
and wait until complete waits for all repetitions to finish.

### 27.1 On-screen display

When visual delivery is enabled, the cue text is shown on-screen at very high
brightness. The foreground service wakes the screen and presents the cue via the
active Activity window, then turns the screen off after the configured
`onScreenTimeoutSeconds` to preserve battery. On-screen delivery is most useful
when the app is visible; with the screen off and the phone in a pocket, the
audio channels (TTS and sounds) remain the primary delivery.

---

# 28. Audio Focus

Use Android Audio Focus with:

```text
AUDIOFOCUS_GAIN_TRANSIENT
```

and navigation/guidance-style audio attributes.

Desired behavior:

```text
Music playing
     ↓
GPX cue
     ↓
request transient audio focus
     ↓
music ducks/pauses
     ↓
"Turn right"
     ↓
release focus
     ↓
music resumes
```

Test this behavior with the actual music application and Bluetooth devices used during rides.

---

# 29. TTS

Initialize Android `TextToSpeech` from the foreground service.

Persist the selected voice identifier rather than merely a language name.

Expose an abstraction such as:

```kotlin
suspend fun speak(text: String)
```

Use `UtteranceProgressListener` to determine when speech finishes.

The playback queue must not start the next item until the TTS utterance has completed.

---

# 30. Sound Playback

For short sound effects such as the built-in on-track and off-track cue sounds,
and cue sounds (e.g. `chime.wav`, `on-route.wav`, `off-route.wav`), use
`SoundPool`.

Preload configured sounds when the ride starts, including cue sounds from the
sound file registry (Section 47) and notification sounds.
Use WAV or OGG format for compatibility with SoundPool.

Then, per DeliveryOptions:

```text
Sound → SoundPool   (if sound enabled)
Text  → TTS         (if tts enabled)
Text  → On-screen   (if visual enabled)
```

The built-in on-track and off-track cues (Section 31–33) are delivered through
this same path, so they respect the active DeliveryOptions — e.g. a
sound-based cue plays only when sound delivery is enabled.

All sound files — shipped defaults and user imports, see Section 47 — resolve
through the local sounds directory and must be short (under ~1 s, under ~250 KB)
WAV or OGG so `SoundPool` can preload them reliably without `TIMED_OUT` errors.

---

# 31. On-Track Heartbeat

Configuration:

```text
distance since last cue
```

should mean:

> Notify the user after they have traveled this many meters without a route cue.

For example:

```text
setting = 2 km

cue at 10 km
     ↓
ride
     ↓
12 km
     ↓
on-track cue (built-in)
```

Maintain:

```kotlin
lastRouteCueProgress
lastOnTrackNotificationProgress
```

Trigger when:

```text
routeProgress - lastRouteCueProgress
    >= configuredDistance
```

Do not count off-track notifications as route cues.

---

# 32. On-Track Notification After Recovery

When the state changes:

```text
OFF_ROUTE → ON_ROUTE
```

immediately queue the configured built-in on-track cue.

Example:

```text
ON ROUTE
   ↓
OFF ROUTE
   ↓
off-track cue (built-in)
   ↓
route recovered
   ↓
on-track cue (built-in)
```

This notification should happen regardless of the normal heartbeat distance.

---

# 33. Off-Route Notifications

When entering `OFF_ROUTE`:

```text
play immediately
```

Then maintain:

```kotlin
lastOffRouteNotificationTime
```

While still off route:

```text
if elapsed >= configured interval:
    deliver the off-track cue
```

Example:

```text
OFF ROUTE
   ↓
sound
   ↓
30 sec
   ↓
sound
   ↓
30 sec
   ↓
sound
```

---

# 34. GPS Location Handling

Use Android's fused location provider.

Request location updates using the configured interval.

For example:

```text
GPS interval = 5 seconds
```

The OS may not provide updates at exactly that interval.

The route engine must tolerate:

```text
5 sec
8 sec
20 sec
45 sec
```

between fixes.

This is another reason cue detection must inspect the movement segment between fixes rather than only the current GPS coordinate.

If GPS is temporarily unavailable, the engine holds its last known state rather than making aggressive route-state transitions; no fresh fix means no cue detection and no state change until a new location arrives.

---

# 35. GPS Accuracy

Every location update includes an accuracy estimate.

Use it as part of route-state confidence.

For example:

```text
accuracy = 6m
```

is substantially more trustworthy than:

```text
accuracy = 80m
```

At minimum, classify extremely poor fixes as low-confidence and avoid making aggressive route-state transitions based on a single bad measurement.

---

# 36. Route-State Hysteresis

Do not use a single threshold:

```text
≤ 30m = on route
> 30m = off route
```

GPS noise can produce:

```text
29m
31m
28m
32m
30m
33m
```

and cause:

```text
ON
OFF
ON
OFF
ON
OFF
```

Instead use hysteresis.

Example:

```text
ON_ROUTE threshold  = routeDeviationThreshold (30m)
OFF_ROUTE threshold = routeDeviationThreshold + hysteresis (45m)
```

Thus:

```text
OFF → ON
    distance <= routeDeviationThreshold

ON → OFF
    distance >= routeDeviationThreshold + hysteresis
```

The `routeDeviationThreshold` is the user-configured setting.
The hysteresis is an internal constant (default 15m).

This accounts for GPS noise: a reading slightly below the threshold won't
immediately cause a state transition, and a large backward jump due to
inaccuracy won't reset route progress.

---

# 37. GPS Accuracy and Hysteresis

Optionally account for GPS accuracy when determining an effective threshold.

A conceptual implementation:

```text
effective threshold =
    min(
        max(configured routeDeviationThreshold + hysteresis, GPS accuracy),
        maximumAllowed
    )
```

Do not allow a very poor GPS accuracy estimate to produce an arbitrarily large route deviation threshold.

---

# 38. Foreground Service

The ride tracker should run as an Android:

```text
ForegroundService
```

with the location foreground-service type.

The foreground service owns:

- location updates
- route engine
- cue queue
- TTS
- audio playback
- ride state

The Activity must not own these responsibilities.

The Activity can disappear when:

- the screen turns off
- another application is opened
- the Activity is recreated

The foreground service should continue operating independently.

---

# 39. Persistent Foreground Notification

Show a persistent notification such as:

```text
GPX Cues
Ride in progress
12.4 km
ON ROUTE
```

Optionally include a STOP action.

This also provides a useful debugging/status mechanism while the screen is off.

---

# 40. Battery Optimization

Use the foreground location service as the primary mechanism for background reliability.

Additionally support:

```text
ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
```

as requested by the specification.

Treat battery optimization exemption as a setup/reliability measure rather than as a replacement for the foreground service.

Configuration UI can display:

```text
Background operation

Battery optimization:
Disabled
```

with an action to request exemption if necessary.

---

# 41. Permissions

Plan for the permissions required by the target Android SDK, including as applicable:

```text
ACCESS_FINE_LOCATION
ACCESS_COARSE_LOCATION
FOREGROUND_SERVICE
FOREGROUND_SERVICE_LOCATION
POST_NOTIFICATIONS
```

Request permissions through the UI before START.

Design around foreground-service location operation rather than assuming that unrestricted background location access is necessary.

Test against the exact Android version and target SDK used by the APK.

---

# 42. Ride Lifecycle

Use an explicit application/session state machine:

```text
IDLE
 │
 │ select GPX
 ▼
ROUTE_LOADED
 │
 │ START
 ▼
STARTING
 │
 ├──────────────┐
 ▼              ▼
ON_ROUTE      OFF_ROUTE
 │              │
 └──────┬───────┘
        │
        │ STOP
        ▼
STOPPING
        │
        ▼
IDLE
```

This prevents UI state and service state from becoming inconsistent.

---

# 43. STOP Behavior

When STOP is pressed:

```text
1. Stop accepting new GPS fixes.
2. Stop location updates.
3. Stop the route engine.
4. Finish the currently playing cue.
5. Discard queued future cues.
6. Release audio focus.
7. Stop or release TTS as appropriate.
8. Stop the foreground service.
9. Return UI to idle.
```

Do not allow queued cues to continue playing after the user has explicitly stopped the ride.

---

# 44. Process Death

Persist enough state to reconstruct an active session:

```text
active GPX
route identifier
route progress
route point index
route state
delivered cue IDs
last GPS position
```

A simple serialized state object or DataStore entry is sufficient.

On service restart:

```text
restore route
     ↓
restore progress
     ↓
obtain fresh GPS
     ↓
re-evaluate route
     ↓
continue
```

Do not attempt to reconstruct every transient event.

---

# 45. Configuration Persistence

Use Android DataStore Preferences.

Persist:

```text
cueRadiusMeters
routeDeviationThresholdMeters
ttsVoiceId
deliveryVisual
deliveryTts
deliverySound
onTrackCue
onTrackDistanceMeters
offTrackCue
offTrackIntervalSeconds
gpsIntervalSeconds
onScreenTimeoutSeconds
```

Potentially also persist:

```text
hysteresisMeters
```

but initially keep hysteresis internal rather than exposing another configuration option.

---

# 46. Configuration UI

Suggested screen:

```text
GPX Cues
────────────────────────

Route
    [ Select GPX ]

Cue delivery
    ☐ On-screen (visual)
    ☐ TTS (audio)
    ☑ Sounds (audio)

Cue radius
    15 m

Route deviation
    30 m

Sound files
    [ Manage ]

GPS interval
    5 sec

TTS voice
    English (US)

Built-in on-track cue
    [ Select sound ]

On-track heartbeat
    Every [ 2.0 ] km without a cue

Built-in off-track cue
    [ Select sound ]

Off-track notification
    Every [ 30 ] sec

On-screen notification time
    5 sec

Background operation
    Battery optimization: Disabled

────────────────────────

              [ START ]
```

Cue delivery checkboxes default to the persisted values but can be adjusted per ride.

During a ride:

```text
GPX Cues

Route: Lakeshore.gpx

● ON ROUTE

Distance: 18.4 km
Next cue: 0.7 km

              [ STOP ]
```

A map is not necessary for version 1.

---

# 47. Audio File Selection

Use the Storage Access Framework to import sound files.

Launch:

```text
ACTION_OPEN_DOCUMENT
```

with:

```text
CATEGORY_OPENABLE
```

and MIME type:

```text
audio/*
```

Allow **multiple** selection so the user can import many sounds at once (e.g.
a batch downloaded from Freesound or Pixabay). Request persistable read
permission only where supported.

Copy each selected file into the app's **local sounds directory** (see below)
and index it by basename. After copying, the app no longer depends on the
original `content://` URI, so imports survive document revocation or storage
unmount.

## Sound File Registry

Sound files are managed through a **filename-keyed registry backed by the local
sounds directory**.

- When a GPX cue references a sound by filename (e.g., `chime.wav`), the app
  looks up the matching file in the registry **by basename**.
- The user can import any number of sound files via the configuration screen's
  "Sound files" section, which opens the Storage Access Framework (multiple
  selection). Each imported file is copied into the local sounds directory and
  becomes immediately selectable.
- The built-in on-track and off-track cue sounds are also selected from this
  registry.
- If a cue references a sound filename that is not in the registry, the sound
  portion of the cue is skipped (the text portion, if present, still plays).
- Android does not ship standard named sound files that can be reliably
  referenced by basename, so the app ships its own defaults (see below) and the
  user can import their own WAV or OGG files.

## Local Sounds Directory

There is a single app-private directory that backs the registry:

```text
<app-files-dir>/sounds/
```

(`context.getFilesDir()/sounds`.) Everything selectable in the app lives here,
so a user who simply drops or imports many files into it gets them all as
selectable options with no further wiring:

- **Shipped defaults.** On first run, the app copies a small set of default cue
  sounds (e.g. `on_track.wav`, `off_track.wav`, `chime.wav`, `beep.wav`,
  `drumstick.wav`, `clap.wav`) from `res/raw` into this directory. Using one
  source of truth means user imports and shipped defaults are treated
  identically and can override each other by basename.
- **User imports.** Files pulled in via the Storage Access Framework above are
  copied here by basename. Re-importing a file with the same name overwrites the
  previous copy.
- **Discovery.** The configuration UI's `Sound files` / `Select sound` pickers
  are populated by scanning this directory (basename -> selectable item). No
  external URI persistence is required for playback.

## Downloading Sounds

The app itself never downloads sounds (no network dependency during setup or
rides). Sources the user can fetch files from and then import into the local
sounds directory:

- **Pixabay** — Pixabay License (effectively CC0); free for commercial use, no
  attribution required, redistributable. Good for short UI chimes/beeps/claps.
  `https://pixabay.com/sound-effects/`
- **OpenGameArt** — filter to CC0 for license-clean SFX.
  `https://opengameart.org/`
- **Freesound** — enormous; **filter by license "Creative Commons 0"** before
  bundling/importing (CC-BY requires in-app attribution, which is awkward for a
  sound-effects app). Useful per-type search links:
  - chimes/dings:
    `https://freesound.org/search/?q=chime&f=license:"Creative%20Commons%200"`
  - clicks/blips: same license filter, `q=blip`
  - rimshot/drumstick: same license filter, `q=drumstick`
  - claps: same license filter, `q=clap`

For the shipped defaults and any user experiments, prefer short clips
(**< 1 s**, **< 250 KB**) encoded as **WAV or OGG** so `SoundPool` can preload
them reliably (see Section 30).

---

# 48. Route Engine API

The route engine should have a small interface:

```kotlin
class RouteEngine(
    private val route: Route,
    private val config: RouteConfig
) {
    fun start(initialLocation: Location): List<RouteEvent>

    fun processLocation(location: Location): List<RouteEvent>

    fun stop()
}
```

Events:

```kotlin
sealed class RouteEvent {
    data object WentOnRoute : RouteEvent()

    data object WentOffRoute : RouteEvent()

    data class CueTriggered(
        val cue: RouteCue
    ) : RouteEvent()

    data object OnTrackHeartbeat : RouteEvent()
}
```

The Android service should not need to understand route geometry.

---

# 49. Example Cue Processing

Route:

```text
0m ─── 100m ─── 200m ─── 300m ─── 400m
                              ↑
                         "turn right"
```

GPS:

```text
GPS #1 → 250m
GPS #2 → 350m
```

The engine sees:

```text
previous progress = 250m
current progress  = 350m
```

and identifies a cue at 300m.

It then checks whether the movement segment between the two GPS positions passed within the cue radius.

If so:

```text
CueTriggered("turn right")
```

The playback layer produces:

```text
TTS: "turn right"
```

even though neither GPS fix necessarily lies within the cue radius.

---

# 50. Multiple Cues Between GPS Fixes

Suppose:

```text
GPS #1 = 1000m
GPS #2 = 1500m

Cue A = 1100m
Cue B = 1200m
Cue C = 1400m
```

The engine should emit:

```text
CueTriggered(A)
CueTriggered(B)
CueTriggered(C)
```

in route order.

The playback queue then plays:

```text
A
wait
B
wait
C
```

No cues are dropped.

---

# 51. Closely Spaced Cues

If:

```text
Cue A = 1000m
Cue B = 1001m
Cue C = 1002m
```

the route engine still produces:

```text
A
B
C
```

The playback queue serializes them.

No cue interrupts another cue.

---

# 52. Cue Behind the Rider

GPS noise may cause:

```text
previous progress = 1000m
current progress  = 980m
```

Do not repeatedly process the 980–1000m interval as newly traveled route.

The delivered-cue set protects against duplicates, but the route matcher should also suppress small backward movements as GPS noise.

---

# 53. Route Crossing Itself

Example:

```text
          │
          │
──────────┼──────────
          │
          │
```

The same geographic position can occur at multiple route positions.

A global nearest-point search might jump from:

```text
route progress = 4000m
```

to:

```text
route progress = 12000m
```

even though the rider has only traveled a few meters.

The stateful route matcher instead searches around the previous route progress and therefore preserves the correct occurrence.

---

# 54. GPS Samples

Maintain:

```kotlin
previousLocation
currentLocation
```

and:

```kotlin
previousRoutePosition
currentRoutePosition
```

These are required for segment-based cue detection.

The complete GPS history does not need to be permanently stored.

---

# 55. Logging

Implement structured debug logging from the beginning.

For each GPS fix:

```text
12:41:23
GPS 42.12345,-87.12345
accuracy 8m
route distance 13.42m
route progress 1823m
state ON_ROUTE
```

State transitions:

```text
12:44:11
ON_ROUTE -> OFF_ROUTE
distance = 51m
```

Cue events:

```text
12:45:03
CUE #17
route progress = 2142m
distance from cue = 12m
```

This will be invaluable during real-world testing.

---

# 56. Development Debug Screen

Create a development-only screen displaying:

```text
GPS
    latitude / longitude
    accuracy
    speed
    last fix time

Route
    state
    distance from route
    route progress
    nearest segment
    next cue

Audio
    queue length
    current cue
    TTS state

Service
    running
    last GPS timestamp
```

This should make diagnosing real-world GPS and route-matching behavior much easier.

---

# 57. Unit Tests

The route engine should have extensive JVM unit tests and should not require Android.

## Basic cue

```text
route:
0m, 100m, 200m

cue:
100m

GPS:
50m → 120m

expected:
cue fires
```

## Missed GPS fix

```text
GPS:
50m → 150m

cue:
100m

expected:
cue fires
```

## Outside cue radius

```text
GPS segment remains 50m from cue
cueRadius = 30m

expected:
no cue
```

## Duplicate fixes

```text
GPS:
50m
100m
105m
110m

expected:
one cue
```

---

# 58. Loop and Crossing Tests

Construct synthetic routes where geographic coordinates overlap.

Verify that:

```text
route progress = 100m
```

does not unexpectedly jump to:

```text
route progress = 5000m
```

when both locations have essentially identical latitude/longitude.

Test multiple cues around the crossing as well.

---

# 59. Deviation Tests

Simulate:

```text
route:
A ───────────── B
             ↑
          deviation
```

Expected state sequence:

```text
ON
OFF
OFF
OFF
ON
```

Expected notifications:

```text
off-track cue
on-track cue
```

exactly once for the corresponding state transitions.

---

# 60. Playback Queue Tests

Submit:

```text
Cue A
Cue B
Cue C
```

rapidly.

Verify:

```text
A starts
A finishes
B starts
B finishes
C starts
C finishes
```

Never:

```text
A starts
B interrupts A
```

---

# 61. Background Testing

Test on a real Android phone.

## Screen off

```text
START
lock phone
put phone in pocket
ride
```

Verify GPS and audio continue.

## Music

```text
start music
start GPX Cues
trigger TTS
```

Verify the cue is clearly audible and music resumes appropriately.

## Bluetooth

Test:

- phone speaker
- headphones
- Bluetooth headphones
- other intended audio devices

## Battery

Test a multi-hour ride.

---

# 62. GPS Testing

Test:

1. walking
2. cycling
3. dense urban environment
4. open roads
5. route crossing itself
6. poor GPS conditions
7. screen off
8. phone in pocket
9. intentional route deviation
10. route recovery

The most important test is intentionally missing a route segment and then returning to the route.

---

# 63. Implementation Phases

## Phase 1 — Project Skeleton

Implement:

- Android project
- Kotlin
- Compose UI
- permissions
- DataStore
- basic screens
- APK build/install

Deliverable:

```text
App opens and configuration works.
```

---

## Phase 2 — GPX Import

Implement:

- Storage Access Framework
- GPX XML parser
- namespace handling
- route model
- cue parsing
- validation
- sound-file handling

Deliverable:

```text
Select GPX
    ↓
Trackpoints: 4,823
Distance: 37.4 km
Cues: 27
```

---

## Phase 3 — Route Geometry Engine

Implement:

- latitude/longitude → local XY
- route segment distances
- cumulative route distances
- point-to-segment distance
- nearest-route calculation

Write unit tests before continuing.

Deliverable:

```text
GPS coordinate
      ↓
distance from route
route progress
nearest segment
```

---

## Phase 4 — Route State Machine

Implement:

- initial route acquisition
- ON_ROUTE
- OFF_ROUTE
- hysteresis
- route recovery
- progress tracking
- self-intersection handling

This is the core milestone.

Deliverable:

```text
GPS simulation
      ↓
correct ON/OFF state
correct route progress
```

---

## Phase 5 — Cue Detection

Implement:

- cue route positions
- segment/circle intersection
- passed-cue detection
- delivered cue IDs
- multiple cues
- missed GPS fixes

Deliverable:

```text
GPS stream
    ↓
correct cue events
```

---

## Phase 6 — Audio

Implement:

- foreground service
- TTS
- SoundPool
- audio focus
- serial playback queue
- sound configuration

Deliverable:

```text
GPS
 ↓
cue
 ↓
audible notification
```

---

## Phase 7 — Background Reliability

Implement:

- foreground notification
- location foreground service
- battery optimization flow
- screen-off behavior
- process restart recovery
- persistent session state

Deliverable:

```text
phone locked
phone in pocket
music playing
ride continues
```

---

## Phase 8 — Real-World Validation

Build a test GPX containing:

- 10+ cues
- closely spaced cues
- a loop
- a route crossing
- a temporary deviation
- a cue passed between GPS fixes

Then ride it in the real world.

Only after that should the following be tuned:

- cue radius
- route deviation threshold
- hysteresis
- GPS interval
- recovery search distance
- heartbeat distance

---

# 64. Initial Configuration Defaults

Recommended initial values:

| Setting | Default |
|---|---:|
| Cue radius | 15 m |
| Route deviation threshold | 30 m |
| GPS interval | 5 sec |
| On-track heartbeat | 2 km |
| Off-route notification | 30 sec |
| Route hysteresis | 15 m |
| Normal progress search | ±500 m |
| Route recovery search | ±2 km |

These are starting points for testing, not necessarily final values.

---

# 65. MVP Acceptance Test

Version 1 should be considered complete when this complete scenario works reliably:

```text
Load GPX
   ↓
START while 100m away
   ↓
OFF_ROUTE sound
   ↓
move toward route
   ↓
ON_ROUTE sound
   ↓
follow route
   ↓
GPS fixes every 5 seconds
   ↓
pass a cue between two GPS fixes
   ↓
cue plays
   ↓
pass two cues rapidly
   ↓
both play sequentially
   ↓
route crosses itself
   ↓
correct route occurrence is used
   ↓
temporarily leave route
   ↓
OFF_ROUTE sound
   ↓
return to route
   ↓
ON_ROUTE sound
   ↓
continue
   ↓
previously delivered cue is never repeated
   ↓
screen remains off
   ↓
music is ducked/paused for cues
   ↓
STOP
   ↓
tracking and audio cease
```

---

# 66. Key Design Decision

The application should **not** be implemented as:

```text
GPS → nearest GPX point → cue
```

Instead:

```text
                     GPX
                      │
                      ▼
          ┌─────────────────────┐
          │ Route + progress    │
          │ 0 ... N meters      │
          └──────────┬──────────┘
                     │
GPS ─────────────────▼
              Route Matcher
                     │
          ┌──────────┴──────────┐
          │                     │
      ON_ROUTE              OFF_ROUTE
          │                     │
          ▼                     ▼
  progress tracking       recovery search
          │
          ▼
 cue crossing detection
          │
          ▼
    CuePlaybackQueue
          │
      ┌───┴────┐
      ▼        ▼
     TTS      Sound
```

The fundamental abstraction is **route progress**, not latitude/longitude.

That one decision solves the hardest requirements:

- routes that cross themselves
- repeated latitude/longitude
- missed GPS fixes
- temporary deviations
- route interception
- exactly-once cues
- correct cue ordering
- recovery near the user's previous position

The route matcher and cue detector should therefore receive the largest share of engineering and testing effort. The UI, GPX import, TTS, and foreground-service plumbing are comparatively straightforward once this core is well isolated.
