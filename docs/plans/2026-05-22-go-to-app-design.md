# go_to_app — Design

A simple, local-first compass app that points to a saved destination, with distance and walk-time readouts. Built in Flutter for Android and mobile web.

## Goals

- Show a compass arrow that points to a saved destination.
- Display distance (m below 1000, km with two decimals at/above).
- Display two walk times: at a fixed pace (default 5 km/h) and at the user's current measured pace.
- Let the user save multiple named destinations and swap the active one.
- Keep everything local. No accounts, no network calls, no analytics.

## Non-goals

- Turn-by-turn navigation.
- Route planning, road following, or anything map-routing related.
- Cloud sync, sharing, or multi-user features.
- iOS native build at v1 (Flutter keeps the option open).
- Desktop / laptop web (no magnetometer).

## Platforms

- **Android** (min SDK 24).
- **Mobile web** (Android Chrome, iOS Safari). HTTPS required for Geolocation and DeviceOrientation.

## Tech stack

- Flutter (stable) + Dart.
- Riverpod for state.
- go_router for navigation.
- Drift for local SQLite (works on mobile + web via `sql.js`).
- shared_preferences for settings.
- flutter_map for the map picker (OpenStreetMap, no API key).
- geolocator for GPS.
- flutter_compass for heading on mobile; a custom `CompassService` wraps `DeviceOrientationEvent` on web (`webkitCompassHeading` on iOS Safari, `deviceorientationabsolute` on Android Chrome).
- permission_handler for runtime permissions on mobile.
- wakelock_plus to keep the screen on while the compass screen is visible.

## Screens

1. **Onboarding** (first launch only) — welcome, location permission, add first destination.
2. **Compass** (home) — the main view. Top bar with active-destination dropdown and settings icon.
3. **Destinations list** — view, edit, delete, mark active, add new.
4. **Add / edit destination** — name plus one of: use current location, pick on map, enter coordinates.
5. **Settings** — walking pace, proximity radius, magnetic vs true north, keep screen on, proximity haptic, on-target haptic.

## Compass screen layout

```
┌─────────────────────────────────┐
│  ▾ Car            ⚙              │  active destination + settings
├─────────────────────────────────┤
│           ╱ ╲                    │
│          ╱   ╲      N            │
│         │  ↑  │   W   E          │  rotating dial, fixed arrow
│          ╲   ╱      S            │
│           ╲ ╱                    │
│         Bearing 47°              │
├─────────────────────────────────┤
│         347 m                    │  distance
│  Walk: 4 min  · arrive 14:35     │  fixed-pace walk time + ETA
│  At your pace: 6 min             │  current-speed walk time
│  GPS ±8 m                        │  accuracy
└─────────────────────────────────┘
```

- Arrow turns green and a haptic fires when within ±10° of target heading (on-target haptic).
- Distance text turns green and a haptic fires once when entering the proximity radius (default 50 m).
- Long-press the distance to copy the destination's coordinates to the clipboard.

## Core logic

### Heading

- `SensorManager` `TYPE_ROTATION_VECTOR` (via flutter_compass on mobile); `DeviceOrientationEvent` on web.
- Bearing to target = great-circle bearing from current fix to target.
- Arrow rotation = `bearingToTarget − deviceAzimuth`, normalized to (−180°, 180°].
- Low-pass filter on the device azimuth (α ≈ 0.15) to stop jitter.
- True north correction via `GeomagneticField` declination, computed locally from lat/lon/altitude/time.

### Distance

- Great-circle distance (Vincenty under the hood from geolocator).
- Format: `< 1000 m` → `"347 m"`; `≥ 1000 m` → `"3.42 km"` (two decimals).

### Walk time — fixed pace

- Default 5.0 km/h, configurable 2–8 km/h in settings.
- `seconds = distance_m / pace_m_s`. Formatted as `"12 min"` or `"1 h 23 min"`.
- Also show ETA wall-clock time ("arrive 14:35").

### Walk time — current speed

- `Position.speed` from geolocator.
- Rolling 10-second average to smooth GPS noise.
- If smoothed speed < 0.5 m/s, show `"—"`.

### Update cadence

