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

### 1. [promo_code_screen.dart](lib/features/order/screens/promo_code_screen.dart) *(514 lines)*

**Purpose:** Dedicated UI screen for voucher code validation and application

**Contains:**
- `PromoCodeScreen` - StatefulWidget that receives initial voucher code, discount value, amount, and discount type
- `_PromoCodeScreenState` - State management with local variables
- `_applyVoucher()` - Direct HTTP API call to validate voucher
- `_extractDiscountInfo()` - Robust discount extraction from multiple response formats with type detection
- `_showRemoveVoucherDialog()` - Confirmation dialog to remove voucher
- UI widgets for input, success/empty states, and illustrations

**Key Responsibilities:**
- Manages local state: `_appliedVoucher`, `_discountAmount`, `_discountValue`, `_discountType`, `_discountCurrency`, `_isPromoValid`
- Validates voucher codes against API
- Extracts discount amount and type from various response formats
- Supports both fixed and percentage-based discounts
- Displays success/failure states with illustrations
- Returns applied voucher data including discount type back to parent screen
- Handles voucher removal with confirmation

**State Properties:**
```dart
late final TextEditingController _voucherController;
String? _appliedVoucher;           // Applied voucher code
double? _discountAmount;           // Raw discount amount (e.g., 50.0)
String? _discountCurrency;         // Currency (RM or empty for percentage)
String? _discountValue;            // Formatted discount (e.g., "50.00")
String? _discountType;             // "fixed" or "percentage"
bool _isPromoValid;                // Is voucher valid?
```

**Key Methods:**

