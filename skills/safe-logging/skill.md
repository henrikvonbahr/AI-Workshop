# Safe logging

## Purpose

Support operation, diagnosis and security monitoring without unnecessarily exposing information.

## Use this skill when

Adding, changing or reviewing log statements, logged properties, diagnostic payloads or exception handling, at any level and in any environment.

## Do not use this skill to decide

Logging libraries, retention, platform access controls, domain-specific sensitivity classifications, approved audit requirements or masking schemes.

## Rules in priority order

1. **Prevent disclosure.** Never log passwords, private keys, access or refresh tokens, session identifiers, API keys, authorization headers or other authentication material. Do not log full national identifiers, payment data, health data, precise addresses or complete user-provided objects. Structured properties and debug logs are not exceptions. Use masking only when explicitly defined by the domain; do not invent hashing or reversible substitutes.
2. **Preserve application correctness.** Keep intended behavior and exception propagation intact. Log exceptions at the layer responsible for handling or terminating the operation, not at every layer. Do not copy potentially sensitive exception messages into extra fields.
3. **Retain minimum necessary operational context.** Log operations and outcomes, not request bodies, DTOs, domain objects or serialized payloads by default. Where appropriate, include safe operation names, approved non-sensitive internal IDs, existing request or trace IDs, outcomes and durations. Use stable event descriptions with separate named properties, not messages constructed from untrusted values. Choose Information for expected lifecycle/business outcomes, Warning for unexpected but handled situations, Error for failures requiring investigation, and Debug/Trace for detailed, still-safe diagnostics.
4. **Improve readability and style last.** Follow existing logging conventions without weakening the preceding rules. The standard governs when examples or annotations contradict it; stack traces are not categorically forbidden or automatically safe.

## Procedure

1. Inspect data handled by the code, including exception contents and values exposed through serialization.
2. Identify the operation, outcome and responsible exception-handling layer. Select the smallest safe context and appropriate level.
3. Generate or revise a stable event using the existing logger’s supported structured properties. Omit payloads, secrets and unnecessary sensitive values.
4. Review every emitted value, including exception details; check formatting semantics, duplicate exception logging and unchanged application behavior.

## Common failure modes

- Logging a whole object because it is convenient or structured.
- Treating debug/development logs, hashes or ad hoc masking as safe.
- Concatenating untrusted values into event descriptions or using unsupported formatting.
- Copying exception messages into additional fields or logging the same failure at multiple layers.
- Using Error for routine success, or assuming passing an exception is always safe or always forbidden.

## When information is missing

Omit values of unknown sensitivity and retain safe operation/outcome context. Request domain guidance if the omitted value is necessary. Prefer an approved internal ID over an email address; if no safe identifier exists, log without one.

## Self-review checklist

- [ ] Every logged value has been inspected; no secrets or prohibited sensitive data remain.
- [ ] No default body, DTO, object or serialized-payload logging remains.
- [ ] Event descriptions are stable, properties are separate, and formatting matches the existing logger.
- [ ] Exception logging occurs at the responsible layer without duplication or copied sensitive messages.
- [ ] Context is minimal and safe; levels match actual outcomes.
- [ ] Application behavior is preserved; unknown-sensitive values are omitted or escalated.