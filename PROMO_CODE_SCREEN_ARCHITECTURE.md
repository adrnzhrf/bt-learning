# Promo Code Screen - Architecture & Code Explanation

## 📋 Overview

The **Promo Code Screen** (also called Voucher Code Screen) is a dedicated modal screen where customers apply discount voucher/promo codes to their battery purchase orders. It handles promo code validation, discount extraction from API responses, and displays the discount amount. The screen is accessed from the Order Confirmation Screen and returns the applied code and discount details back to the parent screen.

---

## 🏗️ Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│  UI Layer                                           │
│  [PromoCodeScreen] (462 lines)                      │
│  - Voucher input field                              │
│  - Apply/Remove buttons                             │
│  - Success/Empty states                             │
│  - Discount display                                 │
└────────────────┬────────────────────────────────────┘
                 │ uses
┌────────────────▼────────────────────────────────────┐
│  HTTP Client (Direct)                               │
│  [PromoCodeScreen._applyVoucher()]                  │
│  - Direct API call to validate endpoint             │
│  - No BLoC pattern (StatefulWidget)                │
│  - Local state management                           │
└────────────────┬────────────────────────────────────┘
                 │ also used by
┌────────────────▼────────────────────────────────────┐
│  Service Layer (for Order Confirmation)             │
│  [OrderConfirmationService]                         │
│  - validatePromoCode()                             │
│  - Discount extraction & normalization              │
│  - Multiple response format support                 │
└────────────────┬────────────────────────────────────┘
                 │ calls
┌────────────────▼────────────────────────────────────┐
│  API Endpoints                                      │
│  /api/partners/v2/promo_codes/validate (PromoScreen)
│  /api/partners/v2/orders/calculate (OrderConfirm) │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Key Files & Their Responsibilities

### 1. [promo_code_screen.dart](lib/features/order/screens/promo_code_screen.dart) *(462 lines)*

**Purpose:** Dedicated UI screen for voucher code validation and application

**Contains:**
- `PromoCodeScreen` - StatefulWidget that receives initial voucher code (optional)
- `_PromoCodeScreenState` - State management with local variables
- `_applyVoucher()` - Direct HTTP API call to validate voucher
- `_extractDiscountAmount()` - Robust discount extraction from multiple response formats
- `_showRemoveVoucherDialog()` - Confirmation dialog to remove voucher
- UI widgets for input, success/empty states, and illustrations

**Key Responsibilities:**
- Manages local state: `_appliedVoucher`, `_discountAmount`, `_discountValue`, `_isPromoValid`
- Validates voucher codes against API
- Extracts discount amount from various response formats
- Displays success/failure states with illustrations
- Returns applied voucher data back to parent screen
- Handles voucher removal with confirmation

**State Properties:**
```dart
late final TextEditingController _voucherController;
String? _appliedVoucher;           // Applied voucher code
double? _discountAmount;           // Raw discount amount (e.g., 50.0)
String? _discountCurrency;         // Currency (RM)
String? _discountValue;            // Formatted discount (e.g., "50.00")
bool _isPromoValid;                // Is voucher valid?
```

**Key Methods:**

```dart
// Apply/validate voucher code
Future<void> _applyVoucher() {
  // 1. Send POST request to /api/partners/v2/promo_codes/validate
  // 2. Parse response for discount
  // 3. Update local state
  // 4. Update UI with success/error
}

// Extract discount from various API response formats
double _extractDiscountAmount(Map<String, dynamic> data) {
  // Try: data['promo_code']['value']
  // Fallback: data['promo_discount']
  // Fallback: data['discount']
  // Fallback: data['discount_amount']
  // Fallback: data['amount']
}

// Show confirmation dialog for removing voucher
void _showRemoveVoucherDialog() {
  // Display BTConfirmDialog
  // Clear local state on confirmation
}
```

**API Endpoint (Direct):**
- **Endpoint**: `POST /api/partners/v2/promo_codes/validate`
- **Base URL**: `https://staging.jualbaterikereta.com`
- **Auth Token**: `b0d2a19821a841691af917e54e3a75d4`

**Request Payload:**
```json
{
  "code": "BATTERIUNEWFD"
}
```

**Response Format:**
```json
{
  "code": "BATTERIUNEWFD",
  "promo_code": {
    "value": "RM50"
  },
  "promo_discount": 50.00,
  "discount": 50.00,
  "discount_amount": 50.00,
  "amount": 50.00,
  "promo_details": {
    "discount_type": "fixed",
    "discount_value": 50.00,
    "minimum_order_amount": 200.00
  }
}
```

