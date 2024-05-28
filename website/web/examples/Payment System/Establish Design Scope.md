---
sidebar_position: 1
---

# Scope

1. Not handling credit card payment processing. Use third party providers such as Stripe, Braintree, Square
2. Global application but support 1 currency
3. Payment Transactions = 1m / day
4. Support pay in, pay out flow
5. Reconciliation for any inconsistencies

## Functional Requirement
1. Pay-in flow: payment system receives money from customers
2. Pay-out flow: payment systems send money to sellers around the world

## Non Functional Requirement
1. Reliability and fault tolerance: Failed payments must be handled correctly
2. Reconciliation process between internal services (payment system, accounting system, etc) and external services (payment service providers). This is to ensure payment information across all systems are consistent

## Back-of-the-envelope estimation
1. 1 million / day ==> 40k / hr == > 10-12 tps (low)

