# Payment State Machine - State Transition Table

## Overview

This document describes the complete state machine for payment transactions in the OrchestrPay platform. The state
machine manages the lifecycle of a payment from creation through completion or cancellation, including fraud check
processing, manual review, and compensation flows.

## State Transition Table

| Current State         | Next State                   | Event                                  |
|-----------------------|------------------------------|----------------------------------------|
| -                     | CREATED                      | PAYMENT_CREATED                        |
| CREATED               | DEBITED                      | CUSTOMER_DEBITED                       |
| CREATED               | CANCELLED                    | PAYMENT_CREATION_FAILED                |
| DEBITED               | FRAUD_CHECKING               | FRAUD_CHECK_INITIATED                  |
| DEBITED               | REFUNDING                    | DEBIT_COMPENSATION_REQUIRED            |
| FRAUD_CHECKING        | FRAUD_APPROVED               | FRAUD_CHECK_PASSED                     |
| FRAUD_CHECKING        | FRAUD_REJECTED               | FRAUD_CHECK_FAILED                     |
| FRAUD_CHECKING        | MANUAL_REVIEW_PENDING        | MANUAL_REVIEW_REQUIRED                 |
| FRAUD_CHECKING        | FRAUD_APPROVED               | FRAUD_CHECK_TIMEOUT (cut-off time)     |
| FRAUD_CHECKING        | FRAUD_CHECKING               | FRAUD_CHECK_RETRY                      |
| MANUAL_REVIEW_PENDING | FRAUD_APPROVED               | MANUAL_REVIEW_APPROVED                 |
| MANUAL_REVIEW_PENDING | FRAUD_REJECTED               | MANUAL_REVIEW_REJECTED                 |
| MANUAL_REVIEW_PENDING | FRAUD_APPROVED               | MANUAL_REVIEW_TIMEOUT (cut-off time)   |
| FRAUD_APPROVED        | CREDITED                     | MERCHANT_CREDITED                      |
| FRAUD_APPROVED        | REFUNDING                    | CREDIT_FAILED                          |
| FRAUD_REJECTED        | REFUNDING                    | FRAUD_REJECTION_CONFIRMED              |
| CREDITED              | NOTIFYING_SUCCESS            | SUCCESS_NOTIFICATION_INITIATED         |
| NOTIFYING_SUCCESS     | COMPLETED                    | SUCCESS_NOTIFICATION_SENT              |
| NOTIFYING_SUCCESS     | COMPLETED                    | SUCCESS_NOTIFICATION_FAILED (eventual) |
| REFUNDING             | REFUNDED                     | CUSTOMER_REFUNDED                      |
| REFUNDING             | REFUND_FAILED                | REFUND_OPERATION_FAILED                |
| REFUNDED              | NOTIFYING_FAILURE            | FAILURE_NOTIFICATION_INITIATED         |
| REFUNDED              | CANCELLING                   | FRAUD_ALERT_SENT (parallel)            |
| NOTIFYING_FAILURE     | CANCELLED                    | FAILURE_NOTIFICATION_SENT              |
| NOTIFYING_FAILURE     | CANCELLED                    | FAILURE_NOTIFICATION_FAILED (eventual) |
| REFUND_FAILED         | MANUAL_INTERVENTION_REQUIRED | REFUND_RETRY_EXHAUSTED                 |
| CANCELLED             | CANCELLED                    | CANCELLATION_CONFIRMED                 |

## Detailed State Descriptions

### Initial States

#### CREATED

- **Description**: Payment record has been created and transaction ID reserved
- **Entry Point**: Beginning of saga
- **Business Meaning**: Payment intent registered in the system

### Processing States

#### DEBITED

- **Description**: Money has been successfully debited from customer's account
- **Business Meaning**: Funds are on hold, awaiting fraud verification
- **Critical**: Money is in limbo - must be resolved to COMPLETED or REFUNDED

#### FRAUD_CHECKING

- **Description**: Transaction is being analyzed by FraudCheck service
- **Business Meaning**: Automated fraud checks are in progress (internal rules + external providers)
- **Timeout**: Subject to retry policy and circuit breaker
- **Special Case**: On timeout (cut-off time), defaults to FRAUD_APPROVED

#### MANUAL_REVIEW_PENDING

