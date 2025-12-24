# Payment & Deposit System - Lower Level Design

## Overview
The Payment & Deposit System handles secure payment processing for property deposits, rent payments, and transaction management in the Nest Tanzania Rental Platform. This system integrates with local and international payment providers to support Tanzania's payment ecosystem.

## Architecture Components

### AWS Services
- **AWS AppSync**: GraphQL API for payment operations
- **AWS Lambda**: Payment processing and webhook handling
- **DynamoDB**: Transaction records and payment history
- **AWS Secrets Manager**: Payment provider credentials
- **AWS SQS**: Asynchronous payment processing
- **AWS SNS**: Payment notifications
- **AWS EventBridge**: Payment event routing

### Payment Providers Integration
- **M-Pesa (Vodacom)**: Mobile money (primary for Tanzania)
- **Tigo Pesa**: Mobile money alternative
- **Airtel Money**: Mobile money alternative
- **Stripe**: International cards and digital wallets
- **Bank Transfers**: Local bank integration (CRDB, NMB, etc.)

### Lambda Functions
- **PaymentFunction**: `nest/src/handlers/graphql-payment.ts`
- **PaymentWebhookFunction**: `nest/src/handlers/payment-webhook.ts`
- **PaymentProcessorFunction**: `nest/src/handlers/payment-processor.ts`

### Database Tables
- **payments** (Materialized View): Current payment state
- **payment-events** (Event Store): Immutable payment event history
- **transactions**: Individual transaction records
- **payment-methods**: User saved payment methods
- **refunds**: Refund processing records

## Data Models

### Payment Types & Status
```typescript
enum PaymentType {
  DEPOSIT = 'DEPOSIT'
  RENT = 'RENT'
  SERVICE_FEE = 'SERVICE_FEE'
  SECURITY_DEPOSIT = 'SECURITY_DEPOSIT'
  REFUND = 'REFUND'
}

enum PaymentStatus {
  PENDING = 'PENDING'
  PROCESSING = 'PROCESSING'
  COMPLETED = 'COMPLETED'
  FAILED = 'FAILED'
  CANCELLED = 'CANCELLED'
  REFUNDED = 'REFUNDED'
  PARTIALLY_REFUNDED = 'PARTIALLY_REFUNDED'
}

enum PaymentMethod {
  MPESA = 'MPESA'
  TIGO_PESA = 'TIGO_PESA'
  AIRTEL_MONEY = 'AIRTEL_MONEY'
  CREDIT_CARD = 'CREDIT_CARD'
  DEBIT_CARD = 'DEBIT_CARD'
  BANK_TRANSFER = 'BANK_TRANSFER'
  DIGITAL_WALLET = 'DIGITAL_WALLET'
}

enum Currency {
  TZS = 'TZS'  // Tanzanian Shilling
  USD = 'USD'  // US Dollar
}
```

### Core Payment Models
```typescript
interface Payment {
  paymentId: ID
  applicationId?: ID
  propertyId: ID
  payerId: ID
  payeeId: ID
  paymentType: PaymentType
  amount: PaymentAmount
  paymentMethod: PaymentMethod
  status: PaymentStatus
  description: string
  metadata: PaymentMetadata
  providerData: ProviderPaymentData
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
  completedAt?: AWSDateTime
  expiresAt?: AWSDateTime
}

interface PaymentAmount {
  amount: number
  currency: Currency
  originalAmount?: number
  originalCurrency?: Currency
  exchangeRate?: number
  fees: PaymentFees
}

interface PaymentFees {
  platformFee: number
  processingFee: number
  providerFee: number
  totalFees: number
}

interface PaymentMetadata {
  userAgent?: string
  ipAddress?: string
  deviceId?: string
  location?: string
  referenceNumber?: string
  notes?: string
}

interface ProviderPaymentData {
  providerId: string
  providerTransactionId?: string
  providerReference?: string
  providerStatus?: string
  providerResponse?: any
  webhookData?: any
}
```

### Transaction Models
```typescript
interface Transaction {
  transactionId: ID
  paymentId: ID
  type: TransactionType
  amount: number
  currency: Currency
  status: TransactionStatus
  providerTransactionId?: string
  createdAt: AWSDateTime
  processedAt?: AWSDateTime
}

enum TransactionType {
  CHARGE = 'CHARGE'
  REFUND = 'REFUND'
  TRANSFER = 'TRANSFER'
  FEE = 'FEE'
}

enum TransactionStatus {
  PENDING = 'PENDING'
  SUCCESS = 'SUCCESS'
  FAILED = 'FAILED'
}
```

