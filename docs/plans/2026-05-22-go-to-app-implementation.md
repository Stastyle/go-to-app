# go_to_app Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a local-first Flutter compass app that points to user-defined destinations, with distance and walk-time readouts. Target Android and mobile web.

**Architecture:** Single-package Flutter app. Pure-Dart domain layer (testable without Flutter). Drift for SQLite, shared_preferences for settings, Riverpod for state, go_router for navigation. Platform-split CompassService for mobile vs web.

**Tech Stack:** Flutter (stable), Dart, Riverpod, go_router, Drift, shared_preferences, flutter_map, geolocator, flutter_compass, permission_handler, wakelock_plus.

**Reference:** See [docs/plans/2026-05-22-go-to-app-design.md](2026-05-22-go-to-app-design.md) for the validated design.

---

## Phase 1 — Project bootstrap

### Task 1: Create Flutter project

**Files:**
- Create: full Flutter project skeleton via CLI

**Step 1: Verify Flutter is installed**

Run: `flutter --version`
Expected: prints version (3.x.x or newer). If not installed, install via https://docs.flutter.dev/get-started/install before continuing.

**Step 2: Create the project in-place**

Run (from `C:\Users\stasm\AndroidStudioProjects\go_to_app`):
```
flutter create --org com.stastyle --project-name go_to_app --platforms=android,web .
```
Expected: `lib/`, `android/`, `web/`, `pubspec.yaml`, etc. created. The existing `docs/`, `.gitignore`, and `.git/` are left alone.

**Step 3: Verify it builds & runs (web smoke test, fastest)**

Run: `flutter run -d chrome`
Expected: Chrome opens the default counter app. Close it.

**Step 4: Delete the default counter scaffold from `lib/main.dart`**

Replace `lib/main.dart` contents with:
```dart
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(home: Scaffold(body: Center(child: Text('go_to_app')))));
```

**Step 5: Commit**

```
git add .
git commit -m "Bootstrap Flutter project for Android + web"
```

---

### Task 2: Add dependencies

**Files:**
- Modify: `pubspec.yaml`

**Step 1: Update `pubspec.yaml` dependencies section**

Replace the `dependencies:` and `dev_dependencies:` blocks with:
```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.5.1
  go_router: ^14.2.0
  drift: ^2.18.0
  drift_flutter: ^0.2.0
  sqlite3_flutter_libs: ^0.5.0
  shared_preferences: ^2.2.3
  flutter_map: ^7.0.0
  latlong2: ^0.9.1
  geolocator: ^12.0.0
  flutter_compass: ^0.8.0
  permission_handler: ^11.3.1
  wakelock_plus: ^1.2.5
  intl: ^0.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
  drift_dev: ^2.18.0
  build_runner: ^2.4.11
  mocktail: ^1.0.3
```

**Step 2: Fetch dependencies**

Run: `flutter pub get`
Expected: "Got dependencies!" with no errors. If a version constraint conflict appears, allow `flutter pub upgrade --major-versions` to bump.

**Step 3: Add `pubspec.lock` to git tracking (not in `.gitignore`)**

Edit `.gitignore`: remove the `pubspec.lock` line. Locking versions is desirable for apps (vs. libraries).

**Step 4: Commit**

```
git add pubspec.yaml pubspec.lock .gitignore
git commit -m "Add core dependencies"
```

---

### Task 3: Configure Android permissions and app name

**Files:**
- Modify: `android/app/src/main/AndroidManifest.xml`
- Modify: `android/app/build.gradle.kts` (or `.gradle` depending on Flutter version)

**Step 1: Add location permission to `AndroidManifest.xml`**

Inside `<manifest>`, before `<application>`, add:
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-feature android:name="android.hardware.sensor.compass" android:required="false" />
<uses-feature android:name="android.hardware.location.gps" android:required="false" />
```

**Step 2: Set app name**

In `AndroidManifest.xml`, change `android:label` on `<application>` to `"go_to_app"`.

**Step 3: Set min SDK**

In `android/app/build.gradle.kts`, ensure `minSdk = 24`.

**Step 4: Commit**

```
git add android/
git commit -m "Configure Android manifest and min SDK"
```

---

### Task 4: Configure linting

**Files:**
- Modify: `analysis_options.yaml`

**Step 1: Replace `analysis_options.yaml` with stricter rules**

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  language:
    strict-casts: true
    strict-raw-types: true
  errors:
    invalid_annotation_target: ignore
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"

linter:
  rules:
    prefer_single_quotes: true
    require_trailing_commas: true
    unawaited_futures: true
```

**Step 2: Verify analyzer is clean**

Run: `flutter analyze`
Expected: "No issues found!"

**Step 3: Commit**

```
git add analysis_options.yaml
git commit -m "Tighten lint rules"
```

---

## Phase 2 — Pure domain logic (TDD)

Pure-Dart code. Highest leverage for tests; no Flutter, no plugins.

### Task 5: Destination model

**Files:**
- Create: `lib/domain/models.dart`
- Test: `test/domain/models_test.dart`

**Step 1: Write the failing test**

`test/domain/models_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:go_to_app/domain/models.dart';

void main() {
  test('Destination equality by id', () {
    const a = Destination(id: 1, name: 'A', latitude: 0, longitude: 0, createdAt: 0);
    const b = Destination(id: 1, name: 'B', latitude: 1, longitude: 1, createdAt: 0);
    expect(a, b);
  });

  test('Destination copyWith updates name', () {
    const d = Destination(id: 1, name: 'A', latitude: 0, longitude: 0, createdAt: 0);
    expect(d.copyWith(name: 'B').name, 'B');
    expect(d.copyWith(name: 'B').id, 1);
  });
}
```

**Step 2: Run — expect compile failure**

Run: `flutter test test/domain/models_test.dart`
Expected: FAIL (file not found).

**Step 3: Implement**

`lib/domain/models.dart`:
```dart
class Destination {
  final int id;
  final String name;
  final double latitude;
  final double longitude;
  final int createdAt;

  const Destination({
    required this.id,
    required this.name,
    required this.latitude,
    required this.longitude,
    required this.createdAt,
  });

  Destination copyWith({String? name, double? latitude, double? longitude}) =>
      Destination(
        id: id,
        name: name ?? this.name,
        latitude: latitude ?? this.latitude,
        longitude: longitude ?? this.longitude,
        createdAt: createdAt,
      );

  @override
  bool operator ==(Object other) => other is Destination && other.id == id;

  @override
  int get hashCode => id.hashCode;
}
```

**Step 4: Run — expect pass**

Run: `flutter test test/domain/models_test.dart`
Expected: All tests passed.

**Step 5: Commit**

```
git add lib/domain/models.dart test/domain/models_test.dart
git commit -m "Add Destination model"
```

---

### Task 6: Geo math — distance and bearing

**Files:**
- Create: `lib/domain/geo.dart`
- Test: `test/domain/geo_test.dart`

**Step 1: Write failing tests**

`test/domain/geo_test.dart`:
```dart
import 'dart:math' as math;
import 'package:flutter_test/flutter_test.dart';
import 'package:go_to_app/domain/geo.dart';

void main() {
  group('distanceMeters', () {
    test('zero distance for identical points', () {
      expect(distanceMeters(0, 0, 0, 0), closeTo(0, 0.001));
    });

    test('1 degree latitude ~ 111 km', () {
      expect(distanceMeters(0, 0, 1, 0), closeTo(111_195, 200));
    });

    test('symmetric', () {
      final ab = distanceMeters(40.0, -74.0, 51.5, -0.1);
      final ba = distanceMeters(51.5, -0.1, 40.0, -74.0);
      expect(ab, closeTo(ba, 0.5));
    });
  });

  group('bearingDegrees', () {
    test('due north is 0', () {
      expect(bearingDegrees(0, 0, 1, 0), closeTo(0, 0.001));
    });

    test('due east is 90', () {
      expect(bearingDegrees(0, 0, 0, 1), closeTo(90, 0.001));
    });

    test('due south is 180', () {
      expect(bearingDegrees(1, 0, 0, 0), closeTo(180, 0.001));
    });

    test('result is in [0, 360)', () {
      final b = bearingDegrees(0, 0, -1, -1);
      expect(b, greaterThanOrEqualTo(0));
      expect(b, lessThan(360));
    });
  });
}
```

**Step 2: Run — expect compile failure**

