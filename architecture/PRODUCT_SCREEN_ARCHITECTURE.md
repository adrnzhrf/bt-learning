# Product Screen - Architecture & Code Explanation

## 📋 Overview

The **Product Screen** (also called Product Battery Screen) is where customers select a compatible battery for their vehicle in the Bateriku battery purchase flow. It displays API-fetched products with detailed specifications, pricing, warranty information, and visual selection indicators. The screen uses **BLoC pattern** for state management and follows clean architecture principles.

---

## 🏗️ Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│  UI Layer                                           │
│  [ProductBatteryScreen] (451 lines)                │
│  - Renders product cards                            │
│  - Handles user selection                           │
│  - Navigates to order confirmation                 │
└────────────────┬────────────────────────────────────┘
                 │ listens to
┌────────────────▼────────────────────────────────────┐
│  Business Logic (BLoC Pattern)                      │
│  [BatteryPurchaseCubit] (341 lines)                │
│  - selectBatteryFromProducts()                     │
│  - selectBattery()                                 │
│  - setProductsFromRoute()                          │
│  - applyPromoCode()                                │
└────────────────┬────────────────────────────────────┘
                 │ delegates to
┌────────────────▼────────────────────────────────────┐
│  Service Layer                                      │
│  [BatteryPurchaseService] (API calls)              │
│  - getQuote()                                      │
│  - checkLocationService()                          │
│  - createOrder()                                   │
└────────────────┬────────────────────────────────────┘
                 │ calls
┌────────────────▼────────────────────────────────────┐
│  API Clients & Data Models                          │
│  [ProductsService] - Product data                  │
│  [ProductItem] - Product model                     │
│  [OrderService] - Order calculations               │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Key Files & Their Responsibilities

### 1. [product_battery_screen.dart](lib/features/battery_purchase/screens/product_battery_screen.dart) *(451 lines)*

**Purpose:** Main UI screen for battery product selection

**Contains:**
- `ProductBatteryScreen` - StatelessWidget that receives route arguments
- `_ProductBatteryScreenContent` - Main content with BLoC integration
- Product image loading with network/asset fallback
- Product card UI with selection states
- Floating action button for confirmation
- `_buildDisplayBatteries()` - Converts ProductItem objects to display format

**Key Features:**
- Displays vehicle plate number in AppBar title
- Shows count of compatible batteries
- Product cards with:
  - Battery image (100x100)
  - Name and brand
  - Formatted price in RM
  - Warranty information
  - Description (if available)
  - "Recommended" badge
  - Availability indicator
  - Selection highlighting (green border + background)
- FAB "Confirm & Proceed" button (visible only when product selected)

**State Dependencies:**
- Uses `BatteryPurchaseCubit` via `BlocBuilder`
- Reads `state.apiProducts` for product list
- Reads `state.selectedBattery` for selection state
- Reads `state.brandId` for brand information

---

### 2. [battery_purchase_cubit.dart](lib/bloc/battery_purchase/battery_purchase_cubit.dart) *(341 lines)*

**Purpose:** Business logic orchestration for battery purchase flow

**Key Methods:**

```dart
class BatteryPurchaseCubit extends Cubit<BatteryPurchaseState> {
  
  // Set products and brand ID from route arguments
  void setProductsFromRoute({
    required List<ProductItem> products,
    int? brandId,
  })
  
  // Select battery from API products and get quote
  Future<void> selectBatteryFromProducts({
    required String batteryId,
    required double latitude,
    required double longitude,
    String? promoCode,
  })
  
  // Alternative selection method with quote generation
  Future<void> selectBattery({
    required String batteryId,
    required double latitude,
    required double longitude,
    String? promoCode,
  })
  
  // Create battery purchase order
  Future<void> createOrder({...})
  
  // Apply promo code (recalculates quote)
  void applyPromoCode(String promoCode)
  
  // Reset entire purchase flow
  void reset()
  
  // Set customer/vehicle/location info
  void setCustomerInfo(CustomerInfo customer)
  void setVehicleInfo(VehicleInfo vehicle)
  void setLocation(LocationInfo location)
  void setTradeIn(bool hasTradeIn)
  
  // Clear all errors
  void clearErrors()
}
```