### Payment Method Models
```typescript
interface SavedPaymentMethod {
  paymentMethodId: ID
  userId: ID
  type: PaymentMethod
  isDefault: boolean
  metadata: PaymentMethodMetadata
  createdAt: AWSDateTime
  lastUsedAt?: AWSDateTime
}

interface PaymentMethodMetadata {
  // M-Pesa
  phoneNumber?: string
  
  // Card
  last4?: string
  brand?: string
  expiryMonth?: number
  expiryYear?: number
  
  // Bank
  bankName?: string
  accountNumber?: string
  
  // Common
  nickname?: string
  isVerified: boolean
}
```

## API Operations

### Payment Initiation

#### Initiate Deposit Payment
```graphql
mutation InitiateDepositPayment($input: InitiatePaymentInput!) {
  initiateDepositPayment(input: $input) {
    paymentId
    status
    amount {
      amount
      currency
      fees {
        platformFee
        processingFee
        totalFees
      }
    }
    paymentUrl
    qrCode
    instructions
    expiresAt
  }
}
```

**Input:**
```typescript
interface InitiatePaymentInput {
  applicationId: ID
  propertyId: ID
  paymentType: PaymentType
  amount: number
  currency: Currency
  paymentMethod: PaymentMethod
  paymentMethodId?: ID // For saved payment methods
  phoneNumber?: string // For mobile money
  returnUrl?: string
  metadata?: PaymentMetadataInput
}
```

**Implementation Flow:**
1. Validate application and property
2. Calculate fees and total amount
3. Create payment record with PENDING status
4. Initialize payment with provider
5. Generate payment instructions/URL
6. Return payment details to client
7. Set up payment monitoring

#### Process Payment
```graphql
mutation ProcessPayment($paymentId: ID!, $input: ProcessPaymentInput!) {
  processPayment(paymentId: $paymentId, input: $input) {
    paymentId
    status
    transactionId
    completedAt
    receipt {
      receiptNumber
      downloadUrl
    }
  }
}
```

**Input:**
```typescript
interface ProcessPaymentInput {
  providerTransactionId?: string
  confirmationCode?: string
  additionalData?: AWSJSON
}
```

### Payment Management

#### Get Payment Details
```graphql
query GetPayment($paymentId: ID!) {
  getPayment(paymentId: $paymentId) {
    paymentId
    applicationId
    propertyId
    amount {
      amount
      currency
      fees {
        totalFees
      }
    }
    status
    paymentMethod
    description
    createdAt
    completedAt
    transactions {
      transactionId
      type
      amount
      status
      createdAt
    }
  }
}
```

#### List User Payments
```graphql
query ListUserPayments($userId: ID!, $status: PaymentStatus, $limit: Int, $nextToken: String) {
  listUserPayments(userId: $userId, status: $status, limit: $limit, nextToken: $nextToken) {
    payments {
      paymentId
      propertyId
      amount {
        amount
        currency
      }
      status
      paymentMethod
      createdAt
    }
    nextToken
    count
  }
}
```

### Refund Management

#### Initiate Refund
```graphql
mutation InitiateRefund($input: InitiateRefundInput!) {
  initiateRefund(input: $input) {
    refundId
    paymentId
    amount {
      amount
      currency
    }
    status
    estimatedCompletionDate
    reason
  }
}
```

**Input:**
```typescript
interface InitiateRefundInput {
  paymentId: ID
  amount?: number // Partial refund amount
  reason: RefundReason
  notes?: string
}

enum RefundReason {
  CANCELLED_APPLICATION = 'CANCELLED_APPLICATION'
  PROPERTY_UNAVAILABLE = 'PROPERTY_UNAVAILABLE'
  LANDLORD_CANCELLATION = 'LANDLORD_CANCELLATION'
  DUPLICATE_PAYMENT = 'DUPLICATE_PAYMENT'
  FRAUDULENT_TRANSACTION = 'FRAUDULENT_TRANSACTION'
  OTHER = 'OTHER'
}
```

### Payment Methods Management

