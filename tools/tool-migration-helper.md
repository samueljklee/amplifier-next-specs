# tool-migration-helper

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-module-tool-migration-helper`

## Overview

Assists with database schema migrations, data migrations, and API version migrations. Generates migration scripts, validates safety, plans rollback strategies, and monitors migration execution.

### Value Proposition

| Without | With |
|---------|------|
| Manually write migration scripts | Auto-generate from schema diff |
| Hope rollback works | Validated rollback scripts before deployment |
| Migration issues discovered in prod | Pre-flight safety checks |
| Complex data migrations are scary | Step-by-step execution with checkpoints |

### Use Cases

1. **Schema migration**: Generate migration from model changes
2. **Safety validation**: Check migration for data loss risks
3. **Rollback planning**: Generate and test rollback scripts
4. **Data migration**: Transform data during schema changes
5. **Migration monitoring**: Track progress of long-running migrations

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "migration_helper",
    "description": """
    Database and API migration assistance.

    Operations:
    - generate: Generate migration from schema changes
    - validate: Check migration safety
    - plan_rollback: Generate rollback strategy
    - execute: Run migration with monitoring
    - status: Check migration status

    Supports: PostgreSQL, MySQL, SQLite, MongoDB, and framework ORMs.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["generate", "validate", "plan_rollback", "execute", "status"],
                "description": "Operation to perform"
            },
            "migration_type": {
                "type": "string",
                "enum": ["schema", "data", "api"],
                "default": "schema",
                "description": "Type of migration"
            },
            "source": {
                "type": "string",
                "description": "Source schema/version (for generate)"
            },
            "target": {
                "type": "string",
                "description": "Target schema/version (for generate)"
            },
            "migration_file": {
                "type": "string",
                "description": "Migration file path (for validate/execute)"
            },
            "dry_run": {
                "type": "boolean",
                "default": true,
                "description": "Simulate without applying changes"
            }
        },
        "required": ["operation"]
    }
}
```

### Output Schema

```python
@dataclass
class MigrationHelperOutput:
    operation: str

    # For generate
    migration: GeneratedMigration | None = None

    # For validate
    validation: MigrationValidation | None = None

    # For plan_rollback
    rollback: RollbackPlan | None = None

    # For execute
    execution: ExecutionResult | None = None

    # For status
    status: MigrationStatus | None = None

@dataclass
class GeneratedMigration:
    name: str
    up_script: str                      # Forward migration SQL
    down_script: str                    # Rollback SQL
    changes: list[SchemaChange]
    estimated_duration: str
    data_at_risk: bool

@dataclass
class SchemaChange:
    type: str                           # add_column | drop_column | rename | alter_type | add_index | etc.
    table: str
    column: str | None
    details: dict
    reversible: bool
    data_loss_risk: str                 # none | low | high

@dataclass
class MigrationValidation:
    safe: bool
    warnings: list[ValidationWarning]
    errors: list[ValidationError]
    recommendations: list[str]

@dataclass
class ValidationWarning:
    type: str
    message: str
    location: str
    suggestion: str

@dataclass
class ValidationError:
    type: str
    message: str
    location: str
    blocking: bool

@dataclass
class RollbackPlan:
    script: str
    steps: list[RollbackStep]
    data_recovery: DataRecoveryPlan | None
    estimated_duration: str
    tested: bool

@dataclass
class RollbackStep:
    order: int
    description: str
    sql: str
    verification: str                   # Query to verify step succeeded

@dataclass
class ExecutionResult:
    success: bool
    duration_seconds: float
    rows_affected: int
    steps_completed: list[str]
    errors: list[str]
    rollback_triggered: bool
```

---

## Architecture

### Migration Generator

```python
class MigrationGenerator:
    """Generate migrations from schema differences."""

    async def generate(
        self,
        source: str,
        target: str,
        migration_type: str
    ) -> GeneratedMigration:
        # Parse schemas
        source_schema = await self._parse_schema(source)
        target_schema = await self._parse_schema(target)

        # Compute differences
        changes = self._diff_schemas(source_schema, target_schema)

        # Generate SQL
        up_script = self._generate_up_script(changes)
        down_script = self._generate_down_script(changes)

        # Analyze risks
        data_at_risk = any(c.data_loss_risk != "none" for c in changes)

        return GeneratedMigration(
            name=self._generate_name(changes),
            up_script=up_script,
            down_script=down_script,
            changes=changes,
            estimated_duration=self._estimate_duration(changes),
            data_at_risk=data_at_risk
        )

    def _diff_schemas(
        self,
        source: Schema,
        target: Schema
    ) -> list[SchemaChange]:
        changes = []

        # Compare tables
        for table in target.tables:
            if table.name not in source.table_names:
                changes.append(SchemaChange(
                    type="create_table",
                    table=table.name,
                    details={"columns": table.columns},
                    reversible=True,
                    data_loss_risk="none"
                ))
            else:
                # Compare columns
                source_table = source.get_table(table.name)
                changes.extend(self._diff_columns(source_table, table))

        # Detect dropped tables
        for table_name in source.table_names:
            if table_name not in target.table_names:
                changes.append(SchemaChange(
                    type="drop_table",
                    table=table_name,
                    details={},
                    reversible=False,
                    data_loss_risk="high"
                ))

        return changes
```

