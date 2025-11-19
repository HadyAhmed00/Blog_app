# Etisalat Wallet Payment Gateway Integration Guide

## Overview

This guide provides step-by-step instructions for integrating the Etisalat Wallet payment gateway using WebView into the existing payment system architecture.

---

## Integration Analysis

### Key Findings

- **Integration Type**: LiteBox/Iframe (uses WebView in mobile)
- **Signature Algorithm**: SHA-256 HMAC (same as Stryve!)
- **Payment Detection**: Callbacks (completeCallback, errorCallback, cancelCallback)
- **Authentication**: No separate auth step needed - uses SecureHash in configuration

### Effort Required

| Task | Effort | Notes |
|------|--------|-------|
| Remote datasource methods | 🟢 Low | 2-3 simple API calls |
| Gateway implementation | 🟢 Low | Similar to Stryve |
| WebView setup | 🟢 Low | Load JS library + configure |
| **Total** | **🟢 30-45 min** | Can reuse 80% of Stryve code |

---

## Step-by-Step Integration Plan

### Step 1: Add Remote Datasource Methods

Add these methods to `ConfirmPaymentRemoteDatasource`:

```dart
// In confirm_payment_remote_datasource.dart

/// Validate Etisalat Lightbox configuration before showing payment
Future<Resource<bool>> getEtisalatLightboxConfiguration({
  required String merchantId,
  required String terminalId,
  required String amount,
  required String orderId,
  required String merchantReference,
  required String transactionTime,
  required String secureHash,
}) async {
  try {
    final response = await dio.post(
      '$baseUrl/ValidateLightbox',
      data: {
        'MerchantId': merchantId,
        'TerminalId': terminalId,
        'AmountTrxn': amount,
        'OrderId': orderId,
        'MerchantReference': merchantReference,
        'DateTimeLocalTrxn': transactionTime,
        'SecureHash': secureHash,
      },
    );

    if (response.statusCode == 200) {
      return SuccessState(true);
    } else {
      return ErrorState('Failed to validate Etisalat configuration');
    }
  } catch (e) {
    return ErrorState('Etisalat validation error: $e');
  }
}

/// Check payment status (optional but recommended)
Future<Resource<EtisalatPaymentStatus>> checkEtisalatPaymentStatus({
  required String merchantReference,
}) async {
  try {
    final response = await dio.post(
      '$baseUrl/CheckTxnStatus',
      data: {
        'MerchantReference': merchantReference,
        'MerchantId': 'YOUR_MERCHANT_ID',
        'TerminalId': 'YOUR_TERMINAL_ID',
        'DateTimeLocalTrxn': DateFormat('yyyyMMddHHmmss').format(DateTime.now()),
        'SecureHash': generateSecureHash(...),
      },
    );

    if (response.statusCode == 200) {
      final status = EtisalatPaymentStatus.fromJson(response.data);
      return SuccessState(status);
    }
    return ErrorState('Failed to check payment status');
  } catch (e) {
    return ErrorState('Error checking payment status: $e');
  }
}
```

---

### Step 2: Create Etisalat Payment Status Model

Create a new file: `lib/features/payment/features/confirm_payment/data/model/etisalat_payment_status.dart`

```dart
class EtisalatPaymentStatus {
  final bool success;
  final bool isPaid;
  final bool isTahweelPaid;
  final bool isMVisaPaid;
  final String? networkReference;
  final String? paidThrough;
  final int? amount;
  final String? systemReference;

  EtisalatPaymentStatus({
    required this.success,
    required this.isPaid,
    required this.isTahweelPaid,
    required this.isMVisaPaid,
    this.networkReference,
    this.paidThrough,
    this.amount,
    this.systemReference,
  });

  factory EtisalatPaymentStatus.fromJson(Map<String, dynamic> json) {
    return EtisalatPaymentStatus(
      success: json['Success'] ?? false,
      isPaid: json['IsPaid'] ?? false,
      isTahweelPaid: json['IsTahweelPaid'] ?? false,
      isMVisaPaid: json['IsMVisaPaid'] ?? false,
      networkReference: json['NetworkReference'],
      paidThrough: json['PaidThrough'],
      amount: json['AmountTrxn'],
      systemReference: json['SystemReference']?.toString(),
    );
  }
}
```

---

### Step 3: Create Etisalat Gateway

Create a new file: `lib/features/payment/features/confirm_payment/data/datasource/gateways/etisalat_payment_gateway.dart`

