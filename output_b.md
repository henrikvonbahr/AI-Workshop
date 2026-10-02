```java
import java.time.Duration;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.TimeoutException;

public final class PaymentService {
    private static final System.Logger LOG =
            System.getLogger(PaymentService.class.getName());
    private static final Duration TIMEOUT = Duration.ofSeconds(5);

    private final PaymentProvider provider;

    public PaymentService(PaymentProvider provider) {
        this.provider = Objects.requireNonNull(provider);
    }

    // Tokenized card data; never include this value in logs.
    public record CardData(String token) {
        public CardData {
            if (token == null || token.isBlank())
                throw new IllegalArgumentException("Card token is required");
        }

        @Override public String toString() {
            return "CardData[REDACTED]";
        }
    }

    public enum Outcome { ACCEPTED, REJECTED, UNKNOWN }
    public record Result(UUID orderId, Outcome outcome, String code) {}
    public enum ProviderDecision { ACCEPTED, DECLINED }

    public interface PaymentProvider {
        // Adapter must enforce the timeout and use orderId as an idempotency key.
        ProviderDecision charge(UUID orderId, CardData card, long amountCents,
                                Duration timeout)
                throws TimeoutException, ProviderFailure;
    }

    public static final class ProviderFailure extends Exception {
        public ProviderFailure(String message, Throwable cause) {
            super(message, cause);
        }
    }

    public Result pay(UUID orderId, CardData card, long amountCents) {
        // Invalid input fails before any provider call.
        Objects.requireNonNull(orderId, "orderId");
        Objects.requireNonNull(card, "card");
        if (amountCents <= 0)
            throw new IllegalArgumentException("Amount must be positive");

        Result result;
        System.Logger.Level level = System.Logger.Level.INFO;
        try {
            ProviderDecision decision = Objects.requireNonNull(
                    provider.charge(orderId, card, amountCents, TIMEOUT),
                    "Missing provider decision");

            result = decision == ProviderDecision.ACCEPTED
                    ? new Result(orderId, Outcome.ACCEPTED, "approved")
                    : new Result(orderId, Outcome.REJECTED, "declined");
        } catch (TimeoutException e) {
            // A timeout does not establish whether the provider charged the card.
            result = new Result(orderId, Outcome.UNKNOWN, "provider_timeout");
            level = System.Logger.Level.WARNING;
        } catch (ProviderFailure | RuntimeException e) {
            result = new Result(orderId, Outcome.UNKNOWN, "provider_failure");
            level = System.Logger.Level.ERROR;
        }

        // Logging is outside the provider exception boundary. No card data,
        // provider messages, or exception payloads are logged.
        LOG.log(level, "orderId={0} amountCents={1} outcome={2} code={3}",
                orderId, amountCents, result.outcome(), result.code());
        return result;
    }
}
```