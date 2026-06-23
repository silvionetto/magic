# Rust Mapping Guidance for the IR

This document shows recommended Rust mappings for IR constructs and patterns for DB mapping (sqlx/diesel) and async execution.

Crate layout (suggested)
- crate::ir             -- generated IR structs (serde)
- crate::engine         -- execution engine for rules and flows
- crate::adapters       -- adapter trait impls and configs
- crate::db             -- repository layer, sqlx/diesel models
- crate::expr           -- expression AST/evaluator or compiled bridge
- crate::api            -- HTTP/api layer if needed

Rust type mappings
------------------
- IR object -> Rust struct: derive(Serialize, Deserialize, Clone, Debug)
- Identifier -> String (newtype `struct Id(String)` optional)
- FieldType -> enum FieldType { Integer, Float, String, Boolean, DateTime, Date, Time, Decimal, Binary, Enum(Vec<String>), Object, Array }
- Field.default -> Option<serde_json::Value> or typed DefaultValue enum
- Expression -> enum Expression { RawScript(String), Ast(Box<AstNode>) }
- Action -> enum Action { Assign{...}, DbOperation{...}, CallAdapter{...}, StartFlow{...}, ShowMessage{...} }

Example: Form struct

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Form {
  pub id: String,
  pub name: String,
  pub entity_binding: Option<String>,
  pub controls: Vec<Control>,
  pub rules: Vec<Rule>,
  pub metadata: Option<Metadata>,
}

Expression evaluation and rule execution
---------------------------------------
Option A (interpreter):
- Implement EvalContext { fields: HashMap<String,Value>, vars: HashMap<String,Value>, services: Arc<ServiceRegistry> }
- impl Expression { async fn eval(&self, ctx: &mut EvalContext) -> Result<Value, EvalError> }
- For AST nodes, implement evaluation deterministically; for RawScript, delegate to an embedded engine (rhai or WASM sandbox).

Option B (compile-to-Rust):
- For critical/constant rules, generate Rust code from AST and compile a plugin/WASM; use `wasmtime` or `wasmer` for sandboxing.

Adapter trait
-------------
Use async-trait to simplify async trait methods:

#[async_trait]
pub trait Adapter: Send + Sync {
  async fn call(&self, operation: &str, input: serde_json::Value) -> Result<serde_json::Value, AdapterError>;
}

Implement HTTP adapters using `reqwest::Client` and configuration deserialized from AdapterDescriptor.

DB mapping patterns
-------------------
sqlx (async):

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct Customer {
  pub customer_id: i64,
  pub name: String,
  pub email: String,
  pub status: String,
  pub created_at: chrono::DateTime<chrono::Utc>,
}

let mut tx = pool.begin().await?;
sqlx::query!("INSERT INTO customers (name,email,status,created_at) VALUES ($1,$2,$3,$4) ON CONFLICT (customer_id) DO UPDATE ...", name, email, status, created_at).execute(&mut tx).await?;
tx.commit().await?;

Diesel (synchronous API unless you use async-diesel wrappers):

table! { customers (customer_id) { customer_id -> Int8, name -> Text, email -> Text, status -> Text, created_at -> Timestamp } }

#[derive(Insertable, Queryable)]
#[table_name = "customers"]
pub struct Customer { ... }

Transactions
------------
- Use sqlx::Pool and begin/commit/rollback for transactional rules.
- Ensure external calls that must not be inside DB transactions are marked and executed outside the transaction scope.

Flow execution patterns
-----------------------
- Represent nodes as `enum NodeKind` and flows as `struct Flow { nodes: Vec<Node>, transitions: Vec<Transition> }`.
- Implement an async executor: `async fn execute_flow(flow:&Flow, input: Value, ctx: &mut ExecContext) -> Result<Value, EngineError>`.
- For parallelSplit: spawn tokio tasks and join handles (tokio::spawn or futures::future::join_all).
- Persist long-running flow state to DB using a flow_instance table; operations must be idempotent.

Events & Correlation
--------------------
- Event envelope: `struct Event { id: Uuid, correlation_id: Option<Uuid>, origin: String, created_at: DateTime<Utc>, payload: Value }`.
- Use `tokio::sync::broadcast` for in-process pub/sub; use NATS/Kafka/RabbitMQ for distributed systems.

Error handling
--------------
- Use `thiserror::Error` to define EngineError/AdapterError.
- Return Result<T, EngineError> and instrument with `tracing` for observability.

Partial info and raw script fallback
------------------------------------
- Preserve rawScript in AST nodes and mark metadata.partial=true.
- Provide a `ScriptEngine` abstraction that can execute raw scripts (Rhai, WASM). Prefer WASM for stronger sandboxing and cross-language portability.

Testing and determinism
-----------------------
- Mock adapters and DB layer for deterministic unit tests.
- Tests assert field validation, rule ordering, transaction rollbacks, flow branching and event correlation.

Notes
-----
- Prefer serde-based IR structs so IR JSON can be loaded directly into Rust types.
- Use feature flags to swap ORMs (sqlx vs diesel) at compile time.

