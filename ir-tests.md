# Unit tests and validation checks to preserve semantics

These tests exercise IR-level semantics and are intended to be implemented as Rust async unit tests (tokio::test).

1) Field validation enforcement
- Setup: instantiate form with email set to invalid value.
- Exercise: trigger onChange for email.
- Assert: validation action produces an error state on the email field and prevents save.

2) OnChange rule execution order
- Setup: two onChange rules for the same field with defined priorities.
- Exercise: change the field.
- Assert: actions execute in priority order; state changes from the first are visible to the second.

3) Transactional onSave rollbacks
- Setup: onSave rule with two DB operations: insert then a failing constraint.
- Exercise: call save.
- Assert: no partial write persisted; transaction was rolled back.

4) Flow decision branching
- Setup: flow with decision node containing two guarded transitions.
- Exercise: start flow with input that selects second branch.
- Assert: executed path is the second branch; side-effects executed once.

5) Adapter error propagation and fallback
- Setup: rule that calls adapter which fails.
- Exercise: run rule.
- Assert: configured onFailure actions execute; when adapter is marked `awaitResponse=false` the rule continues.

6) Event correlation and correlation IDs
- Setup: startFlow from rule with correlation id.
- Exercise: flow nodes raise events.
- Assert: events preserve correlation id and can be correlated to original request.

7) Partial script preservation
- Setup: import with an opaque script that cannot be AST-parsed.
- Exercise: serialize/deserialize IR and run a fallback ScriptEngine.
- Assert: rawScript persisted and fallback executed (or failure reported) without losing other IR data.

Implementation notes
- Mock adapters using `wiremock` or a simple mock trait impl.
- Use an in-memory SQLite (sqlx supports sqlite) for testing transaction semantics.
- Use deterministic clocks (freeze time) for timestamp-dependent tests.

