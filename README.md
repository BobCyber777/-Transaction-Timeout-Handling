# -Transaction-Timeout-Handling
What happens if a transaction times out?


### Transaction Timeout Handling

A transaction timeout is treated as an **indeterminate state**, not automatically as a failure.

If Tejada Financial sends a transaction to the BaaS provider and the request times out, the system does **not** immediately retry the financial operation as a new transaction. Doing so could create a duplicate debit, credit, transfer, or payment if the provider actually processed the original request.

The transaction is retained with its existing **idempotency key** and placed into a controlled pending/unknown state.

The recovery process is:

**Transaction Request → Idempotency Key → Provider Request → Timeout → Indeterminate State → Status Verification → Reconciliation → Final State**

The system then queries the provider using the original provider reference or idempotency key when available, or reconciles against the provider's transaction records/webhooks.

There are three possible outcomes:

**1. Provider confirms success**
The existing transaction is finalized and the corresponding ledger state is confirmed. No second financial operation is created.

**2. Provider confirms failure**
The transaction is safely marked failed and the appropriate ledger workflow is completed.

**3. Provider cannot yet determine the state**
The transaction remains pending/exceptional and is retried for **status inquiry**, not blindly retried as a new financial transaction.

The critical invariant is:

> **A network timeout must never be interpreted as permission to submit the financial operation again.**

Retries are therefore **idempotent and state-aware**.

This prevents the classic distributed-finance failure:

> **“The request timed out, so we sent it again — and the customer was charged twice.”**

The final state is additionally verified through reconciliation so that an eventual provider-side completion cannot be lost simply because the original request timed out.

### Safety Guarantee

The system is designed so that:

**Timeout ≠ Failure**
**Retry ≠ New Transaction**
**Unknown ≠ Success**
**Unknown ≠ Failure**

The transaction remains recoverable until its actual financial state is established.
