# Saga Transaction Registry for OrchestrPay Payment Process

## Transaction Overview

Based on the payment process description, the following transactions have been identified for the Saga orchestration:

| Transaction               | Description                                                    | Type          | Compensation            |
|---------------------------|----------------------------------------------------------------|---------------|-------------------------|
| CREATE_PAYMENT            | Create payment record in the system and reserve transaction ID | Compensatable | CANCEL_PAYMENT          |
| DEBIT_CUSTOMER_ACCOUNT    | Debit money from customer's account                            | Compensatable | REFUND_CUSTOMER_ACCOUNT |
| FRAUD_CHECK_REQUEST       | Submit transaction to FraudCheck service for analysis          | Retriable     | -                       |
| AWAIT_FRAUD_DECISION      | Wait for fraud check result (automatic/manual review/timeout)  | Retriable     | -                       |
| CREDIT_MERCHANT_ACCOUNT   | Transfer money to the merchant/counterparty account            | Pivot         | -                       |
| SEND_SUCCESS_NOTIFICATION | Send success notification to customer                          | Retriable     | -                       |
| SEND_FRAUD_ALERT          | Send alert to security system about suspicious transaction     | Retriable     | -                       |
| SEND_FAILURE_NOTIFICATION | Send failure/refund notification to customer                   | Retriable     | -                       |
| CANCEL_PAYMENT            | Cancel payment record and release transaction ID               | Compensating  | -                       |
| REFUND_CUSTOMER_ACCOUNT   | Return money to customer's account                             | Compensating  | -                       |

## Transaction Details

### Forward Transactions

#### CREATE_PAYMENT

- **Type**: Compensatable
- **Description**: Creates a payment record in the Payment Service database with status "PENDING" and reserves a unique
  transaction ID. This is the entry point of the saga.
- **Compensation**: CANCEL_PAYMENT
- **Rationale**: This operation can be compensated by canceling the payment record if subsequent steps fail.

#### DEBIT_CUSTOMER_ACCOUNT

- **Type**: Compensatable
- **Description**: Debits the specified amount from the customer's account. The account balance is decreased and the
  transaction is recorded.
- **Compensation**: REFUND_CUSTOMER_ACCOUNT
- **Rationale**: This is a critical financial operation that must be compensated if fraud check fails or any subsequent
  step encounters an error. Money cannot remain in limbo.

#### FRAUD_CHECK_REQUEST

- **Type**: Retriable
- **Description**: Sends the transaction details to the FraudCheck Service for analysis. The service checks against
  internal rules and external providers (AML, GeoRisk, etc.).
- **Compensation**: None (retriable)
- **Rationale**: This is an idempotent request operation that can be retried with timeout and circuit breaker patterns.
  It doesn't change state in external systems.

#### AWAIT_FRAUD_DECISION

- **Type**: Retriable
- **Description**: Waits for the fraud check result. Can receive three responses: APPROVED, REJECTED, or MANUAL_REVIEW.
  For manual review, waits up to 20 minutes for operator decision. Implements cut-off-time logic - if no response within
  timeout, transaction is approved by default.
- **Compensation**: None (retriable)
- **Rationale**: This is a polling/waiting operation that can be retried. Uses saga state machine to maintain
  transaction state during manual review periods.

#### CREDIT_MERCHANT_ACCOUNT

- **Type**: Pivot
- **Description**: Transfers money to the merchant/counterparty account. This is the point of no return in the saga.
- **Compensation**: None (pivot transaction)
- **Rationale**: This is the pivot point of the saga. Once money is credited to the merchant, we cannot automatically
  reverse it through saga compensation. If fraud is detected post-factum, a separate refund process must be initiated
  through business procedures. All compensatable transactions must complete before this step.

#### SEND_SUCCESS_NOTIFICATION

- **Type**: Retriable
- **Description**: Sends notification to the customer via Notification Service confirming successful payment completion.
- **Compensation**: None (retriable)
- **Rationale**: Notification is a non-critical operation that can be retried. Even if it fails, the payment is
  complete. Can use eventual consistency for delivery.

### Compensation Transactions

#### CANCEL_PAYMENT

- **Type**: Compensating
- **Description**: Updates payment record status to "CANCELLED" and releases the reserved transaction ID. Logs the
  cancellation reason.
- **Rationale**: Compensates CREATE_PAYMENT by cleaning up the payment record when the saga fails before money movement.

#### REFUND_CUSTOMER_ACCOUNT

- **Type**: Compensating
- **Description**: Credits back the debited amount to the customer's account. Creates a refund transaction record linked
  to the original payment.
