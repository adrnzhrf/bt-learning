# Order Confirmation Screen - Architecture & Code Explanation

## 📋 Overview

The **Order Confirmation Screen** is the checkout/order review page in Bateriku's mobile app where customers confirm battery purchase details, apply promo codes, see pricing breakdowns, and create orders. It's been **refactored from 2,198 lines to a clean architecture** separating concerns across layers.

## 🆕 Latest Updates (February 2026)

### New Features & Changes

**Service Layer Enhancements:**
- ✅ **Singleton pattern** implemented for `OrderConfirmationService` and `OrderService`
- ✅ **Advanced promo validation** with `PromoDetails` extraction (discount type, value, minimum amount)
- ✅ **Multi-source discount resolution** - intelligently extracts discount from multiple API response sources
- ✅ **Comprehensive debug logging** for promo code validation troubleshooting

**API Changes:**
- ✅ **New parameters for order creation**: `leadReason` and `orderType` for better order tracking and categorization
- ✅ **Flexible product ID support** - accepts both String (e.g., "astra-ns70l") and int IDs
- ✅ **Enhanced error handling** with detailed API logging

**Cubit & State Management:**
- ✅ **OrderCreationSuccess** now contains full `OrderCreationResult` object for richer data access
- ✅ Better state isolation with combined `OrderConfirmationUpdated` state for multiple concerns
- ✅ Improved error propagation with individual error states per operation

---

## 🏗️ Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│  UI Layer                                           │
│  [OrderConfirmationScreen] (800 lines)             │
│  - Renders UI components                            │
│  - Displays BLoC state                             │
│  - Handles user interactions                        │
└────────────────┬────────────────────────────────────┘
                 │ listens to
┌────────────────▼────────────────────────────────────┐
│  Business Logic (BLoC Pattern)                      │
│  [OrderConfirmationCubit] (164 lines)              │
│  - calculateOrder()                                │
│  - validatePromoCode()                            │
│  - loadProducts()                                  │
│  - createOrder()                                  │
└────────────────┬────────────────────────────────────┘
                 │ delegates to
┌────────────────▼────────────────────────────────────┐
│  Service Layer                                      │
│  [OrderConfirmationService] (300 lines)           │
│  - Singleton instance access                       │
│  - Smart discount resolution                       │
│  - PromoDetails extraction                         │
│  - Comprehensive error handling                    │
└────────────────┬────────────────────────────────────┘
                 │ calls
┌────────────────▼────────────────────────────────────┐
│  API Clients                                        │
│  [OrderService] (381 lines) - Order calculations  │
│  [ProductsService] - Product data                 │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Key Files & Their Responsibilities

### 1. [order_confirmation_cubit.dart](lib/bloc/order_confirmation/order_confirmation_cubit.dart) *(164 lines)*

**Purpose:** Business logic orchestration

**Key Methods:**
```dart
class OrderConfirmationCubit extends Cubit<OrderConfirmationState> {
  final OrderConfirmationService _orderService;
  
  // Calculate order with optional promo code
  Future<void> calculateOrder({
    required double latitude,
    required double longitude,
    required dynamic productId,
    String? promoCode,
    int tradeIn = 0,
  })
  
  // Load available battery products
  Future<void> loadProducts({
    required String userName,
    required String userPhone,
    required String userEmail,
    required double latitude,
    required double longitude,
    required String vehiclePlateNumber,
  })
  
  // Validate promo code
  Future<void> validatePromoCode({
    required double latitude,
    required double longitude,
    required int productId,
    required String promoCode,
    int tradeIn = 0,
    double? discountAmount,
  })
  
  // Create order and get payment link
  Future<void> createOrder({...})
}
```

**State Management:**
- `OrderCalculationLoading` → API is calculating
- `OrderCalculationSuccess` → Got calculation with totals
- `OrderCalculationError` → Calculation failed
- `ProductsLoading` → Loading product list
- `ProductsLoaded` → Products fetched successfully
- `ProductsError` → Failed to load products
- `PromoValidationLoading` → Validating promo code
- `PromoValidationSuccess` → Promo is valid
- `PromoValidationError` → Promo validation failed
- `OrderCreationLoading` → Creating order
- `OrderCreationSuccess` → Order created, get payment link
- `OrderCreationError` → Order creation failed

---

### 2. [order_confirmation_service.dart](lib/core/services/order_confirmation_service.dart) *(300 lines)*

**Purpose:** API communication and service coordination with enhanced promo validation