### Safety Validator

```python
class MigrationValidator:
    """Validate migration safety."""

    def validate(self, migration: GeneratedMigration) -> MigrationValidation:
        warnings = []
        errors = []

        for change in migration.changes:
            # Check for data loss
            if change.data_loss_risk == "high":
                errors.append(ValidationError(
                    type="data_loss",
                    message=f"Dropping {change.table}.{change.column} will cause data loss",
                    location=f"{change.table}.{change.column}",
                    blocking=True
                ))

            # Check for irreversible changes
            if not change.reversible:
                warnings.append(ValidationWarning(
                    type="irreversible",
                    message=f"Change to {change.table} cannot be automatically rolled back",
                    location=change.table,
                    suggestion="Create manual rollback script and backup data first"
                ))

            # Check for locking issues
            if change.type in ["add_index", "alter_type"] and self._is_large_table(change.table):
                warnings.append(ValidationWarning(
                    type="lock_risk",
                    message=f"This operation may lock {change.table} for extended time",
                    location=change.table,
                    suggestion="Consider using CONCURRENTLY option or off-peak deployment"
                ))

        recommendations = self._generate_recommendations(migration, warnings, errors)

        return MigrationValidation(
            safe=len(errors) == 0,
            warnings=warnings,
            errors=errors,
            recommendations=recommendations
        )
```

---

## Examples

### Example 1: Generate Migration

```python
# Input
{
    "operation": "generate",
    "source": "HEAD~1",                 # Previous commit's schema
    "target": "HEAD"                    # Current schema
}

# Output
{
    "migration": {
        "name": "20240115_add_payment_retry_columns",
        "up_script": """
-- Add retry tracking columns to payments table
ALTER TABLE payments ADD COLUMN retry_count INTEGER DEFAULT 0;
ALTER TABLE payments ADD COLUMN last_retry_at TIMESTAMP;
ALTER TABLE payments ADD COLUMN retry_reason TEXT;

-- Add index for retry queries
CREATE INDEX idx_payments_retry ON payments (retry_count, last_retry_at);
""",
        "down_script": """
-- Remove retry tracking
DROP INDEX IF EXISTS idx_payments_retry;
ALTER TABLE payments DROP COLUMN retry_reason;
ALTER TABLE payments DROP COLUMN last_retry_at;
ALTER TABLE payments DROP COLUMN retry_count;
""",
        "changes": [
            {
                "type": "add_column",
                "table": "payments",
                "column": "retry_count",
                "reversible": true,
                "data_loss_risk": "none"
            },
            {
                "type": "add_index",
                "table": "payments",
                "details": {"columns": ["retry_count", "last_retry_at"]},
                "reversible": true,
                "data_loss_risk": "none"
            }
        ],
        "estimated_duration": "< 1 minute",
        "data_at_risk": false
    }
}
```

### Example 2: Validate Migration

```python
# Input
{
    "operation": "validate",
    "migration_file": "migrations/20240115_remove_legacy_auth.sql"
}

# Output
{
    "validation": {
        "safe": false,
        "errors": [
            {
                "type": "data_loss",
                "message": "Dropping column users.legacy_token will delete 45,000 rows of data",
                "location": "users.legacy_token",
                "blocking": true
            }
        ],
        "warnings": [
            {
                "type": "irreversible",
                "message": "DROP COLUMN cannot be automatically rolled back",
                "suggestion": "Backup users.legacy_token before migration"
            },
            {
                "type": "foreign_key",
                "message": "sessions.user_token references users.legacy_token",
                "suggestion": "Drop foreign key constraint first"
            }
        ],
        "recommendations": [
            "1. Create backup: SELECT id, legacy_token INTO users_legacy_backup FROM users",
            "2. Update sessions to use new auth: UPDATE sessions SET auth_type = 'new'",
            "3. Drop foreign key: ALTER TABLE sessions DROP CONSTRAINT fk_user_token",
            "4. Then proceed with migration"
        ]
    }
}
```

---

## Configuration

```yaml
tool-migration-helper:
  # Database connections
  databases:
    primary:
      type: postgresql
      connection_string: "${DATABASE_URL}"

  # ORM integration
  orm:
    framework: sqlalchemy              # sqlalchemy | django | alembic
    models_path: "src/models/"

  # Safety thresholds
  safety:
    large_table_rows: 100000           # Tables larger than this get warnings
    max_lock_seconds: 60               # Warn if operation might lock longer
    require_backup_for_drops: true

  # Migration settings
  migration:
    directory: "migrations/"
    naming: "timestamp_description"     # timestamp_description | sequential
```

---

## Security Considerations

- Executes SQL against databases
- Requires database credentials
- Dry-run mode recommended before production

---

## Dependencies

### Required
- `sqlparse` - SQL parsing
- Database drivers (psycopg2, mysql-connector, etc.)

### Optional
- `alembic` - SQLAlchemy migrations
- `django` - Django migrations

---

## Open Questions

1. **Multi-database migrations**: Support coordinated migrations across databases?
2. **Zero-downtime patterns**: Generate online schema change scripts?
3. **Data transformation**: Support complex data migrations with Python?
4. **Approval workflow**: Require approval for risky migrations?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