**Navigation:**
- **Route**: `/order/promo-code`
- **Arguments**: Optional initial voucher data
- **Return**: `Map<String, dynamic>` with voucher code and discount
  ```dart
  {
    'voucherCode': String,
    'discountValue': String,  // "50.00"
    'discountAmount': double  // 50.0
  }
  ```

---

### 2. [order_confirmation_service.dart](lib/core/services/order_confirmation_service.dart) *(Promo validation section)*

**Purpose:** Service layer for promo code validation used by Order Confirmation Screen

**Key Method:**

```dart
Future<PromoValidationResult> validatePromoCode({
  required double latitude,
  required double longitude,
  required int productId,
  required String promoCode,
  int tradeIn = 0,
  double? discountAmount,
})
```

**Responsibilities:**
- Validates promo code via order calculation API
- Extracts discount from API response
- Falls back to discount passed from PromoCodeScreen
- Handles multiple response formats
- Returns `PromoValidationResult` with discount and calculation details

**API Endpoint (Service):**
- **Endpoint**: `POST /api/partners/v2/orders/calculate`
- **Purpose**: Calculate order total and validate promo code

**Request Payload:**
```json
{
  "order": {
    "latitude": -3.1456,
    "longitude": 101.6964,
    "product_id": 5,
    "promo_code": "BATTERIUNEWFD",
    "trade_in": 0
  }
}
```

**Response:**
```json
{
  "subtotal": 450.00,
  "delivery_fee": 15.00,
  "promo_discount": 50.00,
  "trade_in_discount": 0.00,
  "total": 415.00,
  "promo_code": {
    "value": "RM50"
  },
  "promo_details": {
    "discount_type": "fixed",
    "discount_value": 50.00,
    "minimum_order_amount": 200.00
  }
}
```

**Methods:**

```dart
// Extract numeric value from promo_code object
double _extractPromoValue(dynamic promoCode)

// Validate promo code and get calculation
Future<PromoValidationResult> validatePromoCode({...})

// Load products
Future<ProductsResult> loadProducts({...})

// Create order
Future<OrderCreationResult> createOrder({...})
```

---

### 3. [order_confirmation_screen.dart](lib/features/order/screens/order_confirmation_screen.dart) *(Promo integration)*

**Purpose:** Integrates promo code functionality into the checkout flow

**Key Integration Points:**

```dart
// Navigate to promo code screen
GestureDetector(
  onTap: () {
    Navigator.pushNamed(
      context,
      '/order/promo-code',
      arguments: {
        'voucherCode': voucherCode,
        'discountValue': voucherDiscountValue,
        'discountAmount': voucherDiscountAmount,
      },
    ).then((value) {
      if (value != null && value is Map<String, dynamic>) {
        setState(() {
          voucherCode = value['voucherCode'] as String?;
          voucherDiscountValue = value['discountValue'] as String?;
          voucherDiscountAmount = value['discountAmount'] as double?;
        });
        // Validate promo via service
        if (voucherCode != null) {
          _validatePromoCode(voucherCode!);
        }
      }
    });
  },
)

// Validate promo code via service
void _validatePromoCode(String code) {
  context.read<OrderConfirmationCubit>().validatePromoCode(
    latitude: locationState.latitude!,
    longitude: locationState.longitude!,
    productId: productIdForPromo,
    promoCode: code,
    tradeIn: hasTradeIn ? 1 : 0,
    discountAmount: voucherDiscountAmount,
  );
}
```

**Local State Variables:**
```dart
String? voucherCode;              // Applied voucher code
String? voucherDiscountValue;     // Formatted discount ("50.00")
double? voucherDiscountAmount;    // Raw discount amount (50.0)
bool _isValidatingPromo = false;  // Loading state
bool _isPromoValid = false;       // Validation result
String? _promoMessage;            // Success message
String? _promoValidationError;    // Error message
```

---

## 💾 Data Models

### PromoValidationResult

```dart
class PromoValidationResult {
  final bool isValid;
  final String message;
  final OrderCalculation? calculation;
  final PromoDetails? promoDetails;

  PromoValidationResult.valid({
    required this.message,
    this.calculation,
    this.promoDetails,
  }) : isValid = true;

  PromoValidationResult.invalid({
    required this.message,
    this.calculation,
    this.promoDetails,
  }) : isValid = false;
}
```