**Implementation Details:**
- **Singleton pattern** - Ensures single instance across app (`OrderConfirmationService.instance`)
- **Advanced promo validation** - Extracts discount values from multiple sources
- **Detailed logging** - Debug logging for troubleshooting promo code validation

**Key Methods:**
```dart
class OrderConfirmationService {
  static final OrderConfirmationService instance = OrderConfirmationService._();

  // Delegates to OrderService for calculation
  Future<OrderCalculationResult> calculateOrderWithPromo({
    required double latitude,
    required double longitude,
    required dynamic productId,
    String? promoCode,
    int tradeIn = 0,
  })
  
  // Fetch available products
  Future<ProductsResult> loadProducts({
    required String userName,
    required String userPhone,
    required String userEmail,
    required double latitude,
    required double longitude,
    required String vehiclePlateNumber,
  })
  
  // Validate promo code by calculating order with smart discount extraction
  Future<PromoValidationResult> validatePromoCode({
    required double latitude,
    required double longitude,
    required int productId,
    required String promoCode,
    int tradeIn = 0,
    double? discountAmount,
  })
  
  // Create order and return payment details
  Future<OrderCreationResult> createOrder({...})
}
```

**Promo Validation Features:**
- **Multi-source discount extraction** - Tries multiple sources (parameter, calculation, promo_code object, discount field)
- **PromoDetails parsing** - Extracts discount type, value, and minimum order amount from API response
- **Flexible response handling** - Handles different API response formats

**Key Classes:**
```dart
class PromoValidationResult {
  final bool isValid;
  final String message;
  final OrderCalculation? calculation;
  final PromoDetails? promoDetails;

  PromoValidationResult.valid({...}) // Success case
  PromoValidationResult.invalid({...}) // Failure case
}

class PromoDetails {
  final String? discountType;        // e.g., "fixed", "percentage"
  final double? discountValue;       // Discount amount/percentage
  final double? minimumOrderAmount;  // Minimum purchase requirement
  
  factory PromoDetails.fromJson(Map<String, dynamic> json)
  Map<String, dynamic> toJson()
}
```

**Responsibilities:**
- Delegates to `OrderService` for order calculations
- Delegates to `ProductsService` for product data
- Smart discount amount resolution (fallback chain)
- Enhanced promo details extraction from API
- Comprehensive error handling with detailed logging

---

### 3. [order_service.dart](lib/core/services/order_service.dart) *(381 lines)*

**Purpose:** Core order API client for calculations and creation

**Singleton Implementation:**
```dart
class OrderService {
  String get _baseUrl => EnvConfig.batteryPurchaseApiUrl;
  static final OrderService instance = OrderService._();
  OrderService._();

  static const String _calculateEndpoint = '/api/partners/v2/orders/calculate';
  static const String _createOrderEndpoint = '/api/partners/v2/orders';
}
```

**Key Methods:**
```dart
// Calculate order total with discounts and fees
Future<OrderCalculationResult> calculateOrder({
  required String token,
  required double latitude,
  required double longitude,
  required dynamic productId,    // Accepts String or int
  String? promoCode,
  int tradeIn = 0,
})

// Create new order with customer and payment details
Future<OrderCreationResult> createOrder({
  required String token,
  required String userName,
  required String userPhone,
  required String userEmail,
  required double latitude,
  required double longitude,
  required String address,
  required String vehiclePlateNumber,
  required dynamic productId,    // Accepts String or int
  required int brandId,
  String paymentType = 'billplz',
  int tradeIn = 0,
  String? promoCode,
  String? notes,
  String? redirectUrl,
  String? leadReason,            // NEW: For order tracking/analytics
  String? orderType,             // NEW: For order categorization
})
```

**API Endpoints:**
- `POST /api/partners/v2/orders/calculate` - Calculate order with discounts
- `POST /api/partners/v2/orders` - Create order and initiate payment

**Parses Responses:**
- `OrderCalculationResult` - Contains OrderCalculation model with pricing breakdown
- `OrderCreationResult` - Contains OrderCreationResponse with order ID and payment URL

---

### 4. [order_confirmation_state.dart](lib/bloc/order_confirmation/order_confirmation_state.dart) *(140 lines)*

**Purpose:** State definitions for the Cubit

