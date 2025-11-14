# FusionRules

FusionRules is a high-performance, asynchronous rule analysis engine built with Apache DataFusion for Python.

It is designed to plug into any Python data pipeline that can provide Apache Arrow tables, and evaluate complex rule logic in a fully asynchronous way.

-----

## Key Features

  - **High-performance in-memory analysis** Built on top of Apache Arrow and DataFusion (Rust core). SQL queries are executed directly in memory without copying data.

  - **Flexible rule DSL** Rules are defined in human-readable `YAML` files. Rules can be hot-reloaded and are highly flexible.

  - **Concurrent SQL execution** Multiple `conditions` (SQL queries) defined in a single rule are executed concurrently via `asyncio`.

  - **Input-source agnostic** The engine core only accepts `pyarrow.Table` objects. This means it can be integrated into any Python application that can produce Arrow data.

  - **Rich logical judgment** Supports `if/then`-style `judgments` blocks, allowing complex boolean and arithmetic expressions over query results to produce a final “arbitration result”.

  - **Fully asynchronous design** From core to implementation, everything is built around `asyncio`, making it a natural fit for high-concurrency I/O scenarios.

-----

## Architecture Philosophy

The core idea behind `FusionRules` is to **decouple the “data source” from the “analysis engine”**.

The engine API expects a dictionary of `pyarrow.Table` objects. This allows it to be embedded into any data pipeline that can provide Arrow tables.

Typical flow:

1.  Your system produces data (Kafka, HTTP API, gRPC, files, etc.).
2.  An adapter converts raw data (JSON, binary, etc.) into pyarrow.Table objects.
3.  FusionRules runs all configured rules and returns a final decision such as "confirm", "review\_required", "no\_match", etc.

-----

## Rule DSL Syntax

All logic in FusionRules is driven by YAML rule files.

Each rule consists of four key sections:

1.  `meta` – Rule metadata (ID, name, enabled flag).
2.  `conditions` – A mapping from symbolic names to SQL queries. These queries are executed concurrently by DataFusion.
3.  `judgments` – An ordered list of if/return pairs. The engine evaluates them in order and returns the return value for the first if expression that evaluates to True.
4.  `default_return` – Fallback value if none of the judgments match.

Example: `rules.yaml`

```yaml
meta:
  id: "RULE_001_SUSPICIOUS_TX"
  name: "Suspicious Large Transaction Rule"
  enabled: true

# 1. 'conditions' - DataFusion executes all SQL concurrently
conditions:
  flagged_users: |
    SELECT user_id, risk_score
    FROM users_table
    WHERE status = 'flagged'

  high_tx: |
    SELECT user_id, SUM(amount) as total_amount
    FROM transactions_table
    WHERE amount > 5000
    GROUP BY user_id
    HAVING SUM(amount) > 10000

# 2. 'judgments' - evaluated by asteval in order
#    You can reference the variable names defined in 'conditions'
#    (e.g., flagged_users). The engine injects helper attributes:
#    .count, .scalar, .columns, etc.
judgments:
  # Access .count
  - if: "flagged_users.count > 0 and high_tx.count > 0"
    return: "confirm"

  # Access .columns['column_name'] and use injected functions like 'sum'
  - if: "flagged_users.count > 0 and sum(flagged_users.columns['risk_score']) > 150"
    return: "review_required"
    
# 3. 'default_return' - fallback value when nothing matches
default_return: "no_match"
```

## Condition Results & Helpers

For each condition (e.g., `flagged_users`, `high_tx`), the engine exposes:

  * `obj.count` – number of rows in the result.
  * `obj.columns['column_name']` – column values for the given name.
  * `obj.scalar` – a convenience accessor for single-value results (implementation-specific).
  * Injected helper functions (e.g., `sum`, `len`, etc.) for use in expressions.

This allows you to write concise yet expressive logical rules on top of SQL results.

-----

## Core Tech Stack

  * **Python 3.12+** Uses `asyncio` as the foundation for concurrency.
  * **Apache DataFusion** (`datafusion`)  
    The core SQL query engine (Rust backend with Python bindings).
  * **Apache Arrow** (`pyarrow`)  
    The in-memory columnar data format powering all computations.
  * **PyYAML** For loading and parsing the YAML-based rule DSL.
  * **Asteval** For safely evaluating `judgments` expressions over condition outputs.