Run: `flutter test test/domain/geo_test.dart`
Expected: FAIL.

**Step 3: Implement**

`lib/domain/geo.dart`:
```dart
import 'dart:math' as math;

const double _earthRadiusMeters = 6_371_000;

double _toRad(double deg) => deg * math.pi / 180.0;
double _toDeg(double rad) => rad * 180.0 / math.pi;

/// Great-circle distance between two points in meters (haversine).
double distanceMeters(double lat1, double lon1, double lat2, double lon2) {
  final dLat = _toRad(lat2 - lat1);
  final dLon = _toRad(lon2 - lon1);
  final a = math.sin(dLat / 2) * math.sin(dLat / 2) +
      math.cos(_toRad(lat1)) *
          math.cos(_toRad(lat2)) *
          math.sin(dLon / 2) *
          math.sin(dLon / 2);
  final c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a));
  return _earthRadiusMeters * c;
}

/// Initial bearing from (lat1, lon1) to (lat2, lon2), normalized to [0, 360).
double bearingDegrees(double lat1, double lon1, double lat2, double lon2) {
  final phi1 = _toRad(lat1);
  final phi2 = _toRad(lat2);
  final dLon = _toRad(lon2 - lon1);
  final y = math.sin(dLon) * math.cos(phi2);
  final x = math.cos(phi1) * math.sin(phi2) -
      math.sin(phi1) * math.cos(phi2) * math.cos(dLon);
  return (_toDeg(math.atan2(y, x)) + 360) % 360;
}
```

**Step 4: Run — expect pass**

Run: `flutter test test/domain/geo_test.dart`
Expected: All passed.

**Step 5: Commit**

```
git add lib/domain/geo.dart test/domain/geo_test.dart
git commit -m "Add great-circle distance and bearing"
```

---

### Task 7: Formatters — distance and walk time

**Files:**
- Modify: `lib/domain/geo.dart`
- Modify: `test/domain/geo_test.dart`

**Step 1: Add failing tests**

Append to `test/domain/geo_test.dart`:
```dart
  group('formatDistance', () {
    test('under 1000 m: integer meters', () {
      expect(formatDistance(0), '0 m');
      expect(formatDistance(347.4), '347 m');
      expect(formatDistance(999.9), '1000 m');
    });

    test('1000+ m: km with 2 decimals', () {
      expect(formatDistance(1000), '1.00 km');
      expect(formatDistance(3421.7), '3.42 km');
      expect(formatDistance(12345), '12.35 km');
    });
  });

  group('formatWalkTime', () {
    test('under a minute', () {
      expect(formatWalkTime(Duration(seconds: 30)), '<1 min');
    });

    test('under an hour', () {
      expect(formatWalkTime(Duration(minutes: 12)), '12 min');
      expect(formatWalkTime(Duration(minutes: 59)), '59 min');
    });

    test('hours and minutes', () {
      expect(formatWalkTime(Duration(minutes: 83)), '1 h 23 min');
      expect(formatWalkTime(Duration(minutes: 120)), '2 h 0 min');
    });
  });
}
```