- **Description**: Transaction requires manual review by security operator
- **Business Meaning**: Automated checks inconclusive, human decision needed
- **Wait Time**: Up to 20 minutes
- **Timeout Behavior**: If no decision within 20 minutes, defaults to FRAUD_APPROVED via cut-off time

#### FRAUD_APPROVED

- **Description**: Fraud checks passed (automated, manual, or by timeout default)
- **Business Meaning**: Transaction cleared for merchant credit (approaching pivot point)
- **Next Critical Step**: CREDIT_MERCHANT_ACCOUNT (pivot transaction)

#### FRAUD_REJECTED

- **Description**: Fraud checks failed - transaction identified as suspicious
- **Business Meaning**: Transaction will be blocked, compensation flow initiated
- **Action Required**: Refund customer, alert security system

#### CREDITED

- **Description**: Money successfully transferred to merchant account (PIVOT POINT PASSED)
- **Business Meaning**: Transaction is essentially complete, only notifications remain
- **Irreversible**: Cannot compensate through saga, post-factum refunds require separate process

### Compensation States

#### REFUNDING

- **Description**: Compensation in progress - returning money to customer
- **Entry Triggers**:
    - Fraud rejection
    - Credit failure
    - System errors after debit
- **Business Meaning**: Executing REFUND_CUSTOMER_ACCOUNT compensation transaction

#### REFUNDED

- **Description**: Money successfully returned to customer account
- **Business Meaning**: Compensation complete, preparing notifications
- **Next Steps**: Send notifications, cancel payment record

#### REFUND_FAILED

- **Description**: Refund operation failed after all retries exhausted
- **Business Meaning**: CRITICAL - manual intervention required to prevent money loss
- **Escalation**: Alert operations team, create incident

### Notification States

#### NOTIFYING_SUCCESS

- **Description**: Sending success notification to customer
- **Business Meaning**: Transaction complete, informing customer
- **Retriable**: Can retry, but doesn't block completion

#### NOTIFYING_FAILURE

- **Description**: Sending failure/refund notification to customer
- **Business Meaning**: Transaction failed/refunded, informing customer
- **Retriable**: Can retry, but doesn't block cancellation

### Terminal States

#### COMPLETED

- **Description**: Payment successfully completed end-to-end
- **Business Meaning**: Happy path complete - money transferred, customer notified
- **Final State**: No further transitions

#### CANCELLED

- **Description**: Payment cancelled and all compensations executed
- **Business Meaning**: Transaction failed but properly cleaned up
- **Final State**: No further transitions

#### MANUAL_INTERVENTION_REQUIRED

- **Description**: System cannot automatically resolve the situation
- **Business Meaning**: CRITICAL - requires human intervention
- **Causes**: Refund failures, system errors
- **Final State**: Requires external resolution

## State Transition Scenarios

### Scenario 1: Happy Path (Automatic Approval)

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ FRAUD_CHECK_PASSED
FRAUD_APPROVED
  ↓ MERCHANT_CREDITED
CREDITED
  ↓ SUCCESS_NOTIFICATION_INITIATED
NOTIFYING_SUCCESS
  ↓ SUCCESS_NOTIFICATION_SENT
COMPLETED
```

### Scenario 2: Fraud Rejection Path

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ FRAUD_CHECK_FAILED
FRAUD_REJECTED
  ↓ FRAUD_REJECTION_CONFIRMED
REFUNDING
  ↓ CUSTOMER_REFUNDED
REFUNDED
  ↓ FAILURE_NOTIFICATION_INITIATED
NOTIFYING_FAILURE
  ↓ FAILURE_NOTIFICATION_SENT
CANCELLED
```

### Scenario 3: Manual Review Approved

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ MANUAL_REVIEW_REQUIRED
MANUAL_REVIEW_PENDING
  ↓ MANUAL_REVIEW_APPROVED (operator decision within 20 min)
FRAUD_APPROVED
  ↓ MERCHANT_CREDITED
CREDITED
  ↓ SUCCESS_NOTIFICATION_INITIATED
NOTIFYING_SUCCESS
  ↓ SUCCESS_NOTIFICATION_SENT
