# Location Screen - Map Integration & Flow

## 📍 Overview

The **Location Screen** is a reusable, shared screen component that handles location selection via map interaction and search. It supports both Google Maps and Huawei Maps depending on the device type, providing seamless location-based battery purchasing experience.

---

## 🎯 How Location Screen is Called

### 1. Entry Point - Battery Purchase Flow

```dart
// From HomeScreen or any screen
Navigator.pushNamed(context, '/battery-purchase/location');
```

### 2. Route Registration (main.dart)

```dart
routes: {
  '/battery-purchase/location': (context) => const LocationScreen(
    nextRoute: '/battery-purchase/vehicle',
  ),
  
  '/tyre-patching/location': (context) => const LocationScreen(
    nextRoute: '/tyre-patching/vehicle',
  ),
  
  '/home/location': (context) => const LocationScreen(),
}
```

### 3. Alternative Usage - Jumpstart Feature

For jumpstart feature, there's a wrapper screen:

```dart
// jumpstart_location_screen.dart
class JumpstartLocationScreen extends StatelessWidget {
  const JumpstartLocationScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return const LocationScreen(
      nextRoute: '/jumpstart/vehicle',
    );
  }
}
```

---

## 🏗️ Architecture & Components

### LocationScreen Widget Structure

**File:** [lib/shared/screens/location_screen.dart](lib/shared/screens/location_screen.dart)

```
LocationScreen (StatefulWidget)
├── Constructor Parameters:
│   └── nextRoute: String? (optional)
│
└── _LocationScreenState
    ├── Map Controllers
    │   ├── GoogleMapController? _mapController
    │   └── huawei.HuaweiMapController? _huaweiMapController
    │
    ├── Location Data
    │   ├── LatLng? _currentPosition
    │   ├── String _currentAddress
    │   └── huawei.LatLng? _huaweiCenterPosition
    │
    ├── UI State
    │   ├── bool _isCheckingPermission
    │   ├── bool _hasPermission
    │   ├── bool _isDraggingMap
    │   └── List<PlaceSuggestion> _suggestions
    │
    └── Services
        ├── MapService? _mapsService
        └── LocationCubit (via context.read)
```

---

## 🗺️ Map Service Architecture

### Two-Layer Map Service System

#### Layer 1: Abstract Interface

**File:** [lib/core/services/map_service.dart](lib/core/services/map_service.dart)

```dart
abstract class MapService {
  /// Get current location
  Future<Position?> getCurrentLocation();

  /// Get address from coordinates
  Future<String> getAddressFromCoordinates(double lat, double lng);

  /// Search for places
  Future<List<PlaceSuggestion>> searchPlaces(
    String query, {
    double? latitude,
    double? longitude,
  });

  /// Get place details
  Future<PlaceDetails?> getPlaceDetails(String placeId);

  /// Get distance between two coordinates
  double getDistanceBetween(
    double startLat,
    double startLng,
    double endLat,
    double endLng,
  );
}
```

#### Layer 2: Factory Pattern

**File:** [lib/core/services/map_service_factory.dart](lib/core/services/map_service_factory.dart)

```dart
class MapServiceFactory {
  static MapService? _instance;
  static bool? _isHuawei;

  /// Get the appropriate map service based on device type
  static Future<MapService> getInstance() async {
    if (_instance != null) {
      return _instance!;
    }

    _isHuawei = await DeviceUtils.isHuaweiDevice();

    if (_isHuawei!) {
      _instance = HuaweiMapsService();
    } else {
      _instance = GoogleMapsService();
    }

    return _instance!;
  }

  /// Check if current device is using Huawei Maps
  static Future<bool> isUsingHuaweiMaps() async {
    if (_isHuawei != null) {
      return _isHuawei!;
    }
    _isHuawei = await DeviceUtils.isHuaweiDevice();
    return _isHuawei!;
  }
}
```

#### Implementations

- **GoogleMapsService** [lib/core/services/google_maps_service.dart](lib/core/services/google_maps_service.dart)
- **HuaweiMapsService** [lib/core/services/huawei_maps_service.dart](lib/core/services/huawei_maps_service.dart)

**Decision Logic:** Device detection via `DeviceUtils.isHuaweiDevice()`

---

## 🔄 Complete User Flow

### Step-by-Step Process

```
┌─────────────────────────────────┐
│   1. Enter Location Screen      │
│   Route: /battery-purchase/...  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  2. Initialize Map Service      │
│   - Detect device type          │
│   - Load Google or Huawei Maps  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  3. Request Location Permission │
│   - Check/Request from OS       │
│   - Show loading state          │
└────────────┬────────────────────┘
             │
             ├─Yes──┐
             │      ▼
             │  Get current
             │  location via GPS
             │
             └──No──┐
                    ▼
              User must search
              manually or enable
```

