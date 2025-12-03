# hooks-audit-trail

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-audit-trail`

## Overview

Comprehensive audit logging for compliance and security. Creates immutable records of all agent operations, decisions, and changes. Designed for enterprise environments with regulatory requirements.

### Value Proposition

| Without | With |
|---------|------|
| No record of AI actions | Complete audit trail |
| Compliance gaps | SOC2/HIPAA/GDPR ready |
| Incident investigation difficult | Full replay capability |
| "What did the AI do?" | Detailed operation log |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-audit-trail
    config:
      # Storage configuration
      storage:
        type: file                      # file | s3 | elasticsearch | splunk
        path: "${AUDIT_LOG_PATH}"       # For file storage
        # s3:
        #   bucket: "audit-logs"
        #   prefix: "amplifier/"

      # What to log
      capture:
        session_lifecycle: true         # start, end
        prompts: true                   # User inputs
        responses: true                 # Agent outputs
        tool_operations: true           # All tool calls
        file_changes: true              # File content changes
        approvals: true                 # User approvals
        errors: true                    # Errors and failures

      # Data handling
      retention:
        days: 365                       # Keep logs for 1 year
        archive_after_days: 90          # Move to cold storage

      # Privacy
      privacy:
        redact_secrets: true            # Redact detected secrets
        hash_user_ids: false            # Hash user identifiers
        exclude_patterns:               # Don't log matching content
          - "password"
          - "credit_card"

      # Integrity
      integrity:
        sign_entries: true              # Cryptographically sign entries
        chain_hashes: true              # Hash chain for tamper detection

      # Format
      format:
        type: jsonl                     # jsonl | json | parquet
        include_timestamp: true
        include_session_id: true
        include_user_id: true
```

### Audit Entry Schema

```python
@dataclass
class AuditEntry:
    # Identity
    entry_id: str                       # Unique entry ID
    timestamp: datetime                 # ISO 8601
    session_id: str
    user_id: str | None

    # Event classification
    event_type: str                     # session:start, tool:execute, etc.
    event_category: str                 # lifecycle | operation | data | error

    # Event details
    action: str                         # create, read, update, delete, execute
    resource_type: str                  # file, tool, prompt, response
    resource_id: str | None             # File path, tool name, etc.

    # Operation details
    input: dict | None                  # Sanitized input
    output: dict | None                 # Sanitized output
    outcome: str                        # success | failure | denied

    # Context
    metadata: dict                      # Additional context
    previous_hash: str | None           # For hash chain
    signature: str | None               # For signed entries

@dataclass
class AuditQuery:
    """Query parameters for audit search."""
    session_id: str | None
    user_id: str | None
    event_type: str | None
    resource_type: str | None
    start_time: datetime | None
    end_time: datetime | None
    outcome: str | None
```

---

## Architecture

```python
class AuditTrailHook:
    """Comprehensive audit logging hook."""

    # Events to capture
    CAPTURED_EVENTS = [
        "session:start",
        "session:end",
        "prompt:submit",
        "prompt:complete",
        "tool:pre",
        "tool:post",
        "tool:error",
        "approval:required",
        "approval:granted",
        "approval:denied",
        "policy:violation",
        "error:*",
    ]

    def __init__(self, config: AuditTrailConfig):
        self.config = config
        self.storage = self._create_storage(config.storage)
        self.sanitizer = AuditSanitizer(config.privacy)
        self.signer = AuditSigner(config.integrity) if config.integrity.sign_entries else None
        self.previous_hash: str | None = None

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Check if we should capture this event
        if not self._should_capture(event):
            return HookResult(action="continue")

        # Create audit entry
        entry = self._create_entry(event, data)

        # Sanitize sensitive data
        entry = self.sanitizer.sanitize(entry)

        # Add integrity features
        if self.config.integrity.chain_hashes:
            entry.previous_hash = self.previous_hash
            self.previous_hash = self._compute_hash(entry)

        if self.signer:
            entry.signature = self.signer.sign(entry)

        # Store entry
        await self.storage.store(entry)

        # Always continue - audit is observational only
        return HookResult(action="continue")

    def _create_entry(self, event: str, data: dict) -> AuditEntry:
        """Create audit entry from event."""
        return AuditEntry(
            entry_id=str(uuid.uuid4()),
            timestamp=datetime.utcnow(),
            session_id=data.get("session_id", "unknown"),
            user_id=data.get("user_id"),
            event_type=event,
            event_category=self._categorize_event(event),
            action=self._extract_action(event, data),
            resource_type=self._extract_resource_type(event, data),
            resource_id=self._extract_resource_id(event, data),
            input=self._extract_input(event, data),
            output=self._extract_output(event, data),
            outcome=self._determine_outcome(event, data),
            metadata=self._extract_metadata(event, data),
        )


class AuditSanitizer:
    """Sanitize audit entries to remove sensitive data."""

    def __init__(self, config: PrivacyConfig):
        self.config = config
        self.secret_patterns = self._compile_patterns()

    def sanitize(self, entry: AuditEntry) -> AuditEntry:
        """Remove or redact sensitive information."""
        if self.config.redact_secrets:
            if entry.input:
                entry.input = self._redact_secrets(entry.input)
            if entry.output:
                entry.output = self._redact_secrets(entry.output)

        if self.config.hash_user_ids and entry.user_id:
            entry.user_id = hashlib.sha256(entry.user_id.encode()).hexdigest()[:16]

        return entry

    def _redact_secrets(self, data: dict) -> dict:
        """Recursively redact secrets from data."""
        if isinstance(data, dict):
            return {k: self._redact_secrets(v) for k, v in data.items()}
        if isinstance(data, str):
            for pattern in self.secret_patterns:
                data = pattern.sub("[REDACTED]", data)
            return data
        return data


class AuditSigner:
    """Sign audit entries for integrity verification."""

    def __init__(self, config: IntegrityConfig):
        self.private_key = self._load_private_key(config.key_path)

    def sign(self, entry: AuditEntry) -> str:
        """Create signature for audit entry."""
        payload = self._serialize_for_signing(entry)
        signature = self.private_key.sign(payload.encode())
        return base64.b64encode(signature).decode()

    def verify(self, entry: AuditEntry) -> bool:
        """Verify audit entry signature."""
        payload = self._serialize_for_signing(entry)
        signature = base64.b64decode(entry.signature)
        try:
            self.public_key.verify(signature, payload.encode())
            return True
        except Exception:
            return False
```