**State Hierarchy:**
```dart
abstract class OrderConfirmationState extends Equatable {}

// Initial state
class OrderConfirmationInitial

// Calculation states
class OrderCalculationLoading
class OrderCalculationSuccess {
  final OrderCalculation calculation;
}
class OrderCalculationError {
  final String message;
}

// Products loading states
class ProductsLoading
class ProductsLoaded {
  final List<ProductItem> products;
  final int? brandId;
}
class ProductsError

// Promo validation states
class PromoValidationLoading
class PromoValidationSuccess {
  final String message;
  final double discountAmount;
  final OrderCalculation? calculation;
}
class PromoValidationError

// Order creation states
class OrderCreationLoading
class OrderCreationSuccess {
  final OrderCreationResult result;  // NEW: Contains full result object
}
class OrderCreationError

// Combined state (if needed)
class OrderConfirmationUpdated {
  final OrderCalculation? calculation;
  final List<ProductItem>? products;
  final String? calculationError;
  final String? productsError;
  final String? promoError;
}
```

**State Features:**
- All states extend `Equatable` for equality comparison
- Props lists for proper state comparison
- Type-safe state access in widgets

---

## 💾 Data Models

### OrderCalculation

```dart
class OrderCalculation {
  final double subtotal;           // Product price
  final double deliveryFee;        // Shipping cost
  final double promoDiscount;      // Discount from promo code
  final double tradeInDiscount;    // Discount from trade-in vehicle
  final double total;              // Final amount to pay
  final String? promoCode;         // Promo code used
  final bool isPromoValid;         // Is the promo code valid?
  
  // Factory constructor for API JSON parsing
  factory OrderCalculation.fromJson(Map<String, dynamic> json)
  
  // Convert back to JSON
  Map<String, dynamic> toJson()
}
```

**Example:**
```dart
OrderCalculation(
  subtotal: 450.00,
  deliveryFee: 15.00,
  promoDiscount: 50.00,
  tradeInDiscount: 20.00,
  total: 395.00,  // 450 + 15 - 50 - 20
  promoCode: 'BATTERIUNEWFD',
  isPromoValid: true,
)
```

### OrderCalculationResult

```dart
class OrderCalculationResult {
  final bool isSuccess;
  final OrderCalculation? calculation;
  final String? error;
  
  // Success constructor
  OrderCalculationResult.success({required this.calculation})
  
  // Failure constructor
  OrderCalculationResult.failure(this.error)
}
```

### ProductItem

```dart
class ProductItem {
  final String id;           // e.g., "astra-ns70l"
  final String name;         // e.g., "ASTRA NS70L"
  final int priceCents;      // Price in cents (divide by 100 for RM)
  final String warranty;     // e.g., "36 months"
  final String? imageUrl;    // Product image
  final String? description; // Product details
}
```

### OrderCreationResponse

```dart
class OrderCreationResponse {
  final String orderId;          // Generated order ID
  final String status;           // e.g., "pending_payment"
  final String? paymentUrl;      // Payment gateway URL
  final String? paymentId;       // Payment provider's ID
  final double totalAmount;      // Amount to pay
}
```

### OrderCreationResult

```dart
class OrderCreationResult {
  final bool isSuccess;
  final OrderCreationResponse? order;
  final String? error;

  OrderCreationResult.success({required this.order})
      : isSuccess = true, error = null;

  OrderCreationResult.failure(this.error)
      : isSuccess = false, order = null;
}
```

### PromoDetails

```dart
class PromoDetails {
  final String? discountType;     // e.g., "fixed", "percentage"
  final double? discountValue;    // Discount amount/percentage
  final double? minimumOrderAmount; // Minimum purchase requirement

  factory PromoDetails.fromJson(Map<String, dynamic> json)
  Map<String, dynamic> toJson()
}
```

---

## 🔄 Data Flow Examples

### Example 1: User Applies Promo Code

