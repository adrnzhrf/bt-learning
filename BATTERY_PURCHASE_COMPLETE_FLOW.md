# Battery Purchase Complete Flow - Location to Payment

## 🎯 Complete User Journey

```
┌─────────────────────────────────────────────────────────────────┐
│                          HOME SCREEN                            │
│                    User taps "Battery Purchase"                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Route: /battery-purchase/location
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              LOCATION SELECTION SCREEN                          │
│  - Select location via GPS or Map search                        │
│  - Confirm address                                              │
│  - Store location in LocationCubit                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Route: /battery-purchase/vehicle
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              VEHICLE DETAILS SCREEN                             │
│  - Enter plate number, name, phone, email                       │
│  - Validate form                                                │
│  - Call ProductsService.getProducts() API                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ ProductsService returns products
                           │ & brandId
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              PRODUCT SELECTION SCREEN                           │
│  - Display available batteries                                  │
│  - User selects battery                                         │
│  - BatteryPurchaseCubit.selectBatteryFromProducts()            │
│  - Get price quote from API                                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Navigate to OrderConfirmation
                           │ (Manual or automatic)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│          ORDER CONFIRMATION SCREEN (Checkout)                   │
│  - Review: Battery selection, location, vehicle                 │
│  - Apply promo/voucher code                                     │
│  - Toggle trade-in option                                       │
│  - See price breakdown with discounts                           │
│  - User confirms and taps "Pay Now"                             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ OrderService.createOrder() API
                           │ Returns: paymentUrl + orderId
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│          PAYMENT WEBVIEW SCREEN (BotaPay Gateway)               │
│  - Load payment URL in in-app browser                           │
│  - User completes payment                                       │
│  - Monitor URL for success/failure patterns                     │
│  - Intercept deep link redirect                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Deep link detected:
                           │ myapp://payment/redirecting?...
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│     PAYMENT DEEP LINK REDIRECTING SCREEN                        │
│  - Extract parameters from deep link                            │
│  - Pass to Payment Status Screen                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Route: /payment/status
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│          PAYMENT STATUS SCREEN                                  │
│  - Show processing state (3 seconds)                            │
│  - Display success/failure result                               │
│  - Show order confirmation details                              │
│  - Action buttons: Home, Track Order                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │ User taps Home or Track
                           ▼
                    ┌──────────────┐
                    │  HOME SCREEN │
                    └──────────────┘
```

---

## 📁 Code Files Related to the Flow

### **Step 1: Location Selection**

