# Payment Method Selection Architecture

## Overview

Users can choose between different payment methods for each provider:
- **Stryve**: Card Payment
- **Etisalat**: Card Payment OR Wallet Payment (different WebViews)

---

## Architecture Design

### Payment Method Types

```
PaymentMethod
├── Card
├── Wallet (Etisalat only)
└── Digital (future)
```

### New Data Models

```
PaymentMethodType enum:
  - card          (Stryve, Etisalat)
  - etisalatWallet (Etisalat only)
  - mobileMoney   (future)

PaymentGatewayConfig:
  - providerType: PaymentProviderType
  - paymentMethod: PaymentMethodType
  - displayName: String
  - supportedMethods: List<PaymentMethodType>
```

---

## Step 1: Create PaymentMethodType Enum

File: `lib/features/payment/features/confirm_payment/data/enums/payment_method_type.dart`

```dart
enum PaymentMethodType {
  card('card', 'Card Payment'),
  etisalatWallet('etisalat_wallet', 'Etisalat Wallet'),
  mobileMoney('mobile_money', 'Mobile Money'),
  bankTransfer('bank_transfer', 'Bank Transfer');

  final String id;
  final String displayName;

  const PaymentMethodType(this.id, this.displayName);

  factory PaymentMethodType.fromId(String id) {
    return values.firstWhere(
      (method) => method.id == id,
      orElse: () => throw Exception('Unknown payment method: $id'),
    );
  }
}
```

---

## Step 2: Create Gateway Configuration Model

File: `lib/features/payment/features/confirm_payment/data/model/payment_gateway_config.dart`

```dart
class PaymentGatewayConfig {
  final String id; // e.g., "stryve_card", "etisalat_card", "etisalat_wallet"
  final String displayName; // e.g., "Stryve", "Etisalat Card", "Etisalat Wallet"
  final PaymentProviderType providerType;
  final PaymentMethodType paymentMethod;
  final String? description;
  final String? icon; // Icon asset path
  final bool isEnabled;

  PaymentGatewayConfig({
    required this.id,
    required this.displayName,
    required this.providerType,
    required this.paymentMethod,
    this.description,
    this.icon,
    this.isEnabled = true,
  });
}
```

---

## Step 3: Create Payment Gateway Selector Service

File: `lib/features/payment/features/confirm_payment/domain/usecases/get_available_payment_gateways_usecase.dart`

```dart
import 'package:injectable/injectable.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/data/enums/payment_provider_type.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/data/model/payment_gateway_config.dart';

@lazySingleton
class GetAvailablePaymentGatewaysUseCase {
  /// Returns all available payment gateway options
  List<PaymentGatewayConfig> getAllAvailableGateways() {
    return [
      // Stryve - Card only
      PaymentGatewayConfig(
        id: 'stryve_card',
        displayName: 'Stryve',
        providerType: PaymentProviderType.stryve,
        paymentMethod: PaymentMethodType.card,
        description: 'Pay with your credit or debit card',
        icon: 'assets/icons/stryve.png',
        isEnabled: true,
      ),

      // Etisalat - Card Payment
      PaymentGatewayConfig(
        id: 'etisalat_card',
        displayName: 'Etisalat Card',
        providerType: PaymentProviderType.etisalatWallet,
        paymentMethod: PaymentMethodType.card,
        description: 'Pay with your credit or debit card via Etisalat',
        icon: 'assets/icons/etisalat_card.png',
        isEnabled: true,
      ),

      // Etisalat - Wallet Payment
      PaymentGatewayConfig(
        id: 'etisalat_wallet',
        displayName: 'Etisalat Wallet',
        providerType: PaymentProviderType.etisalatWallet,
        paymentMethod: PaymentMethodType.etisalatWallet,
        description: 'Pay with your Etisalat Wallet balance',
        icon: 'assets/icons/etisalat_wallet.png',
        isEnabled: true,
      ),

      // Future: More payment methods
      // PaymentGatewayConfig(
      //   id: 'paymob_card',
      //   displayName: 'PayMOB',
      //   providerType: PaymentProviderType.payMOB,
      //   paymentMethod: PaymentMethodType.card,
      //   description: 'Pay with PayMOB',
      //   isEnabled: false, // Not yet implemented
      // ),
    ];
  }

  /// Returns gateways for a specific provider
  List<PaymentGatewayConfig> getGatewaysForProvider(PaymentProviderType providerType) {
    return getAllAvailableGateways()
        .where((gateway) => gateway.providerType == providerType && gateway.isEnabled)
        .toList();
  }

  /// Get a specific gateway config
  PaymentGatewayConfig? getGatewayConfig(String gatewayId) {
    return getAllAvailableGateways().firstWhere(
      (gateway) => gateway.id == gatewayId,
      orElse: () => null,
    );
  }
}
```