### User Interactions

1. **GPS Detection (if permission granted)**
   - Current location detected automatically
   - Marker placed on map
   - Address fetched via reverse geocoding

2. **Drag Map to Change Location**
   - User drags map center
   - Marker bounces during drag animation
   - Address updates on camera idle

3. **Search Location**
   - User taps search field
   - Expands to full search UI
   - Type location query
   - See suggestions via autocomplete
   - Tap suggestion to select

4. **Confirm Location**
   - User taps "Use Address" button
   - Location saved to LocationCubit
   - Navigate to next route with location data

---

## 💾 State Management - LocationCubit

**File:** [lib/bloc/location/location_cubit.dart](lib/bloc/location/location_cubit.dart)

### LocationCubit Class

```dart
class LocationCubit extends Cubit<LocationState> {
  LocationCubit() : super(const LocationState());

  /// Save the selected location
  void saveLocation({
    required double latitude,
    required double longitude,
    required String address,
  }) {
    emit(state.copyWith(
      latitude: latitude,
      longitude: longitude,
      address: address,
      hasLocation: true,
    ));
  }

  /// Clear the saved location
  void clearLocation() {
    emit(const LocationState());
  }

  /// Get the current address
  String? get currentAddress => state.address;

  /// Check if location is saved
  bool get hasLocation => state.hasLocation;
}
```

### LocationState Class

**File:** [lib/bloc/location/location_state.dart](lib/bloc/location/location_state.dart)

```dart
class LocationState extends Equatable {
  final double? latitude;
  final double? longitude;
  final String? address;
  final bool hasLocation;

  const LocationState({
    this.latitude,
    this.longitude,
    this.address,
    this.hasLocation = false,
  });

  LocationState copyWith({
    double? latitude,
    double? longitude,
    String? address,
    bool? hasLocation,
  }) {
    return LocationState(
      latitude: latitude ?? this.latitude,
      longitude: longitude ?? this.longitude,
      address: address ?? this.address,
      hasLocation: hasLocation ?? this.hasLocation,
    );
  }

  @override
  List<Object?> get props => [latitude, longitude, address, hasLocation];
}
```

---

## 🎬 Lifecycle & Key Methods

### Initialization Phase

```dart
@override
void initState() {
  super.initState();
  WidgetsBinding.instance.addObserver(this);
  
  // Setup marker animation controller
  _markerController = AnimationController(
    vsync: this,
    duration: const Duration(milliseconds: 800),
    value: 0.0,
  );
  
  // Initialize map service (Google or Huawei)
  _initializeMapService();
}

Future<void> _initializeMapService() async {
  _mapsService = await MapServiceFactory.getInstance();
  _isHuaweiDevice = await MapServiceFactory.isUsingHuaweiMaps();
  _checkPermissionAndLocate();
}
```

### Location Permission Check

```dart
Future<void> _checkPermissionAndLocate() async {
  if (!mounted) return;
  setState(() => _isCheckingPermission = true);
  
  try {
    LocationPermission permission = await Geolocator.checkPermission();

    if (permission == LocationPermission.denied) {
      // Request permission from user
      permission = await Geolocator.requestPermission();
    }

    if (permission == LocationPermission.whileInUse ||
        permission == LocationPermission.always) {
      // Permission granted - get current position
      final position = await Geolocator.getCurrentPosition();
      
      setState(() {
        _currentPosition = LatLng(position.latitude, position.longitude);
        _hasPermission = true;
      });
      
      // Fetch address from coordinates
      _updateAddressFromPosition();
    } else {
      // Permission denied
      setState(() => _hasPermission = false);
    }
  } finally {
    if (mounted) {
      setState(() => _isCheckingPermission = false);
    }
  }
}
```

### Map Creation & Camera Handling

#### For Google Maps

