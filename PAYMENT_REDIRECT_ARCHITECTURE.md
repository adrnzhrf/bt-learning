# Payment Redirect Screen Architecture

## Overview

The payment system in Bateriku Customer Mobile consists of 4 interconnected screens that handle the complete payment flow from order confirmation to payment result display. This document explains the architecture, flow, and code implementation of each screen.

---

## System Architecture

### Payment Flow Diagram

```
Order Confirmation Screen
    ↓ (User confirms order)
    ↓ API creates order → Returns paymentUrl + redirectUrl
    ↓
Payment WebView Screen (BotaPay Gateway)
    ├─ Loads payment gateway
    ├─ Monitors URL for success/failure patterns
    └─ Intercepts deep links
    ↓
Payment Deep Link Redirecting Screen
    ├─ Extracts parameters from deep link
    └─ Passes to status screen
    ↓
Payment Status Screen
    ├─ Shows processing state (3 sec)
    └─ Displays success/failure result
```

---

## 1. Order Confirmation Screen

**File:** `lib/features/order/screens/order_confirmation_screen.dart`

**Purpose:** Checkout screen where users review order details and initiate payment.

### Key Responsibilities

- Display product selection and pricing
- Collect customer and vehicle information
- Apply voucher/promo codes
- Validate order details
- Initiate payment by calling API

### Order Creation Flow

```dart
// Build redirect URL for deep linking back to app after payment
final redirectUrl = 'myapp://payment-result?orderId=${Uri.encodeComponent(_selectedProductStringId ?? '')}&amount=${Uri.encodeComponent(total.toStringAsFixed(2))}';

// Create order via API
context.read<OrderConfirmationCubit>().createOrder(
  userName: _customerInfo!['name'] as String? ?? '',
  userPhone: _customerInfo!['phone'] as String? ?? '',
  userEmail: _customerInfo!['email'] as String? ?? '',
  latitude: locationState.latitude!,
  longitude: locationState.longitude!,
  address: locationState.address ?? 'Unknown location',
  vehiclePlateNumber: _vehicleInfo?['plateNumber'] as String? ?? '',
  productId: _selectedProductStringId!,
  brandId: _brandId!,
  paymentType: 'billplz',
  tradeIn: hasTradeIn ? 1 : 0,
  promoCode: voucherCode,
  notes: _orderNotes,
  redirectUrl: redirectUrl,
);
```

### On Order Success

```dart
void _handleOrderCreationSuccess(OrderCreationResponse? order) {
  if (order != null &&
      order.paymentUrl != null &&
      order.paymentUrl!.isNotEmpty) {
    _openPaymentInBrowser(order.paymentUrl!);
  } else {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Text('No payment URL received'),
        backgroundColor: Colors.red,
      ),
    );
  }
}

void _openPaymentInBrowser(String paymentUrl) {
  print('Opening payment URL in webview: $paymentUrl');

  // Validate URL
  if (!paymentUrl.startsWith('http://') &&
      !paymentUrl.startsWith('https://')) {
    print('Invalid URL: must start with http:// or https://');
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Text('Invalid payment URL'),
        backgroundColor: Colors.red,
      ),
    );
    return;
  }

  Navigator.of(context).push(
    MaterialPageRoute(
      builder: (context) => PaymentWebViewScreen(paymentUrl: paymentUrl),
    ),
  );
}
```

### Key State Variables

```dart
// Product selection
String? _selectedProductStringId;          // Store original string ID from API
ProductItem? _selectedProductItem;          // Store selected product

// Customer & order info
Map<String, dynamic>? _customerInfo;        // Name, phone, email
Map<String, dynamic>? _vehicleInfo;         // Plate number, etc.
String? _orderNotes;                        // Additional notes

// Pricing calculations
double get subtotal => batteryPrice;
double get deliveryFee => _apiCalculation?.deliveryFee ?? 0.0;
double get tradeInDiscount => hasTradeIn ? 20.00 : 0.0;
double get voucherDiscount => voucherDiscountAmount ?? 0.0;
double get total => _apiCalculation?.total ?? (subtotal + deliveryFee - tradeInDiscount - voucherDiscount);

// Trade-in & voucher
bool hasTradeIn = false;
String? voucherCode;
double? voucherDiscountAmount;
```