- GPS at 1 Hz.
- Sensor at SENSOR_DELAY_GAME (~50 Hz).
- UI throttled to ~10 Hz.

## Data model

### Drift (SQLite)

```dart
class Destinations extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  RealColumn get latitude => real()();
  RealColumn get longitude => real()();
  IntColumn get createdAt => integer()();
}
```

### Settings (shared_preferences)

| Key | Type | Default |
|---|---|---|
| `activeDestinationId` | int? | null |
| `walkingPaceKmh` | double | 5.0 |
| `proximityRadiusM` | int | 50 |
| `useTrueNorth` | bool | true |
| `keepScreenOn` | bool | true |
| `proximityHaptic` | bool | true |
| `onTargetHaptic` | bool | true |
| `onboardingComplete` | bool | false |

## Project structure

```
lib/
├── main.dart
├── app.dart                  # MaterialApp + router
├── data/
│   ├── db.dart               # Drift database
│   ├── destinations_dao.dart
│   └── settings_service.dart
├── domain/
│   ├── geo.dart              # bearing, distance, formatters
│   └── models.dart
├── services/
│   ├── location_service.dart
│   └── compass_service.dart  # mobile + web impls
├── features/
│   ├── onboarding/
│   ├── compass/
│   ├── destinations/
│   └── settings/
└── ui/
    └── widgets/
```

## Permissions

### Mobile

- Location is the only runtime permission needed.
- Onboarding asks once with rationale: "We need your location to point the compass at your destinations. Nothing leaves your phone."
- On deny: app still works for adding destinations by coordinates or map; compass screen shows a "Grant location permission" CTA.
- On permanent deny: same CTA, but it opens app Settings.

### Web

- Geolocation prompt fires the first time the user taps "Use my location."
- On iOS Safari, the first time we need orientation, show a "Tap to enable compass" button that calls `DeviceOrientationEvent.requestPermission()` inside a user gesture.

## Error states

| Condition | UI |
|---|---|
| No active destination | Empty state: "Add your first destination →" |
| Location permission missing | CTA card on top of compass |
| GPS searching (no fix yet) | Dial visible; distance / walk shown as "—"; caption "Acquiring GPS…" |
| GPS fix stale (> 10 s) | Amber caption "GPS signal weak" |
| No magnetometer | Hide dial; show large "Bearing 47°" text instead |
| Stationary (< 0.5 m/s) | "At your pace: —" |
| Within proximity radius | Distance green; one haptic; "You've arrived" caption |

## Edge cases

- **Antipodal target** (> 20 000 km): computes fine; no special case.
- **Same-spot target**: bearing undefined → arrow hidden, distance "0 m", proximity haptic fires.
- **Deleted active destination**: auto-pick the most recently created remaining one; empty state if list is empty.
- **Backgrounded app**: stop sensor + GPS listeners on pause; resume on resume.
- **Keep-screen-on** toggle applies only on the compass screen.

## Testing

### Unit (Dart, fast)

- `geo.dart` — distance, bearing, formatters.
- Speed smoothing — rolling 10-second averager.
- Low-pass filter — convergence and noise rejection on synthetic input.
- Active-destination fallback when the active one is deleted.

### Widget

- Compass empty state.
- Permission-missing CTA renders when permission is denied.
- Distance turns green inside proximity radius.
- Stationary "At your pace: —".
- Destination dropdown lists all saved destinations with the active one marked.

### Golden (light)

- Compass dial snapshots at 0°, 90°, 180°.

### Skipped at v1

- End-to-end tests on real devices.
- Comprehensive mocking of sensor / GPS streams beyond simple Riverpod fakes.

## Build / release

- **Android:** `flutter build apk --release`, `flutter build appbundle` for Play Store.
- **Web:** `flutter build web`; host any static host with HTTPS.
- No backend, no CI required at v1. A GitHub Actions workflow running `flutter test` + `flutter analyze` on push is a reasonable later addition.

## Future ideas (out of scope for v1)

- Optional altitude delta for hiking.
- Optional notes field per destination.
- Custom icons / colors per destination.
- Compass calibration helper (figure-8 gesture).
- A small home-screen widget that shows distance to the active destination.