---

## Step 4: Update ProcessPaymentUseCase

File: `lib/features/payment/features/confirm_payment/domain/usecases/process_payment_usecase.dart`

Add `paymentMethod` parameter:

```dart
Future<PaymentResult> call({
  required PaymentProviderType providerType,
  required PaymentMethodType paymentMethod, // NEW
  required String supplierReference,
  required String amount,
  required String transactionTime,
  required String secretKey,
  required String description,
  required String returnUrl,
  required String mobileNumber,
}) async {
  try {
    print('🔵 ProcessPaymentUseCase: Starting payment processing');
    print('   Provider: ${providerType.name}');
    print('   Method: ${paymentMethod.displayName}');

    // Get gateway
    final gateway = gatewayFactory.createGateway(providerType, paymentMethod); // UPDATED
    print('✅ ProcessPaymentUseCase: Gateway obtained');

    // ... rest of the flow remains the same
  } catch (e) {
    print('🔴 ProcessPaymentUseCase: Error - $e');
    return PaymentResult.error(
      errorMessage: 'Payment processing failed: ${e.toString()}',
    );
  }
}
```

---

## Step 5: Update PaymentGatewayFactory

File: `lib/features/payment/features/confirm_payment/data/datasource/gateways/payment_gateway_factory.dart`

```dart
class PaymentGatewayFactory {
  final ConfirmPaymentRemoteDatasource remoteDataSource;

  PaymentGatewayFactory(this.remoteDataSource);

  /// Create gateway based on provider and payment method
  PaymentGateway createGateway(
    PaymentProviderType providerType,
    PaymentMethodType paymentMethod, // NEW
  ) {
    switch (providerType) {
      case PaymentProviderType.stryve:
        // Stryve only supports card
        if (paymentMethod != PaymentMethodType.card) {
          throw UnsupportedError(
            'Stryve only supports card payments',
          );
        }
        return StryvePaymentGateway(remoteDataSource);

      case PaymentProviderType.etisalatWallet:
        // Etisalat supports both card and wallet
        if (paymentMethod == PaymentMethodType.card) {
          return EtisalatCardPaymentGateway(remoteDataSource);
        } else if (paymentMethod == PaymentMethodType.etisalatWallet) {
          return EtisalatWalletPaymentGateway(remoteDataSource);
        } else {
          throw UnsupportedError(
            'Etisalat does not support ${paymentMethod.displayName}',
          );
        }

      case PaymentProviderType.qnbHC:
      case PaymentProviderType.qnbMEZA:
      case PaymentProviderType.payMOB:
        throw UnsupportedError(
          '${providerType.name} payment gateway is not yet implemented',
        );
    }
  }
}
```

---

## Step 6: Create Separate Etisalat Gateways

### EtisalatCardPaymentGateway

File: `lib/features/payment/features/confirm_payment/data/datasource/gateways/etisalat_card_payment_gateway.dart`

```dart
class EtisalatCardPaymentGateway implements PaymentGateway {
  final ConfirmPaymentRemoteDatasource remoteDataSource;

  EtisalatCardPaymentGateway(this.remoteDataSource);

  @override
  Future<String> authenticate() async {
    return '';
  }

  @override
  String generateSignature({
    required String secretKey,
    required String message,
  }) {
    final keyBytes = utf8.encode(secretKey);
    final messageBytes = utf8.encode(message);
    final hmac = Hmac(sha256, keyBytes);
    final digest = hmac.convert(messageBytes);
    return digest.toString().toUpperCase();
  }

  @override
  WebViewPaymentStrategy getExecutionStrategy() {
    return WebViewPaymentStrategy(
      createInvoiceFunc: (context) async {
        try {
          print('🔵 EtisalatCardPaymentGateway: Preparing card payment');

          // Lightbox with paymentMethodFromLightBox = 0 (Card only)
          return PaymentResult.success(
            invoiceId: context.supplierReference,
            redirectUrl: 'about:blank', // Will be handled by WebView
            invoiceResponse: null,
          );
        } catch (e) {
          return PaymentResult.error(
            errorMessage: 'Failed to prepare Etisalat card payment: $e',
          );
        }
      },
    );
  }

  // Helper for WebView: pass paymentMethod to Lightbox
  int getLightboxPaymentMethod() {
    return 0; // 0 = Card only
  }
}
```

