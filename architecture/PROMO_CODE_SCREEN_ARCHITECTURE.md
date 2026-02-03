# Promo Code Screen - Architecture & Code Explanation

## 📋 Overview

The **Promo Code Screen** (also called Voucher Code Screen) is a dedicated modal screen where customers apply discount voucher/promo codes to their battery purchase orders. It handles promo code validation, discount extraction from API responses, and displays the discount amount. The screen is accessed from the Order Confirmation Screen and returns the applied code and discount details back to the parent screen.

## 🆕 Latest Updates (February 2026)

### New Features & Changes

**Screen Behavior:**
- ✅ **Auto-redirect on success** - Screen automatically closes after 1 second with valid discount (discount > 0)
- ✅ **Error message display** - Shows "Invalid Code. Try Again" below input field on validation failure
- ✅ **Enhanced error state management** - Separate `_errorMessage` field for input field error display
- ✅ **Smart back button** - Returns data only if voucher applied and valid, otherwise returns null

**Discount Type System:**
- ✅ **Type-aware currency** - Empty string for percentage, "RM" for fixed
- ✅ **Multiple type sources** - Checks `promo_code.value_type` then `promo_details.discount_type`
- ✅ **Default type** - Defaults to "fixed" if type not found in response

**API & Response Handling:**
- ✅ **HTTP direct calls** - Uses `EnvConfig.partnersApiToken` for authorization
- ✅ **Enhanced logging** - Comprehensive emoji-prefixed debug logs throughout
- ✅ **Better error messages** - Displays "Invalid Code. Try Again" instead of generic errors
- ✅ **State error clearing** - Clears `_errorMessage` on successful apply

**UI/UX Improvements:**
- ✅ **Error message field** - Shows validation errors directly below input
- ✅ **Auto-redirect** - 1-second delay before closing on successful validation
- ✅ **Smart navigation** - Back button returns data only if valid promo applied
- ✅ **Mounted check** - Ensures context still valid before pop navigation

---

## 🏗️ Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│  UI Layer                                           │
│  [PromoCodeScreen] (520 lines)                      │
│  - Voucher input field with error display           │
│  - Apply/Remove buttons                             │
│  - Success/Empty states                             │
│  - Discount display                                 │
│  - Auto-redirect on valid discount                  │
└────────────────┬────────────────────────────────────┘
                 │ uses
┌────────────────▼────────────────────────────────────┐
│  HTTP Client (Direct)                               │
│  [PromoCodeScreen._applyVoucher()]                  │
│  - Direct API call to validate endpoint             │
│  - EnvConfig.partnersApiToken for auth              │
│  - No BLoC pattern (StatefulWidget)                │
│  - Local state management with error tracking       │
└────────────────┬────────────────────────────────────┘
                 │ also used by
┌────────────────▼────────────────────────────────────┐
│  Service Layer (for Order Confirmation)             │
│  [OrderConfirmationService]                         │
│  - validatePromoCode()                             │
│  - Discount extraction & normalization              │
│  - Multiple response format support                 │
│  - PromoDetails extraction                          │
└────────────────┬────────────────────────────────────┘
                 │ calls
┌────────────────▼────────────────────────────────────┐
│  API Endpoints                                      │
│  /api/partners/v2/promo_codes/validate (Direct)    │
│  /api/partners/v2/orders/calculate (Service)       │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Key Files & Their Responsibilities

### 1. [promo_code_screen.dart](lib/features/order/screens/promo_code_screen.dart) *(520 lines)*

**Purpose:** Dedicated UI screen for voucher code validation and application

**Architecture:**
- `PromoCodeScreen` - StatefulWidget with constructor parameters for initial voucher data
- `_PromoCodeScreenState` - State management with local variables for voucher/discount tracking
- Error message field for validation feedback
- Auto-redirect logic on successful validation

**Key State Variables:**
```dart
late final TextEditingController _voucherController;
String? _appliedVoucher;           // Applied voucher code (e.g., "BATTERIUNEWFD")
double? _discountAmount;           // Raw discount amount (e.g., 50.0)
String? _discountCurrency;         // Currency: "RM" for fixed, "" for percentage
String? _discountValue;            // Formatted discount (e.g., "50.00")
String? _discountType;             // "fixed" or "percentage"
bool _isPromoValid;                // Is discount > 0?
String? _errorMessage;             // NEW: Error message for input field display
```

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

**Key Methods:**