#### Save Payment Method
```graphql
mutation SavePaymentMethod($input: SavePaymentMethodInput!) {
  savePaymentMethod(input: $input) {
    paymentMethodId
    type
    metadata {
      last4
      phoneNumber
      nickname
    }
    isDefault
  }
}
```

#### List Saved Payment Methods
```graphql
query ListPaymentMethods($userId: ID!) {
  listPaymentMethods(userId: $userId) {
    paymentMethodId
    type
    metadata {
      last4
      phoneNumber
      nickname
      isVerified
    }
    isDefault
    lastUsedAt
  }
}
```

## Service Layer Implementation

### File Structure
```
nest/src/service/handlers/payment/
├── PaymentInitiationService.ts
├── PaymentProcessingService.ts
├── PaymentMethodService.ts
├── RefundService.ts
├── PaymentProviderService.ts
├── PaymentValidationService.ts
├── PaymentNotificationService.ts
└── providers/
    ├── MPesaProvider.ts
    ├── TigoPesaProvider.ts
    ├── StripeProvider.ts
    └── BankTransferProvider.ts
```

### Key Service Classes

#### PaymentInitiationService
```typescript
class PaymentInitiationService {
  constructor(
    private paymentDAO: PaymentDAO,
    private paymentEventsDAO: PaymentEventsDAO,
    private providerService: PaymentProviderService,
    private validationService: PaymentValidationService,
    private feeCalculator: PaymentFeeCalculator
  ) {}

  async initiateDepositPayment(input: InitiatePaymentInput): Promise<PaymentInitiationResponse> {
    // 1. Validate application and property
    await this.validationService.validatePaymentRequest(input);

    // 2. Calculate fees
    const fees = await this.feeCalculator.calculateFees(input.amount, input.paymentMethod);
    const totalAmount = input.amount + fees.totalFees;

    // 3. Create payment record
    const paymentId = generatePaymentId();
    const payment: Payment = {
      paymentId,
      applicationId: input.applicationId,
      propertyId: input.propertyId,
      payerId: input.userId,
      payeeId: await this.getPropertyLandlord(input.propertyId),
      paymentType: input.paymentType,
      amount: {
        amount: input.amount,
        currency: input.currency,
        fees
      },
      paymentMethod: input.paymentMethod,
      status: PaymentStatus.PENDING,
      description: `${input.paymentType} payment for property ${input.propertyId}`,
      metadata: input.metadata || {},
      providerData: { providerId: this.getProviderId(input.paymentMethod) },
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      expiresAt: new Date(Date.now() + 30 * 60 * 1000).toISOString() // 30 minutes
    };

    // 4. Save payment
    await this.paymentDAO.createPayment(payment);

    // 5. Create payment event
    await this.paymentEventsDAO.createPaymentEvent({
      eventId: generateUUID(),
      paymentId,
      eventType: PaymentEventType.PAYMENT_INITIATED,
      eventData: payment,
      timestamp: new Date().toISOString(),
      version: 1
    });

    // 6. Initialize with payment provider
    const providerResponse = await this.providerService.initiatePayment(payment);

    // 7. Update payment with provider data
    await this.updatePaymentProviderData(paymentId, providerResponse);

    return {
      paymentId,
      status: PaymentStatus.PENDING,
      amount: payment.amount,
      paymentUrl: providerResponse.paymentUrl,
      qrCode: providerResponse.qrCode,
      instructions: providerResponse.instructions,
      expiresAt: payment.expiresAt
    };
  }

  private async getPropertyLandlord(propertyId: string): Promise<string> {
    const property = await this.propertyDAO.getPropertyById(propertyId);
    return property?.landlordId || '';
  }
}
```

