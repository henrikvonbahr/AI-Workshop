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