### EtisalatWalletPaymentGateway

File: `lib/features/payment/features/confirm_payment/data/datasource/gateways/etisalat_wallet_payment_gateway.dart`

```dart
class EtisalatWalletPaymentGateway implements PaymentGateway {
  final ConfirmPaymentRemoteDatasource remoteDataSource;

  EtisalatWalletPaymentGateway(this.remoteDataSource);

  @override
  Future<String> authenticate() async {
    return '';
  }

  @override
  String generateSignature({
    required String secretKey,
    required String message,
  }) {
    final keyBytes = utf8.encode(secretKey);
    final messageBytes = utf8.encode(message);
    final hmac = Hmac(sha256, keyBytes);
    final digest = hmac.convert(messageBytes);
    return digest.toString().toUpperCase();
  }

  @override
  WebViewPaymentStrategy getExecutionStrategy() {
    return WebViewPaymentStrategy(
      createInvoiceFunc: (context) async {
        try {
          print('🔵 EtisalatWalletPaymentGateway: Preparing wallet payment');

          // Lightbox with paymentMethodFromLightBox = 1 (Digital/Wallet)
          return PaymentResult.success(
            invoiceId: context.supplierReference,
            redirectUrl: 'about:blank',
            invoiceResponse: null,
          );
        } catch (e) {
          return PaymentResult.error(
            errorMessage: 'Failed to prepare Etisalat wallet payment: $e',
          );
        }
      },
    );
  }

  // Helper for WebView: pass paymentMethod to Lightbox
  int getLightboxPaymentMethod() {
    return 1; // 1 = Digital/Wallet
  }
}
```

---

## Step 7: Create Payment Method Selection Widget

File: `lib/features/payment/features/confirm_payment/presentation/widgets/payment_method_selector.dart`

```dart
import 'package:flutter/material.dart';
import 'package:ibnsina_pharma/features/payment/features/confirm_payment/data/model/payment_gateway_config.dart';

class PaymentMethodSelector extends StatefulWidget {
  final List<PaymentGatewayConfig> availableGateways;
  final Function(PaymentGatewayConfig) onSelected;
  final PaymentGatewayConfig? initialSelection;

  const PaymentMethodSelector({
    Key? key,
    required this.availableGateways,
    required this.onSelected,
    this.initialSelection,
  }) : super(key: key);

  @override
  State<PaymentMethodSelector> createState() => _PaymentMethodSelectorState();
}

class _PaymentMethodSelectorState extends State<PaymentMethodSelector> {
  late PaymentGatewayConfig selectedGateway;

  @override
  void initState() {
    super.initState();
    selectedGateway = widget.initialSelection ?? widget.availableGateways.first;
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const Padding(
          padding: EdgeInsets.only(bottom: 16.0),
          child: Text(
            'Select Payment Method',
            style: TextStyle(
              fontSize: 18,
              fontWeight: FontWeight.bold,
            ),
          ),
        ),
        ListView.builder(
          shrinkWrap: true,
          physics: const NeverScrollableScrollPhysics(),
          itemCount: widget.availableGateways.length,
          itemBuilder: (context, index) {
            final gateway = widget.availableGateways[index];
            final isSelected = selectedGateway.id == gateway.id;

            return GestureDetector(
              onTap: () {
                setState(() => selectedGateway = gateway);
                widget.onSelected(gateway);
              },
              child: Container(
                margin: const EdgeInsets.only(bottom: 12),
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  border: Border.all(
                    color: isSelected ? Colors.blue : Colors.grey[300]!,
                    width: isSelected ? 2 : 1,
                  ),
                  borderRadius: BorderRadius.circular(8),
                  color: isSelected ? Colors.blue.withOpacity(0.1) : Colors.transparent,
                ),
                child: Row(
                  children: [
                    // Icon
                    if (gateway.icon != null)
                      Image.asset(
                        gateway.icon!,
                        width: 40,
                        height: 40,
                        errorBuilder: (context, error, stackTrace) {
                          return const Icon(Icons.payment, size: 40);
                        },
                      )
                    else
                      const Icon(Icons.payment, size: 40),
                    const SizedBox(width: 16),
                    // Info
                    Expanded(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text(
                            gateway.displayName,
                            style: const TextStyle(
                              fontSize: 16,
                              fontWeight: FontWeight.w600,
                            ),
                          ),
                          if (gateway.description != null) ...[
                            const SizedBox(height: 4),
                            Text(
                              gateway.description!,
                              style: TextStyle(
                                fontSize: 12,
                                color: Colors.grey[600],
                              ),
                            ),
                          ],
                        ],
                      ),
                    ),
                    // Radio button
                    Radio<String>(
                      value: gateway.id,
                      groupValue: selectedGateway.id,
                      onChanged: (value) {
                        setState(() => selectedGateway = gateway);
                        widget.onSelected(gateway);
                      },
                    ),
                  ],
                ),
              ),
            );
          },
        ),
      ],
    );
  }
}
```