```dart
import 'dart:convert';
import 'package:crypto/crypto.dart';
import 'package:ibnsina_pharma/core/network/resource.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/data/datasource/remote/confirm_payment_remote_datasource.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/domain/entities/payment_context.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/domain/entities/payment_result.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/domain/entities/webview_payment_strategy.dart';

import 'payment_gateway.dart';

class EtisalatPaymentGateway implements PaymentGateway {
  final ConfirmPaymentRemoteDatasource remoteDataSource;

  EtisalatPaymentGateway(this.remoteDataSource);

  @override
  Future<String> authenticate() async {
    // Etisalat doesn't require explicit authentication
    // The Lightbox handles authentication via SecureHash
    return '';
  }

  @override
  Future<CreatePaymentInvoiceResponse> createInvoice({
    required String authToken,
    required String supplierReference,
    required String amount,
    required String transactionTime,
    required String signature,
    required String description,
    required String returnUrl,
    required String mobileNumber,
  }) async {
    // Not used for Etisalat - Lightbox handles invoice creation
    throw UnsupportedError(
      'Etisalat uses Lightbox configuration instead of explicit invoice creation',
    );
  }

  @override
  Future<bool> postPaymentResult({
    required List<Map<String, dynamic>> paymentResults,
  }) async {
    // Implementation for posting payment results to your backend
    // This is called after payment completion
    return true;
  }

  @override
  String generateSignature({
    required String secretKey,
    required String message,
  }) {
    // Etisalat uses SHA-256 HMAC with hex encoding (uppercase)
    // Different from Stryve which uses base64
    final keyBytes = utf8.encode(secretKey);
    final messageBytes = utf8.encode(message);
    final hmac = Hmac(sha256, keyBytes);
    final digest = hmac.convert(messageBytes);

    // Convert to hex and uppercase
    return digest.toString().toUpperCase();
  }

  @override
  WebViewPaymentStrategy getExecutionStrategy() {
    return WebViewPaymentStrategy(
      createInvoiceFunc: (context) async {
        try {
          print('🔵 EtisalatPaymentGateway: Preparing Lightbox configuration');

          // Validate configuration on backend (optional but recommended)
          final validationResult = await remoteDataSource.getEtisalatLightboxConfiguration(
            merchantId: 'YOUR_MERCHANT_ID', // Configure this
            terminalId: 'YOUR_TERMINAL_ID', // Configure this
            amount: context.amount,
            orderId: context.supplierReference,
            merchantReference: context.supplierReference,
            transactionTime: context.transactionTime,
            secureHash: context.signature,
          );

          if (validationResult is! SuccessState) {
            return PaymentResult.error(
              errorMessage: 'Etisalat validation failed',
            );
          }

          print('✅ EtisalatPaymentGateway: Lightbox configuration ready');

          // Return Lightbox URL - will be loaded in WebView
          return PaymentResult.success(
            invoiceId: context.supplierReference,
            // Lightbox URL will handle the payment UI
            redirectUrl: 'about:blank', // Will be replaced by WebView HTML
            invoiceResponse: null,
          );
        } catch (e) {
          print('🔴 EtisalatPaymentGateway: Error - $e');
          return PaymentResult.error(
            errorMessage: 'Failed to prepare Etisalat payment: ${e.toString()}',
          );
        }
      },
    );
  }
}
```

---

### Step 4: Create Etisalat WebView Widget

Create a new file: `lib/features/payment/features/confirm_payment/presentation/pages/etisalat_web_payment.dart`

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/domain/entities/payment_result.dart';

class EtisalatWebPaymentView extends StatefulWidget {
  final String merchantId;
  final String terminalId;
  final String amount; // In piasters
  final String orderId;
  final String merchantReference;
  final String transactionTime;
  final String secureHash;
  final Function(PaymentResult) onComplete;
  final Function(String) onError;
  final Function() onCancel;

  const EtisalatWebPaymentView({
    Key? key,
    required this.merchantId,
    required this.terminalId,
    required this.amount,
    required this.orderId,
    required this.merchantReference,
    required this.transactionTime,
    required this.secureHash,
    required this.onComplete,
    required this.onError,
    required this.onCancel,
  }) : super(key: key);

  @override
  State<EtisalatWebPaymentView> createState() => _EtisalatWebPaymentViewState();
}

