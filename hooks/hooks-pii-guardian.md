# hooks-pii-guardian

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-pii-guardian`

## Overview

Detects and redacts personally identifiable information (PII) before it's sent to LLM providers. Protects user privacy and ensures compliance with GDPR, CCPA, and HIPAA regulations.

### Value Proposition

| Without | With |
|---------|------|
| PII sent to external LLMs | PII redacted before transmission |
| Compliance violations | Privacy-by-design |
| Data breach risks | Sensitive data never leaves |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-pii-guardian
    config:
      # Detection categories
      detect:
        names: true                     # Person names
        emails: true                    # Email addresses
        phones: true                    # Phone numbers
        ssn: true                       # Social Security Numbers
        credit_cards: true              # Credit card numbers
        addresses: true                 # Physical addresses
        dates_of_birth: true            # Birth dates
        ip_addresses: true              # IP addresses
        medical: true                   # Medical record numbers, conditions
        financial: true                 # Bank accounts, routing numbers

      # Action on detection
      action: redact                    # redact | block | warn

      # Redaction style
      redaction:
        style: placeholder              # placeholder | hash | mask
        placeholder_format: "[{type}]"  # e.g., "[EMAIL]", "[SSN]"

      # Exceptions
      allow:
        patterns:
          - "example@example.com"
          - "test@"
        contexts:
          - "test"
          - "example"
          - "sample"
```

### HookResult

```python
# Redact PII before sending to provider
HookResult(
    action="modify",
    data={
        "messages": [
            {
                "role": "user",
                "content": "Process payment for [NAME] at [EMAIL], card ending [CREDIT_CARD_LAST4]"
            }
        ]
    },
    user_message="Redacted 3 PII items before sending to LLM"
)
```

---

## Architecture

```python
class PIIGuardianHook:
    """Detect and redact PII before LLM transmission."""

    def __init__(self, config: PIIGuardianConfig):
        self.config = config
        self.detectors = self._initialize_detectors()
        self.redactor = PIIRedactor(config.redaction)

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Only process provider requests
        if event != "provider:request":
            return HookResult(action="continue")

        messages = data.get("messages", [])
        redacted_messages = []
        total_redactions = 0

        for message in messages:
            content = message.get("content", "")
            if isinstance(content, str):
                redacted_content, count = self._redact_pii(content)
                redacted_messages.append({**message, "content": redacted_content})
                total_redactions += count
            else:
                redacted_messages.append(message)

        if total_redactions > 0:
            return HookResult(
                action="modify",
                data={**data, "messages": redacted_messages},
                user_message=f"Redacted {total_redactions} PII items"
            )

        return HookResult(action="continue")

    def _redact_pii(self, text: str) -> tuple[str, int]:
        """Detect and redact PII from text."""
        findings = []

        for detector in self.detectors:
            findings.extend(detector.detect(text))

        # Sort by position (reverse) to redact from end to start
        findings.sort(key=lambda f: f.start, reverse=True)

        # Remove duplicates/overlaps
        findings = self._remove_overlaps(findings)

        # Apply redactions
        redacted = text
        for finding in findings:
            redacted = (
                redacted[:finding.start] +
                self.redactor.redact(finding) +
                redacted[finding.end:]
            )

        return redacted, len(findings)


class NameDetector:
    """Detect person names using NER or patterns."""

    def detect(self, text: str) -> list[PIIFinding]:
        findings = []

        # Use spaCy NER if available
        if self.nlp:
            doc = self.nlp(text)
            for ent in doc.ents:
                if ent.label_ == "PERSON":
                    findings.append(PIIFinding(
                        type="name",
                        value=ent.text,
                        start=ent.start_char,
                        end=ent.end_char,
                        confidence=0.9
                    ))

        return findings


class EmailDetector:
    """Detect email addresses."""

    PATTERN = re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b')

    def detect(self, text: str) -> list[PIIFinding]:
        findings = []
        for match in self.PATTERN.finditer(text):
            findings.append(PIIFinding(
                type="email",
                value=match.group(),
                start=match.start(),
                end=match.end(),
                confidence=1.0
            ))
        return findings


class CreditCardDetector:
    """Detect credit card numbers."""

    PATTERNS = [
        re.compile(r'\b4[0-9]{12}(?:[0-9]{3})?\b'),      # Visa
        re.compile(r'\b5[1-5][0-9]{14}\b'),              # Mastercard
        re.compile(r'\b3[47][0-9]{13}\b'),               # Amex
        re.compile(r'\b6(?:011|5[0-9]{2})[0-9]{12}\b'),  # Discover
    ]

    def detect(self, text: str) -> list[PIIFinding]:
        findings = []
        for pattern in self.PATTERNS:
            for match in pattern.finditer(text):
                # Validate with Luhn algorithm
                if self._luhn_check(match.group()):
                    findings.append(PIIFinding(
                        type="credit_card",
                        value=match.group(),
                        start=match.start(),
                        end=match.end(),
                        confidence=1.0
                    ))
        return findings


class PIIRedactor:
    """Apply redaction to PII findings."""

    def redact(self, finding: PIIFinding) -> str:
        if self.config.style == "placeholder":
            return self.config.placeholder_format.format(type=finding.type.upper())
        elif self.config.style == "hash":
            return hashlib.sha256(finding.value.encode()).hexdigest()[:8]
        elif self.config.style == "mask":
            return "*" * len(finding.value)
        return "[REDACTED]"
```