```
User enters promo code "BATTERIUNEWFD"
        ↓
Screen calls: cubit.validatePromoCode(
  latitude: -3.1456,
  longitude: 101.6964,
  productId: 5,
  promoCode: "BATTERIUNEWFD",
  tradeIn: 0
)
        ↓
Cubit emits: PromoValidationLoading state
        ↓
Cubit calls: _orderService.validatePromoCode(...)
        ↓
Service makes API call: 
  POST /api/partners/v2/orders/calculate
  Body: {
    "order": {
      "latitude": -3.1456,
      "longitude": 101.6964,
      "product_id": 5,
      "promo_code": "BATTERIUNEWFD",
      "trade_in": 0
    }
  }
        ↓
API responds with calculation including promoDiscount: 50.00
        ↓
Service extracts:
  - OrderCalculation from response
  - PromoDetails (if available)
  - Smart discount amount from multiple sources
        ↓
Service returns: PromoValidationResult.valid(
  message: 'Voucher applied!',
  calculation: OrderCalculation,
  promoDetails: PromoDetails
)
        ↓
Cubit emits: PromoValidationSuccess(
  message: "Promo code applied successfully",
  discountAmount: 50.00,
  calculation: OrderCalculation
)
        ↓
Screen listens to state change via BlocBuilder
        ↓
UI updates to show:
  ✅ Promo applied successfully
  ✅ New total: RM395.00 (was RM445.00)
  ✅ Discount breakdown: -RM50.00
```

### Example 2: User Selects Battery Product

```
User taps on "ASTRA NS70L" battery option
        ↓
Screen calls: cubit.updateSelectedBattery(
  name: "ASTRA NS70L",
  price: 450.00,
  warranty: "36 Months",
  productId: 5
)
        ↓
Cubit internally recalculates order:
  - Calls calculateOrder() with new productId
  - Emits OrderCalculationLoading
        ↓
API calculates delivery fee based on location + product
        ↓
Cubit emits: OrderCalculationSuccess(calculation)
        ↓
Screen rebuilds with new:
  - Selected battery highlighted
  - New subtotal: RM450.00
  - New delivery fee: RM15.00
  - New total: RM465.00 (or less if promo applied)
```

### Example 3: User Toggles Trade-In Option

```
User taps "Trade-in Vehicle" toggle
        ↓
Screen calls: cubit.toggleTradeIn()
        ↓
Cubit updates internal state: tradeIn = 1
        ↓
Cubit recalculates: calculateOrder(tradeIn: 1)
        ↓
API returns updated calculation with tradeInDiscount: 20.00
        ↓
Cubit emits: OrderCalculationSuccess(calculation)
        ↓
UI updates to show:
  - Trade-in toggle is now ON
  - New discount: -RM20.00
  - New total: RM445.00 (450 + 15 - 20)
```

---

## 🛠️ Key Features

| Feature | Implementation | Notes |
|---------|----------------|-------|
| **Product Selection** | User selects battery → `updateSelectedBattery()` called | Updates cubit state |
| **Price Calculation** | API calculates based on location + product ID | Uses `/orders/calculate` endpoint |
| **Promo Codes** | User enters code → `validatePromoCode()` called | Validates and gets discount amount |
| **Trade-In Discount** | Toggle → `toggleTradeIn()` → Recalculates | RM20 discount hardcoded on mobile |
| **Delivery Fee** | Location-based → API returns fee | Varies by delivery zone |
| **Total Breakdown** | `Subtotal + Delivery - Promo - TradeIn = Total` | All calculated on API side |
| **Product Loading** | Fetches list on screen init → `loadProducts()` | Shows available batteries |

---

## 📊 Pricing Calculation Formula

```dart
// Simple formula
Total = Subtotal + DeliveryFee - PromoDiscount - TradeInDiscount

// Example values
Total = 450.00 + 15.00 - 50.00 - 20.00 = 395.00

// From OrderConfirmationScreen (old):
double get tradeInDiscount => hasTradeIn ? 20.00 : 0.0;
double get voucherDiscount => voucherDiscountAmount ?? 
                              _apiCalculation?.promoDiscount ?? 0.0;
double get subtotal => batteryPrice;
double get deliveryFee => _apiCalculation?.deliveryFee ?? 0.0;
double get total => _apiCalculation?.total ?? 
  (subtotal + deliveryFee - tradeInDiscount - voucherDiscount);
```

**API Response Example:**
```json
{
  "subtotal": 450.00,
  "delivery_fee": 15.00,
  "promo_discount": 50.00,
  "trade_in_discount": 20.00,
  "total": 395.00,
  "final_amount_cents": 39500,
  "promo_code": "BATTERIUNEWFD",
  "is_promo_valid": true
}
```

---

## 🎯 API Endpoints Used

### 1. Calculate Order

| Property | Value |
|----------|-------|
| **Endpoint** | `/api/partners/v2/orders/calculate` |
| **Method** | POST |
| **Purpose** | Calculate order total with discounts and delivery fee |
| **Auth** | Bearer token required |