**State Management:**
- `BatteryPurchaseStatus.initial` → Screen just loaded
- `BatteryPurchaseStatus.loading` → Fetching quote or products
- `BatteryPurchaseStatus.success` → Operation succeeded
- `BatteryPurchaseStatus.failure` → Operation failed

**Responsibilities:**
- Manages selected battery state
- Handles quote fetching from `BatteryPurchaseService`
- Converts `ProductItem` to `BatteryProduct` for internal state
- Manages promo code validation
- Coordinates with multiple services

---

### 3. [battery_purchase_state.dart](lib/bloc/battery_purchase/battery_purchase_state.dart) *(227 lines)*

**Purpose:** Immutable state object for BLoC pattern

**State Properties:**

```dart
class BatteryPurchaseState extends Equatable {
  // Battery selection
  final BatteryPurchaseStatus batteriesStatus;
  final List<BatteryProduct>? availableBatteries;
  final String? batteriesError;
  final BatteryProduct? selectedBattery;
  final List<ProductItem>? apiProducts;        // From ProductsService
  final int? brandId;

  // Quote generation
  final BatteryPurchaseStatus quoteStatus;
  final PriceQuote? currentQuote;
  final String? quoteError;

  // Order management
  final BatteryPurchaseStatus orderStatus;
  final BatteryOrder? currentOrder;
  final String? orderError;

  // Location service check
  final BatteryPurchaseStatus locationServiceStatus;
  final LocationService? locationService;
  final String? locationServiceError;

  // Form data
  final CustomerInfo? customerInfo;
  final LocationInfo? location;
  final VehicleInfo? vehicleInfo;
  final String? promoCode;
  final bool hasTradeIn;
}
```

**Methods:**
- `copyWith()` - Creates new state with selected properties updated
- `==` and `hashCode` - Implemented via `Equatable` for equality comparison

---

### 4. [products_service.dart](lib/core/services/products_service.dart) *(217 lines)*

**Purpose:** API client for fetching compatible battery products

**Key Method:**

```dart
class ProductsService {
  Future<ProductsResult> getProducts({
    required String token,
    required String userName,
    required String userPhone,
    required String userEmail,
    required double latitude,
    required double longitude,
    required String vehiclePlateNumber,
  })
}
```

**API Endpoint:** `POST /api/partners/v2/orders/products`

**Base URL:** From `EnvConfig.batteryPurchaseApiUrl`

**Request Payload:**
```json
{
  "user": {
    "name": "John Doe",
    "phone": "+60123456789",
    "email": "john@example.com"
  },
  "order": {
    "latitude": 3.1390,
    "longitude": 101.6869,
    "vehicle_plate_number": "VBS9925"
  }
}
```

**Response:**
```json
{
  "brand_id": 1,
  "products": [
    {
      "id": "astra-ns70l",
      "name": "ASTRA NS70L",
      "brand": "Astra",
      "price": "RM 450.00",
      "price_cents": 45000,
      "warranty_period": 36,
      "warranty_mileage": 60000,
      "image_url": "https://cdn.bateriku.com/products/astra-ns70l.jpg",
      "is_recommended": true,
      "is_available": true,
      "description": "Premium long-lasting battery",
      "category": "Battery",
      "consumable": true
    }
  ]
}
```

**Returns:** `ProductsResult` object containing list of `ProductItem` objects and `brandId`

---

### 5. [ProductItem Model](lib/core/services/products_service.dart) *(140 lines)*

**Purpose:** Data model representing a single battery product

**Key Properties:**