---

## 2. Payment WebView Screen

**File:** `lib/features/order/screens/payment_webview_screen.dart`

**Purpose:** In-app WebView that displays the BotaPay payment gateway and handles payment result detection.

### Package Dependencies

```dart
import 'package:flutter_inappwebview/flutter_inappwebview.dart';
```

### Key Features

| Feature | Description |
|---------|-------------|
| **URL Loading** | Uses `InAppWebView` to load payment URL |
| **Deep Link Interception** | Intercepts custom URLs (non-HTTP/HTTPS) |
| **Success Detection** | Detects patterns: `paid=true`, `success=true`, `callback` |
| **Failure Detection** | Detects patterns: `paid=false`, `success=false` |
| **Exit Protection** | Confirms before allowing user to exit |
| **Go Back Support** | Allows back navigation within WebView |

### WebView Creation and Configuration

```dart
InAppWebView(
  initialUrlRequest: URLRequest(url: WebUri(widget.paymentUrl)),
  onWebViewCreated: (controller) {
    print('WebView created');
    _webViewController = controller;
  },
  // ... rest of configuration
)
```

### Deep Link Interception

```dart
shouldOverrideUrlLoading: (controller, navigationAction) async {
  final uri = navigationAction.request.url;
  final urlString = uri?.toString() ?? '';
  print('Should override url loading: $urlString');

  // Intercept custom app deep links (non-http/https) such as myapp://
  if (urlString.isNotEmpty &&
      !(urlString.startsWith('http://') ||
          urlString.startsWith('https://'))) {
    try {
      final parsed = Uri.parse(urlString);
      final params =
          Map<String, dynamic>.from(parsed.queryParameters);

      // push replacement to the redirecting handler so app can handle the params
      if (mounted) {
        Navigator.of(context).pushReplacementNamed(
          '/payment/redirecting',
          arguments: params,
        );
      }
    } catch (e) {
      print('Failed to parse deep link: $e');
    }

    return NavigationActionPolicy.CANCEL;
  }

  return NavigationActionPolicy.ALLOW;
}
```

### Payment Success Detection

```dart
// Check for payment success indicators
if (urlString.contains('paid=true') ||
    urlString.contains('paid%5D=true') ||
    urlString.contains('paid]=true') ||
    urlString.contains('success=true') ||
    urlString.contains('success') ||
    urlString.contains('callback')) {
  print('Payment success detected');
  try {
    final uri = Uri.parse(urlString);
    final params =
        Map<String, dynamic>.from(uri.queryParameters);

    if (mounted) {
      Navigator.of(context).pushReplacementNamed(
        '/payment/status',
        arguments: {
          'status': 'success',
          'orderId': params['orderId']?.toString(),
          'amount': params['amount']?.toString(),
          'paymentMethod':
              params['paymentMethod']?.toString(),
          'serviceType': params['serviceType']?.toString(),
        },
      );
    }
  } catch (e) {
    print('Failed to parse payment success URL: $e');
  }
}
```

### Payment Failure Detection

```dart
// Check for payment failure indicators
if (urlString.contains('paid=false') ||
    urlString.contains('paid%5D=false') ||
    urlString.contains('paid]=false') ||
    urlString.contains('success=false')) {
  print('Payment failed detected');
  try {
    final uri = Uri.parse(urlString);
    final params =
        Map<String, dynamic>.from(uri.queryParameters);

    if (mounted) {
      Navigator.of(context).pushReplacementNamed(
        '/payment/status',
        arguments: {
          'status': 'failed',
          'orderId': params['orderId']?.toString(),
          'amount': params['amount']?.toString(),
          'paymentMethod':
              params['paymentMethod']?.toString(),
          'serviceType': params['serviceType']?.toString(),
        },
      );
    }
  } catch (e) {
    print('Failed to parse payment failure URL: $e');
  }
  return;
}
```

### Exit Confirmation Dialog