```dart
// Validate and apply voucher code
Future<void> _applyVoucher() {
  // 1. Validate code is not empty
  // 2. Convert code to uppercase
  // 3. Send POST request to /api/partners/v2/promo_codes/validate
  // 4. Parse response for discount info
  // 5. Extract discount amount and type via _extractDiscountInfo()
  // 6. Update state with type-aware currency
  // 7. Auto-redirect after 1 second if discount > 0
  // 8. Show error message if validation fails
}

// Extract discount amount and type from various API response formats
Map<String, dynamic> _extractDiscountInfo(Map<String, dynamic> data) {
  // Extract type from: promo_code.value_type → promo_details.discount_type → default "fixed"
  // Extract amount from: promo_code.value → promo_discount → discount → discount_amount → amount → 0.0
  // Returns: {'amount': double, 'type': String}
}

// Show remove confirmation dialog
void _showRemoveVoucherDialog() {
  // Display BTConfirmDialog with confirmation
  // Clear all state on confirmation
}

// Smart back button navigation
onPressed: () {
  if (_appliedVoucher != null && _isPromoValid) {
    // Return both code and discount value
    Navigator.pop(context, {
      'voucherCode': _appliedVoucher,
      'discountValue': _discountValue,
      'discountAmount': _discountAmount,
      'discountType': _discountType,
    });
  } else {
    // No valid promo, return null
    Navigator.pop(context);
  }
}
```

**State Initialization:**
- Pre-fills controller and state from initial parameters if provided
- Sets `_isPromoValid = true` if initial voucher code provided
- Sets `_discountCurrency = 'RM'` by default

**API Endpoint (Direct):**
- **Endpoint**: `POST /api/partners/v2/promo_codes/validate`
- **Base URL**: From `EnvConfig.batteryPurchaseApiUrl`
- **Auth**: `EnvConfig.partnersApiToken` in Authorization header

**Request Payload:**
```json
{
  "code": "BATTERIUNEWFD"
}
```

**Response Format (supports multiple sources):**
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
- `"fixed"` - Fixed amount discount (e.g., RM50 off) - Shows "RM" currency
- `"percentage"` - Percentage-based discount (e.g., 10% off) - Shows no currency symbol

**Auto-Redirect Behavior:**
- Only redirects if `discountAmount > 0` (valid discount)
- 1-second delay before auto-redirect
- Checks `if (mounted)` before navigation
- Returns complete discount info including type

**Error Handling:**
- Invalid code: Sets `_errorMessage = 'Invalid Code. Try Again'` and displays below input
- Network error: Catches exception and shows same error message
- Empty code: Silent fail, no API call

**Navigation Returns:**
- **Success**: `Map<String, dynamic>` with voucher code, discount value, amount, and type
- **Failure/Cancelled**: `null`

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
  Headers: Authorization: EnvConfig.partnersApiToken
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
  - Discount amount: 50.0 (from promo_code.value → "RM50" → 50)
    ↓
setState() updates local state:
  _appliedVoucher = "BATTERIUNEWFD"
  _discountAmount = 50.0
  _discountValue = "50.00" (formatted)
  _discountType = "fixed"
  _discountCurrency = "RM" (RM for fixed, "" for percentage)
  _isPromoValid = true (discountAmount > 0)
  _errorMessage = null (cleared on success)
    ↓
UI updates:
  ✅ Input field shows "BATTERIUNEWFD"
  ✅ Remove button visible (green)
  ✅ Success message displays (with illustration)
    ↓
[1 second delay for user to see success state]
    ↓
if (mounted) {
  Navigator.pop(context, {
    'voucherCode': 'BATTERIUNEWFD',
    'discountValue': '50.00',
    'discountAmount': 50.0,
    'discountType': 'fixed'
  })
}
    ↓
Screen closes automatically
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
    "promo_discount": 10.0
  }
    ↓
_extractDiscountInfo() extracts:
  - Discount type: "percentage" (from value_type)
  - Discount amount: 10.0
    ↓
setState() updates:
  _discountType = "percentage"
  _discountValue = "10.00"
  _discountCurrency = "" (empty for percentage - no RM shown)
    ↓
UI displays success state:
  ✅ Message shows discount for percentage format
  ✅ No "RM" currency prefix (only for fixed)
    ↓
[1 second delay]
    ↓
Auto-redirect with:
  {
    'voucherCode': 'SAVE10',
    'discountValue': '10.00',
    'discountAmount': 10.0,
    'discountType': 'percentage'
  }
```

### Example 1c: Invalid Promo Code

```
User enters "INVALIDCODE" in voucher input field
    ↓
API returns non-200 status (e.g., 400, 404)
    ↓