```dart
class ProductItem {
  final String id;                      // Unique product ID
  final String name;                    // Product name (e.g., "ASTRA NS70L")
  final String? brand;                  // Brand name (e.g., "Astra")
  final String category;                // Product category (e.g., "Battery")
  final bool consumable;                // Whether it's consumable
  final String? imageUrl;               // Image URL (network or asset)
  final int sizeRanking;                // Size ranking (for sorting)
  final int warrantyPeriod;             // Warranty in months
  final int? warrantyMileage;           // Warranty in kilometers
  final String price;                   // Formatted price (e.g., "RM 450.00")
  final int priceCents;                 // Price in cents (45000)
  final String? description;            // Product description
  final Map<String, dynamic>? specifications;  // Tech specs (voltage, capacity, etc.)
  final bool isAvailable;               // Is product available?
  final bool isRecommended;             // Is product recommended?
}
```

**Helper Methods:**

```dart
// Getter
String get formattedPrice => price;

// Computed warranty string
String get warranty {
  if (warrantyMileage != null) {
    return '$warrantyPeriod Months / ${(warrantyMileage! / 1000).toInt()},000 KM';
  }
  return '$warrantyPeriod Months';
}

// Serialization
factory ProductItem.fromJson(Map<String, dynamic> json)
Map<String, dynamic> toJson()
```

**Example:**
```dart
ProductItem(
  id: 'astra-ns70l',
  name: 'ASTRA NS70L',
  brand: 'Astra',
  price: 'RM 450.00',
  priceCents: 45000,
  warrantyPeriod: 36,
  warrantyMileage: 60000,
  imageUrl: 'https://cdn.bateriku.com/products/astra-ns70l.jpg',
  isRecommended: true,
  isAvailable: true,
  description: 'Premium long-lasting battery',
)
```

---

### 6. [ProductsResult Class](lib/core/services/products_service.dart)

**Purpose:** Result wrapper for API response

```dart
class ProductsResult {
  final bool isSuccess;
  final List<ProductItem>? products;
  final int? brandId;
  final String? error;

  // Success constructor
  ProductsResult.success({
    required this.products,
    this.brandId,
  })

  // Failure constructor
  ProductsResult.failure(this.error)
}
```

**Usage:**
```dart
if (result.isSuccess && result.products != null) {
  // Handle success
  final products = result.products!;
  final brandId = result.brandId;
} else {
  // Handle error
  final errorMessage = result.error!;
}
```

---

### 7. [BatteryPurchaseService](lib/core/services/battery_purchase_service.dart)

**Purpose:** Higher-level service coordinating battery purchase operations

**Key Methods:**
```dart
class BatteryPurchaseService {
  Future<QuoteResult> getQuote({
    required String batteryId,
    required double latitude,
    required double longitude,
    String? promoCode,
  })
  
  Future<LocationServiceResult> checkLocationService({
    required double latitude,
    required double longitude,
  })
  
  Future<OrderResult> createOrder({
    required String batteryId,
    required CustomerInfo customer,
    required LocationInfo location,
    required VehicleInfo vehicle,
    String? promoCode,
    bool hasTradeIn = false,
  })
}
```

---

## 🔄 Data Flow Examples

### Example 1: Screen Load with Products from API

```
VehicleScreen
  ↓
User fills form (name, phone, email, plate, location)
  ↓
ProductsService.getProducts() API call
  ↓
API returns products list + brandId
  ↓
ProductBatteryScreen receives route arguments:
  {
    'products': [ProductItem, ProductItem, ...],
    'brandId': 1,
    'customerInfo': {...},
    'vehicleInfo': {...},
    'location': {...}
  }
  ↓
BatteryPurchaseCubit.setProductsFromRoute()
  ↓
Emits: state.copyWith(apiProducts: products, brandId: brandId)
  ↓
BlocBuilder rebuilds UI with product list
  ↓
_buildDisplayBatteries() converts ProductItem list to display format
  ↓
UI renders product cards with:
  - Battery image
  - Name, brand, price
  - Warranty info
  - Selection state
```

### Example 2: User Selects a Battery

