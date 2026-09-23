# Task 1 — Test Design & Risk Coverage

## Feature

**Local Fund Transfer**

A retail banking feature allowing a customer to transfer money from an account to an already registered beneficiary.

## Objective

Design a risk-based test strategy that demonstrates coverage of:

- Functional requirements
- Financial integrity
- Authentication and security
- Transfer limits
- Fees
- Failure and recovery scenarios
- Transaction state management
- Integration risks
- Automation and regression strategy
- Release-readiness criteria

## Deliverables

The detailed Task 1 analysis is available in:

- `task-1/test-design-risk.md`

The document covers:

1. **Questions and Assumptions**
    - Clarification points for incomplete requirements
    - Explicit assumptions used for test design

2. **Risk Assessment**
    - P0/P1/P2 risk prioritization
    - Financial, security, operational, and compliance risks

3. **Test Cases**
    - 12 focused, executable test cases
    - Preconditions
    - Steps
    - Expected results
    - Acceptance-criteria traceability

4. **Coverage Decisions**
    - Deliberately excluded coverage
    - Automation vs. manual testing strategy
    - Release-safety criteria

## Test Design Principle

The test strategy prioritizes **financial integrity and security over breadth of low-risk UI coverage**.

Particular emphasis is placed on the boundary where the customer's account is debited:

```text
Validation
    ↓
Authentication
    ↓
**CBS** Debit  ← Financial Risk Boundary
    ↓
### Payment Rail
    ↓
### Beneficiary Credit
````

Failures occurring after the debit require stronger validation of:

- Idempotency
- Transaction state
- Timeout handling
- Reversal
- Recovery
- Reconciliation

## Test Automation Strategy

Business-critical and repeatable scenarios should be automated, preferably at **API**/service level where practical, with a smaller number of critical end-to-end UI tests.

Manual/exploratory testing remains important for:

- External compliance decisions
- Human-review workflows
- Notification behavior
- Usability
- Exploratory edge cases

## Release Readiness

The feature is considered ready for release when:

- All P0/P1 scenarios pass.
- No critical or high-severity defects affecting financial integrity or security remain open.
- Duplicate financial effects are prevented.
- Failure, timeout, reversal, and recovery paths are validated.
- Transaction states remain consistent across the relevant systems.
- Critical automated regression tests pass in CI.
- Transaction auditability and production monitoring are available.
- No unexplained Pending/Unknown transactions remain from testing.
- Business/Product/QA stakeholders approve the release based on the agreed acceptance and risk criteria. 