**Request Payload:**
```json
{
  "order": {
    "latitude": -3.1456,
    "longitude": 101.6964,
    "product_id": "astra-ns70l",
    "promo_code": "BATTERIUNEWFD",
    "trade_in": 1
  }
}
```

**Response:**
```json
{
  "subtotal": 450.00,
  "delivery_fee": 15.00,
  "promo_discount": 50.00,
  "trade_in_discount": 20.00,
  "total": 395.00,
  "final_amount_cents": 39500,
  "promo_code": "BATTERIUNEWFD",
  "is_promo_valid": true
}
```

### 2. Create Order

| Property | Value |
|----------|-------|
| **Endpoint** | `/api/partners/v2/orders` |
| **Method** | POST |
| **Purpose** | Create new order and get payment link |
| **Auth** | Bearer token required |

**Request Body:**
```json
{
  "order": {
    "latitude": -3.1456,
    "longitude": 101.6964,
    "product_id": 5,
    "promo_code": "BATTERIUNEWFD",
    "trade_in": 1,
    "lead_reason": "battery_weak",
    "order_type": "battery_replacement",
    "customer": {
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "0123456789"
    },
    "vehicle": {
      "plate_number": "ABC1234"
    },
    "address": "123 Street Name, City",
    "notes": "Optional delivery notes"
  }
}
```

**Response:**
```json
{
  "order_id": "ORD-20260109-12345",
  "status": "pending_payment",
  "payment_url": "https://payment-gateway.com/pay/...",
  "payment_id": "PYMT-12345",
  "total_amount": 395.00
}
```

**NEW Parameters:**
- `lead_reason` - Tracking reason for the order (from ServiceType)
- `order_type` - Categorization of the order (from ServiceType)

### 3. Load Products

| Property | Value |
|----------|-------|
| **Endpoint** | `/api/partners/v2/products` |
| **Method** | GET |
| **Purpose** | Get available battery products |
| **Query Params** | `latitude`, `longitude`, `user_name`, etc. |

**Response:**
```json
{
  "products": [
    {
      "id": "astra-ns70l",
      "name": "ASTRA NS70L",
      "price_cents": 45000,
      "warranty": "36 months",
      "image_url": "https://...",
      "description": "Premium battery..."
    },
    {
      "id": "astra-ns40l",
      "name": "ASTRA NS40L",
      "price_cents": 35000,
      "warranty": "24 months",
      "image_url": "https://...",
      "description": "Standard battery..."
    }
  ],
  "brand_id": 1
}
```

---

## 🧪 Testing Examples

### Unit Test - Calculate Order

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';

void main() {
  group('OrderConfirmationCubit', () {
    test('calculateOrder emits OrderCalculationSuccess on success', () async {
      // Arrange
      final mockService = MockOrderConfirmationService();
      final expectedCalculation = OrderCalculation(
        subtotal: 450.00,
        deliveryFee: 15.00,
        promoDiscount: 0.0,
        tradeInDiscount: 0.0,
        total: 465.00,
        promoCode: null,
        isPromoValid: false,
      );
      
      when(mockService.calculateOrderWithPromo(
        latitude: anyNamed('latitude'),
        longitude: anyNamed('longitude'),
        productId: anyNamed('productId'),
        promoCode: anyNamed('promoCode'),
        tradeIn: anyNamed('tradeIn'),
      )).thenAnswer((_) async => 
        OrderCalculationResult.success(calculation: expectedCalculation)
      );

      final cubit = OrderConfirmationCubit(orderService: mockService);

      // Act
      await cubit.calculateOrder(
        latitude: -3.1456,
        longitude: 101.6964,
        productId: 5,
      );

      // Assert
      expect(cubit.state, isA<OrderCalculationSuccess>());
      final successState = cubit.state as OrderCalculationSuccess;
      expect(successState.calculation.total, 465.00);
    });

    test('calculateOrder emits OrderCalculationError on failure', () async {
      // Arrange
      final mockService = MockOrderConfirmationService();
      when(mockService.calculateOrderWithPromo(...))
        .thenAnswer((_) async => 
          OrderCalculationResult.failure('Network error')
        );

      final cubit = OrderConfirmationCubit(orderService: mockService);

      // Act
      await cubit.calculateOrder(...);

      // Assert
      expect(cubit.state, isA<OrderCalculationError>());
    });
  });
}
```

### Widget Test - Display Order Summary

```dart
void main() {
  testWidgets('OrderConfirmationScreen displays price breakdown', 
    (WidgetTester tester) async {
    // Arrange
    final mockCubit = MockOrderConfirmationCubit();
    const calculation = OrderCalculation(
      subtotal: 450.00,
      deliveryFee: 15.00,
      promoDiscount: 50.00,
      tradeInDiscount: 20.00,
      total: 395.00,
      promoCode: 'BATTERIUNEWFD',
      isPromoValid: true,
    );

    when(mockCubit.stream)
      .thenAnswer((_) => Stream.value(
        OrderCalculationSuccess(calculation)
      ));

    // Act
    await tester.pumpWidget(
      MaterialApp(
        home: BlocProvider<OrderConfirmationCubit>.value(
          value: mockCubit,
          child: const OrderConfirmationScreen(),
        ),
      ),
    );

    // Assert - Check price breakdown
    expect(find.text('Subtotal: RM450.00'), findsOneWidget);
    expect(find.text('Delivery Fee: RM15.00'), findsOneWidget);
    expect(find.text('Promo Discount: -RM50.00'), findsOneWidget);
    expect(find.text('Trade-in Discount: -RM20.00'), findsOneWidget);
    expect(find.text('Total: RM395.00'), findsOneWidget);
  });
}
```

---

## 🔧 How to Use in Code

### Access Current Calculation

```dart
// In a widget
final cubit = context.read<OrderConfirmationCubit>();
final calculation = cubit.state;