```dart
// Apply/validate voucher code
Future<void> _applyVoucher() {
  // 1. Send POST request to /api/partners/v2/promo_codes/validate
  // 2. Parse response for discount and type
  // 3. Extract discount info (amount + type)
  // 4. Update local state with type-aware currency
  // 5. Update UI with success/error and formatted discount message
  // 6. Show SnackBar with custom styling on error
}

// Extract discount amount and type from various API response formats
Map<String, dynamic> _extractDiscountInfo(Map<String, dynamic> data) {
  // Extract discount type from:
  //   1. data['promo_code']['value_type'] (newer format)
  //   2. data['promo_details']['discount_type'] (fallback)
  //   3. Default to 'fixed' if not found
  
  // Try: data['promo_code']['value'] → extract numeric
  // Fallback: data['promo_discount']
  // Fallback: data['discount']
  // Fallback: data['discount_amount']
  // Fallback: data['amount']
  // Fallback: 0.0
  
  // Returns: {'amount': double, 'type': String}
}

// Show confirmation dialog for removing voucher
void _showRemoveVoucherDialog() {
  // Display BTConfirmDialog with custom styling
  // Clear all state on confirmation
  // Show success message with animation
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
    "value": "RM50",
    "value_type": "fixed"
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

**Supported Discount Types:**
- `"fixed"` - Fixed amount discount (e.g., RM50 off)
- `"percentage"` - Percentage-based discount (e.g., 10% off)

**Constructor Parameters:**
```dart
const PromoCodeScreen({
  Key? key,
  String? initialVoucherCode,      // Pre-filled voucher code
  String? initialDiscountValue,    // Pre-filled discount (e.g., "50.00")
  double? initialDiscountAmount,   // Pre-filled discount amount (e.g., 50.0)
  String? initialDiscountType,     // Pre-filled type: "fixed" or "percentage"
})
```

**Navigation:**
- **Route**: `/order/promo-code`
- **Arguments**: Optional initial voucher data
- **Return**: `Map<String, dynamic>` with voucher code, discount, and type
  ```dart
  {
    'voucherCode': String,         // "BATTERIUNEWFD"
    'discountValue': String,       // "50.00" (formatted)
    'discountAmount': double,      // 50.0 (raw amount)
    'discountType': String         // "fixed" or "percentage"
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

### Discount Info Return Object

```dart
// Returned by _extractDiscountInfo()
Map<String, dynamic> {
  'amount': double,     // Extracted numeric amount (e.g., 50.0)
  'type': String        // "fixed" or "percentage"
}
```

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

### Example 1: User Applies Fixed Promo Code in PromoCodeScreen

```
User enters "BATTERIUNEWFD" in voucher input field
    ↓
User taps "Apply" button or presses Enter
    ↓
_applyVoucher() is called
    ↓
Direct HTTP POST to /api/partners/v2/promo_codes/validate
  Body: { "code": "BATTERIUNEWFD" }
    ↓
API returns 200 with discount data:
  {
    "promo_code": {
      "value": "RM50",
      "value_type": "fixed"
    },
    "promo_discount": 50.00
  }
    ↓
_extractDiscountInfo() extracts:
  - Discount type: "fixed" (from promo_code.value_type)
  - Discount amount: 50.0 (from promo_code.value or promo_discount)
    ↓
setState() updates local state:
  _appliedVoucher = "BATTERIUNEWFD"
  _discountAmount = 50.0
  _discountValue = "50.00" (formatted)
  _discountType = "fixed"
  _discountCurrency = "RM" (RM for fixed, empty for percentage)
  _isPromoValid = true (discountAmount > 0)
    ↓
UI updates:
  ✅ Green "Remove" button appears
  ✅ Success message: "You've applied RM50.00 OFF!"
  ✅ TextField disabled
  ✅ Success illustration (Voucher.png) displays
  ✅ "Voucher applied!" text shows
    ↓
User taps back button
    ↓
Navigator.pop(context, {
  'voucherCode': 'BATTERIUNEWFD',
  'discountValue': '50.00',
  'discountAmount': 50.0,
  'discountType': 'fixed'
})
    ↓
OrderConfirmationScreen receives and validates via service
```

### Example 1b: User Applies Percentage Promo Code

```
User enters "SAVE10" in voucher input field
    ↓
API returns:
  {
    "promo_code": {
      "value": "10",
      "value_type": "percentage"
    },
    "promo_discount": 10.00
  }
    ↓
_extractDiscountInfo() extracts:
  - Discount type: "percentage" (from value_type)
  - Discount amount: 10.0
    ↓
setState() updates:
  _discountType = "percentage"
  _discountValue = "10.00"
  _discountCurrency = "" (empty for percentage)
    ↓
UI displays message: "You've applied 10% OFF!"
  (Note: Currency RM is NOT shown for percentages)
    ↓
Returns: {
  'discountType': 'percentage',
  'discountValue': '10.00',
  'discountAmount': 10.0
}
```
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
_showRemoveVoucherDialog() shows BTConfirmDialog:
  Title: "Do you want to remove this voucher?"
  Buttons: [Remove Voucher] [Cancel]
    ↓
User confirms "Remove Voucher"
    ↓
setState() clears all state:
  _appliedVoucher = null
  _discountAmount = null
  _discountValue = null
  _discountType = null
  _isPromoValid = false
  _voucherController.clear()
  _discountCurrency = null
    ↓
UI updates:
  ✅ Input field re-enabled and cleared
  ✅ "Apply" button visible again
  ✅ Empty state illustration displays (no_voucher.png)
  ✅ "No Voucher" message shows
  ✅ "Oops! No voucher applied yet" description
    ↓
User taps back button
    ↓
Navigator.pop(context)  // No return data (null)
    ↓
OrderConfirmationScreen receives null
    ↓
setState() clears voucher values:
  voucherCode = null
  voucherDiscountAmount = null
  voucherDiscountValue = null
  voucherDiscountType = null
    ↓
_calculateOrder() recalculates without promo
    ↓
UI updates with original total (no discount)
```

---

## 🎨 UI Components

### Voucher Input Section

```
┌─────────────────────────────────────────────────────────────────┐
│  [Voucher Code]    [Apply / Remove]                            │
│  (Bateriku color border, 28px radius)                          │
│  (TextField disabled when applied)                             │
│                                                                 │
│  You've applied RM50.00 OFF!                                   │
│  or                                                             │
│  You've applied 10% OFF!                                        │
│  (Right-aligned, B8B8B8 color text)                            │
└─────────────────────────────────────────────────────────────────┘
```

**Interactive States:**
- **Empty state**: TextField enabled, "Apply" text button visible
- **Applied state**: TextField disabled, "Remove" text button visible (green #AEEA00)
- **Submit action**: Can press Enter when field enabled
- **Focus**: Auto-capitalize to uppercase while typing

### Success State

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│      [Voucher.png Illustration]                                │
│           height: 200                                          │
│      (FallBack: Gift Card Icon)                                │
│                                                                 │
│        Voucher applied!                                        │
│        (fontSize: 24, bold)                                    │
│                                                                 │
│   Enjoy your discount with this voucher                        │
│        (fontSize: 16, dark50)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Empty State

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   [no_voucher.png Illustration]                                │
│           height: 200                                          │
│   (FallBack: Custom Person + Help Icon)                        │
│                                                                 │
│          No Voucher                                            │
│        (fontSize: 24, bold)                                    │
│                                                                 │
│    Oops! No voucher applied yet                                │
│        (fontSize: 16, dark50)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Illustration Error Handling:**
- Uses `errorBuilder` to show fallback icons if assets not found
- Success state fallback: `Icons.card_giftcard` (100px, Bateriku color)
- Empty state fallback: Custom layout with person icon + question marks

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
| **Code Validation** | Direct API POST to promo_codes/validate | Returns discount amount & type |
| **Discount Type Detection** | Extracts from `promo_code.value_type` or `promo_details.discount_type` | Supports "fixed" and "percentage" |
| **Discount Extraction** | Robust multi-format extraction | Handles 5+ response formats |
| **State Management** | StatefulWidget with local variables | No BLoC pattern for this screen |
| **Initial Values** | Constructor accepts pre-filled voucher data | Supports editing existing vouchers |
| **Success/Error UI** | Different illustrations & messages | Visual feedback for user actions |
| **Remove Confirmation** | BTConfirmDialog with custom styling | Prevents accidental removal |
| **Currency Display** | Formatted as "RM50.00" for fixed, "10%" for percentage | Type-aware formatting |
| **Integration** | Returns Map with discount type to parent | Parent validates via service |
| **Service Validation** | Additional validation via order calculation | Double-checks discount amount |
| **Input Transformation** | Auto-capitalization to uppercase | USER INPUT → "BATTERIUNEWFD" |
| **Keyboard Submit** | Enter key triggers apply action | Improves UX |
| **Error Handling** | Custom styled SnackBar (red #E53935, float) | Clear error messaging |
| **Debug Logging** | Extensive logging with emoji prefixes | Helps troubleshooting |
| **Asset Fallbacks** | Icon fallbacks if illustrations missing | App doesn't crash without assets |

---

## 🔐 Error Handling

### Invalid Code
```dart
if (response.statusCode == 200) {
  // Valid code - process discount
  final discountInfo = _extractDiscountInfo(data);
  // Update state
} else {
  // Invalid code - show styled error snackbar
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: const Text(
        'Invalid voucher code',
        style: TextStyle(
          color: Colors.white,
          fontSize: 16,
          fontWeight: FontWeight.w500,
        ),
      ),
      backgroundColor: const Color(0xFFE53935),  // Red
      duration: const Duration(seconds: 3),
      behavior: SnackBarBehavior.floating,
      margin: const EdgeInsets.all(16),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
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
  debugPrint('💥 [Voucher] Exception occurred: $e');
  debugPrint('💥 [Voucher] Stack trace: $stackTrace');
  
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text(
        'Error validating voucher: $e',
        style: const TextStyle(
          color: Colors.white,
          fontSize: 16,
          fontWeight: FontWeight.w500,
        ),
      ),
      backgroundColor: const Color(0xFFE53935),
      duration: const Duration(seconds: 3),
      behavior: SnackBarBehavior.floating,
      margin: const EdgeInsets.all(16),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
    ),
  );
}
```

### Discount Extraction Failure
```dart
// Multiple fallback checks ensure we get discount from some field
// Returns 0.0 as last resort (allows screen to proceed with no discount)
// Discount type defaults to "fixed" if not found
// Returns {'amount': 0.0, 'type': 'fixed'}
```

### Empty Code Submission
```dart
if (_voucherController.text.isEmpty) {
  debugPrint('⚠️ [Voucher] Voucher code is empty');
  // Silent fail - no API call made
}
```

---

## 📊 Discount Extraction Logic

The `_extractDiscountInfo()` method supports multiple API response formats and returns both amount AND type:

```
┌─ Extract Discount Type (Priority) ──────────────────────────────┐
│ 1. data['promo_code']['value_type']                             │
│    └─ Modern format: returns "fixed" or "percentage"            │
│                                                                 │
│ 2. data['promo_details']['discount_type']                       │
│    └─ Fallback format: returns "fixed" or "percentage"          │
│                                                                 │
│ 3. Default: "fixed" (if type not found)                        │
└─────────────────────────────────────────────────────────────────┘