```dart
Future<void> _showExitConfirmDialog() async {
  final confirmed = await BTConfirmDialog.confirm(
    context,
    title: 'Exit Payment Gateway?',
    description:
        'Your payment is not yet completed. Are you sure you want to exit?',
    confirmText: 'Exit',
    cancelText: 'Continue',
    icon: Icons.warning_outlined,
  );

  if (confirmed && mounted) {
    Navigator.of(context).pop();
  }
}
```

### Back Navigation Support

```dart
WillPopScope(
  onWillPop: () async {
    if (await _webViewController.canGoBack()) {
      await _webViewController.goBack();
      return false;
    }
    return true;
  },
  child: Scaffold(
    // ... rest of scaffold
  ),
)
```

---

## 3. Payment Redirect Screen

**File:** `lib/features/payment/screens/payment_redirect_screen.dart`

**Purpose:** Displays payment method selection UI (BotaPay interface). Note: This screen is primarily for UI display; actual payment processing happens in the WebView.

### UI Structure

The screen consists of three main sections:

#### Transaction Details Card
```dart
Container(
  padding: const EdgeInsets.all(20),
  child: Column(
    children: [
      Text('Transaction Details'),
      _buildDetailRow('Amount to Pay', 'MYR800.00', isAmount: true),
      _buildDetailRow('Pay to', 'FleetRSA.my'),
      _buildDetailRow('Order ID', '78762876013033'),
      _buildDetailRow('Reference Number', 'FR99783910'),
      _buildDetailRow('Payment Description', 'Fleet Top-up'),
    ],
  ),
)
```

#### Payment Method Card
Expandable sections for:
- Online Banking
- E-Wallet
- Debit/Credit Card

### Online Banking Options

```dart
final List<Map<String, String>> onlineBanks = [
  {'name': 'Maybank2u', 'icon': '🏦'},
  {'name': 'CIMB Clicks', 'icon': '🏦'},
  {'name': 'Public Bank', 'icon': '🏦'},
  {'name': 'RHB Now', 'icon': '🏦'},
  {'name': 'Ambank', 'icon': '🏦'},
  {'name': 'MyBSN', 'icon': '🏦'},
  {'name': 'Bank Rakyat', 'icon': '🏦'},
  {'name': 'UOB', 'icon': '🏦'},
  {'name': 'Affin Bank', 'icon': '🏦'},
  {'name': 'Bank Islam', 'icon': '🏦'},
  {'name': 'HSBC Online', 'icon': '🏦'},
  {'name': 'Standard Chartered Bank', 'icon': '🏦'},
  {'name': 'Kuwait Finance House', 'icon': '🏦'},
  {'name': 'Bank Muamalat', 'icon': '🏦'},
  {'name': 'OCBC Online', 'icon': '🏦'},
  {'name': 'Alliance Bank', 'icon': '🏦'},
  {'name': 'Hong Leong Connect', 'icon': '🏦'},
  {'name': 'Agrobank', 'icon': '🏦'},
];
```

### E-Wallet Options

```dart
final List<Map<String, String>> eWallets = [
  {'name': 'ShopeePay', 'icon': '🛍️'},
  {'name': 'GrabPay', 'icon': '🚗'},
  {'name': 'Setel', 'icon': '⛽'},
  {'name': "Touch 'n Go eWallet", 'icon': '💳'},
  {'name': 'Boost Wallet', 'icon': '🚀'},
];
```

### Proceed to Payment Handler

```dart
onPressed: selectedBank != null
    ? () async {
        // Build deep link to come back to the app after external payment
        final params = {
          'status': 'success',
          'orderId': '78762876013033',
          'amount': 'MYR800.00',
          'paymentMethod': selectedBank ?? 'Online Banking',
          'serviceType': 'batterypurchase',
        };
        final query = params.entries
            .map((e) =>
                '${Uri.encodeComponent(e.key)}=${Uri.encodeComponent(e.value ?? '')}')
            .join('&');
        final deepLink =
            'myapp://payment/redirecting?$query';

        try {
          // Attempt to launch the deep link (this simulates redirect from gateway)
          await launchUrlString(deepLink);
        } catch (_) {
          // ignore - will fallback to local navigation
        }

        // Navigate locally to the redirecting handler (useful for testing / fallback)
        Navigator.pushReplacementNamed(
          context,
          '/payment/redirecting',
          arguments: params,
        );
      }
    : null,
```