if (calculation is OrderCalculationSuccess) {
  print('Total: ${calculation.calculation.total}');
}
```

### Listen to Calculation Changes

```dart
BlocListener<OrderConfirmationCubit, OrderConfirmationState>(
  listener: (context, state) {
    if (state is OrderCalculationSuccess) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Order calculated: RM${state.calculation.total}')),
      );
    } else if (state is OrderCalculationError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: ${state.message}')),
      );
    }
  },
  child: OrderSummaryWidget(),
);
```

### Build UI Based on State

```dart
BlocBuilder<OrderConfirmationCubit, OrderConfirmationState>(
  builder: (context, state) {
    if (state is OrderCalculationLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    
    if (state is OrderCalculationSuccess) {
      final calc = state.calculation;
      return OrderSummary(
        subtotal: calc.subtotal,
        deliveryFee: calc.deliveryFee,
        promoDiscount: calc.promoDiscount,
        total: calc.total,
      );
    }
    
    if (state is OrderCalculationError) {
      return ErrorWidget(message: state.message);
    }
    
    return const SizedBox.shrink();
  },
);
```

---

## ⚠️ Important Notes

1. **Singleton Pattern** - Both `OrderConfirmationService` and `OrderService` use singleton pattern for consistent access
2. **Product IDs can be String or int** - API accepts both formats (e.g., "astra-ns70l" or 5)
3. **Smart Discount Resolution** - Service tries multiple sources for discount amount (parameter → calculation → promo_code object → discount field)
4. **PromoDetails Extraction** - API responses can include detailed promo information (discount type, value, minimum amount)
5. **Trade-in discount is RM20 hardcoded** - Hardcoded on mobile, verified on server
6. **Promo codes validated via API** - No client-side validation, HTTP 200 indicates valid code
7. **Location is required** - Affects delivery fee calculation and product eligibility
8. **All calculations are decimal** - Store as double, display with 2 decimal places
9. **Order Creation Parameters** - New `leadReason` and `orderType` parameters for better tracking
10. **Enhanced Logging** - Extensive debug logging in promo validation for troubleshooting
11. **State Contains Full Result** - `OrderCreationSuccess` contains complete `OrderCreationResult` object
12. **API responses use cents** - `final_amount_cents` field is in cents, divide by 100

---

## 🚀 Future Enhancements

This architecture makes it easy to add:
- **Different payment methods** - Just extend OrderCreationCubit
- **Loyalty points integration** - Add to calculation API
- **Subscription options** - New product type
- **Advanced promo logic** - Multiple promo codes with stacking
- **Order history** - New service layer
- **Real-time price updates** - WebSocket integration
- **Offline support** - Local caching layer
- **Analytics tracking** - Use `leadReason` and `orderType` for insights

---

## 📚 Related Documentation

- `REFACTORING_START_HERE.md` - Overview of refactoring
- `ORDER_CONFIRMATION_QUICK_REFERENCE.md` - Quick developer reference
- `REFACTORING_ORDER_CONFIRMATION.md` - Detailed architecture guide
- `BATTERY_PURCHASE_API.md` - API documentation