### PromoDetails

```dart
class PromoDetails {
  final String? discountType;           // "fixed" or "percentage"
  final double? discountValue;          // Discount amount/percentage
  final double? minimumOrderAmount;     // Min order for promo validity

  PromoDetails({
    this.discountType,
    this.discountValue,
    this.minimumOrderAmount,
  });

  factory PromoDetails.fromJson(Map<String, dynamic> json)
  Map<String, dynamic> toJson()
}
```

### OrderCalculation (used in validation)

```dart
class OrderCalculation {
  final double subtotal;
  final double deliveryFee;
  final double promoDiscount;           // Discount from promo code
  final double tradeInDiscount;
  final double total;                   // Final amount after discounts
  final String? promoCode;
  final bool isPromoValid;
}
```

---

## 🔄 Data Flow Examples

### Example 1: User Applies Promo Code in PromoCodeScreen

```
User enters "BATTERIUNEWFD" in voucher input field
    ↓
User taps "Apply" button
    ↓
_applyVoucher() is called
    ↓
Direct HTTP POST to /api/partners/v2/promo_codes/validate
  Body: { "code": "BATTERIUNEWFD" }
    ↓
API returns 200 with discount data:
  {
    "promo_code": { "value": "RM50" },
    "promo_discount": 50.00,
    "discount": 50.00
  }
    ↓
_extractDiscountAmount() extracts: 50.0
    ↓
setState() updates local state:
  _appliedVoucher = "BATTERIUNEWFD"
  _discountAmount = 50.0
  _discountValue = "50.00"
  _isPromoValid = true
    ↓
UI updates:
  ✅ Green "Remove" button appears
  ✅ Success message: "You've applied RM50.00 OFF!"
  ✅ Success illustration displays
  ✅ "Voucher applied!" text shows
    ↓
User taps back button
    ↓
Navigator.pop(context, {
  'voucherCode': 'BATTERIUNEWFD',
  'discountValue': '50.00',
  'discountAmount': 50.0
})
    ↓
OrderConfirmationScreen receives data
```

### Example 2: PromoValidation via OrderConfirmationService

```
OrderConfirmationScreen receives promo code from PromoCodeScreen
    ↓
Calls: _validatePromoCode(voucherCode)
    ↓
OrderConfirmationCubit.validatePromoCode(
  latitude: -3.1456,
  longitude: 101.6964,
  productId: 5,
  promoCode: "BATTERIUNEWFD",
  tradeIn: 0,
  discountAmount: 50.0
)
    ↓
Cubit emits: PromoValidationLoading()
    ↓
Service calls: OrderService.calculateOrder()
    ↓
API POST /api/partners/v2/orders/calculate
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
API returns 200 with calculation:
  {
    "subtotal": 450.00,
    "delivery_fee": 15.00,
    "promo_discount": 50.00,
    "total": 415.00
  }
    ↓
Service extracts discount: 50.0
    ↓
Service creates OrderCalculation object
    ↓
Service creates PromoValidationResult.valid(
  message: "Promo code applied! Save RM 50.00",
  calculation: OrderCalculation(...),
  promoDetails: PromoDetails(...)
)
    ↓
Cubit emits: PromoValidationSuccess(
  message: "Promo code applied! Save RM 50.00",
  discountAmount: 50.0,
  calculation: OrderCalculation(...)
)
    ↓
OrderConfirmationScreen listens to state
    ↓
UI updates to show:
  ✅ Green success message
  ✅ Promo code chip with checkmark
  ✅ Updated order total: RM415.00 (was RM465.00)
  ✅ Discount breakdown: -RM50.00
```

### Example 3: User Removes Promo Code

```
User taps "Remove" button on applied voucher
    ↓
_showRemoveVoucherDialog() shows confirmation
    ↓
User confirms "Remove Voucher"
    ↓
setState() clears state:
  _appliedVoucher = null
  _discountAmount = null
  _discountValue = null
  _isPromoValid = false
  _voucherController.clear()
    ↓
UI updates:
  ✅ Input field re-enabled
  ✅ "Apply" button visible again
  ✅ Empty state illustration displays
  ✅ "No Voucher" message shows
    ↓
User taps back button
    ↓
Navigator.pop(context)  // No return data
    ↓
OrderConfirmationScreen receives null
    ↓
setState() clears voucher values:
  voucherCode = null
  voucherDiscountAmount = null
  voucherDiscountValue = null
    ↓
_calculateOrder() recalculates without promo
    ↓
UI updates with original total (no discount)
```