#### PaymentProcessingService
```typescript
class PaymentProcessingService {
  constructor(
    private paymentDAO: PaymentDAO,
    private paymentEventsDAO: PaymentEventsDAO,
    private transactionDAO: TransactionDAO,
    private providerService: PaymentProviderService,
    private notificationService: PaymentNotificationService
  ) {}

  async processPayment(paymentId: string, input: ProcessPaymentInput): Promise<PaymentProcessingResponse> {
    // 1. Get payment record
    const payment = await this.paymentDAO.getPaymentById(paymentId);
    if (!payment) {
      throw new Error('Payment not found');
    }

    if (payment.status !== PaymentStatus.PENDING) {
      throw new Error(`Payment cannot be processed in ${payment.status} status`);
    }

    try {
      // 2. Update status to processing
      await this.updatePaymentStatus(paymentId, PaymentStatus.PROCESSING);

      // 3. Verify payment with provider
      const verificationResult = await this.providerService.verifyPayment(
        payment.paymentMethod,
        payment.providerData.providerId,
        input.providerTransactionId
      );

      if (verificationResult.success) {
        // 4. Create successful transaction
        const transaction = await this.createTransaction({
          paymentId,
          type: TransactionType.CHARGE,
          amount: payment.amount.amount,
          currency: payment.amount.currency,
          providerTransactionId: input.providerTransactionId,
          status: TransactionStatus.SUCCESS
        });

        // 5. Update payment to completed
        await this.updatePaymentStatus(paymentId, PaymentStatus.COMPLETED, {
          completedAt: new Date().toISOString(),
          providerTransactionId: input.providerTransactionId
        });

        // 6. Process platform fees
        await this.processPlatformFees(payment);

        // 7. Update application status
        await this.updateApplicationPaymentStatus(payment.applicationId, 'DEPOSIT_PAID');

        // 8. Send notifications
        await this.notificationService.sendPaymentConfirmation(payment);

        // 9. Generate receipt
        const receipt = await this.generateReceipt(payment, transaction);

        return {
          paymentId,
          status: PaymentStatus.COMPLETED,
          transactionId: transaction.transactionId,
          completedAt: new Date().toISOString(),
          receipt
        };
      } else {
        // Payment failed
        await this.updatePaymentStatus(paymentId, PaymentStatus.FAILED);
        await this.notificationService.sendPaymentFailure(payment, verificationResult.error);

        throw new Error(`Payment verification failed: ${verificationResult.error}`);
      }
    } catch (error) {
      // Handle processing errors
      await this.updatePaymentStatus(paymentId, PaymentStatus.FAILED);
      await this.notificationService.sendPaymentError(payment, error.message);
      throw error;
    }
  }

  private async processPlatformFees(payment: Payment): Promise<void> {
    // Create fee transactions
    const feeTransaction = await this.createTransaction({
      paymentId: payment.paymentId,
      type: TransactionType.FEE,
      amount: payment.amount.fees.platformFee,
      currency: payment.amount.currency,
      status: TransactionStatus.SUCCESS
    });

    // Transfer to platform account
    await this.providerService.transferFunds(
      payment.paymentMethod,
      payment.amount.fees.platformFee,
      'PLATFORM_ACCOUNT'
    );
  }
}
```

#### PaymentProviderService
```typescript
class PaymentProviderService {
  constructor(
    private mpesaProvider: MPesaProvider,
    private tigoPesaProvider: TigoPesaProvider,
    private stripeProvider: StripeProvider,
    private bankTransferProvider: BankTransferProvider
  ) {}

  async initiatePayment(payment: Payment): Promise<ProviderInitiationResponse> {
    const provider = this.getProvider(payment.paymentMethod);
    return provider.initiatePayment(payment);
  }

  async verifyPayment(
    method: PaymentMethod,
    providerId: string,
    transactionId?: string
  ): Promise<ProviderVerificationResponse> {
    const provider = this.getProvider(method);
    return provider.verifyPayment(providerId, transactionId);
  }

  async processRefund(
    payment: Payment,
    refundAmount: number,
    reason: string
  ): Promise<ProviderRefundResponse> {
    const provider = this.getProvider(payment.paymentMethod);
    return provider.processRefund(payment.providerData.providerTransactionId, refundAmount, reason);
  }

  private getProvider(method: PaymentMethod): PaymentProvider {
    switch (method) {
      case PaymentMethod.MPESA:
        return this.mpesaProvider;
      case PaymentMethod.TIGO_PESA:
        return this.tigoPesaProvider;
      case PaymentMethod.CREDIT_CARD:
      case PaymentMethod.DEBIT_CARD:
        return this.stripeProvider;
      case PaymentMethod.BANK_TRANSFER:
        return this.bankTransferProvider;
      default:
        throw new Error(`Unsupported payment method: ${method}`);
    }
  }
}
```

### Payment Provider Implementations