### State Management

```dart
class _PaymentRedirectScreenState extends State<PaymentRedirectScreen> {
  String? selectedBank;
  bool isOnlineBankingExpanded = true;
  bool isEwalletExpanded = false;
  bool isCardExpanded = false;
  
  // ... rest of state
}
```

---

## 4. Payment Deep Link Redirecting Screen

**File:** `lib/features/payment/screens/payment_deeplink_redirecting_screen.dart`

**Purpose:** Intermediate screen that handles deep-link parameters from the payment gateway and transitions to the payment status screen.

### Functionality

This screen serves as a bridge between:
- **External payment gateway** (returns via deep link)
- **Payment status screen** (displays result)

### Deep Link Handling

```dart
class PaymentDeepLinkRedirectingScreen extends StatefulWidget {
  const PaymentDeepLinkRedirectingScreen({super.key});

  @override
  State<PaymentDeepLinkRedirectingScreen> createState() =>
      _PaymentDeepLinkRedirectingScreenState();
}

class _PaymentDeepLinkRedirectingScreenState
    extends State<PaymentDeepLinkRedirectingScreen> {
  Map<String, dynamic> _args = {};

  @override
  void initState() {
    super.initState();
    // Delay slightly to allow ModalRoute to be available in didChangeDependencies
    Timer(const Duration(milliseconds: 50), _handleIncoming);
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    final settingsArgs = ModalRoute.of(context)?.settings.arguments;
    if (settingsArgs is Map<String, dynamic>) {
      _args = Map<String, dynamic>.from(settingsArgs);
    } else if (settingsArgs is Map) {
      _args = Map<String, dynamic>.from(settingsArgs as Map);
    }
  }

  void _handleIncoming() {
    final status = (_args['status'] ?? '').toString().toLowerCase();

    // Build arguments for PaymentStatusScreen
    final targetArgs = <String, dynamic>{
      'status': status,
      'orderId': _args['orderId']?.toString(),
      'amount': _args['amount']?.toString(),
      'paymentMethod': _args['paymentMethod']?.toString(),
      'vehicleNumber': _args['vehicleNumber']?.toString(),
      'serviceType': _args['serviceType']?.toString(),
    };

    // Replace this route with the payment status screen
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (!mounted) return;
      Navigator.of(context).pushReplacementNamed(
        '/payment/status',
        arguments: targetArgs,
      );
    });
  }
}
```

### UI

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    backgroundColor: AppColors.bgLightGrey,
    appBar: AppBar(
      backgroundColor: AppColors.bgWhite,
      elevation: 0,
      centerTitle: true,
      title: const Text('Redirecting...'),
    ),
    body: const SafeArea(
      child: Center(
        child: CircularProgressIndicator(),
      ),
    ),
  );
}
```

### Key Points

- Expects route arguments or query parameters with keys: `status`, `orderId`, `amount`, `paymentMethod`, `vehicleNumber`, `serviceType`
- Uses `pushReplacementNamed` to replace itself with the status screen (not adding to navigation stack)
- Small delay (50ms) ensures ModalRoute is available for reading arguments

---

## 5. Payment Status Screen

**File:** `lib/features/payment/screens/payment_status_screen.dart`

**Purpose:** Displays the payment result (success, failure, or processing state) with transaction details and action buttons.

### State Definition

```dart
enum PaymentProcessState { processing, success, failure }

class PaymentStatusScreen extends StatefulWidget {
  final String? vehicleNumber;
  final BTServiceType? serviceType;
  final String? orderId;
  final String? paymentMethod;
  final String? amount;