---

## 🎨 UI Components

### Voucher Input Section

```
┌─────────────────────────────────────────┐
│  [Voucher Code]    [Apply / Remove]    │
│  (Green border, 28px radius)            │
│                                         │
│  You've applied RM50.00 OFF!           │
│  (Right-aligned, grey text)             │
└─────────────────────────────────────────┘
```

### Success State

```
┌─────────────────────────────────────────┐
│                                         │
│        [Voucher Illustration]           │
│                 (200h)                  │
│                                         │
│        Voucher applied!                │
│   Enjoy your discount with this voucher│
│                                         │
└─────────────────────────────────────────┘
```

### Empty State

```
┌─────────────────────────────────────────┐
│                                         │
│    [No Voucher Illustration]            │
│             (200h)                      │
│                                         │
│          No Voucher                    │
│    Oops! No voucher applied yet        │
│                                         │
└─────────────────────────────────────────┘
```

### Order Confirmation Integration

```
┌──────────────────────────────────────────┐
│  🎫 Voucher Code              [→]        │
│                                          │
│  ✅ BATTERIUNEWFD                       │
│  Promo code applied! Save RM 50.00      │
└──────────────────────────────────────────┘
```

---

## 🎯 Key Features

| Feature | Implementation | Notes |
|---------|----------------|-------|
| **Code Validation** | Direct API POST to promo_codes/validate | Returns discount amount |
| **Discount Extraction** | Robust multi-format extraction | Handles 5+ response formats |
| **State Management** | StatefulWidget with local variables | No BLoC pattern for this screen |
| **Success/Error UI** | Different illustrations & messages | Visual feedback for user actions |
| **Remove Confirmation** | BTConfirmDialog with custom styling | Prevents accidental removal |
| **Currency Display** | Formatted as "RM50.00" | Consistent formatting |
| **Integration** | Returns Map to parent screen | Parent validates via service |
| **Service Validation** | Additional validation via order calculation | Double-checks discount amount |
| **Input Transformation** | Auto-capitalization of code | USER INPUT → "BATTERIUNEWFD" |

---

## 🔐 Error Handling

### Invalid Code
```dart
if (response.statusCode == 200) {
  // Valid code - process discount
} else {
  // Invalid code - show error snackbar
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text('Invalid voucher code'),
      backgroundColor: Colors.red,
    ),
  );
}
```

### Network Error
```dart
try {
  // API call
} catch (e, stackTrace) {
  // Show error snackbar with exception message
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text('Error validating voucher: $e'),
      backgroundColor: Colors.red,
    ),
  );
}
```

### Discount Extraction Failure
```dart
// Multiple fallback checks ensure we get discount from some field
// Returns 0.0 as last resort (allows screen to proceed with no discount)
```

---

## 📊 Discount Extraction Logic

The `_extractDiscountAmount()` method supports multiple API response formats:

```
Priority order:
  1. data['promo_code']['value']      → "RM50" → extract "50"
  2. data['promo_discount']            → 50.0
  3. data['discount']                  → 50.0
  4. data['discount_amount']           → 50.0
  5. data['amount']                    → 50.0
  6. Fallback: 0.0 (no discount)
```

**Example Extraction:**
```dart
final input = "RM50";
final numericValue = input.replaceAll(RegExp(r'[^\d.]'), '');  // "50"
final amount = double.tryParse(numericValue) ?? 0.0;           // 50.0
```

---

## 🧪 Testing Examples