---

## Step 8: Update ConfirmPaymentBloc

File: `lib/features/payment/features/confirm_payment/presentation/bloc/confirm_payment_bloc.dart`

Add payment method selection:

```dart
class ConfirmPaymentBloc extends Bloc<ConfirmPaymentEvent, ConfirmPaymentState> {
  final ProcessPaymentUseCase processPaymentUseCase;
  final PostPaymentResultUseCase postPaymentResultUseCase;
  final GetAvailablePaymentGatewaysUseCase getAvailablePaymentGatewaysUseCase; // NEW

  // ... constructor and other event handlers

  Future<void> _onConfirmPressed(
    ConfirmPressed event,
    Emitter<ConfirmPaymentState> emit,
  ) async {
    try {
      // ... validation code ...

      var transactionTime = DateFormat('yyyyMMddHHmmss', 'en').format(DateTime.now());

      emit(state.copyWith(status: ConfirmPaymentStatus.loading));

      final amount = NumberFormatter.parseFormattedNumber(event.confirmPaymentModel.amount!).toString();

      // UPDATED: Include payment method
      final paymentResult = await processPaymentUseCase.call(
        providerType: event.confirmPaymentModel.paymentProviderType,
        paymentMethod: event.paymentMethod, // NEW - get from event
        supplierReference: 'supplier-$transactionTime',
        amount: amount,
        transactionTime: transactionTime,
        secretKey: PaymentConfig.instance.paymentSK,
        description: '',
        returnUrl: PaymentConfig.instance.paymentReturnUrl,
        mobileNumber: event.confirmPaymentModel.phoneNumber,
      );

      // ... rest of the flow remains the same
    } catch (e) {
      // ... error handling
    }
  }
}
```

---

## Step 9: Update ConfirmPaymentEvent

File: `lib/features/payment/features/confirm_payment/presentation/bloc/confirm_payment_event.dart`

```dart
class ConfirmPressed extends ConfirmPaymentEvent {
  final ConfirmPaymentModel confirmPaymentModel;
  final PaymentMethodType paymentMethod; // NEW

  const ConfirmPressed(
    this.confirmPaymentModel,
    this.paymentMethod, // NEW
  );
}
```

---

## Step 10: Update WebView Widgets

### For Etisalat

Update both `EtisalatCardWebPaymentView` and `EtisalatWalletWebPaymentView`:

```dart
String _buildLightboxHtml() {
  final amountInt = int.parse(widget.amount);

  // Different paymentMethod value based on payment type
  final paymentMethod = isWallet ? 1 : 0; // 0=Card, 1=Digital/Wallet, 2=Both

  return '''
  <html>
  <head>
    <script src="https://upgstaging.egyptianbanks.com:8006/js/Lightbox.js"></script>
  </head>
  <body>
    <script>
      Lightbox.Checkout.configure = {
        OrderId: '${widget.orderId}',
        MID: '${widget.merchantId}',
        TID: '${widget.terminalId}',
        SecureHash: '${widget.secureHash}',
        TrxDateTime: '${widget.transactionTime}',
        AmountTrxn: $amountInt,
        MerchantReference: '${widget.merchantReference}',
        paymentMethodFromLightBox: $paymentMethod, // 0=Card, 1=Wallet, 2=Both
        // ... rest of config
      };
      Lightbox.Checkout.showLightbox();
    </script>
  </body>
  </html>
  ''';
}
```