setState() updates:
  _errorMessage = 'Invalid Code. Try Again'
  _appliedVoucher = null
  _isPromoValid = false
    ↓
UI displays:
  ✅ Error message shown below input field
  ✅ Empty state illustration still visible
  ✅ Input field remains enabled for retry
  ✅ No auto-redirect
    ↓
User can retry or go back manually
```

### Example 2: Service Layer Validation after PromoCodeScreen Returns

```
PromoCodeScreen auto-redirects and closes after 1 second
    ↓
OrderConfirmationScreen receives return value:
  {
    'voucherCode': 'BATTERIUNEWFD',
    'discountValue': '50.00',
    'discountAmount': 50.0,
    'discountType': 'fixed'
  }
    ↓
OrderConfirmationScreen stores in local state:
  voucherCode = "BATTERIUNEWFD"
  voucherDiscountAmount = 50.0
  voucherDiscountValue = "50.00"
  voucherDiscountType = "fixed"
    ↓
Calls: _validatePromoCode(voucherCode)
    ↓
OrderConfirmationCubit.validatePromoCode(
  latitude: -3.1456,
  longitude: 101.6964,
  productId: 5,
  promoCode: "BATTERIUNEWFD",
  tradeIn: 0,
  discountAmount: 50.0  ← Passed from PromoCodeScreen
)
    ↓
Cubit emits: OrderConfirmationState with status = validating
    ↓
OrderConfirmationService calls: OrderService.calculateOrder()
    ↓
API POST /api/partners/v2/orders/calculate with promo validation
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
    "total": 415.00,
    "promo_code": {
      "value": "RM50",
      "value_type": "fixed"
    }
  }
    ↓
Service validates:
  - Extracted discount (50.0) matches promo code discount
  - Creates PromoDetails with type and value
  - Creates PromoValidationResult.valid()
    ↓
Cubit emits: OrderConfirmationState with status = success
    ↓
OrderConfirmationScreen listens to state
    ↓
UI updates to show:
  ✅ Green success message: "Promo code applied!"
  ✅ Promo code chip displays "BATTERIUNEWFD" with checkmark
  ✅ Discount section shows: "-RM50.00"
  ✅ Updated order total: RM415.00 (was RM465.00)
  ✅ Order calculation is accurate and verified
```

### Example 3: User Removes Promo Code and Goes Back

```
Screen State Before Removal:
  - _appliedVoucher = "BATTERIUNEWFD"
  - _discountAmount = 50.0
  - _isPromoValid = true
  - UI shows green "Remove" button
    ↓
User taps green "Remove" button
    ↓
_showRemoveVoucherDialog() displays BTConfirmDialog:
  Title: "Do you want to remove this voucher?"
  Buttons: [Remove Voucher] [Cancel]
    ↓
User taps "Remove Voucher" button
    ↓
setState() clears all promo state:
  _appliedVoucher = null
  _discountAmount = 0.0
  _discountValue = "0.00"
  _discountType = null
  _discountCurrency = ""
  _isPromoValid = false
  _errorMessage = null
  _voucherController.clear()
    ↓
UI reverts to empty state:
  ✅ Input field re-enabled and cleared
  ✅ "Apply" button visible again (was "Remove")
  ✅ Empty state illustration (no_voucher.png) displays
  ✅ "No Voucher" header text shows
  ✅ "Oops! No voucher applied yet" description shows
    ↓
User taps back button (top-left arrow)
    ↓
Smart Back Button Logic Executes:
  if (_appliedVoucher != null && _isPromoValid) {
    // Has valid promo - return discount data
    Navigator.pop(context, {
      'voucherCode': _appliedVoucher,
      'discountValue': _discountValue,
      'discountAmount': _discountAmount,
      'discountType': _discountType,
    });
  } else {
    // No promo applied - return null
    Navigator.pop(context);
  }
    ↓
Since _appliedVoucher = null:
  Navigator.pop(context)  // Returns null (no promo data)
    ↓
OrderConfirmationScreen receives null in result
    ↓
.then((value) {
  if (value != null && value is Map<String, dynamic>) {
    // Only executed if value is not null
    setState(() { ... });
  }
  // If value is null, this block is skipped
  // Voucher state remains unchanged or cleared
})
    ↓
OrderConfirmationScreen state update:
  voucherCode remains null or gets cleared
  voucherDiscountAmount remains null or gets cleared
  voucherDiscountValue remains null or gets cleared
    ↓
_calculateOrder() recalculates without promo
    ↓
UI updates with original total:
  ✅ Discount section hidden
  ✅ Total shows original price (no discount)
  ✅ Promo code chip removed
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
