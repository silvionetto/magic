# Magic/UniPaaS -> Rust Intermediate Representation (IR) Specification

Version: 1.0
Date: 2026-06-22

Purpose
-------
This IR captures the behavioural and structural constructs of Magic XPA / UniPaaS needed to migrate applications to Rust while preserving business logic semantics. The IR is JSON-serializable, extensible, tolerant of partial source material, and designed to map cleanly to Rust types and async runtime patterns.

Design goals
------------
- Behavioral fidelity: express triggers, validations, transactions, data-binding and flow semantics.
- Extensibility: allow opaque/partial nodes when source is incomplete.
- Pragmatic compile targets: map easily to Rust structs, enums, async engines, and ORMs (sqlx/diesel).
- Clear separation: UI form definitions, data model, business rules, workflows, adapters, metadata.

Top level structure
-------------------
A program IR is a JSON object with these high-level bags:
- metadata: provenance, version
- entities: persistent data models (tables)
- forms: UI forms and grids bound to entities/views
- rules: global rules not tied to a single form
- flows: flowcharts / processes
- adapters: external connectors (HTTP, DB, SOAP, custom)
- events: explicit event definitions (optional)

Key types
---------
- Identifier: string (stable unique id)
- Expression: either a textual script (original source) or a structured AST (preferred)
- Action: an imperative operation (assign, call, db-op, start-flow, call-adapter, show-message, navigate)

Data model (entities)
----------------------
Entity {
  id: Identifier,
  name: string,
  tableName?: string,
  fields: Field[],
  keys?: Key[],
  relationships?: Relationship[],
  metadata?: Metadata
}

Field {
  id: Identifier,
  name: string,
  label?: string,
  type: FieldType,        // enum: integer, float, string, boolean, datetime, date, time, decimal, binary, enum, object, array
  nullable: boolean,
  default?: Value | Expression,
  constraints?: Constraints,
  mapping?: { column: string }
}

Constraints supports: required, min/max, precision, regex, enumValues, unique, foreignKey.

Form definitions
----------------
Form {
  id: Identifier,
  name: string,
  entityBinding?: Identifier,   // optional link to an entity
  dataSource?: DataSource,      // queries or views
  layout?: Layout,              // optional UI layout model; preserved but optional for runtime
  controls: Control[],
  rules: Rule[],
  events: EventBinding[],
  lifecycle?: LifecycleScripts, // onLoad/onClose scripts
  metadata?: Metadata
}

Control examples: TextField, ComboBox, Grid, Button, Lookup, DatePicker. Controls reference Fields by id for data-binding.

Rule representation
-------------------
Rule {
  id: Identifier,
  name?: string,
  trigger: Trigger,            // onLoad,onSave,onChange(field),onClick(control),onEnterPage,onLeavePage,onTimer,dbTrigger
  condition?: Expression,      // predicate; evaluated in rule context
  actions: Action[],           // executed sequentially; see atomicity semantics
  priorities?: number,         // optional ordering hint
  transactional?: boolean,     // if true, actions are executed inside DB transaction where applicable
  metadata?: Metadata
}

Trigger model: { type: string, target?: Identifier, params?: object }

Action types (examples):
- assign { target: fieldRef | variable, expr }
- validate { target, validator: { type: regex|range|custom }, message }
- dbOperation { op: insert|update|delete|select, entity, mapping }
- callAdapter { adapterId, operation, inputMapping, awaitResponse }
- startFlow { flowId, input }
- showMessage { severity, text }
- navigate { targetForm, params }
- raiseEvent { eventId, payload }

Action ordering semantics: actions run in defined order. If a rule is transactional, DB-affecting actions are grouped into a transaction. Non-db side effects (external calls) can be configured to run inside or outside the DB transaction by the transactional flag.

Expression language
-------------------
Expression is either:
- rawScript: original UniPaaS script (opaque)
- ast: typed AST (recommended for deterministic mapping)

AST nodes (small set):
- Literal { value }
- FieldRef { fieldId }
- VarRef { name }
- BinaryOp { op, left, right }
- UnaryOp { op, operand }
- Call { functionName, args[] }
- Conditional { condition, thenExpr, elseExpr }

IR supports storing rawScript when AST cannot be recovered. All Expressions are serializable.

Flowchart / Workflow model
--------------------------
Flow {
  id: Identifier,
  name: string,
  nodes: Node[],
  transitions: Transition[],
  startNodeId: Identifier,
  metadata?: Metadata
}

Node types (Node.type): start,end,task(user|system),decision,parallelSplit,parallelJoin,subflow,waitEvent,timer,script

Node {
  id, name, type,
  payload?: { script?: Expression, adapterCall?: AdapterCallDescriptor, assignedForm?: Identifier }
  properties?: object // extensible
}

Transition { id, from: nodeId, to: nodeId, condition?: Expression, isDefault?: boolean }

Execution model: the flow engine evaluates transitions from nodes, executes node payloads. Decision nodes evaluate guards; parallelSplit spawns concurrent branches. ParallelJoin waits for branches. Subflows can be executed synchronously or asynchronously per node.properties.

Event model
-----------
Event {
  id: Identifier,
  name: string,
  source: { type: 'ui'|'db'|'adapter'|'scheduler'|'external' },
  payloadSchema?: JSONSchema,
  metadata?: Metadata
}

Runtime event envelope: { eventId, correlationId, origin, timestamp, payload, metadata }

Adapters / External connectors
-----------------------------
AdapterDescriptor {
  id: Identifier,
  name: string,
  type: 'http'|'soap'|'database'|'grpc'|'queue'|'custom',
  config: object,           // deserializable connector config
  operations: AdapterOperation[],
  auth?: AuthDescriptor,
  retryPolicy?: RetryPolicy,
  metadata?: Metadata
}

AdapterOperation { name, inputSchema?, outputSchema?, sideEffect?: boolean }

Metadata and provenance
-----------------------
Metadata {
  id?: Identifier,
  sourceFile?: string,
  sourceLocation?: { line?: number, column?: number },
  version?: string,
  created?: timestamp,
  modified?: timestamp,
  partial?: boolean, // true if this node was reconstructed or truncated
  annotations?: { [key:string]: string }
}

Partial / unknown constructs
---------------------------
When the importer cannot fully reconstruct an AST or semantics, the IR should preserve the raw text in rawScript and mark the node metadata.partial=true. Engines can optionally attempt to evaluate rawScript in a sandbox (e.g., WASM or embedded scripting engine) or ask for human review.

Transactions & error semantics
------------------------------
- Rules marked transactional will execute DB actions inside a transaction. If any DB action fails, the transaction is rolled back.
- Side-effecting external calls that must be compensated should be modelled explicitly (compensating actions) or executed outside the DB transaction with a compensation flow.

Serialization
-------------
All IR nodes are JSON-serializable. The canonical file name for a program is program.ir.json containing the top-level object.

Versioning & evolution
----------------------
IR MUST include a version and be forward/backward tolerant: unrecognized enum values should be stored as "unknown:<value>" and metadata.partial flagged.

Appendix: Example JSON Schema (see ir-schema.json) and example IR (see sample-form.json).