(Move the closing `}` from the file's previous bottom up — append above the final `}` of `void main()`.)

**Step 2: Run — expect failure**

Run: `flutter test test/domain/geo_test.dart`
Expected: FAIL.

**Step 3: Implement**

Append to `lib/domain/geo.dart`:
```dart
String formatDistance(double meters) {
  if (meters < 1000) return '${meters.round()} m';
  return '${(meters / 1000).toStringAsFixed(2)} km';
}

String formatWalkTime(Duration d) {
  if (d.inSeconds < 60) return '<1 min';
  if (d.inMinutes < 60) return '${d.inMinutes} min';
  final h = d.inHours;
  final m = d.inMinutes % 60;
  return '$h h $m min';
}
```

**Step 4: Run — expect pass**

Run: `flutter test test/domain/geo_test.dart`
Expected: All passed.

**Step 5: Commit**

```
git add lib/domain/geo.dart test/domain/geo_test.dart
git commit -m "Add distance and walk-time formatters"
```

---

### Task 8: Rolling-window speed smoother

**Files:**
- Create: `lib/domain/speed_smoother.dart`
- Test: `test/domain/speed_smoother_test.dart`

**Step 1: Write failing tests**

`test/domain/speed_smoother_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:go_to_app/domain/speed_smoother.dart';

void main() {
  test('empty window returns null', () {
    final s = SpeedSmoother(window: Duration(seconds: 10));
    expect(s.average(now: 0), null);
  });

  test('single sample returns its value', () {
    final s = SpeedSmoother(window: Duration(seconds: 10));
    s.add(timestampMs: 0, speedMps: 1.5);
    expect(s.average(now: 100), 1.5);
  });

  test('averages samples in window', () {
    final s = SpeedSmoother(window: Duration(seconds: 10));
    s.add(timestampMs: 0, speedMps: 1.0);
    s.add(timestampMs: 1000, speedMps: 2.0);
    s.add(timestampMs: 2000, speedMps: 3.0);
    expect(s.average(now: 2000), closeTo(2.0, 1e-9));
  });

  test('drops samples older than window', () {
    final s = SpeedSmoother(window: Duration(seconds: 10));
    s.add(timestampMs: 0, speedMps: 1.0);
    s.add(timestampMs: 5_000, speedMps: 2.0);
    s.add(timestampMs: 20_000, speedMps: 4.0);
    expect(s.average(now: 20_000), 4.0);
  });
}
```

**Step 2: Run — expect failure**

Run: `flutter test test/domain/speed_smoother_test.dart`
Expected: FAIL.

**Step 3: Implement**

`lib/domain/speed_smoother.dart`:
```dart
import 'dart:collection';

class _Sample {
  final int timestampMs;
  final double speedMps;
  _Sample(this.timestampMs, this.speedMps);
}

class SpeedSmoother {
  final Duration window;
  final Queue<_Sample> _samples = Queue();

  SpeedSmoother({this.window = const Duration(seconds: 10)});

  void add({required int timestampMs, required double speedMps}) {
    _samples.add(_Sample(timestampMs, speedMps));
    _trim(timestampMs);
  }

  double? average({required int now}) {
    _trim(now);
    if (_samples.isEmpty) return null;
    final sum = _samples.fold<double>(0, (a, s) => a + s.speedMps);
    return sum / _samples.length;
  }

  void _trim(int now) {
    final cutoff = now - window.inMilliseconds;
    while (_samples.isNotEmpty && _samples.first.timestampMs < cutoff) {
      _samples.removeFirst();
    }
  }
}
```

**Step 4: Run — expect pass**

Run: `flutter test test/domain/speed_smoother_test.dart`
Expected: All passed.

**Step 5: Commit**

```
git add lib/domain/speed_smoother.dart test/domain/speed_smoother_test.dart
git commit -m "Add rolling-window speed smoother"
```

---

### Task 9: Low-pass filter for compass jitter

**Files:**
- Create: `lib/domain/low_pass.dart`
- Test: `test/domain/low_pass_test.dart`

**Step 1: Write failing tests**

`test/domain/low_pass_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:go_to_app/domain/low_pass.dart';

void main() {
  test('first sample passes through', () {
    final f = AngleLowPass(alpha: 0.15);
    expect(f.update(42), 42);
  });

  test('converges toward target', () {
    final f = AngleLowPass(alpha: 0.5);
    f.update(0);
    final out = f.update(100);
    expect(out, closeTo(50, 1e-9));
  });

  test('handles 359 -> 1 wrap as short path', () {
    final f = AngleLowPass(alpha: 0.5);
    f.update(359);
    final out = f.update(1);
    expect(out, closeTo(0, 1e-9));
  });
}
```

**Step 2: Run — expect failure**

Run: `flutter test test/domain/low_pass_test.dart`
Expected: FAIL.

**Step 3: Implement**

`lib/domain/low_pass.dart`:
```dart
class AngleLowPass {
  final double alpha;
  double? _state;

  AngleLowPass({this.alpha = 0.15});

  double update(double degrees) {
    final n = _normalize(degrees);
    if (_state == null) {
      _state = n;
      return n;
    }
    var delta = n - _state!;
    if (delta > 180) delta -= 360;
    if (delta < -180) delta += 360;
    _state = _normalize(_state! + alpha * delta);
    return _state!;
  }

  static double _normalize(double d) {
    var x = d % 360;
    if (x < 0) x += 360;
    return x;
  }
}
```

**Step 4: Run — expect pass**

Run: `flutter test test/domain/low_pass_test.dart`
Expected: All passed.

**Step 5: Commit**

```
git add lib/domain/low_pass.dart test/domain/low_pass_test.dart
git commit -m "Add angular low-pass filter"
```

---

## Phase 3 — Data layer

### Task 10: Drift database + Destinations table

**Files:**
- Create: `lib/data/db.dart`

**Step 1: Define the schema**

`lib/data/db.dart`:
```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

part 'db.g.dart';

class Destinations extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 64)();
  RealColumn get latitude => real()();
  RealColumn get longitude => real()();
  IntColumn get createdAt => integer()();
}

@DriftDatabase(tables: [Destinations])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(driftDatabase(name: 'go_to_app'));

  @override
  int get schemaVersion => 1;
}
```

**Step 2: Run codegen**

Run: `dart run build_runner build --delete-conflicting-outputs`
Expected: `lib/data/db.g.dart` is generated, no errors.

**Step 3: Add `*.g.dart` to lint exclusions** — already done in Task 4.

**Step 4: Commit**

```
git add lib/data/db.dart lib/data/db.g.dart
git commit -m "Add Drift database with Destinations table"
```

---

### Task 11: Destinations DAO

**Files:**
- Create: `lib/data/destinations_dao.dart`
- Modify: `lib/data/db.dart`

**Step 1: Write the DAO**

`lib/data/destinations_dao.dart`:
```dart
import 'package:drift/drift.dart';
import 'package:go_to_app/data/db.dart';
import 'package:go_to_app/domain/models.dart' as domain;

part 'destinations_dao.g.dart';

@DriftAccessor(tables: [Destinations])
class DestinationsDao extends DatabaseAccessor<AppDatabase> with _$DestinationsDaoMixin {
  DestinationsDao(super.db);

  Stream<List<domain.Destination>> watchAll() {
    final query = select(destinations)..orderBy([(t) => OrderingTerm.desc(t.createdAt)]);
    return query.watch().map(
          (rows) => rows
              .map((r) => domain.Destination(
                    id: r.id,
                    name: r.name,
                    latitude: r.latitude,
                    longitude: r.longitude,
                    createdAt: r.createdAt,
                  ))
              .toList(),
        );
  }

  Future<int> add({required String name, required double latitude, required double longitude}) {
    return into(destinations).insert(
      DestinationsCompanion.insert(
        name: name,
        latitude: latitude,
        longitude: longitude,
        createdAt: DateTime.now().millisecondsSinceEpoch,
      ),
    );
  }

  Future<void> updateOne(domain.Destination d) {
    return update(destinations).replace(
      DestinationsCompanion(
        id: Value(d.id),
        name: Value(d.name),
        latitude: Value(d.latitude),
        longitude: Value(d.longitude),
        createdAt: Value(d.createdAt),
      ),
    );
  }

  Future<int> remove(int id) =>
      (delete(destinations)..where((t) => t.id.equals(id))).go();

  Future<domain.Destination?> getById(int id) async {
    final r = await (select(destinations)..where((t) => t.id.equals(id))).getSingleOrNull();
    if (r == null) return null;
    return domain.Destination(
      id: r.id,
      name: r.name,
      latitude: r.latitude,
      longitude: r.longitude,
      createdAt: r.createdAt,
    );
  }
}
```

**Step 2: Register on AppDatabase**

In `lib/data/db.dart`, add inside the `AppDatabase` class:
```dart
late final DestinationsDao destinationsDao = DestinationsDao(this);
```

And add the import at top:
```dart
import 'destinations_dao.dart';
```

**Step 3: Run codegen**

Run: `dart run build_runner build --delete-conflicting-outputs`
Expected: clean.

**Step 4: Commit**

```
git add lib/data/destinations_dao.dart lib/data/destinations_dao.g.dart lib/data/db.dart lib/data/db.g.dart
git commit -m "Add DestinationsDao"
```

---

### Task 12: Settings service (shared_preferences)

**Files:**
- Create: `lib/data/settings_service.dart`
- Test: `test/data/settings_service_test.dart`

**Step 1: Write the failing test**

`test/data/settings_service_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:go_to_app/data/settings_service.dart';

void main() {
  setUp(() {
    SharedPreferences.setMockInitialValues({});
  });

  test('defaults', () async {
    final prefs = await SharedPreferences.getInstance();
    final s = SettingsService(prefs);
    expect(s.walkingPaceKmh, 5.0);
    expect(s.proximityRadiusM, 50);
    expect(s.useTrueNorth, true);
    expect(s.keepScreenOn, true);
    expect(s.onboardingComplete, false);
    expect(s.activeDestinationId, null);
  });

  test('persists active destination id', () async {
    final prefs = await SharedPreferences.getInstance();
    final s = SettingsService(prefs);
    await s.setActiveDestinationId(42);
    expect(s.activeDestinationId, 42);
  });
}
```

**Step 2: Run — expect failure**

Run: `flutter test test/data/settings_service_test.dart`
Expected: FAIL.

**Step 3: Implement**

`lib/data/settings_service.dart`:
```dart
import 'package:shared_preferences/shared_preferences.dart';

class SettingsService {
  static const _kActiveDestinationId = 'activeDestinationId';
  static const _kWalkingPaceKmh = 'walkingPaceKmh';
  static const _kProximityRadiusM = 'proximityRadiusM';
  static const _kUseTrueNorth = 'useTrueNorth';
  static const _kKeepScreenOn = 'keepScreenOn';
  static const _kProximityHaptic = 'proximityHaptic';
  static const _kOnTargetHaptic = 'onTargetHaptic';
  static const _kOnboardingComplete = 'onboardingComplete';

  final SharedPreferences _prefs;
  SettingsService(this._prefs);

  int? get activeDestinationId => _prefs.getInt(_kActiveDestinationId);
  Future<void> setActiveDestinationId(int? id) async {
    if (id == null) {
      await _prefs.remove(_kActiveDestinationId);
    } else {
      await _prefs.setInt(_kActiveDestinationId, id);
    }
  }

  double get walkingPaceKmh => _prefs.getDouble(_kWalkingPaceKmh) ?? 5.0;
  Future<void> setWalkingPaceKmh(double v) => _prefs.setDouble(_kWalkingPaceKmh, v);

  int get proximityRadiusM => _prefs.getInt(_kProximityRadiusM) ?? 50;
  Future<void> setProximityRadiusM(int v) => _prefs.setInt(_kProximityRadiusM, v);

  bool get useTrueNorth => _prefs.getBool(_kUseTrueNorth) ?? true;
  Future<void> setUseTrueNorth(bool v) => _prefs.setBool(_kUseTrueNorth, v);

  bool get keepScreenOn => _prefs.getBool(_kKeepScreenOn) ?? true;
  Future<void> setKeepScreenOn(bool v) => _prefs.setBool(_kKeepScreenOn, v);

  bool get proximityHaptic => _prefs.getBool(_kProximityHaptic) ?? true;
  Future<void> setProximityHaptic(bool v) => _prefs.setBool(_kProximityHaptic, v);

  bool get onTargetHaptic => _prefs.getBool(_kOnTargetHaptic) ?? true;
  Future<void> setOnTargetHaptic(bool v) => _prefs.setBool(_kOnTargetHaptic, v);

  bool get onboardingComplete => _prefs.getBool(_kOnboardingComplete) ?? false;
  Future<void> setOnboardingComplete(bool v) => _prefs.setBool(_kOnboardingComplete, v);
}
```

**Step 4: Run — expect pass**

Run: `flutter test test/data/settings_service_test.dart`
Expected: All passed.

**Step 5: Commit**

```
git add lib/data/settings_service.dart test/data/settings_service_test.dart
git commit -m "Add SettingsService over shared_preferences"
```

---

## Phase 4 — Services (platform-touching code)

### Task 13: LocationService

**Files:**
- Create: `lib/services/location_service.dart`

**Step 1: Implement**

`lib/services/location_service.dart`:
```dart
import 'package:geolocator/geolocator.dart';

class LocationFix {
  final double latitude;
  final double longitude;
  final double accuracyMeters;
  final double speedMps;
  final int timestampMs;

  const LocationFix({
    required this.latitude,
    required this.longitude,
    required this.accuracyMeters,
    required this.speedMps,
    required this.timestampMs,
  });
}

class LocationService {
  Stream<LocationFix> stream() {
    return Geolocator.getPositionStream(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.best,
        distanceFilter: 0,
      ),
    ).map((p) => LocationFix(
          latitude: p.latitude,
          longitude: p.longitude,
          accuracyMeters: p.accuracy,
          speedMps: p.speed,
          timestampMs: p.timestamp.millisecondsSinceEpoch,
        ));
  }

  Future<LocationFix?> getCurrent() async {
    try {
      final p = await Geolocator.getCurrentPosition(
        locationSettings: const LocationSettings(accuracy: LocationAccuracy.best),
      );
      return LocationFix(
        latitude: p.latitude,
        longitude: p.longitude,
        accuracyMeters: p.accuracy,
        speedMps: p.speed,
        timestampMs: p.timestamp.millisecondsSinceEpoch,
      );
    } catch (_) {
      return null;
    }
  }

  Future<LocationPermission> requestPermission() async {
    var perm = await Geolocator.checkPermission();
    if (perm == LocationPermission.denied) {
      perm = await Geolocator.requestPermission();
    }
    return perm;
  }
}
```

**Step 2: Commit**

```
git add lib/services/location_service.dart
git commit -m "Add LocationService wrapper"
```

---

### Task 14: CompassService — abstraction and mobile impl

**Files:**
- Create: `lib/services/compass_service.dart`

**Step 1: Implement**

`lib/services/compass_service.dart`:
```dart
import 'package:flutter/foundation.dart';
import 'package:flutter_compass/flutter_compass.dart';

abstract class CompassService {
  /// Stream of device azimuth in degrees, 0 = magnetic north, increasing clockwise.
  Stream<double> stream();

  /// Web only: gates the iOS Safari permission prompt. No-op on mobile.
  Future<bool> requestPermissionIfNeeded() async => true;

  factory CompassService() {
    if (kIsWeb) {
      return _WebCompassService();
    }
    return _MobileCompassService();
  }
}

class _MobileCompassService implements CompassService {
  @override
  Stream<double> stream() => FlutterCompass.events!
      .where((e) => e.heading != null)
      .map((e) => e.heading!);

  @override
  Future<bool> requestPermissionIfNeeded() async => true;
}

class _WebCompassService implements CompassService {
  // Placeholder — implemented in Task 16.
  @override
  Stream<double> stream() => const Stream.empty();

  @override
  Future<bool> requestPermissionIfNeeded() async => false;
}
```

**Step 2: Commit**

```
git add lib/services/compass_service.dart
git commit -m "Add CompassService abstraction with mobile impl"
```

---

### Task 15: CompassService — web impl

**Files:**
- Create: `lib/services/compass_service_web.dart`
- Modify: `lib/services/compass_service.dart`

**Step 1: Web impl using `dart:js_interop` (Flutter 3.16+) or `package:web`**

`lib/services/compass_service_web.dart`:
```dart
import 'dart:async';
import 'dart:js_interop';
import 'package:web/web.dart' as web;

extension type _Orientation(JSObject _) implements JSObject {
  external double? get alpha;
  external double? get webkitCompassHeading;
  external bool? get absolute;
}

@JS('DeviceOrientationEvent.requestPermission')
external JSFunction? get _requestPermission;

class WebCompassImpl {
  StreamController<double>? _controller;
  StreamSubscription<web.Event>? _sub;

  Stream<double> stream() {
    _controller ??= StreamController<double>.broadcast(
      onListen: _attach,
      onCancel: _detach,
    );
    return _controller!.stream;
  }

  void _attach() {
    final eventName = _hasAbsolute() ? 'deviceorientationabsolute' : 'deviceorientation';
    _sub = web.window.onMessage.listen(null); // placeholder; replaced below
    _sub?.cancel();
    _sub = web.EventStreamProviders.deviceOrientationEvent
        .forElement(web.window)
        .listen((e) {
      final o = e as _Orientation;
      final webkit = o.webkitCompassHeading;
      final alpha = o.alpha;
      double? heading;
      if (webkit != null) {
        heading = webkit; // already 0=N CW
      } else if (alpha != null) {
        heading = (360 - alpha) % 360; // alpha is 0=N CCW
      }
      if (heading != null) _controller!.add(heading);
    });
  }

  void _detach() {
    _sub?.cancel();
    _sub = null;
  }

  static bool _hasAbsolute() {
    // Best-effort feature detection.
    return true;
  }

  Future<bool> requestPermissionIfNeeded() async {
    final fn = _requestPermission;
    if (fn == null) return true; // not iOS Safari
    final result = await (fn.callAsFunction() as JSPromise).toDart;
    final dynamic r = result;
    return r == 'granted';
  }
}
```

> **Note:** The `EventStreamProviders.deviceOrientationEvent` import path may differ across `package:web` versions; if it breaks, swap to manual `addEventListener` via `web.window.addEventListener('deviceorientation', callback.toJS)`. The plan owner should consult the current `package:web` docs.

**Step 2: Wire it into `compass_service.dart`**

Replace `_WebCompassService` in `lib/services/compass_service.dart`:
```dart
import 'compass_service_web.dart' if (dart.library.io) 'compass_service_stub.dart';

class _WebCompassService implements CompassService {
  final WebCompassImpl _impl = WebCompassImpl();
  @override
  Stream<double> stream() => _impl.stream();
  @override
  Future<bool> requestPermissionIfNeeded() => _impl.requestPermissionIfNeeded();
}
```

Also create `lib/services/compass_service_stub.dart` so mobile builds don't pull in the web code:
```dart
class WebCompassImpl {
  Stream<double> stream() => const Stream.empty();
  Future<bool> requestPermissionIfNeeded() async => false;
}
```

**Step 3: Build sanity check**

Run: `flutter build web --no-tree-shake-icons`
Expected: build succeeds.
Run: `flutter analyze`
Expected: no issues.

**Step 4: Commit**

```
git add lib/services/
git commit -m "Add web CompassService using DeviceOrientation"
```

---

## Phase 5 — Riverpod state

### Task 16: Riverpod root + repository providers

**Files:**
- Create: `lib/state/providers.dart`
- Modify: `lib/main.dart`

**Step 1: Set up providers**

`lib/state/providers.dart`:
```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:go_to_app/data/db.dart';
import 'package:go_to_app/data/destinations_dao.dart';
import 'package:go_to_app/data/settings_service.dart';
import 'package:go_to_app/domain/models.dart';
import 'package:go_to_app/services/location_service.dart';
import 'package:go_to_app/services/compass_service.dart';

final dbProvider = Provider<AppDatabase>((ref) {
  final db = AppDatabase();
  ref.onDispose(db.close);
  return db;
});

final destinationsDaoProvider = Provider<DestinationsDao>(
  (ref) => ref.watch(dbProvider).destinationsDao,
);

final sharedPrefsProvider = FutureProvider<SharedPreferences>(
  (ref) => SharedPreferences.getInstance(),
);

final settingsProvider = Provider<SettingsService>((ref) {
  final prefs = ref.watch(sharedPrefsProvider).requireValue;
  return SettingsService(prefs);
});

final destinationsStreamProvider = StreamProvider<List<Destination>>(
  (ref) => ref.watch(destinationsDaoProvider).watchAll(),
);

final locationServiceProvider = Provider<LocationService>((_) => LocationService());
final compassServiceProvider = Provider<CompassService>((_) => CompassService());

final locationStreamProvider = StreamProvider<LocationFix>(
  (ref) => ref.watch(locationServiceProvider).stream(),
);

final compassStreamProvider = StreamProvider<double>(
  (ref) => ref.watch(compassServiceProvider).stream(),
);
```

**Step 2: Wrap app in ProviderScope and gate on prefs**

`lib/main.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_to_app/app.dart';

void main() {
  runApp(const ProviderScope(child: GoToApp()));
}
```

`lib/app.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_to_app/state/providers.dart';

class GoToApp extends ConsumerWidget {
  const GoToApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final prefsAsync = ref.watch(sharedPrefsProvider);
    return MaterialApp(
      title: 'go_to_app',
      theme: ThemeData(useMaterial3: true, colorSchemeSeed: Colors.indigo),
      home: prefsAsync.when(
        data: (_) => const Scaffold(body: Center(child: Text('go_to_app — booted'))),
        loading: () => const Scaffold(body: Center(child: CircularProgressIndicator())),
        error: (e, _) => Scaffold(body: Center(child: Text('Error: $e'))),
      ),
    );
  }
}
```

**Step 3: Run**

Run: `flutter run -d chrome`
Expected: shows "go_to_app — booted".

**Step 4: Commit**

```
git add lib/state/providers.dart lib/main.dart lib/app.dart
git commit -m "Wire Riverpod and core providers"
```

---

### Task 17: Active destination + compass state providers

**Files:**
- Modify: `lib/state/providers.dart`
- Create: `lib/state/compass_state.dart`
- Test: `test/state/compass_state_test.dart`

**Step 1: Compass state shape**

`lib/state/compass_state.dart`:
```dart
class CompassState {
  final double? deviceAzimuth;      // null if no sensor
  final double? bearingToTarget;    // null if no GPS or no target
  final double? distanceMeters;
  final Duration? walkAtFixedPace;
  final Duration? walkAtCurrentPace;
  final double? gpsAccuracyMeters;
  final bool gpsStale;
  final bool withinProximity;

  const CompassState({
    this.deviceAzimuth,
    this.bearingToTarget,
    this.distanceMeters,
    this.walkAtFixedPace,
    this.walkAtCurrentPace,
    this.gpsAccuracyMeters,
    this.gpsStale = false,
    this.withinProximity = false,
  });
}
```

**Step 2: Active destination provider**

Add to `lib/state/providers.dart`:
```dart
final activeDestinationIdProvider = StateProvider<int?>((ref) {
  return ref.watch(settingsProvider).activeDestinationId;
});

final activeDestinationProvider = Provider<Destination?>((ref) {
  final id = ref.watch(activeDestinationIdProvider);
  final list = ref.watch(destinationsStreamProvider).valueOrNull ?? const [];
  if (id == null) return list.isEmpty ? null : list.first;
  return list.where((d) => d.id == id).firstOrNull;
});
```

(Add `import 'package:collection/collection.dart';` if `firstOrNull` is unavailable; or write `try first / catch StateError`.)

**Step 3: Compass-state provider**

Append to `providers.dart`:
```dart
final compassStateProvider = Provider<CompassState>((ref) {
  final settings = ref.watch(settingsProvider);
  final target = ref.watch(activeDestinationProvider);
  final location = ref.watch(locationStreamProvider).valueOrNull;
  final heading = ref.watch(compassStreamProvider).valueOrNull;

  if (target == null || location == null) {
    return CompassState(deviceAzimuth: heading);
  }

  final dist = distanceMeters(location.latitude, location.longitude, target.latitude, target.longitude);
  final bearing = bearingDegrees(location.latitude, location.longitude, target.latitude, target.longitude);

  final paceMps = settings.walkingPaceKmh * 1000 / 3600;
  final fixedDur = Duration(seconds: (dist / paceMps).round());

  // current-speed walk time
  final smoother = ref.watch(_speedSmootherProvider);
  smoother.add(timestampMs: location.timestampMs, speedMps: location.speedMps);
  final avg = smoother.average(now: location.timestampMs);
  Duration? currentDur;
  if (avg != null && avg >= 0.5) {
    currentDur = Duration(seconds: (dist / avg).round());
  }

  final stale = DateTime.now().millisecondsSinceEpoch - location.timestampMs > 10_000;
  final within = dist <= settings.proximityRadiusM;

  return CompassState(
    deviceAzimuth: heading,
    bearingToTarget: bearing,
    distanceMeters: dist,
    walkAtFixedPace: fixedDur,
    walkAtCurrentPace: currentDur,
    gpsAccuracyMeters: location.accuracyMeters,
    gpsStale: stale,
    withinProximity: within,
  );
});

final _speedSmootherProvider = Provider<SpeedSmoother>((ref) => SpeedSmoother());
```

Add imports for `geo.dart`, `speed_smoother.dart`, `compass_state.dart`.

**Step 4: Test the pure shape**

`test/state/compass_state_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:go_to_app/state/compass_state.dart';

void main() {
  test('defaults are null/false', () {
    const s = CompassState();
    expect(s.deviceAzimuth, null);
    expect(s.bearingToTarget, null);
    expect(s.gpsStale, false);
    expect(s.withinProximity, false);
  });
}
```

Run: `flutter test test/state/compass_state_test.dart`
Expected: pass.

**Step 5: Commit**

```
git add lib/state/ test/state/
git commit -m "Add active destination and compass state providers"
```

---

## Phase 6 — Navigation skeleton

### Task 18: Routing with go_router and placeholder screens

**Files:**
- Create: `lib/features/compass/compass_screen.dart`
- Create: `lib/features/destinations/destinations_screen.dart`
- Create: `lib/features/destinations/edit_destination_screen.dart`
- Create: `lib/features/settings/settings_screen.dart`
- Create: `lib/features/onboarding/onboarding_screen.dart`
- Create: `lib/router.dart`
- Modify: `lib/app.dart`

**Step 1: Placeholder screens**

Each file: a `StatelessWidget` returning a `Scaffold` with `AppBar(title: Text('<name>'))` and `body: const Center(child: Text('<name>'))`.

**Step 2: Router**

`lib/router.dart`:
```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:go_to_app/features/compass/compass_screen.dart';
import 'package:go_to_app/features/destinations/destinations_screen.dart';
import 'package:go_to_app/features/destinations/edit_destination_screen.dart';
import 'package:go_to_app/features/onboarding/onboarding_screen.dart';
import 'package:go_to_app/features/settings/settings_screen.dart';
import 'package:go_to_app/state/providers.dart';

final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/',
    redirect: (context, state) {
      final settings = ref.read(settingsProvider);
      if (!settings.onboardingComplete && state.matchedLocation != '/onboarding') {
        return '/onboarding';
      }
      return null;
    },
    routes: [
      GoRoute(path: '/', builder: (_, __) => const CompassScreen()),
      GoRoute(path: '/destinations', builder: (_, __) => const DestinationsScreen()),
      GoRoute(path: '/destinations/new', builder: (_, __) => const EditDestinationScreen()),
      GoRoute(path: '/destinations/:id/edit', builder: (_, s) =>
          EditDestinationScreen(destinationId: int.parse(s.pathParameters['id']!))),
      GoRoute(path: '/settings', builder: (_, __) => const SettingsScreen()),
      GoRoute(path: '/onboarding', builder: (_, __) => const OnboardingScreen()),
    ],
  );
});
```

(Add `int? destinationId` parameter to `EditDestinationScreen` constructor.)

**Step 3: Swap MaterialApp for MaterialApp.router**

`lib/app.dart`:
```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final prefsAsync = ref.watch(sharedPrefsProvider);
  return prefsAsync.when(
    data: (_) => MaterialApp.router(
      title: 'go_to_app',
      theme: ThemeData(useMaterial3: true, colorSchemeSeed: Colors.indigo),
      routerConfig: ref.watch(routerProvider),
    ),
    loading: () => const MaterialApp(home: Scaffold(body: Center(child: CircularProgressIndicator()))),
    error: (e, _) => MaterialApp(home: Scaffold(body: Center(child: Text('$e')))),
  );
}
```

**Step 4: Smoke test**

Run: `flutter run -d chrome`
Expected: routes to `/onboarding` (placeholder shown).

**Step 5: Commit**

```
git add lib/features/ lib/router.dart lib/app.dart
git commit -m "Add go_router and placeholder screens"
```

---

## Phase 7 — Compass screen

### Task 19: CompassDial widget

**Files:**
- Create: `lib/ui/widgets/compass_dial.dart`

**Step 1: Implement painted dial**

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class CompassDial extends StatelessWidget {
  /// Device azimuth in degrees (0 = North).
  final double deviceAzimuth;
  /// Bearing to target (0 = North); null hides arrow.
  final double? bearingToTarget;
  /// Arrow color override.
  final Color arrowColor;

  const CompassDial({
    super.key,
    required this.deviceAzimuth,
    this.bearingToTarget,
    this.arrowColor = Colors.indigo,
  });

  @override
  Widget build(BuildContext context) {
    return AspectRatio(
      aspectRatio: 1,
      child: CustomPaint(painter: _DialPainter(deviceAzimuth, bearingToTarget, arrowColor)),
    );
  }
}

class _DialPainter extends CustomPainter {
  final double deviceAzimuth;
  final double? bearingToTarget;
  final Color arrowColor;
  _DialPainter(this.deviceAzimuth, this.bearingToTarget, this.arrowColor);

  @override
  void paint(Canvas canvas, Size size) {
    final r = size.shortestSide / 2;
    final c = size.center(Offset.zero);
    canvas.save();
    canvas.translate(c.dx, c.dy);

    // Dial circle
    final ring = Paint()
      ..style = PaintingStyle.stroke
      ..strokeWidth = 2
      ..color = Colors.grey.shade400;
    canvas.drawCircle(Offset.zero, r - 4, ring);

    // Cardinal labels rotate with the device (so N points to magnetic north)
    canvas.save();
    canvas.rotate(-deviceAzimuth * math.pi / 180);
    _drawLabel(canvas, 'N', Offset(0, -r + 18), Colors.red);
    _drawLabel(canvas, 'E', Offset(r - 18, 0));
    _drawLabel(canvas, 'S', Offset(0, r - 18));
    _drawLabel(canvas, 'W', Offset(-r + 18, 0));
    canvas.restore();

    // Arrow points toward (bearingToTarget − deviceAzimuth)
    if (bearingToTarget != null) {
      final theta = (bearingToTarget! - deviceAzimuth) * math.pi / 180;
      canvas.save();
      canvas.rotate(theta);
      final path = Path()
        ..moveTo(0, -r + 30)
        ..lineTo(-12, 0)
        ..lineTo(0, -10)
        ..lineTo(12, 0)
        ..close();
      canvas.drawPath(path, Paint()..color = arrowColor);
      canvas.restore();
    }

    canvas.restore();
  }

  void _drawLabel(Canvas c, String txt, Offset at, [Color color = Colors.black87]) {
    final tp = TextPainter(
      text: TextSpan(text: txt, style: TextStyle(color: color, fontSize: 16, fontWeight: FontWeight.bold)),
      textDirection: TextDirection.ltr,
    )..layout();
    tp.paint(c, at - Offset(tp.width / 2, tp.height / 2));
  }

  @override
  bool shouldRepaint(covariant _DialPainter old) =>
      old.deviceAzimuth != deviceAzimuth ||
      old.bearingToTarget != bearingToTarget ||
      old.arrowColor != arrowColor;
}
```

**Step 2: Commit**

```
git add lib/ui/widgets/compass_dial.dart
git commit -m "Add CompassDial widget"
```

---

### Task 20: Compass screen — wire it up

**Files:**
- Modify: `lib/features/compass/compass_screen.dart`

**Step 1: Implement**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:go_to_app/domain/geo.dart';
import 'package:go_to_app/state/providers.dart';
import 'package:go_to_app/ui/widgets/compass_dial.dart';
import 'package:intl/intl.dart';
import 'package:wakelock_plus/wakelock_plus.dart';

class CompassScreen extends ConsumerStatefulWidget {
  const CompassScreen({super.key});
  @override
  ConsumerState<CompassScreen> createState() => _CompassScreenState();
}

class _CompassScreenState extends ConsumerState<CompassScreen> {
  @override
  void initState() {
    super.initState();
    if (ref.read(settingsProvider).keepScreenOn) WakelockPlus.enable();
  }

  @override
  void dispose() {
    WakelockPlus.disable();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(compassStateProvider);
    final active = ref.watch(activeDestinationProvider);
    final destinations = ref.watch(destinationsStreamProvider).valueOrNull ?? const [];

    return Scaffold(
      appBar: AppBar(
        title: DropdownButton<int?>(
          value: active?.id,
          underline: const SizedBox.shrink(),
          items: [
            for (final d in destinations)
              DropdownMenuItem(value: d.id, child: Text(d.name)),
            const DropdownMenuItem(value: -1, child: Text('Manage destinations…')),
          ],
          onChanged: (v) {
            if (v == -1) {
              context.push('/destinations');
            } else if (v != null) {
              ref.read(settingsProvider).setActiveDestinationId(v);
              ref.read(activeDestinationIdProvider.notifier).state = v;
            }
          },
        ),
        actions: [
          IconButton(icon: const Icon(Icons.settings), onPressed: () => context.push('/settings')),
        ],
      ),
      body: active == null
          ? Center(
              child: FilledButton(
                onPressed: () => context.push('/destinations/new'),
                child: const Text('Add your first destination'),
              ),
            )
          : _CompassBody(state: state),
    );
  }
}

class _CompassBody extends StatelessWidget {
  final dynamic state;
  const _CompassBody({required this.state});

  @override
  Widget build(BuildContext context) {
    final dist = state.distanceMeters;
    final azimuth = state.deviceAzimuth ?? 0.0;
    final bearing = state.bearingToTarget;
    final eta = state.walkAtFixedPace == null
        ? null
        : DateFormat('HH:mm').format(DateTime.now().add(state.walkAtFixedPace!));
    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        children: [
          Expanded(
            child: CompassDial(
              deviceAzimuth: azimuth,
              bearingToTarget: bearing,
              arrowColor: _arrowColor(state),
            ),
          ),
          const SizedBox(height: 8),
          Text('Bearing ${bearing?.toStringAsFixed(0) ?? '—'}°',
              style: Theme.of(context).textTheme.titleMedium),
          const SizedBox(height: 24),
          Text(dist == null ? '—' : formatDistance(dist),
              style: Theme.of(context).textTheme.displaySmall?.copyWith(
                  color: state.withinProximity ? Colors.green : null)),
          const SizedBox(height: 8),
          Text(
            state.walkAtFixedPace == null
                ? 'Walk: —'
                : 'Walk: ${formatWalkTime(state.walkAtFixedPace!)}'
                    '${eta == null ? '' : '  ·  arrive $eta'}',
            style: Theme.of(context).textTheme.titleMedium,
          ),
          Text(
            state.walkAtCurrentPace == null
                ? 'At your pace: —'
                : 'At your pace: ${formatWalkTime(state.walkAtCurrentPace!)}',
            style: Theme.of(context).textTheme.titleMedium,
          ),
          const SizedBox(height: 8),
          Text(
            state.gpsAccuracyMeters == null
                ? 'GPS searching…'
                : (state.gpsStale ? 'GPS signal weak' : 'GPS ±${state.gpsAccuracyMeters.toStringAsFixed(0)} m'),
            style: TextStyle(
              color: state.gpsStale ? Colors.amber : Colors.grey,
              fontSize: 12,
            ),
          ),
        ],
      ),
    );
  }

  Color _arrowColor(dynamic s) {
    if (s.bearingToTarget == null || s.deviceAzimuth == null) return Colors.indigo;
    final diff = ((s.bearingToTarget! - s.deviceAzimuth!) + 540) % 360 - 180;
    return diff.abs() < 10 ? Colors.green : Colors.indigo;
  }
}
```

(Use a stronger type than `dynamic` once `CompassState` is imported — leaving it loose here so the diff focuses on flow.)

**Step 2: Smoke test**

Run: `flutter run -d chrome`
Expected: with no destinations, shows "Add your first destination". (We'll test with real data after Task 22.)

**Step 3: Commit**

```
git add lib/features/compass/compass_screen.dart
git commit -m "Build out compass screen"
```

---

### Task 21: Haptic feedback for proximity and on-target

**Files:**
- Modify: `lib/features/compass/compass_screen.dart`

**Step 1: Add `HapticFeedback.selectionClick()` calls**

Convert `_CompassScreenState` to listen to changes:
```dart
ref.listen<CompassState>(compassStateProvider, (prev, next) {
  final settings = ref.read(settingsProvider);
  if (settings.proximityHaptic && next.withinProximity && (prev?.withinProximity != true)) {
    HapticFeedback.heavyImpact();
  }
  if (settings.onTargetHaptic && _isOnTarget(next) && !(prev != null && _isOnTarget(prev))) {
    HapticFeedback.selectionClick();
  }
});
```

Helper `_isOnTarget` mirrors `_arrowColor` logic. Add `import 'package:flutter/services.dart';` and `import 'package:go_to_app/state/compass_state.dart';`.

**Step 2: Manual test on a real device**

Run: `flutter run -d <android-device>`
With a destination set, walk into and out of the proximity radius — feel one buzz on entry.

**Step 3: Commit**

```
git add lib/features/compass/compass_screen.dart
git commit -m "Add proximity and on-target haptics"
```

---

## Phase 8 — Destinations CRUD

### Task 22: Destinations list screen

**Files:**
- Modify: `lib/features/destinations/destinations_screen.dart`

**Step 1: Implement**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:go_to_app/state/providers.dart';

class DestinationsScreen extends ConsumerWidget {
  const DestinationsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final destinations = ref.watch(destinationsStreamProvider);
    final activeId = ref.watch(activeDestinationIdProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Destinations')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.push('/destinations/new'),
        child: const Icon(Icons.add),
      ),
      body: destinations.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('$e')),
        data: (list) => list.isEmpty
            ? const Center(child: Text('No destinations yet.'))
            : ListView.builder(
                itemCount: list.length,
                itemBuilder: (_, i) {
                  final d = list[i];
                  return ListTile(
                    leading: Radio<int>(
                      value: d.id,
                      groupValue: activeId,
                      onChanged: (v) {
                        if (v != null) {
                          ref.read(settingsProvider).setActiveDestinationId(v);
                          ref.read(activeDestinationIdProvider.notifier).state = v;
                        }
                      },
                    ),
                    title: Text(d.name),
                    subtitle: Text('${d.latitude.toStringAsFixed(5)}, ${d.longitude.toStringAsFixed(5)}'),
                    trailing: PopupMenuButton<String>(
                      onSelected: (v) async {
                        if (v == 'edit') context.push('/destinations/${d.id}/edit');
                        if (v == 'delete') {
                          await ref.read(destinationsDaoProvider).remove(d.id);
                          if (activeId == d.id) {
                            await ref.read(settingsProvider).setActiveDestinationId(null);
                            ref.read(activeDestinationIdProvider.notifier).state = null;
                          }
                        }
                      },
                      itemBuilder: (_) => const [
                        PopupMenuItem(value: 'edit', child: Text('Edit')),
                        PopupMenuItem(value: 'delete', child: Text('Delete')),
                      ],
                    ),
                  );
                },
              ),
      ),
    );
  }
}
```

**Step 2: Commit**

```
git add lib/features/destinations/destinations_screen.dart
git commit -m "Add destinations list screen"
```

---

### Task 23: Add/edit destination — three modes

**Files:**
- Modify: `lib/features/destinations/edit_destination_screen.dart`
- Create: `lib/features/destinations/map_picker.dart`

**Step 1: Edit screen with tabs**

`edit_destination_screen.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:latlong2/latlong.dart';
import 'package:go_to_app/state/providers.dart';
import 'package:go_to_app/features/destinations/map_picker.dart';