COMPLETED
```

### Scenario 4: Manual Review Rejected

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ MANUAL_REVIEW_REQUIRED
MANUAL_REVIEW_PENDING
  ↓ MANUAL_REVIEW_REJECTED (operator decision)
FRAUD_REJECTED
  ↓ FRAUD_REJECTION_CONFIRMED
REFUNDING
  ↓ CUSTOMER_REFUNDED
REFUNDED
  ↓ FAILURE_NOTIFICATION_INITIATED
NOTIFYING_FAILURE
  ↓ FAILURE_NOTIFICATION_SENT
CANCELLED
```

### Scenario 5: Cut-off Time Timeout (Default Approval)

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ MANUAL_REVIEW_REQUIRED
MANUAL_REVIEW_PENDING
  ↓ MANUAL_REVIEW_TIMEOUT (20 minutes elapsed, no decision)
FRAUD_APPROVED (default approval by cut-off)
  ↓ MERCHANT_CREDITED
CREDITED
  ↓ SUCCESS_NOTIFICATION_INITIATED
NOTIFYING_SUCCESS
  ↓ SUCCESS_NOTIFICATION_SENT
COMPLETED
```

### Scenario 6: Credit Failure with Compensation

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ FRAUD_CHECK_PASSED
FRAUD_APPROVED
  ↓ CREDIT_FAILED (merchant account error)
REFUNDING
  ↓ CUSTOMER_REFUNDED
REFUNDED
  ↓ FAILURE_NOTIFICATION_INITIATED
NOTIFYING_FAILURE
  ↓ FAILURE_NOTIFICATION_SENT
CANCELLED
```

### Scenario 7: Refund Failure (Critical Path)

```
START
  ↓ PAYMENT_CREATED
CREATED
  ↓ CUSTOMER_DEBITED
DEBITED
  ↓ FRAUD_CHECK_INITIATED
FRAUD_CHECKING
  ↓ FRAUD_CHECK_FAILED
FRAUD_REJECTED
  ↓ FRAUD_REJECTION_CONFIRMED
REFUNDING
  ↓ REFUND_OPERATION_FAILED (after retry exhaustion)
REFUND_FAILED
  ↓ REFUND_RETRY_EXHAUSTED
MANUAL_INTERVENTION_REQUIRED (alert ops team)
```

## Event Definitions

### Forward Flow Events

| Event                          | Trigger                                         | Description                                     |
|--------------------------------|-------------------------------------------------|-------------------------------------------------|
| PAYMENT_CREATED                | CREATE_PAYMENT transaction succeeds             | Payment record created, transaction ID reserved |
| CUSTOMER_DEBITED               | DEBIT_CUSTOMER_ACCOUNT transaction succeeds     | Money debited from customer account             |
| FRAUD_CHECK_INITIATED          | FRAUD_CHECK_REQUEST transaction sent            | Request submitted to FraudCheck service         |
| FRAUD_CHECK_PASSED             | FraudCheck service returns APPROVED             | Automated fraud checks passed                   |
| FRAUD_CHECK_FAILED             | FraudCheck service returns REJECTED             | Automated fraud checks failed                   |
| MANUAL_REVIEW_REQUIRED         | FraudCheck service returns MANUAL_REVIEW        | Human review needed                             |
| FRAUD_CHECK_RETRY              | Fraud check timeout/error (within retry policy) | Retrying fraud check request                    |
| FRAUD_CHECK_TIMEOUT            | Cut-off time reached without response           | Default to approved per business rule           |
| MANUAL_REVIEW_APPROVED         | Operator approves transaction                   | Manual decision: approve                        |
| MANUAL_REVIEW_REJECTED         | Operator rejects transaction                    | Manual decision: reject                         |
| MANUAL_REVIEW_TIMEOUT          | 20 minutes elapsed without operator decision    | Default to approved per cut-off rule            |
| MERCHANT_CREDITED              | CREDIT_MERCHANT_ACCOUNT transaction succeeds    | Money transferred to merchant (PIVOT)           |
| SUCCESS_NOTIFICATION_INITIATED | SEND_SUCCESS_NOTIFICATION transaction starts    | Begin sending success notification              |
| SUCCESS_NOTIFICATION_SENT      | Notification delivered successfully             | Customer notified of success                    |
| SUCCESS_NOTIFICATION_FAILED    | Notification failed (eventual consistency)      | Notification failed but transaction complete    |