---

## Storage Backends

### File Storage

```python
class FileAuditStorage:
    """Store audit logs as local files."""

    async def store(self, entry: AuditEntry) -> None:
        # Rotate files by date
        file_path = self._get_file_path(entry.timestamp)

        async with aiofiles.open(file_path, "a") as f:
            await f.write(json.dumps(asdict(entry)) + "\n")

    def _get_file_path(self, timestamp: datetime) -> Path:
        date_str = timestamp.strftime("%Y-%m-%d")
        return self.base_path / f"audit-{date_str}.jsonl"
```

### S3 Storage

```python
class S3AuditStorage:
    """Store audit logs in S3."""

    async def store(self, entry: AuditEntry) -> None:
        # Buffer entries and flush periodically
        self.buffer.append(entry)

        if len(self.buffer) >= self.batch_size:
            await self._flush()

    async def _flush(self) -> None:
        # Write buffered entries to S3
        key = f"{self.prefix}{datetime.utcnow().isoformat()}.jsonl"
        body = "\n".join(json.dumps(asdict(e)) for e in self.buffer)
        await self.s3.put_object(Bucket=self.bucket, Key=key, Body=body)
        self.buffer.clear()
```

---

## Examples

### Example 1: Tool Execution Audit Entry

```json
{
    "entry_id": "audit-001-abc123",
    "timestamp": "2024-01-15T10:30:45.123Z",
    "session_id": "sess-456",
    "user_id": "alice",
    "event_type": "tool:post",
    "event_category": "operation",
    "action": "execute",
    "resource_type": "tool",
    "resource_id": "filesystem:write",
    "input": {
        "tool": "filesystem",
        "operation": "write",
        "file_path": "src/payments/gateway.py",
        "content_length": 1234
    },
    "output": {
        "success": true,
        "bytes_written": 1234
    },
    "outcome": "success",
    "metadata": {
        "duration_ms": 45,
        "provider": "anthropic",
        "model": "claude-3"
    },
    "previous_hash": "sha256:abc123...",
    "signature": "base64:xyz789..."
}
```

### Example 2: Approval Audit Entry

```json
{
    "entry_id": "audit-002-def456",
    "timestamp": "2024-01-15T10:31:00.000Z",
    "session_id": "sess-456",
    "user_id": "alice",
    "event_type": "approval:granted",
    "event_category": "lifecycle",
    "action": "approve",
    "resource_type": "operation",
    "resource_id": "delete:production/config.yaml",
    "input": {
        "prompt": "Delete production config file?",
        "options": ["Allow", "Deny"]
    },
    "output": {
        "decision": "Allow",
        "reason": "User approved"
    },
    "outcome": "success"
}
```

### Example 3: Session Summary

```python
# Query audit log for session summary
entries = await audit_storage.query(AuditQuery(session_id="sess-456"))

# Produces report:
"""
Session Audit Report: sess-456
User: alice
Duration: 15 minutes
Start: 2024-01-15T10:30:00Z
End: 2024-01-15T10:45:00Z

Operations:
- Files written: 5
- Files read: 12
- Tools executed: 23
- Approvals requested: 2
- Approvals granted: 2
- Errors: 0

File Changes:
1. src/payments/gateway.py (created)
2. src/payments/retry.py (created)
3. tests/test_gateway.py (modified)
4. pyproject.toml (modified)
5. README.md (modified)
"""
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `storage.type` | string | "file" | Storage backend |
| `storage.path` | string | "./audit" | Path for file storage |
| `capture.*` | bool | true | What events to capture |
| `retention.days` | int | 365 | Log retention period |
| `privacy.redact_secrets` | bool | true | Redact detected secrets |
| `privacy.hash_user_ids` | bool | false | Hash user identifiers |
| `integrity.sign_entries` | bool | false | Sign entries |
| `integrity.chain_hashes` | bool | false | Create hash chain |

---

## Compliance Notes

- **SOC 2**: Immutable logs with integrity verification
- **HIPAA**: Redaction of PHI, access logging
- **GDPR**: User identification controls, retention limits
- **PCI-DSS**: Cardholder data redaction

---

## Security Considerations

- Audit logs themselves contain sensitive data
- Signing keys must be protected
- Storage must be access-controlled
- Log tampering detection via hash chains

---

## Dependencies

### Required
- `aiofiles` - Async file operations

### Optional
- `boto3` - S3 storage
- `cryptography` - Entry signing
- `elasticsearch` - ES storage

---

## Open Questions

1. **Real-time streaming**: Stream audit logs to SIEM?
2. **Query interface**: Provide search API within hook?
3. **Log rotation**: Built-in rotation vs external tools?
4. **Alert integration**: Trigger alerts on suspicious patterns?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