### Unit Test - Discount Extraction

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('PromoCodeScreen Discount Extraction', () {
    test('extracts discount from promo_code.value', () {
      const data = {
        'promo_code': {
          'value': 'RM50'
        }
      };
      
      final screen = _PromoCodeScreenState();
      final discount = screen._extractDiscountAmount(data);
      
      expect(discount, 50.0);
    });

    test('falls back to promo_discount field', () {
      const data = {
        'promo_discount': 75.0
      };
      
      final screen = _PromoCodeScreenState();
      final discount = screen._extractDiscountAmount(data);
      
      expect(discount, 75.0);
    });

    test('returns 0.0 when no discount found', () {
      const data = {};
      
      final screen = _PromoCodeScreenState();
      final discount = screen._extractDiscountAmount(data);
      
      expect(discount, 0.0);
    });
  });
}
```

### Widget Test - Promo Code Application

```dart
void main() {
  testWidgets('PromoCodeScreen applies voucher code',
    (WidgetTester tester) async {
    // Arrange
    await tester.pumpWidget(
      MaterialApp(
        home: PromoCodeScreen(),
      ),
    );

    // Act - Enter voucher code
    await tester.enterText(
      find.byType(TextField),
      'BATTERIUNEWFD',
    );

    // Act - Tap Apply button
    await tester.tap(find.text('Apply'));
    await tester.pumpAndSettle();

    // Assert - Success message shows
    expect(find.text("You've applied RM50.00 OFF!"), findsOneWidget);
    expect(find.text('Remove'), findsOneWidget);
    expect(find.text('Apply'), findsNothing);
  });
}
```

---

## ⚠️ Important Notes

1. **Direct HTTP Calls** - PromoCodeScreen makes direct HTTP calls, not via BLoC
2. **Local State Management** - Uses StatefulWidget for local state tracking
3. **Response Format Flexibility** - Handles multiple discount field names
4. **Integration Pattern** - Returns Map to parent, parent validates via service
5. **Auto-Capitalization** - Codes automatically converted to uppercase
6. **No Input Validation** - Any code sent to API for validation
7. **Multiple Response Formats** - API can return discount in different formats
8. **Graceful Fallbacks** - Falls back to 0.0 discount if extraction fails
9. **Illustration Assets** - Uses `Voucher.png` and `no_voucher.png` images
10. **Currency Fixed** - Always shows RM currency (hardcoded as 'RM')

---

## 🚀 Integration Points

### 1. Navigation from OrderConfirmationScreen
```dart
Navigator.pushNamed(
  context,
  '/order/promo-code',
  arguments: {
    'voucherCode': voucherCode,
    'discountValue': voucherDiscountValue,
    'discountAmount': voucherDiscountAmount,
  },
)
```

### 2. Receiving Result
```dart
.then((value) {
  if (value != null && value is Map<String, dynamic>) {
    // Update order confirmation with voucher data
    // Validate via service
  }
})
```

### 3. Service Validation
```dart
context.read<OrderConfirmationCubit>().validatePromoCode(
  promoCode: voucherCode,
  discountAmount: voucherDiscountAmount,
)
```

---

## 🔗 Related Endpoints

| Endpoint | Method | Purpose | Used By |
|----------|--------|---------|---------|
| `/api/partners/v2/promo_codes/validate` | POST | Quick validation | PromoCodeScreen |
| `/api/partners/v2/orders/calculate` | POST | Full calculation & validation | OrderConfirmationService |
| `/api/partners/v2/orders` | POST | Create order with promo | OrderService |

---

## 📚 Related Documentation

- [order_confirmation_screen.dart](lib/features/order/screens/order_confirmation_screen.dart) - Parent screen
- [order_confirmation_service.dart](lib/core/services/order_confirmation_service.dart) - Service validation
- [order_service.dart](lib/core/services/order_service.dart) - Order API client
- `ORDER_CONFIRMATION_ARCHITECTURE.md` - Order checkout flow
- `PRODUCT_SCREEN_ARCHITECTURE.md` - Product selection
- `PAYMENT_REDIRECT_ARCHITECTURE.md` - Payment flow

---

## 🎓 Architecture Highlights

### Strengths
✅ Simple, focused UI component  
✅ Robust discount extraction from multiple formats  
✅ Clear return value pattern  
✅ Graceful error handling  
✅ Visual feedback for user actions  
✅ Confirmation dialog for destructive actions  

### Considerations
⚠️ No BLoC pattern for this screen (could be refactored)  
⚠️ Direct HTTP calls from UI (could use service layer)  
⚠️ Hardcoded API tokens in screen  
⚠️ Hardcoded currency (RM)  
⚠️ Asset dependencies (illustrations)  

### Future Enhancements
- Refactor to use BLoC pattern like other screens
- Extract HTTP logic to service layer
- Support multiple currencies
- Add promo code history/suggestions
- Real-time code validation with debouncing
- Batch apply multiple promo codes
- Promo code expiry date display
- Terms and conditions for promo
- Minimum order validation