### Compensation Flow Events

| Event                          | Trigger                                      | Description                                   |
|--------------------------------|----------------------------------------------|-----------------------------------------------|
| PAYMENT_CREATION_FAILED        | CREATE_PAYMENT transaction fails             | Payment creation error                        |
| DEBIT_COMPENSATION_REQUIRED    | Error after debit but before credit          | Need to refund customer                       |
| FRAUD_REJECTION_CONFIRMED      | Fraud check result processed                 | Begin refund due to fraud                     |
| CREDIT_FAILED                  | CREDIT_MERCHANT_ACCOUNT transaction fails    | Cannot credit merchant, must refund           |
| CUSTOMER_REFUNDED              | REFUND_CUSTOMER_ACCOUNT transaction succeeds | Money returned to customer                    |
| REFUND_OPERATION_FAILED        | REFUND_CUSTOMER_ACCOUNT transaction fails    | Refund failed after retries                   |
| REFUND_RETRY_EXHAUSTED         | All refund retry attempts exhausted          | Critical: needs manual intervention           |
| FAILURE_NOTIFICATION_INITIATED | SEND_FAILURE_NOTIFICATION transaction starts | Begin sending failure notification            |
| FAILURE_NOTIFICATION_SENT      | Failure notification delivered               | Customer notified of failure/refund           |
| FAILURE_NOTIFICATION_FAILED    | Notification failed (eventual consistency)   | Notification failed but cancellation proceeds |
| FRAUD_ALERT_SENT               | SEND_FRAUD_ALERT transaction completes       | Security system alerted                       |
| CANCELLATION_CONFIRMED         | CANCEL_PAYMENT transaction succeeds          | Payment record cancelled                      |

## State Machine Properties

### Invariants

1. **Money Safety**: At any point, money must be either:
    - In customer account (states: CREATED, CANCELLED, REFUNDED, COMPLETED via refund)
    - In transit (states: DEBITED, FRAUD_CHECKING, MANUAL_REVIEW_PENDING, FRAUD_APPROVED, REFUNDING)
    - In merchant account (states: CREDITED, NOTIFYING_SUCCESS, COMPLETED via success)

2. **No Money Limbo**: States DEBITED, FRAUD_CHECKING, MANUAL_REVIEW_PENDING, FRAUD_APPROVED, REFUNDING must eventually
   resolve to terminal states

3. **Pivot Point**: CREDITED state is irreversible via saga compensation

4. **Compensation Correctness**: Every compensatable state has a path to CANCELLED or MANUAL_INTERVENTION_REQUIRED

### Timeouts

1. **Fraud Check Timeout**: Configurable retry policy with circuit breaker
2. **Manual Review Timeout**: 20 minutes → defaults to FRAUD_APPROVED
3. **Cut-off Time**: Overall transaction timeout → defaults to FRAUD_APPROVED
4. **Notification Timeout**: Retry with exponential backoff, eventual consistency

### Idempotency

All state transitions are idempotent - processing the same event multiple times in the same state results in the same
next state.

## Implementation Considerations

### State Persistence

- **Primary Store**: PostgreSQL for durable saga state
- **Cache**: Redis for fast state lookups and fraud check results
- **Audit Log**: All state transitions logged with timestamp, event, and context

### Monitoring & Alerts

1. **Stuck Transactions**: Alert if state duration exceeds threshold
    - FRAUD_CHECKING > 30 seconds
    - MANUAL_REVIEW_PENDING > 20 minutes
    - REFUNDING > 10 seconds

2. **Critical States**: Immediate alert for:
    - REFUND_FAILED
    - MANUAL_INTERVENTION_REQUIRED

3. **Business Metrics**:
    - Completion rate by state path
    - Average time in each state
    - Compensation rate
    - Manual review rate

### Temporal Workflow State Mapping

```
Workflow State = Payment State + Saga Context
- Current activity = Current state
- Workflow variables = Transaction details, timestamps, retry counts
- Timers = Manual review timeout, cut-off time
- Signals = Manual review decisions, operator interventions
```

### Camunda BPMN State Mapping

```
Process Instance State = Payment State
- Service task completion = State transition
- Boundary events = Timeouts, errors
- Compensation handlers = Refund operations
- Message correlation = Manual review responses
```