---

## PII Categories

| Category | Examples | Pattern Type |
|----------|----------|--------------|
| **Names** | "John Smith", "María García" | NER |
| **Emails** | "user@domain.com" | Regex |
| **Phones** | "+1-555-123-4567", "(555) 123-4567" | Regex |
| **SSN** | "123-45-6789" | Regex + validation |
| **Credit Cards** | "4111111111111111" | Regex + Luhn |
| **Addresses** | "123 Main St, City, ST 12345" | NER + Regex |
| **DOB** | "01/15/1990", "January 15, 1990" | Regex |
| **IP Addresses** | "192.168.1.1", "2001:db8::1" | Regex |
| **Medical** | MRN, diagnoses | Pattern matching |
| **Financial** | Bank account, routing numbers | Regex + validation |

---

## Examples

### Example 1: Email and Name Redaction

```python
# Input message to LLM:
"Send confirmation to John Smith at john.smith@company.com"

# After redaction:
"Send confirmation to [NAME] at [EMAIL]"
```

### Example 2: Credit Card Redaction

```python
# Input:
"Process payment for card 4111111111111111"

# After redaction:
"Process payment for card [CREDIT_CARD]"
```

### Example 3: Medical Data Redaction

```python
# Input:
"Patient MRN 12345678 diagnosed with diabetes"

# After redaction:
"Patient [MEDICAL_ID] diagnosed with [MEDICAL_CONDITION]"
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `detect.*` | bool | true | Enable detection category |
| `action` | string | "redact" | redact, block, or warn |
| `redaction.style` | string | "placeholder" | How to redact |
| `redaction.placeholder_format` | string | "[{type}]" | Placeholder template |
| `allow.patterns` | list | [] | Patterns to skip |
| `allow.contexts` | list | [] | Contexts that allow PII |

---

## Compliance Mapping

| Regulation | PII Types | Hook Coverage |
|------------|-----------|---------------|
| **GDPR** | Names, emails, addresses, IDs | Full |
| **CCPA** | Personal info, identifiers | Full |
| **HIPAA** | PHI (medical, SSN, etc.) | Full |
| **PCI-DSS** | Card numbers, CVV | Full |

---

## Security Considerations

- Redaction happens BEFORE data leaves to LLM
- Original data never logged
- NER models run locally (no external calls)
- False negatives possible - defense in depth recommended

---

## Dependencies

### Required
- `re` - Regex matching (stdlib)

### Optional
- `spacy` - Named entity recognition for names/addresses
- `presidio` - Microsoft's PII detection library

---

## Open Questions

1. **Accuracy vs performance**: Full NER vs regex-only?
2. **Reversibility**: Should we support de-redaction for debugging?
3. **Custom PII types**: Allow project-specific PII definitions?
4. **Confidence thresholds**: Skip low-confidence detections?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