#### MPesaProvider
```typescript
class MPesaProvider implements PaymentProvider {
  constructor(
    private httpClient: HttpClient,
    private secretsManager: SecretsManager
  ) {}

  async initiatePayment(payment: Payment): Promise<ProviderInitiationResponse> {
    const credentials = await this.getCredentials();
    
    const request = {
      BusinessShortCode: credentials.shortCode,
      Password: this.generatePassword(credentials),
      Timestamp: this.getTimestamp(),
      TransactionType: 'CustomerPayBillOnline',
      Amount: payment.amount.amount,
      PartyA: payment.metadata.phoneNumber,
      PartyB: credentials.shortCode,
      PhoneNumber: payment.metadata.phoneNumber,
      CallBackURL: `${process.env.API_URL}/webhooks/mpesa`,
      AccountReference: payment.paymentId,
      TransactionDesc: payment.description
    };

    const response = await this.httpClient.post(
      'https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest',
      request,
      {
        headers: {
          'Authorization': `Bearer ${await this.getAccessToken()}`,
          'Content-Type': 'application/json'
        }
      }
    );

    return {
      success: response.ResponseCode === '0',
      providerId: response.CheckoutRequestID,
      paymentUrl: null, // M-Pesa uses STK push
      qrCode: null,
      instructions: `Please check your phone for M-Pesa payment request and enter your PIN to complete the payment of TZS ${payment.amount.amount}`,
      error: response.ResponseCode !== '0' ? response.ResponseDescription : null
    };
  }

  async verifyPayment(checkoutRequestId: string, transactionId?: string): Promise<ProviderVerificationResponse> {
    const credentials = await this.getCredentials();
    
    const request = {
      BusinessShortCode: credentials.shortCode,
      Password: this.generatePassword(credentials),
      Timestamp: this.getTimestamp(),
      CheckoutRequestID: checkoutRequestId
    };

    const response = await this.httpClient.post(
      'https://sandbox.safaricom.co.ke/mpesa/stkpushquery/v1/query',
      request,
      {
        headers: {
          'Authorization': `Bearer ${await this.getAccessToken()}`,
          'Content-Type': 'application/json'
        }
      }
    );

    return {
      success: response.ResultCode === '0',
      status: this.mapMpesaStatus(response.ResultCode),
      transactionId: response.MpesaReceiptNumber,
      amount: response.Amount,
      error: response.ResultCode !== '0' ? response.ResultDesc : null
    };
  }

  private async getAccessToken(): Promise<string> {
    const credentials = await this.getCredentials();
    const auth = Buffer.from(`${credentials.consumerKey}:${credentials.consumerSecret}`).toString('base64');
    
    const response = await this.httpClient.get(
      'https://sandbox.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials',
      {
        headers: {
          'Authorization': `Basic ${auth}`
        }
      }
    );

    return response.access_token;
  }
}
```

#### StripeProvider
```typescript
class StripeProvider implements PaymentProvider {
  private stripe: Stripe;

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      apiVersion: '2023-10-16'
    });
  }

  async initiatePayment(payment: Payment): Promise<ProviderInitiationResponse> {
    try {
      const paymentIntent = await this.stripe.paymentIntents.create({
        amount: Math.round(payment.amount.amount * 100), // Convert to cents
        currency: payment.amount.currency.toLowerCase(),
        payment_method_types: ['card'],
        metadata: {
          paymentId: payment.paymentId,
          propertyId: payment.propertyId,
          applicationId: payment.applicationId || ''
        },
        description: payment.description
      });

      return {
        success: true,
        providerId: paymentIntent.id,
        paymentUrl: null,
        qrCode: null,
        instructions: 'Please complete payment using your card details',
        clientSecret: paymentIntent.client_secret
      };
    } catch (error) {
      return {
        success: false,
        providerId: '',
        paymentUrl: null,
        qrCode: null,
        instructions: '',
        error: error.message
      };
    }
  }

  async verifyPayment(paymentIntentId: string): Promise<ProviderVerificationResponse> {
    try {
      const paymentIntent = await this.stripe.paymentIntents.retrieve(paymentIntentId);

      return {
        success: paymentIntent.status === 'succeeded',
        status: this.mapStripeStatus(paymentIntent.status),
        transactionId: paymentIntent.charges.data[0]?.id,
        amount: paymentIntent.amount / 100,
        error: paymentIntent.status === 'failed' ? 'Payment failed' : null
      };
    } catch (error) {
      return {
        success: false,
        status: 'FAILED',
        transactionId: null,
        amount: 0,
        error: error.message
      };
    }
  }
}
```