```dart
void _onMapCreated(GoogleMapController controller) {
  _mapController = controller;
  
  // Move camera to current position if available
  if (_currentPosition != null) {
    _mapController?.animateCamera(
      CameraUpdate.newLatLngZoom(_currentPosition!, 15),
    );
  }
}

void _onCameraIdle() async {
  if (_ignoreNextCameraMove || _isCheckingPermission) {
    return;
  }

  if (_centerPosition == null || !_isDraggingMap || _mapsService == null) {
    return;
  }

  // Trigger marker bounce animation
  _markerController.animateTo(0.0, duration: Duration(milliseconds: 800));

  // After animation, fetch address for new position
  _idleTimer?.cancel();
  _idleTimer = Timer(const Duration(milliseconds: 600), () async {
    if (!mounted) return;
    
    setState(() => _isDraggingMap = false);
    
    final address = await _mapsService!.getAddressFromCoordinates(
      _centerPosition!.latitude,
      _centerPosition!.longitude,
    );
    
    setState(() {
      _currentPosition = _centerPosition;
      _currentAddress = address;
      _searchController.text = address;
    });
  });
}
```

#### For Huawei Maps

```dart
void _onHuaweiMapCreated(huawei.HuaweiMapController controller) {
  _huaweiMapController = controller;
  
  if (_huaweiCurrentPosition != null) {
    _huaweiMapController?.animateCamera(
      huawei.CameraUpdateFactory.newLatLngZoom(_huaweiCurrentPosition!, 15),
    );
  }
}

Future<void> _onHuaweiCameraIdle() async {
  // Similar logic to Google Maps camera idle handler
  // Updates position and address on map drag
}
```

### Search Functionality

```dart
void _searchPlaces(String query) async {
  if (query.isEmpty) {
    setState(() => _suggestions = []);
    return;
  }

  setState(() => _isLoading = true);

  try {
    final suggestions = await _mapsService?.searchPlaces(
      query,
      latitude: _currentPosition?.latitude,
      longitude: _currentPosition?.longitude,
    );

    if (mounted) {
      setState(() {
        _suggestions = suggestions ?? [];
        _isLoading = false;
      });
    }
  } catch (e) {
    setState(() => _isLoading = false);
  }
}

void _onSuggestionSelected(PlaceSuggestion suggestion) async {
  // Get detailed place information
  final details = await _mapsService?.getPlaceDetails(suggestion.placeId);

  if (details != null) {
    setState(() {
      _currentPosition = LatLng(details.latitude, details.longitude);
      _currentAddress = details.address;
      _searchController.text = details.address;
      _suggestions = [];
      _isSearchExpanded = false;
    });

    // Animate camera to selected location
    if (_mapController != null) {
      _mapController?.animateCamera(
        CameraUpdate.newLatLngZoom(
          LatLng(details.latitude, details.longitude),
          15,
        ),
      );
    }
  }
}
```

### Confirm & Navigation

```dart
// When user taps "Use Address" button
if (_currentAddress.isNotEmpty) {
  // Step 1: Save location to global state (LocationCubit)
  context.read<LocationCubit>().saveLocation(
    latitude: _currentPosition!.latitude,
    longitude: _currentPosition!.longitude,
    address: _currentAddress,
  );

  // Step 2: Navigate to next route with location data
  if (widget.nextRoute != null) {
    Navigator.pushNamed(
      context,
      widget.nextRoute!,
      arguments: {
        'latitude': _currentPosition?.latitude,
        'longitude': _currentPosition?.longitude,
        'address': _currentAddress,
      },
    );
  } else {
    // If no nextRoute, just pop
    Navigator.pop(context);
  }
}
```

---

## 📊 UI Layout Structure

```
┌─────────────────────────────────────┐
│       Location Screen               │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │   Google/Huawei Map         │   │
│  │   (Draggable, Searchable)   │   │
│  │                             │   │
│  │   Center Pin with Marker    │   │
│  │   (Animated bounce)         │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Vehicle Location     [⌕]    │   │
│  ├─────────────────────────────┤   │
│  │                             │   │
│  │ Search field or display of  │   │
│  │ selected location           │   │
│  │                             │   │
│  │ [Use Address] button        │   │
│  │                             │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

When Searching:
┌─────────────────────────────────────┐
│  Search Input         [Search...] [✕]│
├─────────────────────────────────────┤
│  ◎ Suggestion 1 - Location Name     │
│  ─────────────────────────────────────
│  ◎ Suggestion 2 - Location Name     │
│  ─────────────────────────────────────
│  ◎ Suggestion 3 - Location Name     │
└─────────────────────────────────────┘
```

---

## 🔗 Integration with Battery Purchase Flow

### Data Flow from Location Screen

```
LocationScreen
│
├─ User selects location (drag or search)
│
├─ Stores in LocationCubit
│   ├─ latitude
│   ├─ longitude
│   └─ address
│
├─ Navigates to next route with arguments
│   └─ /battery-purchase/vehicle
│
▼
VehicleScreen
│
├─ Reads location from LocationCubit
│
├─ User fills vehicle details
│
├─ Calls ProductsService.getProducts()
│   (passes latitude, longitude)
│
▼
ProductBatteryScreen
│
├─ Receives products via route arguments
│
├─ User selects battery
│
▼
OrderConfirmationScreen
│
├─ Reads location from LocationCubit
│
├─ Creates order with location data
│
▼
Payment Processing
```

