GPX Cues
========

Mobile app that tracks my bike ride or walk along a predetermined route specified by a GPX file
and gives me audio cues (sounds and verbal messages through TTS) based on lat/lon coordinates

# Main Loop

1. I select a GPX file from local storage and tap START.
2. App starts tracking location along the route from the GPX file.
3. When current location is at or near a GPX trkpt with a cue, app plays a sound or says a message.
4. At the end of the ride, I tap STOP.

# Examples of GPX file (fragments)

## Say text through TTS

```
<trkpt lat="0.0" lon="0.0">
  <extensions>
    <gpxcues:cue>
      <gpxcues:text>turn right next intersection</gpxcues:text>
    </gpxcues:cue>
  </extensions>
</trkpt>
```

## Play sound once

```
<trkpt lat="0.0" lon="0.0">
  <extensions>
    <gpxcues:cue>
      <gpxcues:sound>chime.wav</gpxcues:sound>
    </gpxcues:cue>
  </extensions>
</trkpt>
```

## Say text through TTS *and* play sounds 3 times

```
<trkpt lat="0.0" lon="0.0">
  <extensions>
    <gpxcues:cue>
      <gpxcues:text>turn left in 100 feet</gpxcues:text>
      <gpxcues:sound>chime.wav</gpxcues:sound>
      <gpxcues:times>3</gpxcues:times>
    </gpxcues:cue>
  </extensions>
</trkpt>
```

# Constraints and Clarifications

## General

* This app only reads GPX files, never writes. I will create these files with my own tooling

## Following the route

* Each cue can be given only once
* For overlapping or rapid cues - queue them, don't drop them and don't interrupt current cue
* Assume I intend to follow the route from GPX file
  - no need to implement any rerouting logic
* I could tap START while not being on the specified route
  - in this case, detect when I am off route initially and detect when I intercept it
* While riding, I could deviate slightly from the route (typically due to unexpected road closures)
  - in this case, assume I will try to get back on the route as quickly as possible
* App will be getting GPS location at a certain interval
  - if I passed a cue between GPS fix points, the app should deliver it as soon as possible
* I could be passing through the same lat/lon (but different trkpt!) several times during a ride
  - app must track my location and my progress along the route
  - when I intercept the route after being off-track, assume it's the closes trkpt to my last known on-track position
* On-track notifications
  - play some sound when I am on track and last cue was a while ago so that I know that the app
    didn't crash and that I am on the route
  - also play this sound when I get on track after being off-track
* Off-track notifications
  - play sound when I am off-track (both initially and during deviations)
* I could be listening to music while riding
  - app must ensure cues are heard (AUDIOFOCUS_GAIN_TRANSIENT)

# Tech Stack

* Android
* No internet connection (app should work without internet)
* Need to build APK (I will install on a phone in development mode)
* Read GPX files from local storage with Storage Access Framework (I will download manually)
* App must reliably run in the background (ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS)
  - screen will be off and phone will be in a pocket
* GPS could be noisy


# Options for Configuration Screen

* radius - determines a circle around lat/lon where a cue becomes active
* TTS voice - app will give me verbal notifications in this voice
* on-track notification sound
* distance since last cue for on-track notification
* off-track notification sound
* how frequently to send off-track notifications
* GPS fix interval - get GPS location once per this many seconds