  const PaymentStatusScreen({
    super.key,
    this.vehicleNumber,
    this.serviceType,
    this.orderId,
    this.paymentMethod,
    this.amount,
  });
}
```

### State Initialization

```dart
void _startProcessing() {
  setState(() {
    _state = PaymentProcessState.processing;
  });

  AppUtils.showGlobalLoading(
    message: 'Redirecting',
    description:
        'We are securely redirecting you to your bank. Closing or refreshing the window may disrupt the transfer and require you to restart the process.',
  );
  
  // Simulate 3-second processing (demo only)
  _timer = Timer(const Duration(seconds: 3), () {
    if (!mounted) return;
    AppUtils.hideGlobalLoading();
    final isSuccess = Random().nextBool();
    setState(() {
      _state = isSuccess
          ? PaymentProcessState.success
          : PaymentProcessState.failure;
    });
  });
}
```

### State Detection from Arguments

```dart
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  final args = ModalRoute.of(context)?.settings.arguments;
  
  if (args is Map && args['status'] is String) {
    final status = (args['status'] as String).toLowerCase();
    if (status == 'success') {
      _setStateImmediate(PaymentProcessState.success);
    } else if (status == 'failed' || status == 'failure') {
      _setStateImmediate(PaymentProcessState.failure);
    }
  }

  // Check for paid parameter (paid=true/false or paid%5D=true/false)
  if (args is Map && args['paid'] is String) {
    final paidValue = (args['paid'] as String).toLowerCase();
    if (paidValue == 'true') {
      _setStateImmediate(PaymentProcessState.success);
    } else if (paidValue == 'false') {
      _setStateImmediate(PaymentProcessState.failure);
    }
  }

  if (args is Map) {
    if (args['vehicleNumber'] is String) {
      _vehicleNumber = args['vehicleNumber'] as String;
    }
    if (args['serviceType'] is String) {
      final st = (args['serviceType'] as String).toLowerCase();
      _serviceType = st == 'batterypurchase'
          ? BTServiceType.batteryPurchase
          : BTServiceType.batteryJumpstart;
    }
    if (args['orderId'] is String) {
      _orderId = args['orderId'] as String;
    }
    if (args['paymentMethod'] is String) {
      _paymentMethod = args['paymentMethod'] as String;
    }
    if (args['amount'] is String) {
      _amount = args['amount'] as String;
    }
  }
}
```

### Content Building

```dart
Widget _buildStateContent() {
  switch (_state) {
    case PaymentProcessState.processing:
      return BTReceiptWidget(
        status: BTOrderStatus.loading,
        vehicleNumber: _vehicleNumber,
        serviceType: _serviceType,
        title: 'Redirecting',
        description:
            'We are securely redirecting you to your bank. Closing or refreshing the window may disrupt the transfer and require you to restart the process.',
        amountLabel: 'Total Payment',
        amountValue: _amount,
        orderId: _orderId,
        paymentMethod: _paymentMethod,
      );
    case PaymentProcessState.success:
      return BTReceiptWidget(
        status: BTOrderStatus.success,
        vehicleNumber: _vehicleNumber,
        serviceType: _serviceType,
        title: 'Payment Successful!',
        description: 'Yey! Your payment has been successful.',
        amountLabel: 'Total Payment',
        amountValue: _amount,
        orderId: _orderId,
        paymentMethod: _paymentMethod,
      );
    case PaymentProcessState.failure:
      return BTReceiptWidget(
        status: BTOrderStatus.failure,
        vehicleNumber: _vehicleNumber,
        serviceType: _serviceType,
        title: 'Uh-oh, Payment Failed!',
        description: "Sorry, we aren't able to process your payment.",
        amountLabel: 'Total Payment',
        amountValue: _amount,
        paymentMethod: _paymentMethod,
      );
  }
}
```

### Action Buttons

```dart
Widget _buildBottomButtons() {
  if (_state == PaymentProcessState.processing) {
    return const SizedBox.shrink();
  }

  return Padding(
    padding: const EdgeInsets.only(top: 24),
    child: _state == PaymentProcessState.success
        ? Row(
            children: [
              Expanded(
                child: SecondaryButton(
                  text: 'Okay',
                  borderColor: AppColors.bateriku900,
                  foregroundColor: AppColors.bateriku900,
                  onPressed: _handleHome,
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: PrimaryButton(
                  text: 'Track Order',
                  onPressed: _handleTrackOrder,
                ),
              ),
            ],
          )
        : Row(
            children: [
              Expanded(
                child: SecondaryButton(
                  text: 'Retry',
                  borderColor: AppColors.bateriku900,
                  foregroundColor: AppColors.bateriku900,
                  onPressed: () {
                    Navigator.of(context).pop();
                  },
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: PrimaryButton(
                  text: 'Cancel',
                  onPressed: _handleHome,
                ),
              ),
            ],
          ),
  );
}
```

### Navigation Handlers

```dart
void _handleHome() {
  Navigator.of(context).popUntil((route) => route.isFirst);
}

void _handleTrackOrder() {
  // TODO: Navigate to orders tracking screen
  Navigator.of(context).popUntil((route) => route.isFirst);
}
```

---

## Route Configuration

**File:** `lib/main.dart`

```dart
routes: {
  '/payment/redirect': (context) => const PaymentRedirectScreen(),
  '/payment/redirecting': (context) => const PaymentDeepLinkRedirectingScreen(),
  '/payment/status': (context) => const PaymentStatusScreen(),
},
```

---

## Data Flow Summary

### Order → Payment URL

1. Order Confirmation Screen collects customer, vehicle, location, and product info
2. API creates order with collected data
3. API returns: `paymentUrl` (BotaPay gateway) + `redirectUrl` (app deep link)

### Payment Processing

4. Payment WebView loads `paymentUrl` in in-app browser
5. User completes payment in BotaPay gateway
6. Gateway returns via custom deep link: `myapp://payment/redirecting?status=success&orderId=xxx&amount=xxx`

### Result Handling

7. WebView intercepts deep link and navigates to Payment Deep Link Redirecting Screen
8. Redirecting Screen extracts parameters and navigates to Payment Status Screen
9. Status Screen displays result (success/failure) with action buttons

---

## Key Integration Points

| Component | Purpose |
|-----------|---------|
| **BotaPay API** | Payment gateway (URL from order creation) |
| **Deep Links** | `myapp://payment/redirecting` - Returns from payment |
| **Route Parameters** | Passes order/payment info between screens |
| **WebView URL Parsing** | Detects success/failure from URL patterns |
| **BTReceiptWidget** | Displays transaction summary and status |
| **AppUtils** | Global loading dialog management |

---

## Error Handling

### WebView Errors

```dart
onWebViewCreated: (controller) {
  print('WebView created');
  _webViewController = controller;
},
onLoadError: (controller, url, code, message) {
  print('WebView load error: $code - $message');
  setState(() {
    _error = message ?? 'Failed to load payment gateway';
    _isLoading = false;
  });
},
```

### Exit Confirmation

```dart
Future<void> _showExitConfirmDialog() async {
  final confirmed = await BTConfirmDialog.confirm(
    context,
    title: 'Exit Payment Gateway?',
    description:
        'Your payment is not yet completed. Are you sure you want to exit?',
    confirmText: 'Exit',
    cancelText: 'Continue',
    icon: Icons.warning_outlined,
  );

  if (confirmed && mounted) {
    Navigator.of(context).pop();
  }
}
```

---

## Testing Considerations

### Local Testing

- Use `launchUrlString` fallback to simulate deep link returns
- Mock payment gateway responses
- Test success/failure URL detection patterns

### Deep Link Patterns Detected

**Success Indicators:**
- `paid=true`
- `paid%5D=true` (URL encoded)
- `paid]=true`
- `success=true`
- `success`
- `callback`

**Failure Indicators:**
- `paid=false`
- `paid%5D=false` (URL encoded)
- `paid]=false`
- `success=false`

---

## Future Enhancements

1. **Order Tracking**: Implement actual order tracking screen for successful payments
2. **Retry Logic**: Add automatic retry mechanism for failed payments
3. **Payment Methods**: Add more payment methods beyond BotaPay
4. **Analytics**: Track payment flow metrics
5. **Offline Support**: Queue failed payments for retry when online
6. **Receipt Download**: Generate and download payment receipts

---

## Related Files

- `lib/bloc/order_confirmation/` - Order creation and product loading logic
- `lib/core/services/order_service.dart` - API communication for orders
- `lib/shared/widgets/bt_receipt_widget.dart` - Transaction display component
- `lib/core/utils/app_utils.dart` - Global loading dialog utilities