## Data Access Layer (DAO)

### PaymentDAO
```typescript
class PaymentDAO extends BaseDAO {
  async createPayment(payment: Payment): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.paymentsTableName,
      Item: marshall(payment),
      ConditionExpression: 'attribute_not_exists(paymentId)'
    });
  }

  async getPaymentById(paymentId: string): Promise<Payment | null> {
    const result = await this.dynamoClient.getItem({
      TableName: this.paymentsTableName,
      Key: marshall({ paymentId })
    });
    return result.Item ? unmarshall(result.Item) as Payment : null;
  }

  async updatePaymentStatus(
    paymentId: string, 
    status: PaymentStatus, 
    additionalData?: Partial<Payment>
  ): Promise<void> {
    const updateData = {
      status,
      updatedAt: new Date().toISOString(),
      ...additionalData
    };

    const updateExpression = Object.keys(updateData)
      .map(key => `#${key} = :${key}`)
      .join(', ');

    await this.dynamoClient.updateItem({
      TableName: this.paymentsTableName,
      Key: marshall({ paymentId }),
      UpdateExpression: `SET ${updateExpression}`,
      ExpressionAttributeNames: Object.keys(updateData)
        .reduce((acc, key) => ({ ...acc, [`#${key}`]: key }), {}),
      ExpressionAttributeValues: marshall(
        Object.entries(updateData).reduce((acc, [key, value]) => ({ ...acc, [`:${key}`]: value }), {})
      )
    });
  }

  async getPaymentsByUser(
    userId: string, 
    status?: PaymentStatus, 
    limit?: number, 
    nextToken?: string
  ): Promise<PaymentListResponse> {
    let keyCondition = 'payerId = :userId';
    const expressionValues: any = { ':userId': userId };

    if (status) {
      keyCondition += ' AND #status = :status';
      expressionValues[':status'] = status;
    }

    const result = await this.dynamoClient.query({
      TableName: this.paymentsTableName,
      IndexName: 'payerId-createdAt-index',
      KeyConditionExpression: keyCondition,
      ExpressionAttributeNames: status ? { '#status': 'status' } : undefined,
      ExpressionAttributeValues: marshall(expressionValues),
      Limit: limit,
      ExclusiveStartKey: nextToken ? JSON.parse(Buffer.from(nextToken, 'base64').toString()) : undefined,
      ScanIndexForward: false // Most recent first
    });

    return {
      payments: result.Items?.map(item => unmarshall(item) as Payment) || [],
      nextToken: result.LastEvaluatedKey ? 
        Buffer.from(JSON.stringify(result.LastEvaluatedKey)).toString('base64') : undefined,
      count: result.Count || 0
    };
  }
}
```

### TransactionDAO
```typescript
class TransactionDAO extends BaseDAO {
  async createTransaction(transaction: Transaction): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.transactionsTableName,
      Item: marshall({
        ...transaction,
        transactionId: transaction.transactionId || generateTransactionId(),
        createdAt: new Date().toISOString()
      })
    });
  }

  async getTransactionsByPayment(paymentId: string): Promise<Transaction[]> {
    const result = await this.dynamoClient.query({
      TableName: this.transactionsTableName,
      IndexName: 'paymentId-createdAt-index',
      KeyConditionExpression: 'paymentId = :paymentId',
      ExpressionAttributeValues: marshall({ ':paymentId': paymentId })
    });

    return result.Items?.map(item => unmarshall(item) as Transaction) || [];
  }
}
```

## Webhook Handling

### Payment Webhook Handler
```typescript
// nest/src/handlers/payment-webhook.ts
export const paymentWebhookHandler = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
  const provider = event.pathParameters?.provider;
  const body = JSON.parse(event.body || '{}');

  try {
    switch (provider) {
      case 'mpesa':
        await handleMpesaWebhook(body);
        break;
      case 'stripe':
        await handleStripeWebhook(body, event.headers);
        break;
      case 'tigo':
        await handleTigoWebhook(body);
        break;
      default:
        throw new Error(`Unknown payment provider: ${provider}`);
    }

    return {
      statusCode: 200,
      body: JSON.stringify({ success: true })
    };
  } catch (error) {
    console.error('Webhook processing error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

async function handleMpesaWebhook(body: any): Promise<void> {
  const { Body } = body;
  const { stkCallback } = Body;
  
  if (stkCallback) {
    const checkoutRequestId = stkCallback.CheckoutRequestID;
    const resultCode = stkCallback.ResultCode;
    
    // Find payment by provider ID
    const payment = await paymentDAO.getPaymentByProviderId(checkoutRequestId);
    if (!payment) return;

    if (resultCode === 0) {
      // Payment successful
      const callbackMetadata = stkCallback.CallbackMetadata?.Item || [];
      const mpesaReceiptNumber = callbackMetadata.find((item: any) => 
        item.Name === 'MpesaReceiptNumber'
      )?.Value;

      await paymentProcessingService.processPayment(payment.paymentId, {
        providerTransactionId: mpesaReceiptNumber
      });
    } else {
      // Payment failed
      await paymentDAO.updatePaymentStatus(payment.paymentId, PaymentStatus.FAILED);
    }
  }
}
```

## Security & Compliance

### PCI DSS Compliance
- No card data stored in application
- All card processing through Stripe (PCI compliant)
- Secure token-based card storage

### Data Encryption
```typescript
class PaymentEncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly keyId = process.env.PAYMENT_ENCRYPTION_KEY_ID;

  async encryptSensitiveData(data: any): Promise<string> {
    const key = await this.getEncryptionKey();
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher(this.algorithm, key);
    
    let encrypted = cipher.update(JSON.stringify(data), 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return `${iv.toString('hex')}:${encrypted}`;
  }

  async decryptSensitiveData(encryptedData: string): Promise<any> {
    const [ivHex, encrypted] = encryptedData.split(':');
    const key = await this.getEncryptionKey();
    const iv = Buffer.from(ivHex, 'hex');
    
    const decipher = crypto.createDecipher(this.algorithm, key);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return JSON.parse(decrypted);
  }
}
```

### Fraud Detection
```typescript
class FraudDetectionService {
  async analyzePayment(payment: Payment): Promise<FraudAnalysisResult> {
    const riskFactors = [];
    let riskScore = 0;

    // Check payment velocity
    const recentPayments = await this.getRecentPayments(payment.payerId, 24); // 24 hours
    if (recentPayments.length > 5) {
      riskFactors.push('HIGH_VELOCITY');
      riskScore += 30;
    }

    // Check amount patterns
    if (payment.amount.amount > 1000000) { // > 1M TZS
      riskFactors.push('HIGH_AMOUNT');
      riskScore += 20;
    }

    // Check location consistency
    const locationRisk = await this.analyzeLocationRisk(payment);
    riskScore += locationRisk.score;
    riskFactors.push(...locationRisk.factors);

    return {
      riskScore,
      riskLevel: this.calculateRiskLevel(riskScore),
      riskFactors,
      recommendation: this.getRecommendation(riskScore)
    };
  }

  private calculateRiskLevel(score: number): 'LOW' | 'MEDIUM' | 'HIGH' {
    if (score < 30) return 'LOW';
    if (score < 70) return 'MEDIUM';
    return 'HIGH';
  }
}
```

## Performance & Monitoring

### Payment Metrics
```typescript
interface PaymentMetrics {
  totalPayments: number
  successRate: number
  averageProcessingTime: number
  paymentMethodDistribution: { [method: string]: number }
  revenueByPeriod: { [period: string]: number }
  refundRate: number
  fraudDetectionRate: number
}
```

### CloudWatch Dashboards
- Payment success/failure rates by provider
- Processing time percentiles
- Revenue tracking
- Fraud detection alerts
- Webhook delivery success rates

### Error Handling & Retry Logic
```typescript
class PaymentRetryService {
  private readonly maxRetries = 3;
  private readonly retryDelays = [1000, 5000, 15000]; // ms

  async executeWithRetry<T>(
    operation: () => Promise<T>,
    paymentId: string,
    operationType: string
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (attempt < this.maxRetries - 1) {
          await this.delay(this.retryDelays[attempt]);
          await this.logRetryAttempt(paymentId, operationType, attempt + 1, error);
        }
      }
    }

    await this.logRetryFailure(paymentId, operationType, lastError!);
    throw lastError!;
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

This payment system provides comprehensive support for Tanzania's payment ecosystem while maintaining security, compliance, and reliability standards required for financial transactions.