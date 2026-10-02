# Skill-assisted result

Follow the coding skill below while implementing the task. Apply only relevant rules. If a security-relevant detail is unspecified, choose the safer minimal implementation and state the assumption briefly.


SKILL:


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



- Every logged value has been inspected; no secrets or prohibited sensitive data remain.

- No default body, DTO, object or serialized-payload logging remains.

- Event descriptions are stable, properties are separate, and formatting matches the existing logger.

- Exception logging occurs at the responsible layer without duplication or copied sensitive messages.

- Context is minimal and safe; levels match actual outcomes.

- Application behavior is preserved; unknown-sensitive values are omitted or escalated.



TASK:


Implement the following task in java.


Return a compact but understandable implementation. Include small supporting types needed to understand the control flow. After the code, list at most three assumptions.


Implement a compact service or endpoint in a language of your choice that:



- accepts order ID, card data and amount;

- calls a payment provider;

- logs accepted and rejected outcomes;

- handles provider timeout and unexpected failure;

- returns a compact result.



Include enough surrounding types or pseudocode to understand logged values and exception boundaries.


Use the same language and model as the baseline. Return compact, understandable code. Then list at most three assumptions and perform the skill's self-review using concrete code evidence.
```java
import java.math.BigDecimal;
import java.util.Objects;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class PaymentService {
    private static final Logger LOG =
            LoggerFactory.getLogger(PaymentService.class);

    public enum ProviderOutcome { ACCEPTED, REJECTED }
    public enum Status { ACCEPTED, REJECTED, TIMEOUT, FAILED }
    public record Result(Status status) {}

    public record CardData(
            String number, int expiryMonth, int expiryYear, String cvv) {
        // Prevent accidental disclosure through implicit string conversion.
        @Override
        public String toString() {
            return "CardData[contents omitted]";
        }
    }

    public static final class ProviderTimeout extends Exception {}
    public static final class ProviderFailure extends Exception {}

    public interface PaymentProvider {
        ProviderOutcome charge(String orderId, CardData card, BigDecimal amount)
                throws ProviderTimeout, ProviderFailure;
    }

    private final PaymentProvider provider;

    public PaymentService(PaymentProvider provider) {
        this.provider = Objects.requireNonNull(provider);
    }

    public Result pay(String orderId, CardData card, BigDecimal amount) {
        if (orderId == null || orderId.isBlank() || card == null
                || amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("Invalid payment request");
        }

        final ProviderOutcome outcome;
        try {
            outcome = Objects.requireNonNull(
                    provider.charge(orderId, card, amount));
        } catch (ProviderTimeout ignored) {
            LOG.atWarn()
                    .addKeyValue("operation", "payment.charge")
                    .addKeyValue("outcome", Status.TIMEOUT)
                    .log("Payment provider timed out");
            // A timeout does not establish whether a charge occurred.
            return new Result(Status.TIMEOUT);
        } catch (ProviderFailure | RuntimeException ignored) {
            LOG.atError()
                    .addKeyValue("operation", "payment.charge")
                    .addKeyValue("outcome", Status.FAILED)
                    .log("Payment provider operation failed");
            return new Result(Status.FAILED);
        }

        Status status = switch (outcome) {
            case ACCEPTED -> Status.ACCEPTED;
            case REJECTED -> Status.REJECTED;
        };

        LOG.atInfo()
                .addKeyValue("operation", "payment.charge")
                .addKeyValue("outcome", status)
                .log("Payment provider returned a decision");
        return new Result(status);
    }
}```

**Assumptions**



1. Java 17+ and SLF4J 2.x are available; amounts use one configured currency.

2. The provider adapter validates card details, enforces a timeout, and does not log payment payloads or authentication material.

3. This service is the exception-handling boundary: provider failures become results; no automatic retries occur because a timeout can leave the charge outcome unknown.



**Safe-logging self-review**



- Every emitted value is a fixed operation string or controlled status enum. `orderId`, `amount`, and all card fields are omitted; their sensitivity need not be assumed.

- Messages are stable, with separate named properties. Decisions use Information, timeout uses Warning, and unexpected failure uses Error.

- Each path logs once at the handling boundary. No exception, message, cause, stack trace, DTO, or serialized payload is logged. Validation exceptions propagate; JVM `Error`s are not caught. `CardData.toString()` additionally omits its contents.