class _EtisalatWebPaymentViewState extends State<EtisalatWebPaymentView> {
  late WebViewController _controller;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _setupWebViewController();
  }

  void _setupWebViewController() {
    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..setNavigationDelegate(
        NavigationDelegate(
          onPageStarted: (String url) {
            setState(() => _isLoading = true);
          },
          onPageFinished: (String url) {
            setState(() => _isLoading = false);
          },
          onWebResourceError: (WebResourceError error) {
            print('WebView error: ${error.description}');
            widget.onError('WebView error: ${error.description}');
          },
        ),
      )
      ..addJavaScriptChannel(
        'PaymentComplete',
        onMessageReceived: (JavaScriptMessage message) {
          print('✅ Payment complete callback received');
          try {
            final data = json.decode(message.message);
            print('Payment data: $data');

            widget.onComplete(PaymentResult.success(
              invoiceId: data['SystemReference']?.toString() ??
                        data['TxnId']?.toString() ?? '',
              status: 'completed',
            ));
          } catch (e) {
            print('Error parsing payment complete: $e');
            widget.onError('Error parsing payment response: $e');
          }
        },
      )
      ..addJavaScriptChannel(
        'PaymentError',
        onMessageReceived: (JavaScriptMessage message) {
          print('❌ Payment error callback received: ${message.message}');
          widget.onError(message.message);
        },
      )
      ..addJavaScriptChannel(
        'PaymentCancel',
        onMessageReceived: (JavaScriptMessage message) {
          print('🚫 Payment cancelled by user');
          widget.onCancel();
        },
      )
      ..loadRequest(Uri.dataFromString(
        _buildLightboxHtml(),
        mimeType: 'text/html',
        encoding: Encoding.getByName('utf-8'),
      ));
  }

  String _buildLightboxHtml() {
    // Parse amount to integer (in piasters)
    final amountInt = int.parse(widget.amount);

    return '''
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Etisalat Payment</title>
      <script src="https://upgstaging.egyptianbanks.com:8006/js/Lightbox.js"></script>
      <style>
        body { margin: 0; padding: 0; background: #f5f5f5; }
        #loading { text-align: center; padding: 20px; }
      </style>
    </head>
    <body>
      <div id="loading">Loading payment gateway...</div>

      <script>
        console.log('Initializing Etisalat Lightbox...');

        // Configure Lightbox
        Lightbox.Checkout.configure = {
          OrderId: '${widget.orderId}',
          MID: '${widget.merchantId}',
          TID: '${widget.terminalId}',
          SecureHash: '${widget.secureHash}',
          TrxDateTime: '${widget.transactionTime}',
          AmountTrxn: $amountInt,
          MerchantReference: '${widget.merchantReference}',
          paymentMethodFromLightBox: 2, // 0=Card, 1=Digital, 2=Both

          completeCallback: function(data) {
            console.log('Complete callback:', data);
            PaymentComplete.postMessage(JSON.stringify(data));
          },

          errorCallback: function(data) {
            console.log('Error callback:', data);
            const errorMsg = data.error || data.Error || JSON.stringify(data);
            PaymentError.postMessage(errorMsg);
          },

          cancelCallback: function() {
            console.log('Cancel callback');
            PaymentCancel.postMessage('User cancelled payment');
          }
        };

        console.log('Configuration ready, showing Lightbox...');

        // Show Lightbox as popup over the hosted site
        // Use showPaymentPage() for external URL or showLightbox() for popup
        try {
          Lightbox.Checkout.showLightbox();
        } catch (e) {
          console.error('Error showing Lightbox:', e);
          PaymentError.postMessage('Failed to load payment gateway: ' + e.message);
        }
      </script>
    </body>
    </html>
    ''';
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        WebViewWidget(controller: _controller),
        if (_isLoading)
          const Center(
            child: CircularProgressIndicator(),
          ),
      ],
    );
  }
}
```

---

### Step 5: Update Payment Gateway Factory

Edit `lib/features/payment/features/confirm_payment/data/datasource/gateways/payment_gateway_factory.dart`:

```dart
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/data/datasource/gateways/etisalat_payment_gateway.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/data/enums/payment_provider_type.dart';

class PaymentGatewayFactory {
  final ConfirmPaymentRemoteDatasource remoteDataSource;

  PaymentGatewayFactory(this.remoteDataSource);

  PaymentGateway createGateway(PaymentProviderType providerType) {
    switch (providerType) {
      case PaymentProviderType.stryve:
        return StryvePaymentGateway(remoteDataSource);

      case PaymentProviderType.etisalatWallet:
        return EtisalatPaymentGateway(remoteDataSource);

      case PaymentProviderType.qnbHC:
        throw UnsupportedError(
          'QnbHC payment gateway is not yet implemented',
        );

      case PaymentProviderType.qnbMEZA:
        throw UnsupportedError(
          'QnbMEZA payment gateway is not yet implemented',
        );

      case PaymentProviderType.payMOB:
        throw UnsupportedError(
          'PayMOB payment gateway is not yet implemented',
        );
    }
  }
}
```

---

### Step 6: Update Payment Provider Type Enum

Edit `lib/features/payment/features/confirm_payment/data/enums/payment_provider_type.dart`:

```dart
enum PaymentProviderType {
  stryve('stryve'),
  etisalatWallet('etisalat_wallet'),
  qnbHC('qnb_hc'),
  qnbMEZA('qnb_meza'),
  payMOB('paymob');

  final String id;

  const PaymentProviderType(this.id);