┌─ Extract Discount Amount (Priority) ────────────────────────────┐
│ 1. data['promo_code']['value']                                  │
│    └─ String like "RM50" → extract numeric "50" → 50.0         │
│                                                                 │
│ 2. data['promo_discount']                                       │
│    └─ Direct numeric: 50.0                                      │
│                                                                 │
│ 3. data['discount']                                             │
│    └─ Direct numeric: 50.0                                      │
│                                                                 │
│ 4. data['discount_amount']                                      │
│    └─ Direct numeric: 50.0                                      │
│                                                                 │
│ 5. data['amount']                                               │
│    └─ Direct numeric: 50.0                                      │
│                                                                 │
│ 6. Fallback: 0.0 (no discount found)                           │
└─────────────────────────────────────────────────────────────────┘
```

**Example Extraction - Fixed Discount:**
```dart
final input = {"promo_code": {"value": "RM50", "value_type": "fixed"}};
final numericValue = "RM50".replaceAll(RegExp(r'[^\d.]'), '');  // "50"
final amount = double.tryParse(numericValue) ?? 0.0;             // 50.0
final type = "fixed";
return {'amount': 50.0, 'type': 'fixed'};
```

**Example Extraction - Percentage Discount:**
```dart
final input = {"promo_code": {"value": "10", "value_type": "percentage"}};
final numericValue = "10".replaceAll(RegExp(r'[^\d.]'), '');    // "10"
final amount = double.tryParse(numericValue) ?? 0.0;             // 10.0
final type = "percentage";
return {'amount': 10.0, 'type': 'percentage'};
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
2. **Local State Management** - Uses StatefulWidget with multiple state variables
3. **Response Format Flexibility** - Handles multiple discount field names and type formats
4. **Discount Type Support** - Distinguishes between "fixed" (RM) and "percentage" (%) discounts
5. **Type-Aware Currency** - Shows "RM" for fixed, empty for percentage
6. **Integration Pattern** - Returns Map with discount type to parent, parent validates via service
7. **Auto-Capitalization** - Codes automatically converted to uppercase during input
8. **No Input Validation** - Any code sent to API for validation (server validates)
9. **Graceful Fallbacks** - Falls back to 0.0 discount and "fixed" type if extraction fails
10. **Illustration Assets** - Uses `Voucher.png` and `no_voucher.png` images with icon fallbacks
11. **Error SnackBars** - Styled with red background (#E53935), floating, 3s duration, 12px radius
12. **TextField Disabled When Applied** - Prevents editing after voucher applied
13. **Enter Key Support** - Can submit code with keyboard Enter key
14. **Extensive Logging** - All operations logged with emoji prefixes for debugging
15. **Initial Voucher Support** - Constructor accepts pre-filled voucher data for editing scenarios

---

## 🚀 Integration Points

### 1. Navigation from OrderConfirmationScreen
```dart
Navigator.pushNamed(
  context,
  '/order/promo-code',
  arguments: {
    'initialVoucherCode': voucherCode,
    'initialDiscountValue': voucherDiscountValue,
    'initialDiscountAmount': voucherDiscountAmount,
    'initialDiscountType': voucherDiscountType,
  },
)
```

### 2. Receiving Result
```dart
.then((value) {
  if (value != null && value is Map<String, dynamic>) {
    setState(() {
      voucherCode = value['voucherCode'] as String?;
      voucherDiscountValue = value['discountValue'] as String?;
      voucherDiscountAmount = value['discountAmount'] as double?;
      voucherDiscountType = value['discountType'] as String?;
    });
    // Validate via service with discount type
    if (voucherCode != null) {
      _validatePromoCode(voucherCode!);
    }
  }
})
```

### 3. Service Validation
```dart
context.read<OrderConfirmationCubit>().validatePromoCode(
  latitude: locationState.latitude!,
  longitude: locationState.longitude!,
  productId: productIdForPromo,
  promoCode: code,
  tradeIn: hasTradeIn ? 1 : 0,
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
✅ Support for fixed and percentage discounts  
✅ Clear return value pattern with discount type  
✅ Graceful error handling with styled SnackBars  
✅ Visual feedback for user actions  
✅ Confirmation dialog for destructive actions  
✅ Excellent debugging with emoji-prefixed logs  
✅ Icon fallbacks for missing assets  
✅ Type-aware currency display  

### Considerations
⚠️ No BLoC pattern for this screen (could be refactored)  
⚠️ Direct HTTP calls from UI (could use service layer)  
⚠️ Hardcoded API tokens in screen  
⚠️ Hardcoded base URL  
⚠️ Asset dependencies (illustrations) - mitigated with icon fallbacks  
⚠️ No validation for minimum order amount (handled by parent service)  

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