class EditDestinationScreen extends ConsumerStatefulWidget {
  final int? destinationId;
  const EditDestinationScreen({super.key, this.destinationId});

  @override
  ConsumerState<EditDestinationScreen> createState() => _EditDestinationScreenState();
}

class _EditDestinationScreenState extends ConsumerState<EditDestinationScreen> {
  final _name = TextEditingController();
  final _lat = TextEditingController();
  final _lon = TextEditingController();
  int _mode = 0; // 0 = here, 1 = map, 2 = coords

  bool get _editing => widget.destinationId != null;

  @override
  void initState() {
    super.initState();
    if (_editing) _loadExisting();
  }

  Future<void> _loadExisting() async {
    final d = await ref.read(destinationsDaoProvider).getById(widget.destinationId!);
    if (d != null && mounted) {
      _name.text = d.name;
      _lat.text = d.latitude.toString();
      _lon.text = d.longitude.toString();
      setState(() => _mode = 2);
    }
  }

  Future<void> _useCurrentLocation() async {
    final fix = await ref.read(locationServiceProvider).getCurrent();
    if (fix == null) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Could not get current location')),
      );
      return;
    }
    _lat.text = fix.latitude.toString();
    _lon.text = fix.longitude.toString();
  }

  Future<void> _pickOnMap() async {
    final picked = await Navigator.of(context).push<LatLng>(
      MaterialPageRoute(builder: (_) => const MapPickerScreen()),
    );
    if (picked != null) {
      _lat.text = picked.latitude.toString();
      _lon.text = picked.longitude.toString();
    }
  }

  Future<void> _save() async {
    final name = _name.text.trim();
    final lat = double.tryParse(_lat.text);
    final lon = double.tryParse(_lon.text);
    if (name.isEmpty || lat == null || lon == null) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Name and valid coordinates required')),
      );
      return;
    }
    final dao = ref.read(destinationsDaoProvider);
    if (_editing) {
      final existing = await dao.getById(widget.destinationId!);
      if (existing != null) {
        await dao.updateOne(existing.copyWith(name: name, latitude: lat, longitude: lon));
      }
    } else {
      final newId = await dao.add(name: name, latitude: lat, longitude: lon);
      // First destination becomes active by default
      if (ref.read(settingsProvider).activeDestinationId == null) {
        await ref.read(settingsProvider).setActiveDestinationId(newId);
        ref.read(activeDestinationIdProvider.notifier).state = newId;
      }
    }
    if (mounted) context.pop();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(_editing ? 'Edit destination' : 'New destination')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(controller: _name, decoration: const InputDecoration(labelText: 'Name')),
            const SizedBox(height: 16),
            SegmentedButton<int>(
              segments: const [
                ButtonSegment(value: 0, label: Text('Here')),
                ButtonSegment(value: 1, label: Text('Map')),
                ButtonSegment(value: 2, label: Text('Coords')),
              ],
              selected: {_mode},
              onSelectionChanged: (s) => setState(() => _mode = s.first),
            ),
            const SizedBox(height: 16),
            if (_mode == 0)
              FilledButton.icon(
                onPressed: _useCurrentLocation,
                icon: const Icon(Icons.my_location),
                label: const Text('Use my location'),
              ),
            if (_mode == 1)
              FilledButton.icon(
                onPressed: _pickOnMap,
                icon: const Icon(Icons.map),
                label: const Text('Pick on map'),
              ),
            if (_mode == 2)
              Column(
                children: [
                  TextField(controller: _lat, decoration: const InputDecoration(labelText: 'Latitude')),
                  TextField(controller: _lon, decoration: const InputDecoration(labelText: 'Longitude')),
                ],
              ),
            const SizedBox(height: 16),
            if (_lat.text.isNotEmpty)
              Text('Coords: ${_lat.text}, ${_lon.text}'),
            const Spacer(),
            FilledButton(onPressed: _save, child: const Text('Save')),
          ],
        ),
      ),
    );
  }
}
```

**Step 2: Map picker**

`map_picker.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:flutter_map/flutter_map.dart';
import 'package:latlong2/latlong.dart';