### Code Example: Reading Location in Next Screen

```dart
// In VehicleScreen or any downstream screen
@override
Widget build(BuildContext context) {
  final locationState = context.read<LocationCubit>().state;
  
  final latitude = locationState.latitude;
  final longitude = locationState.longitude;
  final address = locationState.address;
  
  return Scaffold(
    // Use location data here
  );
}
```

---

## ✨ Key Features

### 1. Dual Map Support
- **Google Maps** (Standard devices)
- **Huawei Maps** (Huawei devices)
- Automatic detection via `DeviceUtils.isHuaweiDevice()`

### 2. Location Detection Methods
- **GPS:** Automatic detection if permission granted
- **Search:** Autocomplete place search
- **Manual:** Drag map to select location

### 3. Address Reverse Geocoding
- Converts coordinates to readable address
- Updates in real-time during map drag
- Displays in search field

### 4. Marker Animation
- Bouncy animation on map drag
- Haptic feedback during drag
- Smooth camera transitions

### 5. Flexible Navigation
- `nextRoute` parameter for dynamic routing
- Passes location data via route arguments
- Falls back to pop if no nextRoute specified

### 6. Reusable Across Features
- Battery Purchase
- Jumpstart Service
- Tyre Patching
- Any location-based feature

---

## 🛠️ Customization

### Using with Different Next Routes

```dart
// Battery Purchase
LocationScreen(nextRoute: '/battery-purchase/vehicle')

// Jumpstart
LocationScreen(nextRoute: '/jumpstart/vehicle')

// Tyre Patching
LocationScreen(nextRoute: '/tyre-patching/vehicle')

// Custom Usage
LocationScreen(nextRoute: '/custom/next-screen')

// No Navigation (just pop)
LocationScreen()
```

### Accessing Location Data Later

```dart
// In any downstream widget
final locationCubit = context.read<LocationCubit>();

// Get current state
final latitude = locationCubit.state.latitude;
final longitude = locationCubit.state.longitude;
final address = locationCubit.state.address;

// Check if location is saved
if (locationCubit.hasLocation) {
  // Use location data
}

// Get address
final currentAddr = locationCubit.currentAddress;

// Clear location when needed
locationCubit.clearLocation();
```

---

## 📁 Related Files Summary

| File | Purpose |
|------|---------|
| [lib/shared/screens/location_screen.dart](lib/shared/screens/location_screen.dart) | Main Location Screen (1733 lines) |
| [lib/bloc/location/location_cubit.dart](lib/bloc/location/location_cubit.dart) | Location state management |
| [lib/bloc/location/location_state.dart](lib/bloc/location/location_state.dart) | Location state model |
| [lib/core/services/map_service.dart](lib/core/services/map_service.dart) | Map service abstract interface |
| [lib/core/services/map_service_factory.dart](lib/core/services/map_service_factory.dart) | Map service factory (Google/Huawei) |
| [lib/core/services/google_maps_service.dart](lib/core/services/google_maps_service.dart) | Google Maps implementation |
| [lib/core/services/huawei_maps_service.dart](lib/core/services/huawei_maps_service.dart) | Huawei Maps implementation |
| [lib/shared/screens/vehicle_screen.dart](lib/shared/screens/vehicle_screen.dart) | Downstream screen |
| [lib/features/battery_purchase/screens/product_battery_screen.dart](lib/features/battery_purchase/screens/product_battery_screen.dart) | Product selection screen |
| [lib/features/jumpstart/screens/jumpstart_location_screen.dart](lib/features/jumpstart/screens/jumpstart_location_screen.dart) | Jumpstart location wrapper |

---

## 🎯 Summary

The Location Screen is a sophisticated, reusable component that:
- Provides seamless location selection via map interaction
- Supports multiple map providers (Google/Huawei)
- Manages location state globally via LocationCubit
- Integrates with multiple features via dynamic routing
- Handles all edge cases (permissions, no results, network errors)
- Offers excellent UX with animations and real-time address updates

**Entry Point:** `Navigator.pushNamed(context, '/battery-purchase/location')`

**Exit Point:** Location saved to `LocationCubit` → Navigate to next route with location data

This architecture ensures that location data is consistently available throughout the battery purchase and other location-based features in the app.