```
User taps on "ASTRA NS70L" product card
  ↓
InkWell onTap triggers:
  cubit.selectBatteryFromProducts(
    batteryId: 'astra-ns70l',
    latitude: 3.1390,
    longitude: 101.6869
  )
  ↓
Cubit finds product in state.apiProducts
  ↓
Cubit converts ProductItem to BatteryProduct:
  BatteryProduct(
    id: 'astra-ns70l',
    name: 'ASTRA NS70L',
    brand: 'Astra',
    price: 450.00,
    warrantyMonths: '36',
    warrantyKilometers: '60000',
    imageUrl: 'https://...',
    isBestSeller: true,
    isAvailable: true,
  )
  ↓
Emits: state.copyWith(selectedBattery: batteryProduct)
  ↓
UI updates to show:
  - Green border around selected card
  - Light green background
  - FloatingActionButton becomes visible
  ↓
Cubit fetches quote from BatteryPurchaseService:
  getQuote(
    batteryId: 'astra-ns70l',
    latitude: 3.1390,
    longitude: 101.6869
  )
  ↓
Emits: BatteryPurchaseStatus.loading
  ↓
API returns quote with pricing
  ↓
Emits: BatteryPurchaseStatus.success with quote
```

### Example 3: User Confirms and Proceeds

```
User taps "Confirm & Proceed" FloatingActionButton
  ↓
Get selected product from state.apiProducts
  ↓
Build arguments object:
  {
    'productId': selectedProduct.id,
    'brandId': state.brandId,
    'selectedProduct': {
      'id': selectedProduct.id,
      'name': selectedProduct.name,
      'price_cents': selectedProduct.priceCents,
      'warranty_period': selectedProduct.warrantyPeriod,
      'warranty_mileage': selectedProduct.warrantyMileage,
      'brand': selectedProduct.brand,
      'description': selectedProduct.description,
    },
    'products': state.apiProducts,
    'customerInfo': customerInfo,
    'vehicleInfo': vehicleInfo,
    'locationInfo': locationInfo,
  }
  ↓
Navigate to '/order/confirmation' with arguments
  ↓
Order Confirmation Screen loads with selected product
```

### Example 4: Product Image Loading

```
Product card renders battery image
  ↓
_buildProductImage(imageUrl) called
  ↓
Check if URL is network or local:
  - Starts with 'http://' or 'https://' → Image.network()
  - Otherwise → Image.asset()
  ↓
If network image:
  - Shows loading spinner while fetching
  - Displays image on success
  - Falls back to battery icon on error
  ↓
If local asset:
  - Loads from 'assets/images/products/...'
  - Falls back to battery icon on error
  ↓
Image displayed in 100x100 container with rounded corners
```

---

## 🎨 UI Components

### Product Card Structure

```
┌─────────────────────────────────────────────┐
│  [Recommended Badge]                        │  (top-right)
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────┐                               │
│  │ [Image]  │  ASTRA NS70L                  │
│  │  100x100 │  Astra                        │
│  │          │  RM 450.00                    │
│  └──────────┘                               │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ Warranty                            │   │
│  │ 36 Months / 60,000 KM               │   │
│  │ Premium long-lasting battery        │   │
│  └─────────────────────────────────────┘   │
│                                             │
└─────────────────────────────────────────────┘

Selection states:
- Unselected: White bg, grey border
- Selected: Light green (#E8F5E9) bg, green (#9ACD32) border
- Unavailable: Opacity 0.6, disabled
```

### Screen Layout

```
┌──────────────────────────────┐
│ AppBar: Vehicle Plate (VBS9925)
├──────────────────────────────┤
│ "Select your Battery"        │
│ "Here are X compatible..."   │
├──────────────────────────────┤
│ Product Card 1               │
├──────────────────────────────┤
│ Product Card 2               │
├──────────────────────────────┤
│ Product Card 3               │
├──────────────────────────────┤
│ [Spacing for FAB]            │
└──────────────────────────────┘
         ↓↓↓ Floating Action Button ↓↓↓
     [Confirm & Proceed →]
```