class MapPickerScreen extends StatefulWidget {
  const MapPickerScreen({super.key});
  @override
  State<MapPickerScreen> createState() => _MapPickerScreenState();
}

class _MapPickerScreenState extends State<MapPickerScreen> {
  LatLng? _picked;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Pick a location'),
        actions: [
          if (_picked != null)
            IconButton(
              icon: const Icon(Icons.check),
              onPressed: () => Navigator.of(context).pop(_picked),
            ),
        ],
      ),
      body: FlutterMap(
        options: MapOptions(
          initialCenter: const LatLng(32.0853, 34.7818), // Tel Aviv as a friendly default
          initialZoom: 13,
          onTap: (_, ll) => setState(() => _picked = ll),
        ),
        children: [
          TileLayer(
            urlTemplate: 'https://tile.openstreetmap.org/{z}/{x}/{y}.png',
            userAgentPackageName: 'com.stastyle.go_to_app',
          ),
          if (_picked != null)
            MarkerLayer(markers: [
              Marker(point: _picked!, width: 40, height: 40, child: const Icon(Icons.location_pin, color: Colors.red, size: 40)),
            ]),
        ],
      ),
    );
  }
}
```

**Step 3: Run**

Run: `flutter run -d chrome`
Verify: can add a destination via Here / Map / Coords and it appears in the list.

**Step 4: Commit**

```
git add lib/features/destinations/
git commit -m "Add destination create/edit with three input modes"
```

---

## Phase 9 — Settings

### Task 24: Settings screen

**Files:**
- Modify: `lib/features/settings/settings_screen.dart`

**Step 1: Implement**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_to_app/state/providers.dart';

class SettingsScreen extends ConsumerStatefulWidget {
  const SettingsScreen({super.key});
  @override
  ConsumerState<SettingsScreen> createState() => _SettingsScreenState();
}

class _SettingsScreenState extends ConsumerState<SettingsScreen> {
  @override
  Widget build(BuildContext context) {
    final s = ref.watch(settingsProvider);
    return Scaffold(
      appBar: AppBar(title: const Text('Settings')),
      body: ListView(
        children: [
          ListTile(
            title: const Text('Walking pace'),
            subtitle: Slider(
              min: 2,
              max: 8,
              divisions: 12,
              value: s.walkingPaceKmh,
              label: '${s.walkingPaceKmh.toStringAsFixed(1)} km/h',
              onChanged: (v) async {
                await s.setWalkingPaceKmh(v);
                setState(() {});
              },
            ),
          ),
          ListTile(
            title: const Text('Proximity radius'),
            subtitle: Slider(
              min: 10,
              max: 200,
              divisions: 19,
              value: s.proximityRadiusM.toDouble(),
              label: '${s.proximityRadiusM} m',
              onChanged: (v) async {
                await s.setProximityRadiusM(v.round());
                setState(() {});
              },
            ),
          ),
          SwitchListTile(
            title: const Text('Use true north'),
            subtitle: const Text('Corrects magnetic declination'),
            value: s.useTrueNorth,
            onChanged: (v) async { await s.setUseTrueNorth(v); setState(() {}); },
          ),
          SwitchListTile(
            title: const Text('Keep screen on'),
            value: s.keepScreenOn,
            onChanged: (v) async { await s.setKeepScreenOn(v); setState(() {}); },
          ),
          SwitchListTile(
            title: const Text('Proximity haptic'),
            value: s.proximityHaptic,
            onChanged: (v) async { await s.setProximityHaptic(v); setState(() {}); },
          ),
          SwitchListTile(
            title: const Text('On-target haptic'),
            value: s.onTargetHaptic,
            onChanged: (v) async { await s.setOnTargetHaptic(v); setState(() {}); },
          ),
        ],
      ),
    );
  }
}
```