  factory PaymentProviderType.fromId(String id) {
    return values.firstWhere(
      (type) => type.id == id,
      orElse: () => throw Exception('Unknown payment provider: $id'),
    );
  }
}
```

---

### Step 7: Update ConfirmPaymentBloc (Minor Change)

No significant changes needed. The existing logic in `confirm_payment_bloc.dart` will work as-is because `ProcessPaymentUseCase` handles all gateway differences internally.

---

## Key Differences from Stryve

| Aspect | Stryve | Etisalat |
|--------|--------|----------|
| **Signature** | Base64 HMAC | Hex uppercase HMAC |
| **Authentication** | Explicit API call (`authenticate()`) | No auth needed - handled by Lightbox |
| **Invoice Creation** | REST API endpoint | Lightbox JS configuration |
| **Payment Detection** | URL redirect monitoring | JS callbacks (complete/error/cancel) |
| **User Experience** | External redirect | Popup overlay or embedded |
| **Callback Validation** | Webhook notification | JS function callbacks |

---

## Signature Generation Example

For Etisalat, the signature is generated differently from Stryve:

```dart
// Message format (fields sorted alphabetically)
String message = 'DateTimeLocalTrxn=180829144425&MerchantId=11000000025&TerminalId=800022';

// Hex-decoded secret key
String secretKey = '66623430313531632D663137362D346664332D616634392D396531633665336337376230';

// Result: SHA-256 HMAC in uppercase hex
String signature = '55d537dbcd8c6cf390cc11e1c2e3452a8f73a7a15462a531fa71baa443254677';
```

---

## Configuration Required

Before integration, you'll need these credentials from Etisalat:

1. **MerchantId** - Your Etisalat merchant ID (11-18 chars)
2. **TerminalId** - Your terminal ID (6-8 chars)
3. **Merchant Secret Key** - For HMAC signature (hex encoded)
4. **Environment URL** - Staging or Production
   - Staging: `https://upgstaging.egyptianbanks.com:8006`
   - Production: `https://upg.egyptianbanks.com`

Store these in `PaymentConfig`:

```dart
class PaymentConfig {
  static const String etisalatMerchantId = 'YOUR_MERCHANT_ID';
  static const String etisalatTerminalId = 'YOUR_TERMINAL_ID';
  static const String etisalatSecretKey = 'YOUR_SECRET_KEY_HEX';
  static const String etisalatBaseUrl = 'https://upgstaging.egyptianbanks.com:8006';
}
```

---

## Testing Checklist

- [ ] Merchant credentials configured in PaymentConfig
- [ ] EtisalatPaymentGateway class created and registered
- [ ] Signature generation tested (use online HMAC generator)
- [ ] WebView loads Lightbox HTML correctly
- [ ] JS callbacks (complete, error, cancel) are triggered
- [ ] Payment data is properly parsed from callback
- [ ] PostPaymentResultUseCase receives correct data
- [ ] Backend receives payment notification
- [ ] Build successful: `flutter build apk --debug`

---

## Payment Flow Diagram

```
ConfirmPaymentBloc
  ↓
ProcessPaymentUseCase
  ↓ (generates signature)
EtisalatPaymentGateway
  ↓ (returns strategy)
WebViewPaymentStrategy
  ↓
EtisalatWebPaymentView (loads HTML)
  ↓
Lightbox.js (Etisalat)
  ↓ (user completes payment)
JS Callback (complete/error/cancel)
  ↓
PostPaymentResultUseCase
  ↓
Backend Notification
```

---

## Troubleshooting

### Signature Mismatch
- Ensure secret key is hex-decoded
- Fields must be sorted alphabetically
- Use uppercase hex output
- Field format: `name=value&name2=value2`

### Lightbox Not Loading
- Check that `https://upgstaging.egyptianbanks.com:8006/js/Lightbox.js` is accessible
- Verify MerchantId and TerminalId are correct
- Check browser console for JS errors

### Callbacks Not Triggered
- Verify JS channels are added before loading HTML
- Check that function names match exactly: `PaymentComplete`, `PaymentError`, `PaymentCancel`
- Use `console.log()` in JS to debug

### CORS Issues
- WebView should not have CORS restrictions
- Ensure `setJavaScriptMode(JavaScriptMode.unrestricted)` is set

---

## Additional Resources

- [Etisalat Merchant Integration PDF](./Merchant%20Integration%20-%20Wallet.pdf)
- [SHA-256 HMAC Generator](https://www.liavaag.org/English/SHA-Generator/HMAC/)
- [ISO 4217 Currency Codes](https://en.wikipedia.org/wiki/ISO_4217)

---

## Next Steps

1. Obtain Etisalat merchant credentials
2. Create the files listed in Steps 1-4
3. Update factory and enums (Steps 5-6)
4. Build and test
5. Implement backend notification handler
6. Test with real transactions on staging