#### Screen Files:
- **Main:** [lib/shared/screens/location_screen.dart](lib/shared/screens/location_screen.dart#L1) (1300+ lines)
  - Reusable location selection screen
  - Used by both Battery Purchase and Jumpstart features
  - Implements Google Maps and Huawei Maps support
  - GPS location detection
  - Address search with autocomplete

#### Related Files:
- [lib/bloc/location/location_cubit.dart](lib/bloc/location/location_cubit.dart)
  - Stores selected location (latitude, longitude, address)
  
- [lib/core/services/map_service.dart](lib/core/services/map_service.dart)
  - Google Maps API integration
  - Place search and details
  
- [lib/core/services/map_service_factory.dart](lib/core/services/map_service_factory.dart)
  - Factory for creating appropriate map service (Google/Huawei)

#### Navigation:
```dart
// From HomeScreen
Navigator.pushNamed(context, '/battery-purchase/location');

// From LocationScreen (storage in Cubit)
context.read<LocationCubit>().setLocation(
  latitude: _currentPosition.latitude,
  longitude: _currentPosition.longitude,
  address: _currentAddress,
);

// Navigate to vehicle screen
Navigator.pushNamed(context, '/battery-purchase/vehicle');
```

---

### **Step 2: Vehicle Details Form**

#### Screen Files:
- **Main:** [lib/shared/screens/vehicle_screen.dart](lib/shared/screens/vehicle_screen.dart#L1) (450+ lines)
  - Collects customer and vehicle information
  - Form validation
  - Phone number formatting (Malaysia specific)
  - Calls ProductsService API

#### Related Files:
- [lib/core/services/products_service.dart](lib/core/services/products_service.dart) (217 lines)
  - API: `POST /api/partners/v2/orders/products`
  - Sends user location and vehicle data
  - Returns available products and brand ID

#### API Integration:
```dart
// ProductsService.getProducts()
POST /api/partners/v2/orders/products

Request Body:
{
  "user": {
    "name": string,
    "phone": string,
    "email": string
  },
  "order": {
    "latitude": double,
    "longitude": double,
    "vehicle_plate_number": string
  }
}

Response:
{
  "brand_id": int,
  "products": [ProductItem]
}
```

#### Key Logic:
```dart
// In VehicleScreen._fetchAndNavigate()
final result = await ProductsService.instance.getProducts(
  token: token,
  userName: nameText,
  userPhone: cleanPhone,
  userEmail: emailText,
  latitude: locationState.latitude,
  longitude: locationState.longitude,
  vehiclePlateNumber: plateText,
);

// Navigate to product screen with products
if (result.products.isNotEmpty) {
  Navigator.pushNamed(
    context,
    '/battery-purchase/product',
    arguments: {
      'products': result.products,
      'brandId': result.brandId,
      'customerInfo': {...},
      'vehicleInfo': {...},
      'location': {...},
    },
  );
}
```

---

### **Step 3: Product Selection**

#### Screen Files:
- **Main:** [lib/features/battery_purchase/screens/product_battery_screen.dart](lib/features/battery_purchase/screens/product_battery_screen.dart#L1) (509 lines)
  - Display battery products in list format
  - User selects battery by tapping
  - Selected battery highlighted with visual feedback

#### BLoC Files:
- [lib/bloc/battery_purchase/battery_purchase_cubit.dart](lib/bloc/battery_purchase/battery_purchase_cubit.dart) (341 lines)
  - `setProductsFromRoute()` - Initialize products
  - `selectBatteryFromProducts()` - Select battery and get quote
  - State management for battery selection

- [lib/bloc/battery_purchase/battery_purchase_state.dart](lib/bloc/battery_purchase/battery_purchase_state.dart) (227 lines)
  - Stores selected battery
  - Stores current quote (price)
  - Battery selection status (loading/success/failure)

- [lib/bloc/battery_purchase/battery_purchase_event.dart](lib/bloc/battery_purchase/battery_purchase_event.dart)
  - `BatterySelected` event triggered on user selection

#### Service Files:
- [lib/core/services/battery_purchase_service.dart](lib/core/services/battery_purchase_service.dart) (626 lines)
  - `getQuote()` API call - Get price quote for selected battery at location

#### API Integration:
```dart
// BatteryPurchaseService.getQuote()
POST /api/v1/quotes

Request Body:
{
  "battery_id": string,
  "location": {
    "latitude": double,
    "longitude": double
  },
  "promo_code": string (optional)
}

Response:
{
  "basePrice": double,
  "discount": double,
  "deliveryFee": double,
  "totalPrice": double,
  "promoCodeApplied": string
}
```

#### Key Code:
```dart
// ProductBatteryScreen
BlocBuilder<BatteryPurchaseCubit, BatteryPurchaseState>(
  builder: (context, state) {
    return Scaffold(
      body: BatteryList(
        products: state.apiProducts,
        selectedBattery: state.selectedBattery,
        onSelect: (batteryId) {
          context.read<BatteryPurchaseCubit>()
            .selectBatteryFromProducts(
              batteryId: batteryId,
              latitude: locationInfo['latitude'],
              longitude: locationInfo['longitude'],
            );
        },
      ),
      floatingActionButton: ProceedButton(
        onPressed: () {
          Navigator.pushNamed(
            context,
            '/order/confirmation',
            arguments: {...},
          );
        },
      ),
    );
  },
);
```

---

### **Step 4: Order Confirmation (Checkout)**

#### Screen Files:
- **Main:** [lib/features/order/screens/order_confirmation_screen.dart](lib/features/order/screens/order_confirmation_screen.dart#L1) (1900+ lines)
  - Display selected battery, location, vehicle info
  - Show price breakdown
  - Apply promo codes
  - Toggle trade-in option
  - Handle order creation

#### BLoC Files:
- [lib/bloc/order_confirmation/order_confirmation_cubit.dart](lib/bloc/order_confirmation/order_confirmation_cubit.dart)
  - `createOrder()` - Call API to create order and get payment URL
  - `calculateOrder()` - Recalculate total with discounts
  - `validatePromoCode()` - Verify promo code

- [lib/bloc/order_confirmation/order_confirmation_state.dart](lib/bloc/order_confirmation/order_confirmation_state.dart)
  - States: OrderCalculationSuccess, OrderCreationSuccess, etc.

#### Service Files:
- [lib/core/services/order_service.dart](lib/core/services/order_service.dart)
  - `createOrder()` API call

- [lib/core/services/payment_context_service.dart](lib/core/services/payment_context_service.dart)
  - Store payment context before navigation

#### API Integration:
```dart
// OrderService.createOrder()
POST /api/v1/orders

Request Body:
{
  "user_name": string,
  "user_phone": string,
  "user_email": string,
  "latitude": double,
  "longitude": double,
  "address": string,
  "vehicle_plate_number": string,
  "product_id": string,
  "brand_id": int,
  "payment_type": string,      // e.g., "billplz"
  "trade_in": int,              // 0 or 1
  "promo_code": string,         // optional
  "redirect_url": string        // for payment deep link
}

Response:
{
  "orderId": string,
  "paymentUrl": string,         // Payment gateway URL
  "totalAmount": double,
  "vehicleNumber": string,
  "serviceType": string
}
```

#### Key Code:
```dart
// OrderConfirmationScreen._createOrder()
void _createOrder() {
  // Build redirect URL for app deep link after payment
  final redirectUrl = 'myapp://payment/redirecting?orderId=...';

  context.read<OrderConfirmationCubit>().createOrder(
    userName: _customerInfo['name'],
    userPhone: _customerInfo['phone'],
    userEmail: _customerInfo['email'],
    latitude: locationState.latitude,
    longitude: locationState.longitude,
    address: locationState.address,
    vehiclePlateNumber: _vehicleInfo['plateNumber'],
    productId: _selectedProductStringId,
    brandId: _brandId,
    paymentType: 'billplz',
    tradeIn: hasTradeIn ? 1 : 0,
    promoCode: voucherCode,
    redirectUrl: redirectUrl,
  );
}

// On success, handle payment
void _handleOrderCreationSuccess(OrderCreationResponse order) {
  if (order.paymentUrl != null) {
    // Store payment context
    PaymentContextService().setPaymentContext(
      orderId: order.orderId,
      amount: order.totalAmount.toString(),
      vehicleNumber: order.vehicleNumber,
      serviceType: order.serviceType,
      paymentMethod: order.paymentMethod,
    );

    // Navigate to payment webview
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => PaymentWebViewScreen(
          paymentUrl: order.paymentUrl,
        ),
      ),
    );
  }
}
```

#### Related Files:
- [lib/features/order/screens/promo_code_screen.dart](lib/features/order/screens/promo_code_screen.dart)
  - Promo code validation and application
  - Route: `/order/promo-code`

---

### **Step 5: Payment Processing**

#### Screen Files:

**5.1 Payment WebView Screen**
- **File:** [lib/features/order/screens/payment_webview_screen.dart](lib/features/order/screens/payment_webview_screen.dart) (200+ lines)
  - OR: [lib/features/payment/screens/bt_payment_webview_screen.dart](lib/features/payment/screens/bt_payment_webview_screen.dart) (alternative)
  - Load payment gateway URL in in-app WebView
  - Monitor URL changes for payment status
  - Intercept deep link redirects

#### WebView Navigation Logic:
```dart
// PaymentWebViewScreen - Detect payment completion
InAppWebViewClient(
  onLoadStart: (controller, url) {
    // Monitor URL for payment patterns
    final urlString = url.toString();
    
    if (_isPaymentSuccess(urlString)) {
      // Payment successful - navigate to result
      final args = _buildPaymentArguments('success', url);
      Navigator.of(context).pushReplacementNamed(
        '/payment/redirecting',
        arguments: args,
      );
    } else if (_isPaymentFailure(urlString)) {
      // Payment failed
      final args = _buildPaymentArguments('failure', url);
      Navigator.of(context).pushReplacementNamed(
        '/payment/redirecting',
        arguments: args,
      );
    }
  },
);

// Success/failure patterns
static const _paidTruePatterns = [
  'paid=true',
  'status=success',
  'transaction_status=success',
];

static const _paidFalsePatterns = [
  'paid=false',
  'status=failure',
  'transaction_status=failed',
];
```

**5.2 Payment Deep Link Redirecting Screen**
- **File:** [lib/features/payment/screens/payment_deeplink_redirecting_screen.dart](lib/features/payment/screens/payment_deeplink_redirecting_screen.dart)
  - Parse deep link parameters
  - Extract orderId, status, amount
  - Navigate to Payment Status Screen

#### Deep Link Format:
```
myapp://payment/redirecting?
  status=success
  &orderId=ORD-12345
  &amount=395.00
  &transactionId=TXN-67890
```

**5.3 Payment Status Screen**
- **File:** [lib/features/payment/screens/payment_status_screen.dart](lib/features/payment/screens/payment_status_screen.dart) (200+ lines)
  - Display payment result (success/failure)
  - Show order details and confirmation
  - Action buttons: Home, Track Order

#### Key Code:
```dart
// PaymentStatusScreen
class PaymentStatusScreen extends StatefulWidget {
  final String? status;
  final String? orderId;
  final String? amount;
  final String? transactionId;
}

@override
Widget build(BuildContext context) {
  return Scaffold(
    body: Column(
      children: [
        // Processing state (3 seconds)
        if (_isProcessing) BTReceiptWidget(status: BTOrderStatus.loading),
        
        // Success state
        if (!_isProcessing && _isSuccess)
          BTReceiptWidget(status: BTOrderStatus.success),
        
        // Failure state
        if (!_isProcessing && !_isSuccess)
          BTReceiptWidget(status: BTOrderStatus.failure),
        
        // Action buttons
        Padding(
          padding: EdgeInsets.all(16),
          child: Row(
            children: [
              PrimaryButton(
                text: 'Home',
                onPressed: _handleHome,
              ),
              SizedBox(width: 16),
              SecondaryButton(
                text: 'Track Order',
                onPressed: _handleTrackOrder,
              ),
            ],
          ),
        ),
      ],
    ),
  );
}

void _handleHome() {
  Navigator.of(context).popUntil((route) => route.isFirst);
}
```

---

## 🔄 Data Flow Through the App

### Location Data Flow:
```
LocationScreen
    ↓ User selects location
    ↓ Updates LocationCubit state
    ↓ Store: latitude, longitude, address
    ↓
VehicleScreen (reads from LocationCubit)
    ↓ Passes to ProductsService API
    ↓
ProductsService
    ↓ Returns products filtered for location
    ↓
ProductBatteryScreen (receives via route args)
    ↓ Displays products for selection
    ↓
OrderConfirmationScreen (reads from LocationCubit)
    ↓ Includes location in order creation
    ↓
OrderService.createOrder() API
```

### Customer Info Data Flow:
```
VehicleScreen (Form Input)
    ↓ Collects: name, phone, email, plate number
    ↓ Passes via route arguments
    ↓
ProductBatteryScreen
    ↓ Receives customerInfo, vehicleInfo, location
    ↓
OrderConfirmationScreen (route args)
    ↓ Reads from screen state
    ↓
orderConfirmationCubit.createOrder(...)
    ↓
OrderService.createOrder() API
```

### Battery Selection Data Flow:
```
ProductBatteryScreen
    ↓ User selects battery
    ↓ Calls BatteryPurchaseCubit.selectBatteryFromProducts()
    ↓
BatteryPurchaseCubit
    ↓ Calls BatteryPurchaseService.getQuote()
    ↓
API returns PriceQuote
    ↓ Cubit emits BatteryPurchaseState with:
    ├─ selectedBattery
    ├─ currentQuote
    └─ quoteStatus: success
    ↓
ProductBatteryScreen (BlocBuilder) updates UI
    ↓
User taps "Proceed to Checkout"
    ↓
OrderConfirmationScreen (route args with selectedBattery)
    ↓ Display battery details and quote
    ↓
User taps "Pay Now"
    ↓ OrderConfirmationCubit.createOrder()
```

---

## 🚀 Navigation Routes Configuration

All routes are defined in [lib/main.dart](lib/main.dart#L119):

```dart
routes: {
  // Location Selection
  '/battery-purchase/location': (context) => const LocationScreen(
    nextRoute: '/battery-purchase/vehicle',
  ),
  
  // Vehicle Details
  '/battery-purchase/vehicle': (context) => const VehicleScreen(),
  
  // Product Selection
  '/battery-purchase/product': (context) => const ProductBatteryScreen(),
  
  // Order Confirmation
  '/order/confirmation': (context) => const OrderConfirmationScreen(),
  
  // Promo Code
  '/order/promo-code': (context) => PromoCodeScreen(...),
  
  // Payment WebView
  '/payment/redirect': (context) => const PaymentRedirectScreen(),
  
  // Payment Deep Link Handling
  '/payment/redirecting': (context) => const PaymentDeepLinkRedirectingScreen(),
  
  // Payment Status
  '/payment/status': (context) => const PaymentStatusScreen(...),
}
```

---

## 📦 Models & Data Classes

### Location Data:
```dart
// lib/bloc/location/location_state.dart
class LocationState {
  final double? latitude;
  final double? longitude;
  final String? address;
  final bool isLoading;
  final String? error;
}
```

### Customer & Vehicle Info:
```dart
// lib/core/services/products_service.dart
class CustomerInfo {
  final String name;
  final String phone;
  final String email;
}

class VehicleInfo {
  final String plateNumber;
  final bool hasModifications;
}
```

### Product Item:
```dart
// lib/core/services/products_service.dart
class ProductItem {
  final String id;
  final String name;
  final String? brand;
  final String price;           // e.g., "RM 450.00"
  final int priceCents;         // e.g., 45000
  final int warrantyPeriod;     // months
  final int? warrantyMileage;   // km
  final String? category;
  final String? imageUrl;
}
```

### Battery & Quote:
```dart
// lib/bloc/battery_purchase/battery_purchase_state.dart
class BatteryProduct {
  final String id;
  final String name;
  final String brand;
  final double price;
  final String warrantyMonths;
  final String warrantyKilometers;
  final String imageUrl;
}

class PriceQuote {
  final double basePrice;
  final double? discount;
  final double deliveryFee;
  final double totalPrice;
  final String? promoCodeApplied;
}
```

### Order:
```dart
// lib/features/order/models/order_creation_response.dart
class OrderCreationResponse {
  final String orderId;
  final String paymentUrl;
  final double totalAmount;
  final String vehicleNumber;
  final String serviceType;
  final String? paymentMethod;
}
```

---

## 🎯 Key Services Summary

| Service | File | Purpose |
|---------|------|---------|
| **MapService** | [lib/core/services/map_service.dart](lib/core/services/map_service.dart) | Google Maps API - search, geocoding |
| **ProductsService** | [lib/core/services/products_service.dart](lib/core/services/products_service.dart) | Fetch products based on location & vehicle |
| **BatteryPurchaseService** | [lib/core/services/battery_purchase_service.dart](lib/core/services/battery_purchase_service.dart) | Get batteries, quotes, create orders |
| **OrderService** | [lib/core/services/order_service.dart](lib/core/services/order_service.dart) | Create orders, get payment URLs |
| **PaymentContextService** | [lib/core/services/payment_context_service.dart](lib/core/services/payment_context_service.dart) | Store payment info across navigation |
| **LocationCubit** | [lib/bloc/location/location_cubit.dart](lib/bloc/location/location_cubit.dart) | Manage selected location globally |

---

## 🔐 Environment Configuration

All API URLs are configured in:
- [lib/core/config/env_config.dart](lib/core/config/env_config.dart)

```dart
class EnvConfig {
  static const String batteryPurchaseApiUrl = 
    'https://api.bateriku.com/';  // or staging URL
  static const int apiTimeout = 30000;  // 30 seconds
}
```

---

## 🧪 Complete Example: From Start to Payment

```dart
// 1. User taps Battery Purchase from Home
Navigator.pushNamed(context, '/battery-purchase/location');

// 2. LocationScreen - User selects location (lat: 3.15, lng: 101.50, addr: "KL")
context.read<LocationCubit>().setLocation(
  latitude: 3.15,
  longitude: 101.50,
  address: "KL",
);
Navigator.pushNamed(context, '/battery-purchase/vehicle');

// 3. VehicleScreen - User fills form
// name: "John", phone: "0123456789", email: "john@example.com", plate: "WYS9925"
// API Call: ProductsService.getProducts(...)
// Returns: 3 batteries, brandId: 5
Navigator.pushNamed(
  context,
  '/battery-purchase/product',
  arguments: {
    'products': [Battery1, Battery2, Battery3],
    'brandId': 5,
    'customerInfo': {'name': 'John', 'phone': '0123456789', 'email': 'john@example.com'},
    'vehicleInfo': {'plateNumber': 'WYS9925'},
    'location': {'latitude': 3.15, 'longitude': 101.50, 'address': 'KL'},
  },
);

// 4. ProductBatteryScreen - User selects battery
context.read<BatteryPurchaseCubit>().selectBatteryFromProducts(
  batteryId: 'astra-ns70l',
  latitude: 3.15,
  longitude: 101.50,
);
// BatteryPurchaseService.getQuote() returns price: RM 450.00

// 5. User taps "Proceed to Checkout"
Navigator.pushNamed(
  context,
  '/order/confirmation',
  arguments: {
    'selectedBattery': BatteryProduct(...),
    'quote': PriceQuote(basePrice: 450.00, ...),
    'customerInfo': {...},
    'vehicleInfo': {...},
    'location': {...},
  },
);

// 6. OrderConfirmationScreen - User reviews & taps "Pay Now"
context.read<OrderConfirmationCubit>().createOrder(
  userName: 'John',
  userPhone: '60123456789',
  userEmail: 'john@example.com',
  latitude: 3.15,
  longitude: 101.50,
  address: 'KL',
  vehiclePlateNumber: 'WYS9925',
  productId: 'astra-ns70l',
  brandId: 5,
  paymentType: 'billplz',
  tradeIn: 0,
  redirectUrl: 'myapp://payment/redirecting?...',
);
// API returns: orderId: 'ORD-123456', paymentUrl: 'https://payment.gateway.com/...'

// 7. PaymentWebViewScreen - User completes payment in gateway
// Gateway detects success and redirects via deep link:
// myapp://payment/redirecting?status=success&orderId=ORD-123456

// 8. PaymentDeepLinkRedirectingScreen - Extract parameters
final args = ModalRoute.of(context)?.settings.arguments;
Navigator.pushReplacementNamed(
  context,
  '/payment/status',
  arguments: {
    'status': 'success',
    'orderId': 'ORD-123456',
    'amount': '450.00',
  },
);

// 9. PaymentStatusScreen - Show success receipt
// User can tap "Home" to return to home screen
```

---

## ✅ Summary

**Complete Flow:**
- Location Selection → Vehicle Details → Product Selection → Order Confirmation → Payment WebView → Payment Status → Home

**Key Files:**
- Location: `location_screen.dart`
- Vehicle: `vehicle_screen.dart`
- Product: `product_battery_screen.dart`
- Order: `order_confirmation_screen.dart`
- Payment: `payment_webview_screen.dart`, `payment_status_screen.dart`

**Key Services:**
- ProductsService, BatteryPurchaseService, OrderService, MapService

**State Management:**
- LocationCubit, BatteryPurchaseCubit, OrderConfirmationCubit

**APIs:**
- Products, Quotes, Orders, Payment Gateway Integration

This complete flow ensures a seamless battery purchase experience from location selection through payment processing! 🎉