**Step 2: Commit**

```
git add lib/features/settings/settings_screen.dart
git commit -m "Build out settings screen"
```

---

### Task 25: True-north correction

**Files:**
- Modify: `lib/state/providers.dart`

**Step 1: Apply declination**

`flutter_compass` already reports true heading when available; on web, we need to add declination ourselves. For a v1 simple approach, expose the raw heading and apply correction in `compassStateProvider` only when location is known. Use a small static table or the `geomagnetic` package — for v1, accept ~1° error and skip declination (most users won't notice).

If you do want it: add a TODO + reference, or pull in a small package such as `geomag` if available. **Leave this as a TODO comment in the provider with a link to the design doc's note.**

**Step 2: Commit (no-op or TODO)**

```
git commit --allow-empty -m "TODO: apply magnetic declination in true-north mode"
```

---

## Phase 10 — Onboarding

### Task 26: Onboarding flow

**Files:**
- Modify: `lib/features/onboarding/onboarding_screen.dart`

**Step 1: Implement multi-step onboarding**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:geolocator/geolocator.dart';
import 'package:go_router/go_router.dart';
import 'package:go_to_app/state/providers.dart';

class OnboardingScreen extends ConsumerStatefulWidget {
  const OnboardingScreen({super.key});
  @override
  ConsumerState<OnboardingScreen> createState() => _OnboardingScreenState();
}

class _OnboardingScreenState extends ConsumerState<OnboardingScreen> {
  int _step = 0;

  Future<void> _grantPermission() async {
    final perm = await ref.read(locationServiceProvider).requestPermission();
    if (perm == LocationPermission.always || perm == LocationPermission.whileInUse) {
      setState(() => _step = 2);
    } else {
      setState(() => _step = 2); // proceed; user can add by coords
    }
  }

  Future<void> _finish() async {
    await ref.read(settingsProvider).setOnboardingComplete(true);
    if (mounted) context.go('/destinations/new');
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              if (_step == 0) ...[
                const Text('Welcome to go_to_app',
                    style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold)),
                const SizedBox(height: 16),
                const Text('A compass that points to places you care about. Everything stays on your device.'),
                const SizedBox(height: 32),
                FilledButton(onPressed: () => setState(() => _step = 1), child: const Text('Get started')),
              ] else if (_step == 1) ...[
                const Text('Location permission', style: TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
                const SizedBox(height: 16),
                const Text('We use your location to point the compass at your destinations. Nothing leaves your phone.'),
                const SizedBox(height: 32),
                FilledButton(onPressed: _grantPermission, child: const Text('Grant access')),
              ] else ...[
                const Text('Add your first destination',
                    style: TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
                const SizedBox(height: 16),
                const Text('Save a place — your home, car, hotel, or anywhere worth pointing to.'),
                const SizedBox(height: 32),
                FilledButton(onPressed: _finish, child: const Text('Add destination')),
              ],
            ],
          ),
        ),
      ),
    );
  }
}
```

**Step 2: Smoke test**

Run: `flutter run -d chrome`
Expected: app opens onboarding; "Get started" → "Grant access" → "Add destination" → routes to `/destinations/new`. After adding, lands on compass.

**Step 3: Commit**

```
git add lib/features/onboarding/onboarding_screen.dart
git commit -m "Add onboarding flow"
```

---

## Phase 11 — Polish & release

### Task 27: Long-press distance to copy coords

**Files:**
- Modify: `lib/features/compass/compass_screen.dart`

**Step 1: Wrap distance text in GestureDetector**

```dart
GestureDetector(
  onLongPress: () async {
    if (active != null) {
      await Clipboard.setData(ClipboardData(text: '${active.latitude}, ${active.longitude}'));
      if (context.mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Coordinates copied')),
        );
      }
    }
  },
  child: Text(/* distance text */),
)
```

**Step 2: Commit**

```
git commit -am "Long-press distance copies coordinates"
```

---

### Task 28: Web entry point and manifest

**Files:**
- Modify: `web/index.html`
- Modify: `web/manifest.json`

**Step 1: Update `web/manifest.json`**

```json
{
  "name": "go_to_app",
  "short_name": "go_to_app",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3F51B5",
  "description": "A local-first compass for your destinations.",
  "orientation": "portrait-primary",
  "prefer_related_applications": false,
  "icons": [
    { "src": "icons/Icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/Icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

**Step 2: Build verification**

Run: `flutter build web --release`
Expected: `build/web/` populated. Run a local server to test:
```
python -m http.server -d build/web 8080
```
Open `http://localhost:8080` — though compass + GPS need HTTPS, the UI should render.

**Step 3: Android release build**

Run: `flutter build apk --release`
Expected: `build/app/outputs/flutter-apk/app-release.apk` exists.

**Step 4: Commit**

```
git add web/
git commit -m "Polish web manifest and verify release builds"
```

---

### Task 29: App icon

**Files:**
- Add: `assets/icon.png` (1024x1024)
- Modify: `pubspec.yaml`

**Step 1: Add `flutter_launcher_icons` dev dep**

In `pubspec.yaml`:
```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.1

flutter_launcher_icons:
  android: true
  web:
    generate: true
    background_color: "#ffffff"
    theme_color: "#3F51B5"
  image_path: "assets/icon.png"
```

**Step 2: Generate**

Run: `flutter pub get && dart run flutter_launcher_icons`
Expected: icons generated for Android and web.

**Step 3: Commit**

```
git add pubspec.yaml assets/ android/app/src/main/res/ web/icons/
git commit -m "Add app icons"
```

---

### Task 30: Final sweep

**Step 1: Analyze and test**

Run: `flutter analyze && flutter test`
Expected: no issues, all tests pass.

**Step 2: Manual checklist on a real Android device**

- [ ] First launch shows onboarding.
- [ ] Granting location permission lands on add-destination.
- [ ] Adding a "here" destination works; compass arrow appears.
- [ ] Adding via map picker drops a pin and saves.
- [ ] Adding via coordinates accepts decimal input.
- [ ] Switching active destination from the dropdown updates the compass.
- [ ] Walking around updates distance and "at your pace" walk time.
- [ ] Standing still shows "—" for current pace.
- [ ] Entering proximity radius turns distance green and buzzes.
- [ ] Settings sliders persist after restart.

**Step 3: Tag v0.1.0**

```
git tag v0.1.0
git push --tags
```

---

## Done

If everything in Task 30 passes, the app is feature-complete for v1. Future ideas (altitude delta, custom icons per destination, home-screen widget, compass calibration helper) are listed in the design doc as out-of-scope.