---

## 🎯 Navigation Flow

```
Home Screen
    ↓
Battery Purchase Section
    ↓
Vehicle Screen (customer fills form)
    ↓
[ProductsService.getProducts() API call]
    ↓
Product Battery Screen ← Route: '/battery-purchase/product'
    ↓
User selects battery
    ↓
[BatteryPurchaseCubit.selectBatteryFromProducts()]
    ↓
Confirm & Proceed button
    ↓
Order Confirmation Screen ← Route: '/order/confirmation'
    ↓
Apply promo, review total, pay
    ↓
Payment Gateway
    ↓
Order Status Screen
```

---

## 🛠️ Key Features

| Feature | Implementation | Notes |
|---------|----------------|-------|
| **Product Fetch** | `ProductsService.getProducts()` | Called before reaching product screen |
| **Product Display** | `_buildDisplayBatteries()` converts ProductItem to Map | Maps to display format |
| **Image Loading** | Network or local asset with fallbacks | Gracefully handles errors |
| **Selection State** | `state.selectedBattery` tracks selection | Updates UI with visual feedback |
| **Recommended Badge** | `ProductItem.isRecommended` flag | Shows "Recommended" label |
| **Availability** | `ProductItem.isAvailable` flag | Disables unavailable products |
| **Warranty Display** | Combines period + mileage | Formatted string via getter |
| **Price Display** | `ProductItem.price` (formatted) | RM format with 2 decimals |
| **Quote Generation** | `selectBatteryFromProducts()` calls `getQuote()` | Optional, for delivery fee |
| **Navigation** | Route arguments with full product data | Passes to order confirmation |

---

## 💾 Data Models Summary

### ProductItem (from API)
```dart
{
  id: String,
  name: String,
  brand: String?,
  price: String,           // "RM 450.00"
  priceCents: int,         // 45000
  warrantyPeriod: int,     // 36
  warrantyMileage: int?,   // 60000
  imageUrl: String?,
  isRecommended: bool,
  isAvailable: bool,
  description: String?,
  specifications: Map?
}
```

### BatteryProduct (internal state)
```dart
{
  id: String,
  name: String,
  brand: String,
  price: double,           // 450.00
  warrantyMonths: String,  // "36"
  warrantyKilometers: String,  // "60000"
  imageUrl: String,
  isBestSeller: bool,
  isAvailable: bool
}
```

### Route Arguments
```dart
{
  'products': List<ProductItem>,
  'brandId': int,
  'customerInfo': Map,
  'vehicleInfo': Map,
  'location': Map
}
```

---

## 🔐 Error Handling

### Image Loading Errors
```dart
// Network image fails
errorBuilder: (context, error, stackTrace) {
  return Icon(Icons.battery_charging_full, size: 60);
}

// Asset image fails
errorBuilder: (context, error, stackTrace) {
  return Icon(Icons.battery_charging_full, size: 60);
}
```

### API Errors
```dart
if (!result.isSuccess) {
  // result.error contains error message
  // UI can display error or retry
}
```

### Availability Handling
```dart
if (isAvailable) {
  // Enable product selection
  onTap: () { selectBatteryFromProducts(...) }
} else {
  // Disable product, show reduced opacity
  onTap: null,
  Opacity(opacity: 0.6)
}
```

---

## 🧪 Testing Examples

### Unit Test - Product Selection

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';

