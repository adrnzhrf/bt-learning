# Order Confirmation Screen - Architecture & Code Explanation

## 📋 Overview

The **Order Confirmation Screen** is the checkout/order review page in Bateriku's mobile app where customers confirm battery purchase details, apply promo codes, see pricing breakdowns, and create orders. It's been **refactored from 2,198 lines to a clean architecture** separating concerns across layers.

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
│  [OrderConfirmationCubit] (160 lines)              │
│  - calculateOrder()                                │
│  - validatePromoCode()                            │
│  - loadProducts()                                  │
│  - toggleTradeIn()                                │
└────────────────┬────────────────────────────────────┘
                 │ delegates to
┌────────────────▼────────────────────────────────────┐
│  Service Layer                                      │
│  [OrderConfirmationService] (297 lines)           │
│  - API calls to backend                            │
│  - Response parsing                                │
│  - Error handling                                  │
└────────────────┬────────────────────────────────────┘
                 │ calls
┌────────────────▼────────────────────────────────────┐
│  API Clients                                        │
│  [OrderService] - Order calculations              │
│  [ProductsService] - Product data                 │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Key Files & Their Responsibilities

### 1. [order_confirmation_screen.dart](lib/features/order/screens/order_confirmation_screen.dart) *(2,397 lines - Original)*

**Purpose:** Main UI screen (old monolithic implementation)

**Contains:**
- Original implementation with everything mixed together
- 20+ local state variables
- API calls scattered throughout
- Business logic mixed with UI rendering
- ⚠️ Hard to test and maintain

**Status:** Still available for backward compatibility, but refactored version is recommended

---

### 2. [order_confirmation_cubit.dart](lib/bloc/order_confirmation/order_confirmation_cubit.dart) *(160 lines)*

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

### 3. [order_confirmation_service.dart](lib/core/services/order_confirmation_service.dart) *(297 lines)*

**Purpose:** API communication and service coordination

**Key Methods:**
```dart
class OrderConfirmationService {
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
  
  // Validate promo code by calculating order
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

**Responsibilities:**
- Delegates to `OrderService` for order calculations
- Delegates to `ProductsService` for product data
- Parses API responses
- Handles errors
- Implements retry logic

---

### 4. [order_service.dart](lib/core/services/order_service.dart) *(349 lines)*

**Purpose:** Order calculation API client

**Key Method:**
```dart
class OrderService {
  Future<OrderCalculationResult> calculateOrder({
    required String token,
    required double latitude,
    required double longitude,
    required dynamic productId,
    String? promoCode,
    int tradeIn = 0,
  })
}
```

**API Endpoint:** `POST /api/partners/v2/orders/calculate`

**Parses Response into [OrderCalculation](#ordercalculation-data-model) model:**
- `subtotal` - Product price
- `deliveryFee` - Shipping cost
- `promoDiscount` - Discount from promo code
- `tradeInDiscount` - Trade-in vehicle discount
- `total` - Final amount to pay
- `isPromoValid` - Promo code validity
- `promoCode` - Code that was applied

---

### 5. [order_confirmation_state.dart](lib/bloc/order_confirmation/order_confirmation_state.dart) *(140 lines)*

**Purpose:** State definitions for the Cubit

**Contains:**
- All possible states the confirmation screen can be in
- Immutable state objects extending `Equatable`
- Used for BLoC event handling
- Props list for equality comparison

---

## 💾 Data Models

### OrderCalculation (in [order_service.dart](lib/core/services/order_service.dart#L187))

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

### ProductItem (from ProductsService)

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
Service parses response → OrderCalculation
        ↓
Cubit emits: PromoValidationSuccess(
  message: "Promo code applied successfully",
  discountAmount: 50.00
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
    "customer": {
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "0123456789"
    },
    "vehicle": {
      "plate_number": "ABC1234"
    }
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

1. **Original Screen is 2,198 lines** - Very difficult to maintain and test
2. **Refactored version separates concerns** - Better testability and maintainability
3. **BLoC pattern ensures state consistency** - No local state conflicts
4. **Service layer handles all API logic** - Easy to mock for testing
5. **Product IDs can be String or int** - API accepts both formats (e.g., "astra-ns70l" or 5)
6. **Trade-in discount is RM20 hardcoded** - Hardcoded on mobile, verified on server
7. **Promo codes validated via API** - No client-side validation
8. **Location is required** - Affects delivery fee calculation
9. **All calculations are decimal** - Store as double, display with 2 decimal places
10. **API responses use cents** - `final_amount_cents` field is in cents, divide by 100

---

## 🚀 Future Enhancements

This architecture makes it easy to add:
- **Different payment methods** - Just extend OrderCreationCubit
- **Loyalty points integration** - Add to calculation API
- **Subscription options** - New product type
- **Advanced promo logic** - Multiple promo codes
- **Order history** - New service layer
- **Real-time price updates** - WebSocket integration
- **Offline support** - Local caching layer

---

## 📚 Related Documentation

- `REFACTORING_START_HERE.md` - Overview of refactoring
- `ORDER_CONFIRMATION_QUICK_REFERENCE.md` - Quick developer reference
- `REFACTORING_ORDER_CONFIRMATION.md` - Detailed architecture guide
- `BATTERY_PURCHASE_API.md` - API documentation