---

## Step 11: Update Confirm Payment Page

File: `lib/features/payment/features/confirm_payment/presentation/pages/confirm_payment_page.dart`

Add payment method selection:

```dart
class ConfirmPaymentPage extends StatefulWidget {
  // ... existing properties

  @override
  State<ConfirmPaymentPage> createState() => _ConfirmPaymentPageState();
}

class _ConfirmPaymentPageState extends State<ConfirmPaymentPage> {
  late GetAvailablePaymentGatewaysUseCase getAvailablePaymentGatewaysUseCase;
  late List<PaymentGatewayConfig> availableGateways;
  late PaymentGatewayConfig selectedGateway;

  @override
  void initState() {
    super.initState();
    getAvailablePaymentGatewaysUseCase =
      context.read<GetAvailablePaymentGatewaysUseCase>();

    // Get available gateways
    availableGateways = getAvailablePaymentGatewaysUseCase.getAllAvailableGateways();
    selectedGateway = availableGateways.first;
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Payment method selector
        PaymentMethodSelector(
          availableGateways: availableGateways,
          initialSelection: selectedGateway,
          onSelected: (gateway) {
            setState(() => selectedGateway = gateway);
          },
        ),

        // Existing confirm button
        ElevatedButton(
          onPressed: () {
            context.read<ConfirmPaymentBloc>().add(
              ConfirmPressed(
                confirmPaymentModel,
                selectedGateway.paymentMethod, // Pass selected method
              ),
            );
          },
          child: const Text('Confirm Payment'),
        ),
      ],
    );
  }
}
```

---

## Complete Data Flow

```
User selects payment method
  ↓
ConfirmPaymentPage.PaymentMethodSelector
  ↓ (onSelected)
selectedGateway = PaymentGatewayConfig
  ↓
User taps "Confirm Payment"
  ↓
ConfirmPressed(model, paymentMethod)
  ↓
ConfirmPaymentBloc._onConfirmPressed()
  ↓
processPaymentUseCase.call(providerType, paymentMethod, ...)
  ↓
ProcessPaymentUseCase
  ↓
gateway = gatewayFactory.createGateway(providerType, paymentMethod)
  ↓
EtisalatCardPaymentGateway OR EtisalatWalletPaymentGateway
  ↓
getExecutionStrategy() returns WebViewPaymentStrategy
  ↓
WebView loads HTML with correct Lightbox config
  ↓
User completes payment in Lightbox
  ↓
JS callback triggered (completeCallback/errorCallback/cancelCallback)
  ↓
PostPaymentResultUseCase sends to backend
```

---

## Summary of Changes

### New Files
1. `payment_method_type.dart` - Enum for payment methods
2. `payment_gateway_config.dart` - Gateway configuration model
3. `get_available_payment_gateways_usecase.dart` - Gateway selector
4. `etisalat_card_payment_gateway.dart` - Card-only gateway
5. `etisalat_wallet_payment_gateway.dart` - Wallet-only gateway
6. `payment_method_selector.dart` - UI widget for selection

### Updated Files
1. `payment_gateway_factory.dart` - Support payment method parameter
2. `process_payment_usecase.dart` - Support payment method parameter
3. `confirm_payment_bloc.dart` - Handle payment method selection
4. `confirm_payment_event.dart` - Add payment method to events
5. `confirm_payment_page.dart` - Show payment method selector
6. `etisalat_web_payment_view.dart` - Use correct Lightbox config

---

## Testing Scenarios

- [ ] Select Stryve (Card only)
- [ ] Select Etisalat Card
- [ ] Select Etisalat Wallet
- [ ] Each opens correct payment page
- [ ] Lightbox shows correct payment method options
- [ ] Payment completes successfully for each method
- [ ] Backend receives correct payment method info