void main() {
  group('BatteryPurchaseCubit Product Selection', () {
    test('selectBatteryFromProducts emits success state', () async {
      // Arrange
      final mockService = MockBatteryPurchaseService();
      final products = [
        ProductItem(
          id: 'astra-ns70l',
          name: 'ASTRA NS70L',
          brand: 'Astra',
          price: 'RM 450.00',
          priceCents: 45000,
          warrantyPeriod: 36,
          warrantyMileage: 60000,
          category: 'Battery',
        ),
      ];

      final cubit = BatteryPurchaseCubit(batteryService: mockService);
      cubit.emit(cubit.state.copyWith(apiProducts: products));

      // Act
      await cubit.selectBatteryFromProducts(
        batteryId: 'astra-ns70l',
        latitude: 3.1390,
        longitude: 101.6869,
      );

      // Assert
      expect(cubit.state.selectedBattery, isNotNull);
      expect(cubit.state.selectedBattery!.id, 'astra-ns70l');
      expect(cubit.state.selectedBattery!.name, 'ASTRA NS70L');
    });
  });
}
```

### Widget Test - Product Display

```dart
void main() {
  testWidgets('ProductBatteryScreen displays product cards',
    (WidgetTester tester) async {
    // Arrange
    final mockCubit = MockBatteryPurchaseCubit();
    final products = [
      ProductItem(id: '1', name: 'Product 1', ...),
      ProductItem(id: '2', name: 'Product 2', ...),
    ];

    when(mockCubit.stream).thenAnswer((_) => Stream.value(
      mockCubit.state.copyWith(apiProducts: products)
    ));

    // Act
    await tester.pumpWidget(
      MaterialApp(
        home: BlocProvider<BatteryPurchaseCubit>.value(
          value: mockCubit,
          child: const ProductBatteryScreen(),
        ),
      ),
    );

    // Assert
    expect(find.text('Select your Battery'), findsOneWidget);
    expect(find.text('Product 1'), findsOneWidget);
    expect(find.text('Product 2'), findsOneWidget);
  });
}
```

---

## ⚠️ Important Notes

1. **Products from API only** - No hardcoded fallback products
2. **Route arguments required** - Screen expects products list in navigation args
3. **Product IDs can be String** - Both string IDs (e.g., "astra-ns70l") and numeric IDs (e.g., 5) are supported
4. **Images support both network and local assets** - Graceful fallback to icon
5. **Warranty can be time or distance based** - Or both combined
6. **Selection is visual feedback only** - Actual selection happens when proceeding
7. **BLoC manages state consistently** - No local state conflicts
8. **Quote fetching is optional** - Can be deferred to order confirmation
9. **Location data is required** - For delivery fee calculation
10. **Brand ID is optional** - Used for analytics and filtering

---

## 🚀 Future Enhancements

This architecture makes it easy to add:
- **Filtering by brand, warranty, price** - Add to product display logic
- **Product comparison** - Compare multiple selected products
- **Advanced search** - Filter and sort products
- **Product reviews** - Display ratings and reviews
- **Inventory levels** - Show stock quantity
- **Real-time availability** - WebSocket updates
- **Quick filters** - Filter by warranty period, price range
- **Wishlist functionality** - Save products for later
- **Product details modal** - Expandable full details view
- **Size recommendations** - AI-based battery sizing

---

## 📚 Related Files

- [product_battery_screen.dart](lib/features/battery_purchase/screens/product_battery_screen.dart) - Main UI
- [battery_purchase_cubit.dart](lib/bloc/battery_purchase/battery_purchase_cubit.dart) - State management
- [battery_purchase_state.dart](lib/bloc/battery_purchase/battery_purchase_state.dart) - State definitions
- [products_service.dart](lib/core/services/products_service.dart) - API client
- [battery_purchase_service.dart](lib/core/services/battery_purchase_service.dart) - Business logic service
- [vehicle_screen.dart](lib/shared/screens/vehicle_screen.dart) - Previous screen (calls ProductsService)
- [order_confirmation_screen.dart](lib/features/order/screens/order_confirmation_screen.dart) - Next screen

---

## 🔗 Related Documentation

- `ORDER_CONFIRMATION_ARCHITECTURE.md` - Order review screen architecture
- `BATTERY_PURCHASE_API.md` - Backend API specifications
- `DEVELOPMENT.md` - Development setup guide
- `MODULE_CREATION_GUIDE.md` - Creating new modules