- **Rationale**: Compensates DEBIT_CUSTOMER_ACCOUNT by returning money to the customer when fraud check fails or system
  errors occur.

### Alert/Notification Transactions

#### SEND_FRAUD_ALERT

- **Type**: Retriable
- **Description**: Sends alert to the security system when a transaction is identified as fraudulent, including
  transaction details for analysis.
- **Compensation**: None (retriable)
- **Rationale**: This is a notification operation that should be retried until successful. It's important for security
  monitoring but doesn't affect transaction outcome.

#### SEND_FAILURE_NOTIFICATION

- **Type**: Retriable
- **Description**: Sends notification to customer informing about payment failure or refund, including the reason (fraud
  detection, system error, etc.).
- **Compensation**: None (retriable)
- **Rationale**: Customer notification about failure/refund. Can be retried with idempotency to ensure delivery.

## Saga Flow Patterns

### Happy Path (Approved Transaction)

1. CREATE_PAYMENT
2. DEBIT_CUSTOMER_ACCOUNT
3. FRAUD_CHECK_REQUEST
4. AWAIT_FRAUD_DECISION → APPROVED
5. CREDIT_MERCHANT_ACCOUNT (pivot)
6. SEND_SUCCESS_NOTIFICATION

### Fraud Detected Path

1. CREATE_PAYMENT
2. DEBIT_CUSTOMER_ACCOUNT
3. FRAUD_CHECK_REQUEST
4. AWAIT_FRAUD_DECISION → REJECTED
5. **Compensation Flow**:
    - REFUND_CUSTOMER_ACCOUNT (compensate step 2)
    - CANCEL_PAYMENT (compensate step 1)
6. SEND_FRAUD_ALERT
7. SEND_FAILURE_NOTIFICATION

### Manual Review Path

1. CREATE_PAYMENT
2. DEBIT_CUSTOMER_ACCOUNT
3. FRAUD_CHECK_REQUEST
4. AWAIT_FRAUD_DECISION → MANUAL_REVIEW
5. **Wait state** (up to 20 minutes)
6. **Manual decision received**:
    - If APPROVED → Continue to CREDIT_MERCHANT_ACCOUNT
    - If REJECTED → Execute compensation flow

### Timeout/Cut-off Path

1. CREATE_PAYMENT
2. DEBIT_CUSTOMER_ACCOUNT
3. FRAUD_CHECK_REQUEST
4. AWAIT_FRAUD_DECISION → TIMEOUT (cut-off-time exceeded)
5. **Default to APPROVED**
6. CREDIT_MERCHANT_ACCOUNT (pivot)
7. SEND_SUCCESS_NOTIFICATION

## Key Architectural Considerations

### Resilience Patterns Applied

1. **Retry Policy**: Applied to FRAUD_CHECK_REQUEST, AWAIT_FRAUD_DECISION, and all notification transactions with
   exponential backoff
2. **Circuit Breaker**: Protects FraudCheck Service calls to prevent cascade failures
3. **Bulkhead**: Isolates resources for fraud check operations to prevent resource exhaustion
4. **Rate Limiter**: Controls request rate to external fraud check providers
5. **Transactional Outbox**: Ensures reliable event publishing for saga state changes and compensations

### State Management

- Saga state stored in PostgreSQL for durability
- Redis caches intermediate results (fraud check responses, limits) for fast lookups
- State machine tracks: PENDING → DEBITED → FRAUD_CHECKING → FRAUD_APPROVED/FRAUD_REJECTED → COMPLETED/REFUNDED

### Idempotency

- All transactions use idempotency keys (transaction ID) to safely handle retries
- DEBIT/REFUND operations check for duplicate processing
- Notifications use deduplication to prevent multiple deliveries

### Monitoring & Traceability

- Each saga instance has unique correlation ID for distributed tracing
- State transitions logged for audit and debugging
- Metrics collected for each step duration and failure rates
- Alerts configured for stuck sagas (exceeding SLA) and high compensation rates

## Implementation Notes for Orchestrator

### Temporal Implementation (Primary)

- Use Temporal Workflows for saga orchestration
- Activities for each transaction step
- Compensation activities automatically triggered on failure
- Built-in retry and timeout handling
- Durable timers for manual review waiting period
- Parallel execution where possible (e.g., notifications)

### Camunda Implementation (Alternative)

- BPMN process definition with compensation events
- Service tasks for each transaction
- Boundary events for timeouts and errors
- Compensation handlers attached to activities
- Message correlation for manual review responses
- Parallel gateways for concurrent operations
