# Magic → Rust Migration Framework

A framework for migrating Magic XPA / UniPaaS applications to Rust while preserving business logic semantics.

## Overview

This project provides:

1. **Intermediate Representation (IR)** - A JSON-serializable specification capturing Magic/UniPaaS constructs (entities, forms, rules, flows, adapters)
2. **JSON Schema** - Validation for IR documents
3. **Testing Harness** - Rust crate for equivalence testing between original and migrated logic
4. **Sample IR** - Example customer application demonstrating the IR structure

## Project Structure

```
magic/
├── ir-spec.md          # IR specification document
├── ir-schema.json      # JSON Schema for IR validation
├── sample-form.json    # Example IR for a customer form application
├── rust-mapping.md     # Mapping guide from IR to Rust types
├── ir-tests.md         # Testing strategy for IR validation
├── testing-harness/    # Equivalence testing framework
│   ├── README.md
│   ├── TEST_STRATEGY.md
│   ├── run_tests.ps1
│   └── rust-equivalence/    # Rust crate for premium calculation
│       ├── Cargo.toml
│       ├── src/lib.rs
│       └── tests/
└── docs/               # Reference PDFs (Magic XPA 4.11 docs)
```

## Quick Start

### Validate an IR Document

```bash
# Using any JSON Schema validator (e.g., ajv, jsonschema)
npx ajv validate -s ir-schema.json -d sample-form.json
```

### Run Equivalence Tests

```bash
cd testing-harness/rust-equivalence
cargo test
```

Or use the PowerShell script:
```powershell
cd testing-harness
./run_tests.ps1
```

## IR Specification

The IR captures:
- **Entities** - Persistent data models with fields, keys, relationships
- **Forms** - UI forms with controls, rules, event bindings
- **Rules** - Business logic with triggers, conditions, actions
- **Flows** - Workflow/flowchart definitions
- **Adapters - **Adapters** - External connectors (HTTP, SOAP, DB, gRPC, etc.)
- **Metadata** - Provenance, versioning, partial reconstruction flags

See [ir-spec.md](ir-spec.md) for full specification.

## Testing Harness

The `rust-equivalence` crate demonstrates equivalence testing for a premium calculation function. It provides:

- `PolicyInput` / `PolicyResult` structs matching the IR data model
- `calculate_premium()` function implementing the business logic
- Property-based tests using `proptest`
- Golden file tests comparing against expected outputs

See [testing-harness/README.md](testing-harness/README.md) and [TEST_STRATEGY.md](testing-harness/TEST_STRATEGY.md) for details.

## Migration Workflow

1. **Extract** - Parse Magic/UniPaaS source into IR (tooling TBD)
2. **Validate** - Verify IR against `ir-schema.json`
3. **Map** - Use `rust-mapping.md` to generate Rust types and modules
4. **Implement** - Write Rust handlers for rules, flows, adapters
5. **Test** - Use testing harness for equivalence validation
6. **Deploy** - Run on Rust async runtime (tokio/async-std)

## Documentation

- [IR Specification](ir-spec.md) - Complete IR format definition
- [Rust Mapping Guide](rust-mapping.md) - IR to Rust type mapping
- [Testing Strategy](ir-tests.md) - Validation and equivalence testing approach
- [Testing Harness](testing-harness/README.md) - Rust equivalence testing details

## License

MIT